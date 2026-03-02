"""
core/merge.py
Junção Science + Portal + Arquivo 3 com REDE/UF/CLUSTER/etc.
Garante outer join: nenhuma linha perdida.
"""
from __future__ import annotations
from typing import Dict, List, Optional, Tuple

import pandas as pd

from app.core.map_rules import (
    OUTPUT_COLUMNS, build_output_row, build_ref_index,
)
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

_SCI = "_sci"
_POR = "_por"
_ALL_OUTPUT = OUTPUT_COLUMNS + ["_source_tag", "_central_src",
                                  "_tipo_rota_src", "_arq3_match", "_cnl_val"]

_NAN_STRINGS = {"nan", "none", "null", "<na>", "n/a", "na", "#n/a", ""}

def _clean_val(v) -> str:
    """Converte qualquer valor para string limpa. NaN, None, 'nan' → ''."""
    if v is None:
        return ""
    try:
        import math
        if isinstance(v, float) and math.isnan(v):
            return ""
    except (TypeError, ValueError):
        pass
    s = str(v).strip()
    return "" if s.lower() in _NAN_STRINGS else s


def _prefix(df: pd.DataFrame, pfx: str) -> pd.DataFrame:
    return df.rename(columns={c: f"{c}{pfx}" for c in df.columns})


def build_merged_df(
    science_df: pd.DataFrame,
    portal_df: pd.DataFrame,
    ref_df: Optional[pd.DataFrame],
    uf_map: Dict[str, str],
    join_keys_sci: List[str],   # colunas em science_df usadas para junção
    join_keys_por: List[str],   # colunas em portal_df usadas para junção
    join_type: str = "outer",
    config: Optional[Dict] = None,
    cn_to_uf_map: Optional[Dict[str, str]] = None,
) -> Tuple[pd.DataFrame, Dict]:
    """
    Une Science + Portal (outer join padrão).
    Aplica regras de negócio linha a linha.
    Retorna (merged_df_com_OUTPUT_COLUMNS + _aux, relatorio).
    """
    config = config or {}
    ref_index = build_ref_index(ref_df, config)
    rows_sci = len(science_df)
    rows_por = len(portal_df)

    # ── Junção ──────────────────────────────────────────────────────────
    if join_keys_sci and join_keys_por and \
       all(k in science_df.columns for k in join_keys_sci) and \
       all(k in portal_df.columns  for k in join_keys_por):
        merged = _exact_join(science_df, portal_df,
                              join_keys_sci, join_keys_por, join_type)
    else:
        log.warning("Sem chaves de junção válidas — concatenação posicional.")
        merged = _positional_join(science_df, portal_df)

    # ── Aplicar regras por linha ─────────────────────────────────────────
    sci_cols = list(science_df.columns)
    por_cols = list(portal_df.columns)
    records = []

    for _, row in merged.iterrows():
        sci_row = {c: _clean_val(row.get(f"{c}{_SCI}")) for c in sci_cols}
        por_row = {c: _clean_val(row.get(f"{c}{_POR}")) for c in por_cols}

        # source_tag: distingue origem da linha
        has_sci = any(v for v in sci_row.values())
        has_por = any(v for v in por_row.values())
        if has_sci and has_por:
            tag = "BOTH"
        elif has_sci:
            tag = "SCIENCE"
        else:
            tag = "PORTAL"

        cfg = dict(config)
        cfg["_cn_to_uf_map"] = cn_to_uf_map or {}
        out_row = build_output_row(sci_row, por_row, tag, ref_index, uf_map, cfg)
        records.append(out_row)

    result = pd.DataFrame(records)
    # Garante colunas de saída mesmo se vazia
    for col in OUTPUT_COLUMNS:
        if col not in result.columns:
            result[col] = ""

    report = {
        "rows_science":       rows_sci,
        "rows_portal":        rows_por,
        "rows_merged":        len(result),
        "join_type":          join_type,
        "arq3_match_count":   int(result.get("_arq3_match", pd.Series()).eq("True").sum()),
        "uf_missing":         int(result["UF"].eq("").sum()),
        "cluster_missing":    int(result["CLUSTER"].eq("").sum()),
        "source_breakdown":   result["_source_tag"].value_counts().to_dict()
                              if "_source_tag" in result.columns else {},
    }
    log.info("Merge: %d linhas (sci=%d por=%d arq3_matches=%d)",
             len(result), rows_sci, rows_por, report["arq3_match_count"])
    return result, report


def _exact_join(sci: pd.DataFrame, por: pd.DataFrame,
                sci_keys: List[str], por_keys: List[str],
                join_type: str) -> pd.DataFrame:
    sci_w = _prefix(sci.copy(), _SCI)
    por_w = _prefix(por.copy(), _POR)
    l_on = [f"{k}{_SCI}" for k in sci_keys]
    r_on = [f"{k}{_POR}" for k in por_keys]
    # Normaliza chaves
    for c in l_on: sci_w[c] = sci_w[c].str.strip().str.upper()
    for c in r_on: por_w[c] = por_w[c].str.strip().str.upper()
    merged = pd.merge(sci_w, por_w, left_on=l_on, right_on=r_on, how=join_type)
    merged = merged.fillna("")
    log.info("Exact join: %d linhas (type=%s)", len(merged), join_type)
    return merged


def _positional_join(sci: pd.DataFrame, por: pd.DataFrame) -> pd.DataFrame:
    sci_w = _prefix(sci.copy(), _SCI)
    por_w = _prefix(por.copy(), _POR)
    max_rows = max(len(sci_w), len(por_w))
    sci_w = sci_w.reindex(range(max_rows))
    por_w = por_w.reindex(range(max_rows))
    return pd.concat([sci_w.reset_index(drop=True),
                      por_w.reset_index(drop=True)], axis=1).fillna("")


def count_join_matches(science_df: pd.DataFrame, portal_df: pd.DataFrame,
                        sci_key: str, por_key: str) -> Dict:
    if sci_key not in science_df.columns or por_key not in portal_df.columns:
        return {"error": "coluna não encontrada"}
    sci_vals = set(science_df[sci_key].str.strip().str.upper().dropna())
    por_vals = set(portal_df[por_key].str.strip().str.upper().dropna())
    return {
        "sci_unique": len(sci_vals),
        "por_unique": len(por_vals),
        "matches":    len(sci_vals & por_vals),
        "sci_only":   len(sci_vals - por_vals),
        "por_only":   len(por_vals - sci_vals),
    }
