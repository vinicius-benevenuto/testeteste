"""
core/map_rules.py

Estratégia de lookup no Arquivo 3:
  Força bruta bidirecional — para cada linha de entrada (Science ou Portal),
  tenta TODAS as colunas da linha contra TODAS as colunas do Arquivo 3.
  A primeira combinação que produzir match único retorna o registro do Arq3.

  Ordem de preferência configurável; colunas "ruidosas" (datas, numéricos
  curtos, IDs sequenciais) são descartadas automaticamente.
"""
from __future__ import annotations
import unicodedata
from typing import Any, Dict, List, Optional, Set, Tuple

from app.utils.logging_utils import get_logger
from app.utils.cnl_utils import clean_cnl, clean_cn

log = get_logger(__name__)

OUTPUT_COLUMNS = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]

_INVALID = {"nan", "none", "null", "n/a", "na", "#n/a", ""}

# Colunas de entrada que NUNCA devem ser usadas como chave de lookup
# (evita falsos matches com datas, CNLs, sequenciais, flags etc.)
_SKIP_INPUT: Set[str] = {
    _n for _n in (
        "ROW_NUMBER","DATA ATIVACAO","DATA DESATIVACAO","DT_ATIVACAO",
        "DT_CADASTRO","DT_EXECUCAO","ID ROTA","SEQUENCIAL",
        "CNL","CNL_PPI","PPI","OPC","DPC","NUM SSI","TRAF_CURSADO",
        "BILHETADOR","EOT","OC","INTERCON","ECR","CSP","SENTIDO",
        "SINALIZACAO","TIPO_REGISTRO","CATEGORIA","CATEG_DESCRICAO",
        "ROTA_EXCLUSIVA","RESP_CADASTRO","RESP_EXECUCAO","OBS","GATEWAY",
        "TRAFEGO","TRAF","CIRC","OCUP","TMR","CANAIS","CAPS","MAIOR1ERL",
        "STATUS","I","Z","X",
    )
}

# Colunas do Arquivo 3 que NUNCA são chaves de lookup
_SKIP_ARQ3: Set[str] = {
    _n for _n in (
        "REDE","X","Z","I","OBS","STATUS","CANAIS","CAPS","TMR",
        "MAIOR1ERL","CIRC_ATIVOS","CIRC_BLOQ","OCUP_TOTAL",
        "TRAF_ENTR","TRAF_SAIDA","TMR ENT","TMR SAIDA",
        "MAX DE CAPACIDADE","MAX DE CIRCUITOS ATIVOS",
        "MAX DE CIRCUITOS BLOQUEADOS","MAX DE OCUPACAO TOTAL",
        "MAX DE TRAFEGO ENTRADA","MAX DE TRAFEGO SAIDA",
        "MEDIA DE TMR ENTRADA","MEDIA DE TMR SAIDA",
        "TRAFEGO TOTAL (ERL)",
    )
}

# Colunas de entrada com PRIORIDADE para Science e Portal (tentadas primeiro)
_PRIORITY_SCI = [
    "Central Interna", "Central Origem", "COD_CCC Interna",
    "COD_CCC Origem", "Descrição",
]
_PRIORITY_POR = [
    "CENTRAL", "LABEL_E", "LABEL_S", "ROTA_E", "ROTA_S", "DESIGNACAO",
]


# ── utilidades ────────────────────────────────────────────────────────────────

def _norm(s: str) -> str:
    s = unicodedata.normalize("NFKD", str(s or ""))
    return "".join(c for c in s if not unicodedata.combining(c)).upper().strip()

_normalize_key = _norm


def _s(v: Any) -> str:
    s = str(v or "").strip()
    return "" if s.lower() in _INVALID else s


def _find_col(cols: List[str], *candidates: str) -> str:
    nm = {_norm(c): c for c in cols}
    for cand in candidates:
        cn = _norm(cand)
        if cn in nm:
            return nm[cn]
        for col_n, col_orig in nm.items():
            if cn in col_n:
                return col_orig
    return ""


def _is_lookup_val(v: str) -> bool:
    """Valor candidato a chave: não vazio, não puramente numérico curto."""
    if not v or len(v) < 3:
        return False
    if v.isdigit() and len(v) < 6:
        return False
    return True


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
    v = coalesce(por.get(pcol, ""), sci.get(scol, ""),
                 sci.get("Central Interna", ""))
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


# ── construção do índice ───────────────────────────────────────────────────────

def build_ref_index(ref_df, config: Optional[Dict] = None) -> Dict[str, Any]:
    """
    Constrói um índice multi-coluna do Arquivo 3.

    Resultado:
      {
        "_by_col": {
          "NOME_COLUNA_ARQ3_NORM": {
            "VALOR_NORM": melhor_rec,
            ...
          },
          ...
        },
        "_col_map":   { NORM → nome_original },   # colunas do Arq3
        "_skip_arq3": set de colunas ignoradas,
        "_col_cluster": nome original da coluna CLUSTER,
        "_col_uf":      nome original da coluna UF,
      }

    O lookup varre todos os pares (coluna_input × coluna_arq3) em ordem de
    prioridade até encontrar match.
    """
    if ref_df is None or (hasattr(ref_df, "empty") and ref_df.empty):
        return {"_by_col": {}, "_col_map": {}, "_skip_arq3": set(),
                "_col_cluster": "", "_col_uf": ""}

    cols   = list(ref_df.columns)
    cfg    = config or {}
    NONE   = "(nenhuma)"

    def _pick(cfg_key: str, *subs: str) -> str:
        manual = cfg.get(cfg_key, "")
        if manual and manual != NONE and manual in cols:
            return manual
        return _find_col(cols, *subs)

    col_central = _pick("arq3_central_col", "Central", "CENTRAL")
    col_uf      = _pick("arq3_uf_col",      "UF", "ESTADO", "Estado", "SIGLA")
    col_cluster = _pick("arq3_cluster_col",
                        "CLUSTER", "Cluster", "CLÚSTER", "AGRUPAMENTO",
                        "CLUSTER DE ROTEAMENTO", "CLUSTER NOME")
    col_tipo    = _pick("arq3_tipo_rota_col",
                        "Tipo de Rota", "TIPO DE ROTA", "TIPO_ROTA")
    col_rotulos = _pick("arq3_rotulos_col",
                        "Rótulos de Linha", "RÓTULOS DE LINHA",
                        "Rotulos de Linha", "ROTULOS DE LINHA", "ROTULO", "LABEL_E")
    col_rede    = _pick("arq3_rede_col",    "REDE", "Rede")
    col_op      = _pick("arq3_operadora_col","OPERADORA", "Operadora", "EMPRESA")
    col_den     = _pick("arq3_denominacao_col",
                        "Denominação","DENOMINAÇÃO","Denominacao","DENOMINACAO",
                        "Denominacão","DESCRIÇÃO","Descrição")

    log.info("── Arquivo 3 ──────────────────────────────────────────")
    log.info("  Central=%r  UF=%r  CLUSTER=%r  Rótulos=%r",
             col_central, col_uf, col_cluster, col_rotulos)
    if not col_cluster:
        log.error("  ⚠️  CLUSTER não encontrado! Colunas: %s", cols)
    if not col_uf:
        log.error("  ⚠️  UF não encontrada! Colunas: %s", cols)

    # Determina quais colunas do Arq3 usar como chaves de lookup
    col_map     = {_norm(c): c for c in cols}
    skip_arq3   = {_norm(c) for c in cols if _norm(c) in _SKIP_ARQ3}
    # Nunca indexar UF como chave (SP, AM etc. são demasiado curtos e ambíguos)
    if col_uf:
        skip_arq3.add(_norm(col_uf))

    # Constrói índice: by_col[col_arq3_norm][valor_norm] → melhor_rec
    by_col: Dict[str, Dict[str, Any]] = {}
    # Para ANY (por Central): guarda melhor por score
    best_any:   Dict[str, Tuple[int, Any]] = {}  # central_norm → (score, rec)

    for _, row in ref_df.iterrows():
        # Monta registro normalizado
        cen  = _s(row.get(col_central, "")).upper() if col_central else ""
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

        sc = _score_rec(rec)

        # Indexa CADA coluna do Arq3 (exceto as ignoradas)
        for arq3_col in cols:
            cn = _norm(arq3_col)
            if cn in skip_arq3:
                continue
            val = _s(row.get(arq3_col, ""))
            if not _is_lookup_val(_norm(val)):
                continue
            val_n = _norm(val)
            if cn not in by_col:
                by_col[cn] = {}
            # Guarda melhor registro por score para este valor nesta coluna
            existing = by_col[cn].get(val_n)
            if existing is None or _score_rec(existing) < sc:
                by_col[cn][val_n] = rec

            # Rótulos: indexa cada parte separada por " | "
            if arq3_col == col_rotulos and " | " in val:
                for part in val.split(" | "):
                    pn = _norm(part.strip())
                    if _is_lookup_val(pn):
                        existing_p = by_col[cn].get(pn)
                        if existing_p is None or _score_rec(existing_p) < sc:
                            by_col[cn][pn] = rec

        # Atualiza best_any por Central
        if cen:
            prev_sc, _ = best_any.get(cen, (-1, None))
            if sc > prev_sc:
                best_any[cen] = (sc, rec)

    n_centrais = len(best_any)
    n_chaves   = sum(len(v) for v in by_col.values())
    log.info("Indexado: %d centrais | %d colunas Arq3 indexadas | %d chaves totais",
             n_centrais, len(by_col), n_chaves)
    for cen, (_, rec) in list(best_any.items())[:8]:
        uf_ok = "✅" if rec.get("UF")      else "❌"
        cl_ok = "✅" if rec.get("CLUSTER") else "❌"
        log.info("  %-22s %sUF=%-4s %sCLUSTER=%s",
                 cen, uf_ok, rec.get("UF",""), cl_ok, rec.get("CLUSTER",""))

    return {
        "_by_col":      by_col,
        "_col_map":     col_map,
        "_skip_arq3":   skip_arq3,
        "_col_cluster": col_cluster,
        "_col_uf":      col_uf,
        "_best_any":    {cen: rec for cen, (_, rec) in best_any.items()},
    }


# ── lookup força bruta ────────────────────────────────────────────────────────

def _brute_lookup(row_dict: Dict[str, str],
                  ref_index: Dict[str, Any],
                  priority_cols: List[str]) -> Optional[Dict]:
    """
    Para cada coluna do row_dict, tenta fazer match em qualquer coluna do Arq3.

    Estratégia em 3 passos:
      1. Tenta priority_cols do input × todas as colunas Arq3 (em ordem)
      2. Tenta todas as demais colunas do input × todas as colunas Arq3
      3. Tenta sub-partes dos valores (split por _ e por espaço)
    """
    by_col  = ref_index.get("_by_col", {})
    if not by_col:
        return None

    # Colunas Arq3 em ordem de preferência
    # (Central primeiro, Rótulos segundo, resto depois)
    arq3_cols_ordered = sorted(
        by_col.keys(),
        key=lambda c: (
            0 if "CENTRAL" in c else
            1 if "ROTULO" in c or "LABEL" in c else
            2
        )
    )

    skip_input = {_norm(c) for c in _SKIP_INPUT}

    def _try_value(val_n: str) -> Optional[Dict]:
        if not _is_lookup_val(val_n):
            return None
        for arq3_col_n in arq3_cols_ordered:
            rec = by_col[arq3_col_n].get(val_n)
            if rec:
                return rec
        return None

    # Passo 1: colunas prioritárias
    for input_col in priority_cols:
        # tenta a coluna exata e variações de case
        for col, val in row_dict.items():
            if _norm(col) != _norm(input_col):
                continue
            v = _s(val)
            if not v:
                continue
            rec = _try_value(_norm(v))
            if rec:
                log.debug("Match P1: input_col=%r val=%r → CLUSTER=%s",
                          col, v, rec.get("CLUSTER",""))
                return rec

    # Passo 2: todas as colunas do input não-skip
    for col, val in row_dict.items():
        if _norm(col) in skip_input:
            continue
        v = _s(val)
        if not v:
            continue
        rec = _try_value(_norm(v))
        if rec:
            log.debug("Match P2: input_col=%r val=%r → CLUSTER=%s",
                      col, v, rec.get("CLUSTER",""))
            return rec

    # Passo 3: sub-partes dos valores (split por _ e espaço)
    for col, val in row_dict.items():
        if _norm(col) in skip_input:
            continue
        v = _s(val)
        if not v or len(v) < 5:
            continue
        # Tenta prefixos de underscore (ex: "MBCAFPA_AFAFA2" → "MBCAFPA")
        parts = v.replace("_", " ").split()
        for part in parts:
            pn = _norm(part)
            if _is_lookup_val(pn):
                rec = _try_value(pn)
                if rec:
                    log.debug("Match P3 (subpart): input_col=%r part=%r → CLUSTER=%s",
                              col, part, rec.get("CLUSTER",""))
                    return rec

    return None


def lookup_ref(central: str, tipo_rota: str, rotulos: str,
               ref_index: Dict[str, Any],
               label_e: str = "", label_s: str = "") -> Optional[Dict]:
    """Alias de compatibilidade — usa _brute_lookup internamente."""
    row = {"Central": central, "Tipo de Rota": tipo_rota,
           "Rótulos de Linha": rotulos, "LABEL_E": label_e, "LABEL_S": label_s}
    return _brute_lookup(row, ref_index, _PRIORITY_POR + _PRIORITY_SCI)


# ── linha de saída ─────────────────────────────────────────────────────────────

def build_output_row(sci_row: Dict, por_row: Dict, source_tag: str,
                     ref_index: Dict[str, Any],
                     uf_map: Dict[str, str],
                     config: Dict) -> Dict:

    cfg = config or {}

    tipo_rota, tr_src = derive_tipo_rota(
        por_row, sci_row,
        pcol=cfg.get("tipo_rota_portal_col", "TIPO_ROTA"),
        scol=cfg.get("tipo_rota_sci_col", "Sinalização da Rota"),
    )

    le_col = cfg.get("label_e_col", "LABEL_E")
    ls_col = cfg.get("label_s_col", "LABEL_S")
    rotulos_portal = derive_rotulos(por_row, le=le_col, ls=ls_col,
                                    concat=cfg.get("concat_labels", True),
                                    sep=cfg.get("label_sep", " | "))

    # ── Lookup Arquivo 3: força bruta por source ──────────────────────
    if source_tag == "SCIENCE":
        priority = _PRIORITY_SCI
        ref = _brute_lookup(sci_row, ref_index, priority)
    elif source_tag == "PORTAL":
        priority = _PRIORITY_POR
        ref = _brute_lookup(por_row, ref_index, priority)
    else:  # BOTH
        ref = (_brute_lookup(por_row, ref_index, _PRIORITY_POR)
               or _brute_lookup(sci_row, ref_index, _PRIORITY_SCI))

    def _ref(key: str) -> str:
        return _s(ref.get(key, "")) if ref else ""

    # ── Central ──────────────────────────────────────────────────────
    central, cen_src = derive_central(
        por_row, sci_row,
        pcol=cfg.get("central_portal_col", "CENTRAL"),
        scol=cfg.get("central_sci_col", "Central Origem"),
    )

    # ── UF ──────────────────────────────────────────────────────────
    uf = _ref("UF")

    if not uf:
        cn_to_uf = cfg.get("_cn_to_uf_map", {})
        if cn_to_uf:
            for cn_col in (cfg.get("cn_sci_col", ""), "Área Ponta B", "Area Ponta B"):
                if cn_col and cn_col not in ("(nenhuma)", ""):
                    cn_val = clean_cn(sci_row.get(cn_col, ""))
                    if cn_val:
                        uf = cn_to_uf.get(cn_val, "")
                        if uf:
                            break

    if not uf and uf_map:
        cnl_sci = clean_cnl(sci_row.get(cfg.get("cnl_sci_col", "CNL"), ""))
        cnl_por = clean_cnl(coalesce(
            por_row.get(cfg.get("cnl_por_col", "CNL_PPI"), ""),
            por_row.get("CNL", ""), por_row.get("PPI", ""),
        ))
        cnl_val = cnl_sci or cnl_por
        if cnl_val:
            uf = uf_map.get(cnl_val, "") or uf_map.get(cnl_val.lstrip("0"), "")

    # ── Restantes ────────────────────────────────────────────────────
    cluster     = _ref("CLUSTER")
    rotulos     = _ref("Rótulos de Linha") or rotulos_portal
    denominacao = _ref("Denominação") or coalesce(
        sci_row.get(cfg.get("denominacao_sci_col", "Descrição"), ""),
        por_row.get("DESIGNACAO", ""),
    )
    operadora = derive_operadora(
        _ref("OPERADORA"), por_row, sci_row,
        op_col=cfg.get("operadora_sci_col", "Operadora Origem"),
    )

    cnl_log = (clean_cnl(sci_row.get(cfg.get("cnl_sci_col", "CNL"), ""))
               or clean_cnl(por_row.get(cfg.get("cnl_por_col", "CNL_PPI"), "")))

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

