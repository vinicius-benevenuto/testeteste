"""
core/merge.py
Delega ao pipeline.py que implementa o processo oficial de 9 etapas.
Mantém assinatura compatível com o código existente.
"""
from __future__ import annotations
from typing import Dict, List, Optional, Tuple

import pandas as pd

from app.core.pipeline import run_pipeline, OUTPUT_COLUMNS
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

_ALL_OUTPUT = OUTPUT_COLUMNS + [
    "_source_tag", "_chave", "_arq3_match", "_nova_rota",
]


def build_merged_df(
    science_df: pd.DataFrame,
    portal_df: pd.DataFrame,
    ref_df: Optional[pd.DataFrame],
    uf_map: Dict[str, str],
    join_keys_sci: List[str],
    join_keys_por: List[str],
    join_type: str = "outer",
    config: Optional[Dict] = None,
    cn_to_uf_map: Optional[Dict[str, str]] = None,
) -> Tuple[pd.DataFrame, Dict]:
    """
    Ponto de entrada principal.
    Executa o pipeline oficial de 9 etapas e retorna (result_df, report).
    """
    cfg = config or {}
    result, report = run_pipeline(
        sci_df  = science_df,
        por_df  = portal_df,
        arq3_df = ref_df,
        config  = cfg,
    )
    return result, report


def count_join_matches(science_df, portal_df, sci_key, por_key):
    if sci_key not in science_df.columns or por_key not in portal_df.columns:
        return {"error": "coluna nao encontrada"}
    sci_vals = set(science_df[sci_key].str.strip().str.upper().dropna())
    por_vals = set(portal_df[por_key].str.strip().str.upper().dropna())
    return {
        "sci_unique": len(sci_vals),
        "por_unique": len(por_vals),
        "matches":    len(sci_vals & por_vals),
        "sci_only":   len(sci_vals - por_vals),
        "por_only":   len(por_vals - sci_vals),
    }
