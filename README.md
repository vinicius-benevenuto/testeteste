"""
core/merge.py
Lógica de junção e fusão das tabelas Science + Portal.
Garante que nenhuma linha seja perdida (outer join padrão).
"""
from __future__ import annotations
from typing import Dict, List, Optional, Tuple

import pandas as pd

from app.core.map_rules import OUTPUT_COLUMNS, apply_rule
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

_SCI_PREFIX  = "_sci"
_POR_PREFIX  = "_por"
_EMPTY_ROW: Dict = {}


def _prefix_cols(df: pd.DataFrame, prefix: str) -> pd.DataFrame:
    """Adiciona prefixo a todas as colunas para evitar colisões após merge."""
    return df.rename(columns={c: f"{c}{prefix}" for c in df.columns})


def _safe_get(row: pd.Series, col: str, prefix: str) -> str:
    """Lê valor de linha com prefixo, retornando '' se não encontrar."""
    key = f"{col}{prefix}"
    val = row.get(key, "")
    return str(val or "").strip()


def build_merged_df(
    science_df: pd.DataFrame,
    portal_df: pd.DataFrame,
    mapping: Dict,
    join_keys: List[str],        # colunas alvo (ex.: ["Central", "Tipo de Rota"])
    join_type: str = "outer",    # "inner" | "left" | "right" | "outer"
    fuzzy: bool = False,
    fuzzy_threshold: int = 90,
    literal_defaults: Optional[Dict] = None,
) -> Tuple[pd.DataFrame, Dict]:
    """
    Une Science + Portal e aplica o mapeamento de colunas.
    Retorna (merged_df_com_colunas_alvo, relatorio_dict).

    O resultado sempre contém exatamente OUTPUT_COLUMNS.
    """
    rows_sci = len(science_df)
    rows_por = len(portal_df)

    # --- Mapeamento de chaves de junção nas tabelas originais -------------
    # join_keys são nomes das colunas ALVO (ex.: "Central").
    # Precisamos encontrar qual coluna de cada tabela corresponde a cada chave.
    sci_key_cols: List[str] = []
    por_key_cols: List[str] = []

    for key in join_keys:
        rule = mapping.get(key, {})
        sources = rule.get("sources", [])
        sci_src = next((s["col"] for s in sources if s["table"] == "science"), None)
        por_src = next((s["col"] for s in sources if s["table"] == "portal"), None)
        if sci_src and sci_src in science_df.columns:
            sci_key_cols.append(sci_src)
        else:
            sci_key_cols.append(None)
        if por_src and por_src in portal_df.columns:
            por_key_cols.append(por_src)
        else:
            por_key_cols.append(None)

    # --- Junção -----------------------------------------------------------
    if not join_keys or all(k is None for k in sci_key_cols) or all(k is None for k in por_key_cols):
        # Sem chaves válidas: cross-product ou apenas concatenação lateral
        log.warning("Sem chaves de junção válidas — usando concatenação posicional.")
        result = _positional_merge(science_df, portal_df, mapping, literal_defaults)
    elif fuzzy:
        result = _fuzzy_merge(science_df, portal_df, sci_key_cols, por_key_cols,
                              join_type, fuzzy_threshold, mapping, literal_defaults)
    else:
        result = _exact_merge(science_df, portal_df, sci_key_cols, por_key_cols,
                              join_type, mapping, literal_defaults)

    report = {
        "rows_science": rows_sci,
        "rows_portal":  rows_por,
        "rows_merged":  len(result),
        "join_type":    join_type,
        "join_keys":    join_keys,
        "fuzzy":        fuzzy,
        "null_counts":  {c: int(result[c].eq("").sum()) for c in OUTPUT_COLUMNS},
    }
    log.info("Merge concluído: %d linhas (sci=%d por=%d)", len(result), rows_sci, rows_por)
    return result[OUTPUT_COLUMNS], report


# ---------------------------------------------------------------------------

def _apply_mapping_to_row(sci_row: Dict, por_row: Dict,
                           mapping: Dict, defaults: Optional[Dict]) -> Dict:
    out = {}
    for col in OUTPUT_COLUMNS:
        rule = mapping.get(col, {"type": "literal", "value": ""})
        out[col] = apply_rule(rule, sci_row, por_row, defaults)
    return out


def _positional_merge(sci: pd.DataFrame, por: pd.DataFrame,
                       mapping: Dict, defaults: Optional[Dict]) -> pd.DataFrame:
    """Merge posicional quando não há chaves — combina linha a linha até esgotar."""
    max_rows = max(len(sci), len(por))
    records = []
    empty_sci = {c: "" for c in sci.columns}
    empty_por = {c: "" for c in por.columns}
    for i in range(max_rows):
        sci_row = sci.iloc[i].to_dict() if i < len(sci) else empty_sci
        por_row = por.iloc[i].to_dict() if i < len(por) else empty_por
        records.append(_apply_mapping_to_row(sci_row, por_row, mapping, defaults))
    return pd.DataFrame(records, columns=OUTPUT_COLUMNS)


def _exact_merge(sci: pd.DataFrame, por: pd.DataFrame,
                  sci_key_cols: List[Optional[str]],
                  por_key_cols: List[Optional[str]],
                  join_type: str, mapping: Dict,
                  defaults: Optional[Dict]) -> pd.DataFrame:
    """Merge exato usando as chaves mapeadas."""
    # Prepara cópias com prefixo para evitar colisões
    sci_w = _prefix_cols(sci.copy(), _SCI_PREFIX)
    por_w = _prefix_cols(por.copy(), _POR_PREFIX)

    left_on  = [f"{k}{_SCI_PREFIX}" for k in sci_key_cols if k]
    right_on = [f"{k}{_POR_PREFIX}" for k in por_key_cols if k]

    if len(left_on) != len(right_on) or not left_on:
        log.warning("Chaves assimétricas — usando concatenação posicional.")
        return _positional_merge(sci, por, mapping, defaults)

    # Normaliza chaves para case-insensitive
    for col in left_on:
        sci_w[col] = sci_w[col].str.strip().str.upper()
    for col in right_on:
        por_w[col] = por_w[col].str.strip().str.upper()

    merged = pd.merge(sci_w, por_w, left_on=left_on, right_on=right_on, how=join_type)
    log.info("Exact merge: %d linhas (join=%s)", len(merged), join_type)

    # Aplica mapeamento
    sci_cols_orig = list(sci.columns)
    por_cols_orig = list(por.columns)
    records = []
    for _, row in merged.iterrows():
        sci_row = {c: row.get(f"{c}{_SCI_PREFIX}", "") for c in sci_cols_orig}
        por_row = {c: row.get(f"{c}{_POR_PREFIX}", "") for c in por_cols_orig}
        records.append(_apply_mapping_to_row(sci_row, por_row, mapping, defaults))

    return pd.DataFrame(records, columns=OUTPUT_COLUMNS)


def _fuzzy_merge(sci: pd.DataFrame, por: pd.DataFrame,
                  sci_key_cols: List[Optional[str]],
                  por_key_cols: List[Optional[str]],
                  join_type: str, threshold: int,
                  mapping: Dict, defaults: Optional[Dict]) -> pd.DataFrame:
    """
    Fuzzy merge usando thefuzz (python-levenshtein) se disponível.
    Fallback para exact merge se biblioteca ausente.
    """
    try:
        from thefuzz import process as fuzz_process  # type: ignore
    except ImportError:
        log.warning("thefuzz não instalado — usando exact merge.")
        return _exact_merge(sci, por, sci_key_cols, por_key_cols,
                             join_type, mapping, defaults)

    # Usa apenas a primeira chave para fuzzy (simplificação)
    sci_key = sci_key_cols[0]
    por_key = por_key_cols[0]
    if not sci_key or not por_key:
        return _exact_merge(sci, por, sci_key_cols, por_key_cols,
                             join_type, mapping, defaults)

    por_values = por[por_key].fillna("").str.strip().tolist()
    records = []
    used_por_indices = set()

    for _, sci_row in sci.iterrows():
        sci_val = str(sci_row.get(sci_key, "") or "").strip()
        if not sci_val:
            records.append(_apply_mapping_to_row(sci_row.to_dict(), {}, mapping, defaults))
            continue

        match = fuzz_process.extractOne(sci_val, por_values, score_cutoff=threshold)
        if match:
            matched_val, score, idx = match
            por_row = por.iloc[idx].to_dict()
            used_por_indices.add(idx)
            records.append(_apply_mapping_to_row(sci_row.to_dict(), por_row, mapping, defaults))
        else:
            records.append(_apply_mapping_to_row(sci_row.to_dict(), {}, mapping, defaults))

    # Outer: adiciona linhas do portal sem match
    if join_type in ("outer", "right"):
        for i, por_row in por.iterrows():
            if i not in used_por_indices:
                records.append(_apply_mapping_to_row({}, por_row.to_dict(), mapping, defaults))

    log.info("Fuzzy merge: %d linhas (threshold=%d%%)", len(records), threshold)
    return pd.DataFrame(records, columns=OUTPUT_COLUMNS)


def count_join_matches(science_df: pd.DataFrame, portal_df: pd.DataFrame,
                        sci_key: str, por_key: str) -> Dict:
    """Conta matches exatos entre dois conjuntos de chaves (para preview)."""
    sci_vals = set(science_df[sci_key].str.strip().str.upper().dropna())
    por_vals = set(portal_df[por_key].str.strip().str.upper().dropna())
    matches = sci_vals & por_vals
    return {
        "sci_unique": len(sci_vals),
        "por_unique": len(por_vals),
        "matches": len(matches),
        "sci_only": len(sci_vals - por_vals),
        "por_only": len(por_vals - sci_vals),
    }
