"""
core/map_rules.py — Regras de mapeamento.

Estratégia de lookup no Arquivo 3 (em ordem de prioridade):
  1. Central + Tipo de Rota  (match exato)
  2. Central + Rótulos de Linha
  3. Central sozinha (ANY — melhor registro por score)
  4. Rótulo individual (LABEL_E ou LABEL_S do Portal) → match direto no Arq3
     └─ Fallback crítico quando Central do Portal ≠ Central do Arquivo 3

UF adicional:
  5. CN/DDD direto
  6. CNL → CN → UF
"""
from __future__ import annotations
import unicodedata
from typing import Any, Dict, List, Optional, Tuple

from app.utils.logging_utils import get_logger
from app.utils.cnl_utils import clean_cnl, clean_cn

log = get_logger(__name__)

OUTPUT_COLUMNS = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]

_INVALID = {"nan", "none", "null", "n/a", "na", "#n/a", ""}

_BR_UF = {
    "AC","AL","AP","AM","BA","CE","DF","ES","GO",
    "MA","MT","MS","MG","PA","PB","PR","PE","PI",
    "RJ","RN","RS","RO","RR","SC","SP","SE","TO",
}


# ─── utilidades ─────────────────────────────────────────────────────────────

def _norm(s: str) -> str:
    """Remove acentos, uppercase — para comparação insensível."""
    s = unicodedata.normalize("NFKD", str(s or ""))
    return "".join(c for c in s if not unicodedata.combining(c)).upper().strip()

_normalize_key = _norm   # alias público (usado em pages.py)


def _s(v: Any) -> str:
    s = str(v or "").strip()
    return "" if s.lower() in _INVALID else s


def _find_col(cols: List[str], *candidates: str) -> str:
    """Encontra coluna: exato accent-insensitive, depois por substring."""
    norm_map = {_norm(c): c for c in cols}
    for cand in candidates:
        cn = _norm(cand)
        if cn in norm_map:
            return norm_map[cn]
        for col_n, col_orig in norm_map.items():
            if cn in col_n:
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


def derive_tipo_rota(por: Dict, sci: Dict,
                     pcol="TIPO_ROTA", scol="Sinalização da Rota") -> Tuple[str, str]:
    v = coalesce(por.get(pcol, ""), sci.get(scol, ""))
    return v.upper(), ("PORTAL" if _s(por.get(pcol, "")) else "SCIENCE")


def derive_central(por: Dict, sci: Dict,
                   pcol="CENTRAL", scol="Central Origem") -> Tuple[str, str]:
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


def _score_rec(rec: Dict) -> int:
    return sum(1 for k in ("UF", "CLUSTER", "Rótulos de Linha",
                            "OPERADORA", "Denominação", "Tipo de Rota")
               if rec.get(k, ""))


# ─── índice do Arquivo 3 ─────────────────────────────────────────────────────

def build_ref_index(ref_df, config: Optional[Dict] = None) -> Dict[str, Any]:
    """
    Constrói dois índices a partir do Arquivo 3:

    idx_central  — chaveado por Central (3 variações: TR, RL, ANY)
    idx_rotulo   — chaveado por cada valor individual de Rótulo de Linha
                   → fallback quando Central do Portal ≠ Central do Arquivo 3

    Para |ANY|: guarda o registro de MAIOR score por Central
    (evita pegar linha vazia quando a Central aparece múltiplas vezes).
    """
    if ref_df is None or (hasattr(ref_df, "empty") and ref_df.empty):
        return {"_central": {}, "_rotulo": {}}

    cols = list(ref_df.columns)
    cfg  = config or {}
    NONE = "(nenhuma)"

    def _pick(cfg_key: str, *subs: str) -> str:
        manual = cfg.get(cfg_key, "")
        if manual and manual != NONE and manual in cols:
            return manual
        return _find_col(cols, *subs)

    col_central = _pick("arq3_central_col",
        "Central", "CENTRAL", "Central Origem", "CENTRAL_ORIGEM")
    col_uf      = _pick("arq3_uf_col",
        "UF", "ESTADO", "Estado", "SIGLA", "SIGLA_UF")
    col_cluster = _pick("arq3_cluster_col",
        "CLUSTER", "Cluster", "CLÚSTER", "Clúster",
        "CLUSTER DE ROTEAMENTO", "CLUSTER_NOME", "CLUSTER NOME",
        "AGRUPAMENTO", "GRUPO DE ROTEAMENTO", "CLUS")
    col_tipo    = _pick("arq3_tipo_rota_col",
        "Tipo de Rota", "TIPO DE ROTA", "TIPO_DE_ROTA", "TIPO_ROTA", "TIPO ROTA")
    col_rotulos = _pick("arq3_rotulos_col",
        "Rótulos de Linha", "RÓTULOS DE LINHA", "Rotulos de Linha",
        "ROTULOS DE LINHA", "ROTULOS_DE_LINHA", "RÓTULO", "ROTULO", "LABEL_E")
    col_rede    = _pick("arq3_rede_col",    "REDE", "Rede", "NETWORK")
    col_op      = _pick("arq3_operadora_col","OPERADORA", "Operadora", "EMPRESA")
    col_den     = _pick("arq3_denominacao_col",
        "Denominação", "DENOMINAÇÃO", "Denominacao", "DENOMINACAO",
        "Denominacão", "DENOMINACÃO", "DESCRIÇÃO", "Descrição")

    log.info("Arquivo 3: Central=%r UF=%r CLUSTER=%r Rótulos=%r | %d colunas",
             col_central, col_uf, col_cluster, col_rotulos, len(cols))

    if not col_central:
        log.error("Arquivo 3: coluna Central NÃO encontrada! Colunas: %s", cols)
        return {"_central": {}, "_rotulo": {}}
    if not col_cluster:
        log.warning("Arquivo 3: CLUSTER não detectado! Colunas: %s", cols)
    if not col_uf:
        log.warning("Arquivo 3: UF não detectada! Colunas: %s", cols)

    idx_central: Dict[str, Any]  = {}
    best_any:    Dict[str, Dict] = {}
    best_score:  Dict[str, int]  = {}
    idx_rotulo:  Dict[str, Any]  = {}  # rotulo_value → rec

    for _, row in ref_df.iterrows():
        cen  = _s(row.get(col_central, "")).upper()
        if not cen:
            continue

        tipo = _s(row.get(col_tipo,    "")).upper() if col_tipo    else ""
        rot  = _s(row.get(col_rotulos, ""))         if col_rotulos else ""
        uf_v = _s(row.get(col_uf,      ""))         if col_uf      else ""
        cl_v = _s(row.get(col_cluster, ""))         if col_cluster else ""
        re_v = _s(row.get(col_rede,    ""))         if col_rede    else ""
        op_v = _s(row.get(col_op,      ""))         if col_op      else ""
        dn_v = _s(row.get(col_den,     ""))         if col_den     else ""

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

        # Índice por Central
        rot_upper = rot.upper()
        if tipo:
            idx_central.setdefault(f"{cen}|TR|{tipo}", rec)
        if rot_upper:
            idx_central.setdefault(f"{cen}|RL|{rot_upper}", rec)

        # ANY: guarda melhor por score
        sc = _score_rec(rec)
        if sc > best_score.get(cen, -1):
            best_score[cen] = sc
            best_any[cen]   = rec

        # Índice por Rótulo individual
        # Cada linha do Arquivo 3 tem UM rótulo → mapeia diretamente
        if rot:
            # Indexa pela parte antes do " | " também (se concatenado)
            for part in rot.split(" | "):
                part = part.strip().upper()
                if part and part not in idx_rotulo:
                    idx_rotulo[part] = rec

    for cen, rec in best_any.items():
        idx_central[f"{cen}|ANY"] = rec

    n_cen = len(best_any)
    n_rot = len(idx_rotulo)
    log.info("Arquivo 3 indexado: %d centrais, %d rótulos únicos", n_cen, n_rot)

    for cen, rec in list(best_any.items())[:5]:
        uf_ok = "✅" if rec.get("UF")      else "❌"
        cl_ok = "✅" if rec.get("CLUSTER") else "❌"
        log.info("  %-20s %sUF=%-4s %sCLUSTER=%s",
                 cen, uf_ok, rec.get("UF",""), cl_ok, rec.get("CLUSTER",""))

    return {"_central": idx_central, "_rotulo": idx_rotulo}


def lookup_ref(central: str, tipo_rota: str, rotulos: str,
               ref_index: Dict[str, Any],
               label_e: str = "", label_s: str = "") -> Optional[Dict]:
    """
    Busca em cascata:
    1. Central + Tipo de Rota
    2. Central + Rótulos concatenados
    3. Central sozinha (ANY)
    4. LABEL_E direto no índice de rótulos   ← fallback quando Central ≠ Arq3
    5. LABEL_S direto no índice de rótulos
    """
    idx_c = ref_index.get("_central", {})
    idx_r = ref_index.get("_rotulo",  {})

    c = _s(central).upper()
    t = _s(tipo_rota).upper()
    r = _s(rotulos).upper()
    le = _s(label_e).upper()
    ls = _s(label_s).upper()

    # 1–3: por Central
    if c:
        result = (
            (idx_c.get(f"{c}|TR|{t}") if t else None)
            or (idx_c.get(f"{c}|RL|{r}") if r else None)
            or idx_c.get(f"{c}|ANY")
        )
        if result:
            return result

    # 4–5: por Rótulo individual (LABEL_E / LABEL_S do Portal)
    if le and (result := idx_r.get(le)):
        return result
    if ls and (result := idx_r.get(ls)):
        return result

    # 6: tenta cada parte dos rótulos concatenados
    if r:
        for part in r.split(" | "):
            part = part.strip().upper()
            if part and (result := idx_r.get(part)):
                return result

    return None


# ─── linha de saída ──────────────────────────────────────────────────────────

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

    # Rótulos — para lookup e para saída
    le_col = config.get("label_e_col", "LABEL_E")
    ls_col = config.get("label_s_col", "LABEL_S")
    label_e = _s(por_row.get(le_col, ""))
    label_s = _s(por_row.get(ls_col, ""))
    rotulos_portal = derive_rotulos(
        por_row, le=le_col, ls=ls_col,
        concat=config.get("concat_labels", True),
        sep=config.get("label_sep", " | "),
    )

    # Lookup Arquivo 3 — agora com label_e e label_s como fallback
    ref = lookup_ref(central, tipo_rota, rotulos_portal, ref_index,
                     label_e=label_e, label_s=label_s)

    def _ref(key: str) -> str:
        return _s(ref.get(key, "")) if ref else ""

    # UF: Arquivo 3 → CN direto → CNL
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

    log.debug("central=%-15s le=%-25s uf=%-4s cluster=%-12s arq3=%s",
              central, label_e, uf or "(vazio)", cluster or "(vazio)", ref is not None)

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
        "_label_e":         label_e,
    }
