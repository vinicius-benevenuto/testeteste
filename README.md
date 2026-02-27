"""
core/normalize.py
Normalização de DataFrames: labels de colunas, tipos, datas.
Os dados brutos são preservados inalterados; só os headers são normalizados.
"""
from __future__ import annotations
import re
from typing import Dict, List, Optional, Tuple

import pandas as pd

from app.utils.encoding import normalize_column_label, normalize_columns
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

# Padrões de data aceitos
_DATE_FORMATS = ["%d/%m/%Y", "%Y-%m-%d", "%d-%m-%Y", "%d/%m/%y", "%Y/%m/%d"]


def apply_column_normalization(df: pd.DataFrame) -> Tuple[pd.DataFrame, Dict[str, str]]:
    """
    Normaliza os headers do DataFrame:
    - Corrige mojibake
    - Colapsa espaços duplos
    Retorna (df_com_headers_normalizados, {original: normalizado}).
    Os dados das células não são alterados.
    """
    rename_map = normalize_columns(list(df.columns))
    # Lida com duplicatas após normalização
    seen: Dict[str, int] = {}
    final_rename: Dict[str, str] = {}
    for orig, norm in rename_map.items():
        if norm in seen:
            seen[norm] += 1
            norm = f"{norm}_{seen[norm]}"
        else:
            seen[norm] = 0
        final_rename[orig] = norm

    df = df.rename(columns=final_rename)
    changed = {k: v for k, v in final_rename.items() if k != v}
    if changed:
        log.info("Colunas normalizadas: %d renomeadas", len(changed))
    return df, final_rename


def coerce_dates(series: pd.Series) -> pd.Series:
    """Tenta converter série de strings para datas ISO. Mantém original em falha."""
    def _try_parse(val: str) -> str:
        if not val or not isinstance(val, str):
            return val
        for fmt in _DATE_FORMATS:
            try:
                import datetime
                return datetime.datetime.strptime(val.strip(), fmt).strftime("%Y-%m-%d")
            except ValueError:
                continue
        return val
    return series.map(_try_parse)


def infer_and_coerce_types(df: pd.DataFrame) -> pd.DataFrame:
    """
    Tenta inferir e converter tipos (datas, inteiros).
    Preserva os valores originais se a conversão falhar.
    Tudo permanece como string — só adiciona metadados de tipo.
    """
    for col in df.columns:
        col_lower = col.lower()
        if any(kw in col_lower for kw in ("data", "dt_", "date", "ativacao", "desativacao")):
            df[col] = coerce_dates(df[col])
    return df


def strip_whitespace(df: pd.DataFrame) -> pd.DataFrame:
    """Remove espaços em branco do início/fim de todos os valores string."""
    str_cols = df.select_dtypes(include="object").columns
    for col in str_cols:
        df[col] = df[col].str.strip()
    return df
