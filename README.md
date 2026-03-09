"""
core/hash_utils.py
==================
Gera hash_rota: SHA1 sobre as 8 colunas de negócio, normalizado.
Normalização: strip → remove acentos → upper → colapsa espaços → join "||" → SHA1.
"""
from __future__ import annotations
import hashlib
import re
import unicodedata

BUSINESS_COLS: list[str] = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]


def _norm(v) -> str:
    """Normaliza um valor para o hash."""
    s = str(v) if v is not None else ""
    s = s.strip()
    # Remove acentos via NFKD decomposition
    s = unicodedata.normalize("NFKD", s).encode("ASCII", "ignore").decode("ASCII")
    s = s.upper()
    # Colapsa espaços múltiplos
    s = re.sub(r"\s+", " ", s).strip()
    return s


def make_hash_rota(row) -> str:
    """Gera hash_rota a partir de um dict/Series com as colunas de negócio."""
    if hasattr(row, "get"):
        parts = [_norm(row.get(c, "")) for c in BUSINESS_COLS]
    else:
        parts = [_norm(getattr(row, c, "")) for c in BUSINESS_COLS]
    raw = "||".join(parts)
    return hashlib.sha1(raw.encode("utf-8")).hexdigest()


def add_hash_column(df) -> "pd.DataFrame":  # type: ignore[name-defined]
    """Adiciona coluna hash_rota ao DataFrame, se ainda não existir."""
    import pandas as pd
    df = df.copy()
    df["hash_rota"] = df.apply(make_hash_rota, axis=1)
    return df
