"""
core/merge.py
Combina Science (VIVO-SMP) + Portal (VIVO-STFC) + lookup no Arquivo 3.

Estratégia de combinação:
  - Science e Portal são bases INDEPENDENTES (redes diferentes: SMP e STFC).
  - Cada uma é processada separadamente com suas regras de mapeamento.
  - O resultado final é a CONCATENAÇÃO das duas bases processadas.
  - CLUSTER, UF, Rótulos de Linha, Denominação e OPERADORA vêm do Arquivo 3
    pelo lookup por Central (ou por LABEL_E/LABEL_S como fallback).

  Exceção (join opcional): se o usuário configurar chaves de junção no wizard,
  as linhas com Central em comum entre Science e Portal são unidas (BOTH→SMP).
  Linhas sem match ficam como SCIENCE ou PORTAL individualmente.
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


def _row_to_dict(row, cols) -> Dict[str, str]:
    return {c: _clean_val(row.get(c)) for c in cols}


def _process_rows(df: pd.DataFrame, source_tag: str,
                  ref_index: Dict, uf_map: Dict, config: Dict,
                  cn_to_uf_map: Dict) -> List[Dict]:
    """Processa cada linha de um DataFrame com as regras de mapeamento."""
    cols = list(df.columns)
    records = []
    cfg = dict(config)
    cfg["_cn_to_uf_map"] = cn_to_uf_map or {}

    for _, row in df.iterrows():
        if source_tag == "SCIENCE":
            sci_row = _row_to_dict(row, cols)
            por_row = {}
        elif source_tag == "PORTAL":
            sci_row = {}
            por_row = _row_to_dict(row, cols)
        else:  # BOTH — linha unida
            sci_row = {c.removesuffix(_SCI): _clean_val(row.get(c))
                       for c in row.index if c.endswith(_SCI)}
            por_row = {c.removesuffix(_POR): _clean_val(row.get(c))
                       for c in row.index if c.endswith(_POR)}

        out = build_output_row(sci_row, por_row, source_tag,
                               ref_index, uf_map, cfg)
        records.append(out)
    return records


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

    config        = config or {}
    ref_index     = build_ref_index(ref_df, config)
    cn_to_uf_map  = cn_to_uf_map or {}
    rows_sci      = len(science_df)
    rows_por      = len(portal_df)

    records: List[Dict] = []

    use_join = (
        join_keys_sci and join_keys_por
        and all(k in science_df.columns for k in join_keys_sci)
        and all(k in portal_df.columns  for k in join_keys_por)
    )

    if use_join:
        # ── Modo JOIN: une Science+Portal por chave, separa não-matches ──
        log.info("Modo JOIN ativado: sci_keys=%s por_keys=%s", join_keys_sci, join_keys_por)

        sci_w = science_df.copy().rename(columns={c: f"{c}{_SCI}" for c in science_df.columns})
        por_w = portal_df.copy().rename(columns={c: f"{c}{_POR}" for c in portal_df.columns})
        l_on  = [f"{k}{_SCI}" for k in join_keys_sci]
        r_on  = [f"{k}{_POR}" for k in join_keys_por]
        for c in l_on: sci_w[c] = sci_w[c].astype(str).str.strip().str.upper()
        for c in r_on: por_w[c] = por_w[c].astype(str).str.strip().str.upper()

        joined = pd.merge(sci_w, por_w, left_on=l_on, right_on=r_on,
                          how="outer", indicator=True).fillna("")

        # BOTH: linhas com match em ambas
        both_df = joined[joined["_merge"] == "both"].copy()
        # SCIENCE: linhas só no Science
        sci_only = science_df[
            ~science_df[join_keys_sci[0]].str.strip().str.upper().isin(
                portal_df[join_keys_por[0]].str.strip().str.upper()
            )
        ].copy()
        # PORTAL: linhas só no Portal
        por_only = portal_df[
            ~portal_df[join_keys_por[0]].str.strip().str.upper().isin(
                science_df[join_keys_sci[0]].str.strip().str.upper()
            )
        ].copy()

        log.info("Join: %d BOTH, %d só Science, %d só Portal",
                 len(both_df), len(sci_only), len(por_only))

        records.extend(_process_rows(both_df, "BOTH", ref_index, uf_map, config, cn_to_uf_map))
        records.extend(_process_rows(sci_only, "SCIENCE", ref_index, uf_map, config, cn_to_uf_map))
        records.extend(_process_rows(por_only, "PORTAL", ref_index, uf_map, config, cn_to_uf_map))
    else:
        # ── Modo CONCAT: processa cada base independentemente ─────────────
        log.info("Modo CONCAT: Science=%d linhas + Portal=%d linhas", rows_sci, rows_por)
        records.extend(_process_rows(science_df, "SCIENCE", ref_index, uf_map, config, cn_to_uf_map))
        records.extend(_process_rows(portal_df,  "PORTAL",  ref_index, uf_map, config, cn_to_uf_map))

    result = pd.DataFrame(records)
    for col in OUTPUT_COLUMNS:
        if col not in result.columns:
            result[col] = ""

    report = {
        "rows_science":     rows_sci,
        "rows_portal":      rows_por,
        "rows_merged":      len(result),
        "join_type":        "join" if use_join else "concat",
        "arq3_match_count": int(result.get("_arq3_match", pd.Series()).eq("True").sum()),
        "uf_missing":       int(result["UF"].eq("").sum()),
        "cluster_missing":  int(result["CLUSTER"].eq("").sum()),
        "source_breakdown": result["_source_tag"].value_counts().to_dict()
                            if "_source_tag" in result.columns else {},
    }
    log.info("Merge final: %d linhas (sci=%d por=%d arq3=%d UF_vazio=%d CLUSTER_vazio=%d)",
             len(result), rows_sci, rows_por,
             report["arq3_match_count"], report["uf_missing"], report["cluster_missing"])
    return result, report


def count_join_matches(science_df, portal_df, sci_key, por_key):
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

