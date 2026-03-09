"""
db/tf_repository.py
===================
Operações na tabela tabela_final:
  - load()        → DataFrame com as 8 colunas de negócio
  - upsert(df)    → (inserted, skipped) — INSERT OR IGNORE por hash_rota
  - count()       → total de rotas
"""
from __future__ import annotations

import logging
from typing import Tuple

import pandas as pd
from sqlalchemy import text
from sqlalchemy.orm import Session

from app.core.hash_utils import BUSINESS_COLS, make_hash_rota
from app.db.models import TabelaFinal

log = logging.getLogger(__name__)

# Mapeamento: nome de exibição → coluna SQLAlchemy
_COL_MAP = {
    "REDE":             "rede",
    "UF":               "uf",
    "CLUSTER":          "cluster",
    "Tipo de Rota":     "tipo_de_rota",
    "Central":          "central",
    "Rótulos de Linha": "rotulos_de_linha",
    "OPERADORA":        "operadora",
    "Denominação":      "denominacao",
}
_COL_MAP_INV = {v: k for k, v in _COL_MAP.items()}


class TabelaFinalRepository:
    def __init__(self, session: Session) -> None:
        self.session = session

    # ── Leitura ───────────────────────────────────────────────────────────

    def load(self) -> pd.DataFrame:
        """Carrega todas as rotas como DataFrame com colunas de negócio."""
        rows = self.session.query(TabelaFinal).order_by(TabelaFinal.id).all()
        if not rows:
            return pd.DataFrame(columns=BUSINESS_COLS)
        return pd.DataFrame([_to_display(r) for r in rows])

    def count(self) -> int:
        return self.session.query(TabelaFinal).count()

    # ── Escrita (INSERT OR IGNORE via hash_rota) ──────────────────────────

    def upsert(self, df: pd.DataFrame) -> Tuple[int, int]:
        """
        Insere rotas novas, ignora duplicatas por hash_rota.
        Retorna (inseridas, ignoradas).
        """
        if df is None or df.empty:
            return 0, 0

        # Normaliza nomes de colunas para o esperado
        df = _normalize_cols(df)

        # Gera hash para cada linha
        df = df.copy()
        df["_hash"] = df.apply(make_hash_rota, axis=1)

        # Busca hashes existentes em lote
        existing_hashes: set[str] = set(
            h for (h,) in self.session.query(TabelaFinal.hash_rota).all()
        )

        inserted = 0
        skipped  = 0

        for _, row in df.iterrows():
            h = row["_hash"]
            if h in existing_hashes:
                skipped += 1
                continue
            try:
                obj = TabelaFinal(
                    hash_rota        = h,
                    rede             = _sv(row, "REDE"),
                    uf               = _sv(row, "UF"),
                    cluster          = _sv(row, "CLUSTER"),
                    tipo_de_rota     = _sv(row, "Tipo de Rota"),
                    central          = _sv(row, "Central"),
                    rotulos_de_linha = _sv(row, "Rótulos de Linha"),
                    operadora        = _sv(row, "OPERADORA"),
                    denominacao      = _sv(row, "Denominação"),
                )
                self.session.add(obj)
                self.session.flush()
                existing_hashes.add(h)
                inserted += 1
            except Exception as e:
                self.session.rollback()
                log.debug("Insert ignorado (hash_rota duplicado): %s", e)
                skipped += 1

        return inserted, skipped

    def clear(self) -> None:
        """Remove todos os registros (uso interno/debug — não exposto na UI)."""
        self.session.query(TabelaFinal).delete()


# ── Helpers privados ──────────────────────────────────────────────────────

def _to_display(r: TabelaFinal) -> dict:
    return {
        "REDE":             r.rede             or "",
        "UF":               r.uf               or "",
        "CLUSTER":          r.cluster          or "",
        "Tipo de Rota":     r.tipo_de_rota     or "",
        "Central":          r.central          or "",
        "Rótulos de Linha": r.rotulos_de_linha or "",
        "OPERADORA":        r.operadora        or "",
        "Denominação":      r.denominacao      or "",
    }


def _sv(row, col: str) -> str:
    """Safe value: converte para str, None → ''."""
    v = row.get(col, "") if hasattr(row, "get") else getattr(row, col, "")
    if v is None or (isinstance(v, float) and str(v) == "nan"):
        return ""
    return str(v).strip()


def _normalize_cols(df: pd.DataFrame) -> pd.DataFrame:
    """
    Garante que o DataFrame tenha as 8 colunas de negócio.
    Tenta mapear variações comuns de nome.
    """
    rename = {}
    for col in df.columns:
        cu = col.strip().upper()
        if cu == "TIPO DE ROTA" and "Tipo de Rota" not in df.columns:
            rename[col] = "Tipo de Rota"
        elif cu == "ROTULOS DE LINHA" and "Rótulos de Linha" not in df.columns:
            rename[col] = "Rótulos de Linha"
        elif cu == "DENOMINACAO" and "Denominação" not in df.columns:
            rename[col] = "Denominação"
        elif cu == "DENOMINAÇÃO" and "Denominação" not in df.columns:
            rename[col] = "Denominação"
    if rename:
        df = df.rename(columns=rename)
    # Garante que todas as colunas existam
    for c in BUSINESS_COLS:
        if c not in df.columns:
            df[c] = ""
    return df
