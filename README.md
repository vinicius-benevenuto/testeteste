"""
core/map_rules.py
Regras de mapeamento de negócio:
  - REDE: VIVO-SMP (Science) | VIVO-STFC (Portal) | VIVO-SMP (ambos)
  - UF: via CNL→CN→UF (resolvido pelo repositório)
  - CLUSTER/Rótulos/Denominação: via Arquivo 3 com fallbacks
  - Tipo de Rota: Portal.TIPO_ROTA → Science.Sinalização da Rota
  - Central: Portal.CENTRAL → Science.Central Origem
  - OPERADORA: Arquivo 3 → Portal.EMPRESA → Science.Operadora Origem
"""
from __future__ import annotations
from typing import Any, Dict, List, Optional, Tuple
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

OUTPUT_COLUMNS = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]

# ── Regra REDE ────────────────────────────────────────────────────────────

def derive_rede(source_tag: str) -> str:
    """
    source_tag: SCIENCE | PORTAL | BOTH
    BOTH → prioriza SMP (Science contribuiu).
    """
    tag = source_tag.upper()
    if tag in ("SCIENCE", "BOTH"):
        return "VIVO-SMP"
    return "VIVO-STFC"

# ── Coalesce genérico ─────────────────────────────────────────────────────

def coalesce(*values: str) -> str:
    """Retorna o primeiro valor não-vazio."""
    for v in values:
        s = str(v or "").strip()
        if s:
            return s
    return ""

# ── Tipo de Rota ──────────────────────────────────────────────────────────

def derive_tipo_rota(portal_row: Dict, science_row: Dict,
                     portal_col: str = "TIPO_ROTA",
                     sci_col: str = "Sinalização da Rota") -> Tuple[str, str]:
    """Retorna (valor, origem)."""
    v = coalesce(portal_row.get(portal_col, ""), science_row.get(sci_col, ""))
    orig = "PORTAL" if portal_row.get(portal_col, "").strip() else "SCIENCE"
    return v.strip().upper(), orig

# ── Central ───────────────────────────────────────────────────────────────

def derive_central(portal_row: Dict, science_row: Dict,
                   portal_col: str = "CENTRAL",
                   sci_col: str = "Central Origem") -> Tuple[str, str]:
    v = coalesce(portal_row.get(portal_col, ""), science_row.get(sci_col, ""))
    orig = "PORTAL" if portal_row.get(portal_col, "").strip() else "SCIENCE"
    return v.strip().upper(), orig

# ── Rótulos de Linha ──────────────────────────────────────────────────────

def derive_rotulos(portal_row: Dict, label_e: str = "LABEL_E",
                   label_s: str = "LABEL_S", concat: bool = True,
                   sep: str = " | ") -> str:
    e = str(portal_row.get(label_e, "") or "").strip()
    s = str(portal_row.get(label_s, "") or "").strip()
    if concat and e and s:
        return f"{e}{sep}{s}"
    return e or s

# ── OPERADORA ─────────────────────────────────────────────────────────────

def derive_operadora(arq3_val: str, portal_row: Dict,
                     science_row: Dict,
                     sci_op_col: str = "Operadora Origem") -> str:
    return coalesce(
        arq3_val,
        portal_row.get("EMPRESA", ""),
        science_row.get(sci_op_col, ""),
        science_row.get("Operadora destino", ""),
    ).strip().upper()

# ── Match no Arquivo 3 ────────────────────────────────────────────────────

def build_ref_index(ref_df) -> Dict[str, Dict]:
    """
    Constrói índice do Arquivo 3 para lookup rápido.
    Chaves: central|tipo_de_rota  e  central|rotulos
    """
    import pandas as pd
    index: Dict[str, Dict] = {}
    if ref_df is None or (hasattr(ref_df, "empty") and ref_df.empty):
        return index
    for _, row in ref_df.iterrows():
        central = str(row.get("Central", "") or "").strip().upper()
        tipo    = str(row.get("Tipo de Rota", "") or "").strip().upper()
        rotulos = str(row.get("Rótulos de Linha", "") or "").strip().upper()
        rec     = row.to_dict()
        if central and tipo:
            index[f"{central}|TR|{tipo}"] = rec
        if central and rotulos:
            index[f"{central}|RL|{rotulos}"] = rec
    return index


def lookup_ref(central: str, tipo_rota: str, rotulos: str,
               ref_index: Dict[str, Dict]) -> Optional[Dict]:
    """Busca no Arquivo 3 por chave Central+TipoRota, depois Central+Rótulos."""
    c = central.strip().upper()
    t = tipo_rota.strip().upper()
    r = rotulos.strip().upper()
    return (ref_index.get(f"{c}|TR|{t}") or
            ref_index.get(f"{c}|RL|{r}"))

# ── Linha completa ────────────────────────────────────────────────────────

def build_output_row(
    sci_row: Dict,
    por_row: Dict,
    source_tag: str,
    ref_index: Dict[str, Dict],
    uf_map: Dict[str, str],
    config: Dict,
) -> Dict:
    """
    Constrói uma linha de saída com todas as 8 colunas + _source_tag.
    config: dicionário com opções configuráveis pelo usuário.
    """
    # ── Central e Tipo de Rota primeiro (usados como chave de lookup) ──
    central, central_src = derive_central(
        por_row, sci_row,
        portal_col=config.get("central_portal_col", "CENTRAL"),
        sci_col=config.get("central_sci_col", "Central Origem"),
    )
    tipo_rota, tr_src = derive_tipo_rota(
        por_row, sci_row,
        portal_col=config.get("tipo_rota_portal_col", "TIPO_ROTA"),
        sci_col=config.get("tipo_rota_sci_col", "Sinalização da Rota"),
    )

    # ── Rótulos portal (para lookup e fallback) ──
    rotulos_portal = derive_rotulos(
        por_row,
        label_e=config.get("label_e_col", "LABEL_E"),
        label_s=config.get("label_s_col", "LABEL_S"),
        concat=config.get("concat_labels", True),
        sep=config.get("label_sep", " | "),
    )

    # ── Lookup no Arquivo 3 ──
    ref = lookup_ref(central, tipo_rota, rotulos_portal, ref_index)

    # ── REDE ──
    rede = derive_rede(source_tag)

    # ── UF via CNL → CN → UF ──
    cnl_val = coalesce(
        sci_row.get(config.get("cnl_sci_col", "CNL"), ""),
        por_row.get(config.get("cnl_por_col", "CNL_PPI"), ""),
        por_row.get("PPI", ""),
    ).strip()
    uf = uf_map.get(cnl_val, "")
    if not uf and cnl_val:
        uf = uf_map.get(cnl_val.lstrip("0"), "")
    # Fallback: UF do Arquivo 3
    if not uf and ref:
        uf = str(ref.get("UF", "") or "").strip()

    # ── Campos do Arquivo 3 ──
    cluster     = str(ref.get("CLUSTER", "") or "").strip() if ref else ""
    rotulos     = str(ref.get("Rótulos de Linha", "") or "").strip() if ref else rotulos_portal
    denominacao = str(ref.get("Denominação", "") or "").strip() if ref else coalesce(
        sci_row.get(config.get("denominacao_sci_col", "Descrição"), ""),
        por_row.get("DESIGNACAO", ""),
    )
    operadora = derive_operadora(
        str(ref.get("OPERADORA", "") or "").strip() if ref else "",
        por_row, sci_row,
        sci_op_col=config.get("operadora_sci_col", "Operadora Origem"),
    )

    log.debug("Row central=%s tipo=%s tag=%s uf=%s arq3_match=%s",
              central, tipo_rota, source_tag, uf, ref is not None)

    return {
        "REDE":             rede,
        "UF":               uf,
        "CLUSTER":          cluster,
        "Tipo de Rota":     tipo_rota,
        "Central":          central,
        "Rótulos de Linha": rotulos,
        "OPERADORA":        operadora,
        "Denominação":      denominacao,
        "_source_tag":      source_tag,
        "_central_src":     central_src,
        "_tipo_rota_src":   tr_src,
        "_arq3_match":      str(ref is not None),
        "_cnl_val":         cnl_val,
    }
