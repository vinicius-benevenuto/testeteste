"""
core/map_rules.py
Regras de mapeamento de negócio.

Prioridade UF/CLUSTER:
  1. Arquivo 3 via Central + Tipo de Rota  (match exato)
  2. Arquivo 3 via Central + Rótulos
  3. Arquivo 3 via Central sozinha (ANY fallback)
  4. CN direto (DDD/Área Ponta B) → cn_to_uf
  5. CNL → CN → UF via uf_map pré-calculado
"""
from __future__ import annotations
from typing import Any, Dict, List, Optional, Tuple

from app.utils.logging_utils import get_logger
from app.utils.cnl_utils import clean_cnl, clean_cn

log = get_logger(__name__)

OUTPUT_COLUMNS = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]

_INVALID = {"nan", "none", "null", "n/a", "na", "#n/a", ""}


def _clean_str(v: Any) -> str:
    s = str(v or "").strip()
    return "" if s.lower() in _INVALID else s


def _resolve_col(df_columns: List[str], *candidates: str) -> str:
    """Busca coluna case-insensitive. Retorna nome original ou ''."""
    up = {c.strip().upper(): c for c in df_columns}
    for cand in candidates:
        hit = up.get(cand.strip().upper())
        if hit:
            return hit
    return ""


def derive_rede(source_tag: str) -> str:
    if (source_tag or "").upper() in ("SCIENCE", "BOTH"):
        return "VIVO-SMP"
    return "VIVO-STFC"


def coalesce(*values: Any) -> str:
    for v in values:
        s = _clean_str(v)
        if s:
            return s
    return ""


def derive_tipo_rota(por: Dict, sci: Dict,
                     pcol: str = "TIPO_ROTA",
                     scol: str = "Sinalização da Rota") -> Tuple[str, str]:
    v   = coalesce(por.get(pcol, ""), sci.get(scol, ""))
    src = "PORTAL" if _clean_str(por.get(pcol, "")) else "SCIENCE"
    return v.strip().upper(), src


def derive_central(por: Dict, sci: Dict,
                   pcol: str = "CENTRAL",
                   scol: str = "Central Origem") -> Tuple[str, str]:
    v   = coalesce(por.get(pcol, ""), sci.get(scol, ""))
    src = "PORTAL" if _clean_str(por.get(pcol, "")) else "SCIENCE"
    return v.strip().upper(), src


def derive_rotulos(por: Dict, le: str = "LABEL_E",
                   ls: str = "LABEL_S",
                   concat: bool = True, sep: str = " | ") -> str:
    e = _clean_str(por.get(le, ""))
    s = _clean_str(por.get(ls, ""))
    if concat and e and s:
        return f"{e}{sep}{s}"
    return e or s


def derive_operadora(arq3_val: str, por: Dict, sci: Dict,
                     op_col: str = "Operadora Origem") -> str:
    return coalesce(
        arq3_val,
        por.get("EMPRESA", ""),
        sci.get(op_col, ""),
        sci.get("Operadora destino", ""),
    ).strip().upper()


# ── Índice do Arquivo 3 ───────────────────────────────────────────────────

def build_ref_index(ref_df, config: Optional[Dict] = None) -> Dict[str, Any]:
    """
    Indexa Arquivo 3.
    Prioridade para nomes de coluna:
      1. Configuração explícita do wizard (arq3_*_col)
      2. Autodetecção case-insensitive (_resolve_col)

    Cria 3 chaves por linha:
      {CENTRAL}|TR|{TIPO_ROTA}  — match por tipo de rota
      {CENTRAL}|RL|{ROTULOS}    — match por rótulos
      {CENTRAL}|ANY             — fallback: qualquer linha da central
    """
    idx: Dict[str, Any] = {}
    if ref_df is None or (hasattr(ref_df, "empty") and ref_df.empty):
        return idx

    cols    = list(ref_df.columns)
    cfg     = config or {}
    NONE    = "(nenhuma)"

    def _pick(cfg_key: str, *auto_candidates: str) -> str:
        """Usa config do wizard se disponível, senão autodetecta."""
        manual = cfg.get(cfg_key, "")
        if manual and manual != NONE and manual in cols:
            return manual
        return _resolve_col(cols, *auto_candidates)

    col_central = _pick("arq3_central_col",
        "Central", "CENTRAL", "Central Origem", "CENTRAL_ORIGEM")
    col_uf      = _pick("arq3_uf_col",
        "UF", "uf", "Estado", "ESTADO")
    col_cluster = _pick("arq3_cluster_col",
        "CLUSTER", "Cluster", "cluster", "AGRUPAMENTO", "Agrupamento",
        "CLUSTER_NOME", "Cluster Nome", "CLUSTER NOME")
    col_tipo    = _pick("arq3_tipo_rota_col",
        "Tipo de Rota", "TIPO DE ROTA", "TIPO_DE_ROTA", "TIPO_ROTA",
        "Tipo Rota", "TIPO ROTA")
    col_rotulos = _pick("arq3_rotulos_col",
        "Rótulos de Linha", "RÓTULOS DE LINHA", "Rotulos de Linha",
        "ROTULOS DE LINHA", "ROTULOS_DE_LINHA", "LABEL_E", "Rótulo", "ROTULO")
    col_rede    = _pick("arq3_rede_col",    "REDE", "Rede", "rede")
    col_op      = _pick("arq3_operadora_col","OPERADORA", "Operadora", "operadora")
    col_den     = _pick("arq3_denominacao_col",
        "Denominação", "DENOMINAÇÃO", "Denominacao", "DENOMINACAO",
        "Denominacão", "DENOMINACÃO")

    log.info(
        "Arquivo 3 colunas: Central=%r UF=%r Cluster=%r Tipo=%r Rotulos=%r",
        col_central, col_uf, col_cluster, col_tipo, col_rotulos,
    )

    if not col_central:
        log.warning("Arquivo 3: coluna 'Central' não encontrada. Colunas: %s", cols)
        return idx

    if not col_cluster:
        log.warning(
            "Arquivo 3: coluna 'CLUSTER' não encontrada! "
            "CLUSTER ficará vazio. Configure no wizard (seção 8). "
            "Colunas disponíveis: %s", cols
        )
    if not col_uf:
        log.warning(
            "Arquivo 3: coluna 'UF' não encontrada! "
            "Configure no wizard (seção 8). Colunas disponíveis: %s", cols
        )

    for _, row in ref_df.iterrows():
        cen  = _clean_str(row.get(col_central, "")).upper()
        tipo = _clean_str(row.get(col_tipo,    "")).upper() if col_tipo    else ""
        rot  = _clean_str(row.get(col_rotulos, "")).upper() if col_rotulos else ""
        uf_v = _clean_str(row.get(col_uf,      ""))         if col_uf      else ""
        cl_v = _clean_str(row.get(col_cluster, ""))         if col_cluster else ""
        re_v = _clean_str(row.get(col_rede,    ""))         if col_rede    else ""
        op_v = _clean_str(row.get(col_op,      ""))         if col_op      else ""
        dn_v = _clean_str(row.get(col_den,     ""))         if col_den     else ""

        if not cen:
            continue

        rec = {
            "Central":          cen,
            "Tipo de Rota":     tipo,
            "Rótulos de Linha": rot,
            "UF":               uf_v,
            "CLUSTER":          cl_v,
            "REDE":             re_v,
            "OPERADORA":        op_v,
            "Denominação":      dn_v,
        }

        if tipo:
            idx.setdefault(f"{cen}|TR|{tipo}", rec)
        if rot:
            idx.setdefault(f"{cen}|RL|{rot}", rec)
        # ANY garante retorno mesmo sem tipo/rótulos
        idx.setdefault(f"{cen}|ANY", rec)

    n_centrais = sum(1 for k in idx if k.endswith("|ANY"))
    log.info("Arquivo 3 indexado: %d chaves, %d centrais únicas", len(idx), n_centrais)
    return idx


def lookup_ref(central: str, tipo_rota: str, rotulos: str,
               ref_index: Dict[str, Any]) -> Optional[Dict]:
    """Busca em cascata: TR → RL → ANY."""
    c = _clean_str(central).upper()
    t = _clean_str(tipo_rota).upper()
    r = _clean_str(rotulos).upper()
    if not c:
        return None
    return (
        (ref_index.get(f"{c}|TR|{t}") if t else None)
        or (ref_index.get(f"{c}|RL|{r}") if r else None)
        or ref_index.get(f"{c}|ANY")
    )


# ── Linha de saída ────────────────────────────────────────────────────────

def build_output_row(sci_row: Dict, por_row: Dict, source_tag: str,
                     ref_index: Dict[str, Any],
                     uf_map: Dict[str, str],
                     config: Dict) -> Dict:

    central, cen_src = derive_central(
        por_row, sci_row,
        pcol=config.get("central_portal_col", "CENTRAL"),
        scol=config.get("central_sci_col", "Central Origem"),
    )
    tipo_rota, tr_src = derive_tipo_rota(
        por_row, sci_row,
        pcol=config.get("tipo_rota_portal_col", "TIPO_ROTA"),
        scol=config.get("tipo_rota_sci_col", "Sinalização da Rota"),
    )
    rotulos_portal = derive_rotulos(
        por_row,
        le=config.get("label_e_col", "LABEL_E"),
        ls=config.get("label_s_col", "LABEL_S"),
        concat=config.get("concat_labels", True),
        sep=config.get("label_sep", " | "),
    )

    # Arquivo 3 — fonte primária para UF e CLUSTER
    ref = lookup_ref(central, tipo_rota, rotulos_portal, ref_index)

    def _ref(key: str) -> str:
        return _clean_str(ref.get(key, "")) if ref else ""

    # UF em cascata
    uf = _ref("UF")

    if not uf:
        cn_to_uf = config.get("_cn_to_uf_map", {})
        if cn_to_uf:
            cn_col = config.get("cn_sci_col", "")
            if cn_col and cn_col != "(nenhuma)":
                cn_val = clean_cn(sci_row.get(cn_col, ""))
                if cn_val:
                    uf = cn_to_uf.get(cn_val, "")
            if not uf:
                cn_val = clean_cn(sci_row.get("CN", ""))
                if cn_val:
                    uf = cn_to_uf.get(cn_val, "")

    if not uf and uf_map:
        cnl_sci = clean_cnl(sci_row.get(config.get("cnl_sci_col", "CNL"), ""))
        cnl_por = clean_cnl(coalesce(
            por_row.get(config.get("cnl_por_col", "CNL_PPI"), ""),
            por_row.get("CNL", ""),
            por_row.get("PPI", ""),
        ))
        cnl_val = cnl_sci or cnl_por
        if cnl_val:
            uf = uf_map.get(cnl_val, "") or uf_map.get(cnl_val.lstrip("0"), "")

    cluster     = _ref("CLUSTER")
    rotulos     = _ref("Rótulos de Linha") or rotulos_portal
    denominacao = _ref("Denominação") or coalesce(
        sci_row.get(config.get("denominacao_sci_col", "Descrição"), ""),
        por_row.get("DESIGNACAO", ""),
    )
    operadora = derive_operadora(
        _ref("OPERADORA"), por_row, sci_row,
        op_col=config.get("operadora_sci_col", "Operadora Origem"),
    )

    cnl_val_log = (
        clean_cnl(sci_row.get(config.get("cnl_sci_col", "CNL"), ""))
        or clean_cnl(por_row.get(config.get("cnl_por_col", "CNL_PPI"), ""))
    )

    log.debug(
        "Row central=%s tipo=%s tag=%s uf=%s cluster=%s arq3=%s",
        central, tipo_rota, source_tag,
        uf or "(vazio)", cluster or "(vazio)", ref is not None,
    )

    return {
        "REDE":             derive_rede(source_tag),
        "UF":               uf,
        "CLUSTER":          cluster,
        "Tipo de Rota":     tipo_rota,
        "Central":          central,
        "Rótulos de Linha": rotulos,
        "OPERADORA":        operadora,
        "Denominação":      denominacao,
        "_source_tag":      source_tag,
        "_central_src":     cen_src,
        "_tipo_rota_src":   tr_src,
        "_arq3_match":      str(ref is not None),
        "_cnl_val":         cnl_val_log,
    }
