"""
core/map_rules.py
Regras de mapeamento de negócio.
Prioridade UF/CLUSTER:
  1. Arquivo 3 via Central + Tipo de Rota  (match exato)
  2. Arquivo 3 via Central + Rótulos       (match por rótulos)
  3. Arquivo 3 via Central sozinha         (fallback - NOVA REGRA)
  4. CN direto (DDD) → cn_to_uf
  5. CNL → CN → UF via uf_map pré-calculado
"""
from __future__ import annotations
from typing import Any, Dict, Optional, Tuple

from app.utils.logging_utils import get_logger
from app.utils.cnl_utils import clean_cnl, clean_cn

log = get_logger(__name__)

OUTPUT_COLUMNS = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]

_INVALID = {"nan", "none", "null", "n/a", "na", "#n/a", ""}


def derive_rede(source_tag: str) -> str:
    if (source_tag or "").upper() in ("SCIENCE", "BOTH"):
        return "VIVO-SMP"
    return "VIVO-STFC"


def coalesce(*values: Any) -> str:
    for v in values:
        s = str(v or "").strip()
        if s and s.lower() not in _INVALID:
            return s
    return ""


def derive_tipo_rota(por: Dict, sci: Dict,
                     pcol: str = "TIPO_ROTA",
                     scol: str = "Sinalização da Rota") -> Tuple[str, str]:
    v = coalesce(por.get(pcol, ""), sci.get(scol, ""))
    src = "PORTAL" if str(por.get(pcol, "")).strip() else "SCIENCE"
    return v.strip().upper(), src


def derive_central(por: Dict, sci: Dict,
                   pcol: str = "CENTRAL",
                   scol: str = "Central Origem") -> Tuple[str, str]:
    v = coalesce(por.get(pcol, ""), sci.get(scol, ""))
    src = "PORTAL" if str(por.get(pcol, "")).strip() else "SCIENCE"
    return v.strip().upper(), src


def derive_rotulos(por: Dict, le: str = "LABEL_E",
                   ls: str = "LABEL_S",
                   concat: bool = True, sep: str = " | ") -> str:
    e = coalesce(por.get(le, ""))
    s = coalesce(por.get(ls, ""))
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


def build_ref_index(ref_df) -> Dict[str, Any]:
    """
    Indexa Arquivo 3 com 3 tipos de chave:
      {central}|TR|{tipo_rota}   — match exato
      {central}|RL|{rotulos}     — match por rótulos
      {central}|ANY              — fallback: qualquer linha com essa central
    """
    idx: Dict[str, Any] = {}
    if ref_df is None or (hasattr(ref_df, "empty") and ref_df.empty):
        return idx

    for _, row in ref_df.iterrows():
        rec  = row.to_dict()
        cen  = str(row.get("Central", "") or "").strip().upper()
        tipo = str(row.get("Tipo de Rota", "") or "").strip().upper()
        rot  = str(row.get("Rótulos de Linha", "") or "").strip().upper()

        if not cen:
            continue

        if tipo and tipo not in _INVALID:
            idx.setdefault(f"{cen}|TR|{tipo}", rec)

        if rot and rot not in _INVALID:
            idx.setdefault(f"{cen}|RL|{rot}", rec)

        # Chave ANY — garante que qualquer central conhecida retorna UF+CLUSTER
        idx.setdefault(f"{cen}|ANY", rec)

    n_centrais = sum(1 for k in idx if k.endswith("|ANY"))
    log.debug("Arquivo 3 indexado: %d chaves, %d centrais únicas", len(idx), n_centrais)
    return idx


def lookup_ref(central: str, tipo_rota: str, rotulos: str,
               ref_index: Dict[str, Any]) -> Optional[Dict]:
    """
    1. Central + Tipo de Rota
    2. Central + Rótulos
    3. Central sozinha  ← garante UF/CLUSTER mesmo sem Tipo de Rota
    """
    c = central.strip().upper()
    t = tipo_rota.strip().upper() if tipo_rota else ""
    r = rotulos.strip().upper()   if rotulos   else ""
    if not c:
        return None
    return (
        (ref_index.get(f"{c}|TR|{t}") if t and t not in _INVALID else None)
        or (ref_index.get(f"{c}|RL|{r}") if r and r not in _INVALID else None)
        or ref_index.get(f"{c}|ANY")
    )


def build_output_row(sci_row: Dict, por_row: Dict, source_tag: str,
                     ref_index: Dict[str, Any],
                     uf_map: Dict[str, str],
                     config: Dict) -> Dict:

    # ── Derivações base ────────────────────────────────────────────────
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

    # ── Lookup Arquivo 3 ──────────────────────────────────────────────
    ref = lookup_ref(central, tipo_rota, rotulos_portal, ref_index)

    def _ref(key: str) -> str:
        if not ref:
            return ""
        v = str(ref.get(key, "") or "").strip()
        return "" if v.lower() in _INVALID else v

    # ── UF em cascata ─────────────────────────────────────────────────
    uf = ""

    # 1. Arquivo 3 (via Central — funciona mesmo sem Tipo de Rota)
    if ref:
        uf = _ref("UF")

    # 2. CN direto (Área Ponta B → DDD → UF)
    cn_to_uf = config.get("_cn_to_uf_map", {})
    if not uf and cn_to_uf:
        cn_col = config.get("cn_sci_col", "")
        if cn_col and cn_col != "(nenhuma)":
            cn_val = clean_cn(sci_row.get(cn_col, ""))
            if cn_val:
                uf = cn_to_uf.get(cn_val, "")
        if not uf:
            cn_val = clean_cn(sci_row.get("CN", ""))
            if cn_val:
                uf = cn_to_uf.get(cn_val, "")

    # 3. CNL → CN → UF via uf_map (banco)
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

    # ── Restante dos campos ────────────────────────────────────────────
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
        "Row central=%s tipo=%s tag=%s uf=%s cluster=%s arq3_match=%s",
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
