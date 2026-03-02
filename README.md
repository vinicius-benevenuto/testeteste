"""
core/map_rules.py - Regras de mapeamento.
CLUSTER e UF vêm do Arquivo 3, lookup por Central.

Busca de colunas:
  1. Config manual do wizard (arq3_*_col)
  2. Match exato case+accent-insensitive
  3. Match por substring case+accent-insensitive
  4. Scan inteligente: analisa valores das colunas para identificar UF e CLUSTER
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

# Siglas de estados brasileiros para detecção de coluna UF
_BR_UF = {
    "AC","AL","AP","AM","BA","CE","DF","ES","GO",
    "MA","MT","MS","MG","PA","PB","PR","PE","PI",
    "RJ","RN","RS","RO","RR","SC","SP","SE","TO",
}


def _normalize_key(s: str) -> str:
    """Remove acentos, uppercase — para comparação insensível."""
    s = unicodedata.normalize("NFKD", str(s or ""))
    s = "".join(c for c in s if not unicodedata.combining(c))
    return s.upper().strip()


def _s(v: Any) -> str:
    """String limpa."""
    s = str(v or "").strip()
    return "" if s.lower() in _INVALID else s


def _find_col(cols: List[str], *candidates: str) -> str:
    """
    Busca coluna com 3 estratégias (todas accent+case-insensitive):
      1. Match exato
      2. Coluna contém o candidato (substring)
      3. Candidato contém a coluna (coluna é substring do candidato)
    """
    norm_map = {_normalize_key(c): c for c in cols}

    for cand in candidates:
        cand_n = _normalize_key(cand)
        # 1. Exato
        if cand_n in norm_map:
            return norm_map[cand_n]
        # 2. Coluna contém candidato
        for col_n, col_orig in norm_map.items():
            if cand_n in col_n:
                return col_orig
        # 3. Candidato contém coluna (ex: candidato="CLUSTER" contém col="CLUS")
        for col_n, col_orig in norm_map.items():
            if col_n and col_n in cand_n:
                return col_orig
    return ""


def _scan_uf_col(df, cols: List[str], exclude: set) -> str:
    """
    Varre colunas restantes e detecta qual contém siglas de UF como valores.
    Retorna o nome da coluna se >= 80% dos valores não-nulos forem siglas válidas.
    """
    import pandas as pd
    for col in cols:
        if col in exclude:
            continue
        try:
            sample = df[col].dropna().astype(str).str.strip().str.upper()
            sample = sample[sample.str.len() == 2]
            if len(sample) < 5:
                continue
            pct = sample.isin(_BR_UF).mean()
            if pct >= 0.80:
                log.info("Coluna UF detectada por scan de valores: %r (%.0f%% UFs válidas)", col, pct * 100)
                return col
        except Exception:
            continue
    return ""


def _scan_cluster_col(df, cols: List[str], exclude: set) -> str:
    """
    Varre colunas e detecta qual tem valores que parecem códigos de cluster:
    - Curtos (2-30 chars), consistentes, não todos iguais
    - Não são datas, números, UFs
    Heurística: coluna com variabilidade moderada e valores não-numéricos curtos.
    """
    import pandas as pd
    candidates = []
    for col in cols:
        if col in exclude:
            continue
        try:
            sample = df[col].dropna().astype(str).str.strip()
            sample = sample[sample.str.len().between(2, 40)]
            if len(sample) < 5:
                continue
            # Descarta colunas com muitos valores únicos (texto livre) ou poucos (constantes)
            n_unique = sample.nunique()
            if n_unique < 2 or n_unique > len(sample) * 0.8:
                continue
            # Descarta colunas que parecem ser UF (2 chars, todas maiúsculas e válidas)
            if sample.str.len().max() == 2 and sample.str.upper().isin(_BR_UF).mean() > 0.5:
                continue
            # Pontuação: prefere colunas com nome contendo "cluster", "grupo", "agrup"
            name_n = _normalize_key(col)
            score = 0
            if "CLUSTER" in name_n or "CLUS" in name_n:
                score += 10
            if "GRUPO" in name_n or "AGRUP" in name_n:
                score += 5
            if "NOME" in name_n or "NAME" in name_n:
                score += 2
            # Pontuação por padrão de valores (ex: "CL-SP-01", "CLUSTER_SP_01")
            looks_like_code = sample.str.contains(r"[A-Z]{2}[_\-]", regex=True, na=False).mean()
            score += looks_like_code * 5
            candidates.append((score, col))
        except Exception:
            continue

    if not candidates:
        return ""
    candidates.sort(key=lambda x: -x[0])
    best_score, best_col = candidates[0]
    if best_score > 0:
        log.info("Coluna CLUSTER detectada por scan: %r (score=%.1f)", best_col, best_score)
        return best_col
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


def _score_rec(rec: Dict) -> int:
    """Pontua um registro: mais campos preenchidos = melhor."""
    return sum(1 for k in ("UF", "CLUSTER", "Rótulos de Linha",
                            "OPERADORA", "Denominação", "Tipo de Rota")
               if rec.get(k, ""))


# ─────────────────────────────────────────────────────────────────────────────
# ÍNDICE DO ARQUIVO 3
# ─────────────────────────────────────────────────────────────────────────────

def build_ref_index(ref_df, config: Optional[Dict] = None) -> Dict[str, Any]:
    """
    Indexa Arquivo 3 para lookup por Central.

    Detecção de colunas em 4 camadas:
      1. Config manual do wizard
      2. Match exato accent+case-insensitive
      3. Match por substring accent+case-insensitive
      4. Scan inteligente de valores (para UF e CLUSTER)

    ANY: mantém o registro com MAIOR score (mais campos preenchidos) por central,
    garantindo que uma primeira linha vazia não apague dados de linhas seguintes.
    """
    if ref_df is None or (hasattr(ref_df, "empty") and ref_df.empty):
        return {}

    cols = list(ref_df.columns)
    cfg  = config or {}
    NONE = "(nenhuma)"

    def _pick(cfg_key: str, *subs: str) -> str:
        manual = cfg.get(cfg_key, "")
        if manual and manual != NONE and manual in cols:
            return manual
        return _find_col(cols, *subs)

    col_central = _pick("arq3_central_col",
        "Central", "CENTRAL", "Central Origem", "CENTRAL_ORIGEM",
        "CENTRAL ORIGEM", "CENTRAL_DE_ORIGEM")
    col_uf      = _pick("arq3_uf_col",
        "UF", "ESTADO", "Estado", "SIGLA", "SIGLA_UF", "SIGLA UF",
        "ESTADO_UF", "ESTADO UF")
    col_cluster = _pick("arq3_cluster_col",
        "CLUSTER", "Cluster", "CLÚSTER", "Clúster",
        "CLUSTER DE ROTEAMENTO", "CLUSTER_DE_ROTEAMENTO",
        "CLUSTER NOME", "CLUSTER_NOME", "CLUSTER ID", "CLUSTER_ID",
        "AGRUPAMENTO", "AGRUPAMENTO DE ROTAS", "GRUPO DE ROTEAMENTO",
        "GRUPO", "CLUS")
    col_tipo    = _pick("arq3_tipo_rota_col",
        "Tipo de Rota", "TIPO DE ROTA", "TIPO_DE_ROTA", "TIPO_ROTA",
        "Tipo Rota", "TIPO ROTA", "TIPO")
    col_rotulos = _pick("arq3_rotulos_col",
        "Rótulos de Linha", "RÓTULOS DE LINHA", "Rotulos de Linha",
        "ROTULOS DE LINHA", "ROTULOS_DE_LINHA", "RÓTULO", "ROTULO",
        "LABEL_E", "LABEL E", "LABELS")
    col_rede    = _pick("arq3_rede_col",    "REDE", "Rede", "NETWORK")
    col_op      = _pick("arq3_operadora_col","OPERADORA", "Operadora",
                         "EMPRESA", "EMPRESA OPERADORA")
    col_den     = _pick("arq3_denominacao_col",
        "Denominação", "DENOMINAÇÃO", "Denominacao", "DENOMINACAO",
        "Denominacão", "DENOMINACÃO", "DESCRIÇÃO", "Descrição",
        "DESCRICAO", "DESCRICÃO", "DENOMINACAO_ROTA")

    # ── Scan inteligente para colunas não encontradas ────────────────────
    mapped = {c for c in [col_central, col_uf, col_cluster, col_tipo,
                           col_rotulos, col_rede, col_op, col_den] if c}
    if not col_uf:
        col_uf = _scan_uf_col(ref_df, cols, mapped)
        if col_uf:
            mapped.add(col_uf)
    if not col_cluster:
        col_cluster = _scan_cluster_col(ref_df, cols, mapped)
        if col_cluster:
            mapped.add(col_cluster)

    # ── Log de diagnóstico ───────────────────────────────────────────────
    log.info("─── Arquivo 3: detecção de colunas (de %d disponíveis) ───", len(cols))
    log.info("  Central  → %r", col_central)
    log.info("  UF       → %r%s", col_uf,      "" if col_uf      else " ⚠️ NÃO ENCONTRADA")
    log.info("  CLUSTER  → %r%s", col_cluster, "" if col_cluster else " ⚠️ NÃO ENCONTRADA")
    log.info("  Tipo     → %r", col_tipo)
    log.info("  Rótulos  → %r", col_rotulos)
    log.info("  Colunas disponíveis: %s", cols)

    if not col_central:
        log.error("Arquivo 3: coluna Central NÃO encontrada!")
        return {}

    # ── Iteração ─────────────────────────────────────────────────────────
    idx: Dict[str, Any]        = {}
    best_any: Dict[str, Dict]  = {}
    best_score: Dict[str, int] = {}

    for _, row in ref_df.iterrows():
        cen  = _s(row.get(col_central, "")).upper()
        if not cen:
            continue

        tipo = _s(row.get(col_tipo,    "")).upper() if col_tipo    else ""
        rot  = _s(row.get(col_rotulos, "")).upper() if col_rotulos else ""
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

        # TR e RL: primeiro match exato vence
        if tipo:
            idx.setdefault(f"{cen}|TR|{tipo}", rec)
        if rot:
            idx.setdefault(f"{cen}|RL|{rot}", rec)

        # ANY: guarda o MELHOR registro por central (maior número de campos preenchidos)
        sc = _score_rec(rec)
        if sc > best_score.get(cen, -1):
            best_score[cen] = sc
            best_any[cen]   = rec

    for cen, rec in best_any.items():
        idx[f"{cen}|ANY"] = rec

    n_centrais = len(best_any)
    log.info("Arquivo 3 indexado: %d chaves, %d centrais", len(idx), n_centrais)

    # Amostra de verificação
    for cen, rec in list(best_any.items())[:5]:
        uf_ok = "✅" if rec.get("UF")      else "❌"
        cl_ok = "✅" if rec.get("CLUSTER") else "❌"
        log.info("  Central=%-18s %sUF=%-4s %sCLUSTER=%s",
                 cen, uf_ok, rec.get("UF",""), cl_ok, rec.get("CLUSTER",""))

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

    # Arquivo 3 — fonte primária para UF e CLUSTER
    ref = lookup_ref(central, tipo_rota, rotulos_portal, ref_index)

    def _ref(key: str) -> str:
        return _s(ref.get(key, "")) if ref else ""

    # UF: Arquivo 3 → CN/DDD direto → CNL
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

    log.debug("Row central=%-15s uf=%-4s cluster=%-12s arq3=%s",
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
