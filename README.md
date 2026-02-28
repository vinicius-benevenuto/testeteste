"""core/validate.py — Relatório de qualidade e validações."""
from __future__ import annotations
from typing import Dict, List

import pandas as pd
from app.core.map_rules import OUTPUT_COLUMNS
from app.utils.logging_utils import get_logger

log = get_logger(__name__)


def generate_report(df: pd.DataFrame, name: str = "tabela") -> Dict:
    total = len(df)
    cols = []
    for col in df.columns:
        if col.startswith("_"): continue
        series = df[col]
        empty  = int(series.eq("").sum() + series.isna().sum())
        cols.append({
            "coluna":    col,
            "vazios":    empty,
            "pct_vazio": round(empty / total * 100, 1) if total else 0,
            "unicos":    int(series.nunique()),
            "exemplo":   str(series.dropna().iloc[0]) if not series.dropna().empty else "",
        })
    return {"tabela": name, "total_linhas": total,
            "total_colunas": len(df.columns), "colunas": cols}


def check_duplicates(df: pd.DataFrame, key_cols: List[str]) -> Dict:
    existing = [c for c in key_cols if c in df.columns]
    if not existing:
        return {"key_cols": key_cols, "duplicatas": 0, "unicos": len(df)}
    dup = df.duplicated(subset=existing, keep=False)
    return {"key_cols": existing, "total": len(df),
            "duplicatas": int(dup.sum()), "unicos": int(len(df) - dup.sum())}


def quality_summary(merged_df: pd.DataFrame) -> Dict:
    """Relatório de qualidade pós-merge."""
    total = len(merged_df)
    if total == 0:
        return {"total": 0}
    out = {"total": total}
    for col in ["UF", "CLUSTER", "Denominação", "OPERADORA", "Rótulos de Linha"]:
        if col in merged_df.columns:
            missing = int(merged_df[col].eq("").sum() + merged_df[col].isna().sum())
            out[f"{col}_missing"] = missing
            out[f"{col}_pct"]     = round(missing / total * 100, 1)
    if "_arq3_match" in merged_df.columns:
        out["arq3_no_match"] = int(merged_df["_arq3_match"].ne("True").sum())
    if "_source_tag" in merged_df.columns:
        out["source_breakdown"] = merged_df["_source_tag"].value_counts().to_dict()
    return out
