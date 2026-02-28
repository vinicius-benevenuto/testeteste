"""core/normalize.py — Normalização de headers e dados."""
from __future__ import annotations
import re
from typing import Dict, List, Tuple
import pandas as pd
from app.utils.encoding import normalize_column_label, normalize_columns
from app.utils.logging_utils import get_logger

log = get_logger(__name__)
_DATE_FMTS = ["%d/%m/%Y", "%Y-%m-%d", "%d-%m-%Y", "%d/%m/%y"]


def apply_column_normalization(df: pd.DataFrame) -> Tuple[pd.DataFrame, Dict[str, str]]:
    """Normaliza headers (mojibake, espaços). Dados intocados."""
    rename_map = normalize_columns(list(df.columns))
    seen: Dict[str, int] = {}
    final: Dict[str, str] = {}
    for orig, norm in rename_map.items():
        if norm in seen:
            seen[norm] += 1; norm = f"{norm}_{seen[norm]}"
        else:
            seen[norm] = 0
        final[orig] = norm
    df = df.rename(columns=final)
    changed = {k: v for k, v in final.items() if k != v}
    if changed:
        log.info("Colunas normalizadas: %d renomeadas", len(changed))
    return df, final


def strip_whitespace(df: pd.DataFrame) -> pd.DataFrame:
    for col in df.select_dtypes(include="object").columns:
        df[col] = df[col].str.strip()
    return df


def coerce_dates(series: pd.Series) -> pd.Series:
    import datetime
    def _try(v: str) -> str:
        if not isinstance(v, str): return v
        for fmt in _DATE_FMTS:
            try: return datetime.datetime.strptime(v.strip(), fmt).strftime("%Y-%m-%d")
            except ValueError: continue
        return v
    return series.map(_try)


def infer_and_coerce_types(df: pd.DataFrame) -> pd.DataFrame:
    for col in df.columns:
        if any(k in col.lower() for k in ("data", "dt_", "ativacao", "desativacao")):
            df[col] = coerce_dates(df[col])
    return df
