"""
core/map_rules.py - Regras de mapeamento.
UF e CLUSTER vêm do Arquivo 3, lookup por Central.
Detecção de coluna 100% automática por substring.
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


def _s(v: Any) -> str:
    """String limpa — descarta NaN e similares."""
    s = str(v or "").strip()
    return "" if s.lower() in _INVALID else s


def _find_col(cols: List[str], *substrings: str) -> str:
    """
    Encontra coluna cujo nome contenha qualquer um dos substrings (case-insensitive).
    Prioriza match exato, depois parcial.
    """
    up = {c.strip().upper(): c for c in cols}
    # 1. Match exato
    for sub in substrings:
        if sub.upper() in up:
            return up[sub.upper()]
    # 2. Match parcial (coluna contém o substring)
    for sub in substrings:
        for col_up, col_orig in up.items():
            if sub.upper() in col_up:
                return col_orig
    return ""


def derive_rede(tag: str) -> str:
    return "VIVO-SMP" if (tag or "").upper() in ("SCIENCE", "BOTH") else "VIVO-STFC"


def coalesce(*values: Any) -> str:
    for v in values:
        s = _s(v)
        if s:
            return s
    return ""


def derive_tipo_rota(por: Dict, sci: Dict, pcol="TIPO_ROTA",
                     scol="Sinalização da Rota") -> Tuple[str, str]:
    v = coalesce(por.get(pcol, ""), sci.get(scol, ""))
    return v.upper(), ("PORTAL" if _s(por.get(pcol, "")) else "SCIENCE")


def derive_central(por: Dict, sci: Dict, pcol="CENTRAL",
                   scol="Central Origem") -> Tuple[str, str]:
    v = coalesce(por.get(pcol, ""), sci.get(scol, ""))
    return v.upper(), ("PORTAL" if _s(por.get(pcol, "")) else "SCIENCE")


def derive_rotulos(por: Dict, le="LABEL_E", ls="LABEL_S",
                   concat=True, sep=" | ") -> str:
    e = _s(por.get(le, ""))
    s = _s(por.get(ls, ""))
    return f"{e}{sep}{s}" if (concat and e and s) else (e or s)


def derive_operadora(arq3_val: str, por: Dict, sci: Dict,
                     op_col="Operadora Origem") -> str:
    return coalesce(arq3_val, por.get("EMPRESA", ""),
                    sci.get(op_col, ""), sci.get("Operadora destino", "")).upper()


# ─────────────────────────────────────────────────────────────────────────────
# ÍNDICE DO ARQUIVO 3
# ─────────────────────────────────────────────────────────────────────────────

def build_ref_index(ref_df, config: Optional[Dict] = None) -> Dict[str, Any]:
    """
    Indexa Arquivo 3 detectando colunas automaticamente por substring.
    Não depende de nomes exatos — encontra CLUSTER mesmo que se chame
    'CLUSTER DE ROTEAMENTO', 'Cluster Mkt', etc.

    Gera 3 chaves por linha:
      {CENTRAL}|TR|{TIPO}    — match com tipo de rota
      {CENTRAL}|RL|{ROTULOS} — match com rótulos
      {CENTRAL}|ANY          — fallback: qualquer linha da central
    """
    idx: Dict[str, Any] = {}
    if ref_df is None or (hasattr(ref_df, "empty") and ref_df.empty):
        return idx

    cols = list(ref_df.columns)
    cfg  = config or {}
    NONE = "(nenhuma)"

    def _pick(cfg_key: str, *subs: str) -> str:
        """Usa config do wizard se válido, senão autodetecta por substring."""
        manual = cfg.get(cfg_key, "")
        if manual and manual != NONE and manual in cols:
            return manual
        return _find_col(cols, *subs)

    # Detecção de colunas — MUITO permissiva para nunca falhar
    col_central = _pick("arq3_central_col",
        "Central", "CENTRAL", "Central Origem", "CENTRAL_ORIGEM", "CENTRAL ORIGEM")
    col_uf      = _pick("arq3_uf_col",
        "UF", "ESTADO", "Estado", "SIGLA", "SIGLA_UF", "SIGLA UF")
    col_cluster = _pick("arq3_cluster_col",
        "CLUSTER", "Cluster", "AGRUPAMENTO", "Agrupamento",
        "CLUSTER_NOME", "CLUSTER NOME", "CLUSTER_ID", "CLUSTER ID",
        "CLUSTER DE ROTEAMENTO", "CLUS")   # match parcial captura qualquer variação
    col_tipo    = _pick("arq3_tipo_rota_col",
        "Tipo de Rota", "TIPO DE ROTA", "TIPO_DE_ROTA", "TIPO_ROTA",
        "TIPO ROTA", "TIPO", "tipo_de_rota")
    col_rotulos = _pick("arq3_rotulos_col",
        "Rótulos de Linha", "RÓTULOS DE LINHA", "Rotulos de Linha",
        "ROTULOS DE LINHA", "ROTULOS_DE_LINHA", "RÓTULO", "ROTULO", "LABEL_E")
    col_rede    = _pick("arq3_rede_col",    "REDE", "Rede", "NETWORK")
    col_op      = _pick("arq3_operadora_col","OPERADORA", "Operadora", "EMPRESA")
    col_den     = _pick("arq3_denominacao_col",
        "Denominação", "DENOMINAÇÃO", "Denominacao", "DENOMINACAO",
        "Denominacão", "DENOMINACÃO", "DESCRICAO", "DESCRIÇÃO", "Descrição")

    # Log detalhado para depuração
    log.info("─── Arquivo 3: detecção de colunas ───")
    log.info("  Central  → %r", col_central)
    log.info("  UF       → %r", col_uf)
    log.info("  CLUSTER  → %r", col_cluster)
    log.info("  Tipo     → %r", col_tipo)
    log.info("  Rótulos  → %r", col_rotulos)
    log.info("  Todas as colunas disponíveis: %s", cols)

    if not col_central:
        log.error("Arquivo 3 SEM coluna Central! Colunas: %s", cols)
        return idx
    if not col_cluster:
        log.warning("Arquivo 3: coluna CLUSTER não encontrada! "
                    "CLUSTER ficará vazio. Colunas: %s", cols)
    if not col_uf:
        log.warning("Arquivo 3: coluna UF não encontrada! "
                    "UF via Arquivo 3 ficará vazio. Colunas: %s", cols)

    for _, row in ref_df.iterrows():
        cen  = _s(row.get(col_central, "")).upper()
        tipo = _s(row.get(col_tipo,    "")).upper() if col_tipo    else ""
        rot  = _s(row.get(col_rotulos, "")).upper() if col_rotulos else ""
        uf_v = _s(row.get(col_uf,      ""))         if col_uf      else ""
        cl_v = _s(row.get(col_cluster, ""))         if col_cluster else ""
        re_v = _s(row.get(col_rede,    ""))         if col_rede    else ""
        op_v = _s(row.get(col_op,      ""))         if col_op      else ""
        dn_v = _s(row.get(col_den,     ""))         if col_den     else ""

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
        idx.setdefault(f"{cen}|ANY", rec)

    n_centrais = sum(1 for k in idx if k.endswith("|ANY"))
    log.info("Arquivo 3 indexado: %d chaves, %d centrais únicas", len(idx), n_centrais)

    # Amostra para confirmar que CLUSTER está sendo capturado
    sample = [(k, v.get("CLUSTER", ""), v.get("UF", ""))
              for k, v in idx.items() if k.endswith("|ANY")][:5]
    for k, cl, uf in sample:
        log.info("  Amostra: Central=%s UF=%r CLUSTER=%r", k[:-4], uf, cl)

    return idx


def lookup_ref(central: str, tipo_rota: str, rotulos: str,
               ref_index: Dict[str, Any]) -> Optional[Dict]:
    c = _s(central).upper()
    t = _s(tipo_rota).upper()
    r = _s(rotulos).upper()
    if not c:
        return None
    return (
        (ref_index.get(f"{c}|TR|{t}") if t else None)
        or (ref_index.get(f"{c}|RL|{r}") if r else None)
        or ref_index.get(f"{c}|ANY")
    )


# ─────────────────────────────────────────────────────────────────────────────
# LINHA DE SAÍDA
# ─────────────────────────────────────────────────────────────────────────────

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

    # ── Arquivo 3 (fonte primária: UF + CLUSTER) ──────────────────────
    ref = lookup_ref(central, tipo_rota, rotulos_portal, ref_index)

    def _ref(key: str) -> str:
        return _s(ref.get(key, "")) if ref else ""

    # ── UF em cascata ─────────────────────────────────────────────────
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

    cnl_log = (clean_cnl(sci_row.get(config.get("cnl_sci_col", "CNL"), ""))
               or clean_cnl(por_row.get(config.get("cnl_por_col", "CNL_PPI"), "")))

    log.debug("Row central=%s uf=%s cluster=%s arq3=%s",
              central, uf or "(vazio)", cluster or "(vazio)", ref is not None)

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
        "_cnl_val":         cnl_log,
    }

