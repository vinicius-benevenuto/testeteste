"""
core/validate.py
Validação e relatório de qualidade dos dados.
"""
from __future__ import annotations
from typing import Dict, List

import pandas as pd

from app.utils.logging_utils import get_logger

log = get_logger(__name__)


def generate_report(df: pd.DataFrame, name: str = "tabela") -> Dict:
    """Gera relatório de qualidade: nulos, duplicatas, estatísticas básicas."""
    total = len(df)
    report: Dict = {
        "tabela": name,
        "total_linhas": total,
        "total_colunas": len(df.columns),
        "colunas": [],
    }
    for col in df.columns:
        series = df[col]
        empty = int(series.eq("").sum() + series.isna().sum())
        unique = int(series.nunique())
        report["colunas"].append({
            "coluna": col,
            "vazios": empty,
            "pct_vazio": round(empty / total * 100, 1) if total else 0,
            "unicos": unique,
            "exemplo": str(series.dropna().iloc[0]) if not series.dropna().empty else "",
        })
    return report


def check_duplicates(df: pd.DataFrame, key_cols: List[str]) -> Dict:
    """Conta duplicatas pelas colunas-chave definidas."""
    existing = [c for c in key_cols if c in df.columns]
    if not existing:
        return {"key_cols": key_cols, "total": len(df), "duplicatas": 0, "unicos": len(df)}
    dup_mask = df.duplicated(subset=existing, keep=False)
    return {
        "key_cols": existing,
        "total": len(df),
        "duplicatas": int(dup_mask.sum()),
        "unicos": int(len(df) - dup_mask.sum()),
    }


def validate_output(df: pd.DataFrame) -> List[str]:
    """Valida o DataFrame de saída. Retorna lista de avisos."""
    from app.core.map_rules import OUTPUT_COLUMNS
    warnings: List[str] = []
    missing = [c for c in OUTPUT_COLUMNS if c not in df.columns]
    if missing:
        warnings.append(f"⚠️ Colunas faltando na saída: {missing}")
    for col in OUTPUT_COLUMNS:
        if col in df.columns:
            pct_empty = df[col].eq("").mean() * 100
            if pct_empty > 80:
                warnings.append(f"⚠️ '{col}': {pct_empty:.0f}% de valores vazios.")
    return warnings
