"""
core/pipeline.py
================
PROCESSO OFICIAL – SANEAMENTO, VALIDAÇÃO E ENRIQUECIMENTO DE ROTAS

PARTE 1 – Science (Etapas 1-4)
  Etapa 1: filtrar Central que começa com "M"
  Etapa 2: remover rotas desativadas (data_desativacao válida e < hoje)
  Etapa 3: construir chave Central[:7]+"_"+ID_Rota, validar contra Arq3
  Etapa 4: enriquecer com todos os campos do Arq3

PARTE 2 – Portal (Etapas 5-9)
  Etapa 5: converter coluna CENTRAL com tabela de-para fixa
  Etapa 6: escolher 6 chars finais (LABEL_E ou LABEL_S)
  Etapa 7: construir Rótulos de Linha = central_conv[:7]+"_"+label[:6]
  Etapa 8: buscar rótulo no Arq3
  Etapa 9: criar nova rota se não encontrada (VIVO-STFC, UF via CN/DESIGNACAO)
"""
from __future__ import annotations

import re
import unicodedata
from datetime import date
from typing import Any, Dict, List, Optional, Tuple

import pandas as pd

from app.utils.logging_utils import get_logger

log = get_logger(__name__)

# ── Constantes ────────────────────────────────────────────────────────────────

OUTPUT_COLUMNS = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]

# CN (DDD) → UF  —  tabela oficial de DDDs brasileiros
CN_TO_UF: Dict[str, str] = {
    "11": "SP", "12": "SP", "13": "SP", "14": "SP", "15": "SP",
    "16": "SP", "17": "SP", "18": "SP", "19": "SP",
    "21": "RJ", "22": "RJ", "24": "RJ",
    "27": "ES", "28": "ES",
    "31": "MG", "32": "MG", "33": "MG", "34": "MG", "35": "MG",
    "37": "MG", "38": "MG",
    "41": "PR", "42": "PR", "43": "PR", "44": "PR", "45": "PR", "46": "PR",
    "47": "SC", "48": "SC", "49": "SC",
    "51": "RS", "53": "RS", "54": "RS", "55": "RS",
    "61": "DF",   # DDD 61 = Distrito Federal (Brasília)
    "62": "GO", "63": "TO", "64": "GO",
    "65": "MT", "66": "MT",
    "67": "MS",
    "68": "AC",
    "69": "RO",
    "71": "BA", "73": "BA", "74": "BA", "75": "BA", "77": "BA",
    "79": "SE",
    "81": "PE", "82": "AL", "83": "PB", "84": "RN",
    "85": "CE", "86": "PI", "87": "PE", "88": "CE", "89": "PI",
    "91": "PA", "92": "AM", "93": "PA", "94": "PA",
    "95": "RR", "96": "AP", "97": "AM",
    "98": "MA", "99": "MA",
}

# Conversão de CENTRAL no Portal (de-para exato, case-insensitive)
CENTRAL_PORTAL_MAP: Dict[str, str] = {
    "SPO.PL.AS6": "ASIPASA",
    "SPO.JG.AS6": "ASIJGRA",
    "RJO.BA.AS1": "ASIRJOA",
    "BHE.FU.AS1": "ASIBHEA",
    "SDR.CB.AS1": "ASISDRA",
    "BLM.PD.AS1": "ASIBLMA",
    "CTA.PG.AS1": "ASICTAA",
    "PAE.IP.AS1": "ASIPAEA",
}

_INVALID = {"nan", "none", "null", "n/a", "na", "#n/a", ""}
_DATE_1899_1900 = {"1899", "1900"}


# ── Utilidades ────────────────────────────────────────────────────────────────

def _s(v: Any) -> str:
    """Converte para string limpa; retorna '' para nulos/inválidos."""
    if v is None:
        return ""
    try:
        import math
        if isinstance(v, float) and math.isnan(v):
            return ""
    except (TypeError, ValueError):
        pass
    s = str(v).strip()
    return "" if s.lower() in _INVALID else s


def _norm(s: str) -> str:
    """Normaliza para UPPER sem acentos."""
    s = unicodedata.normalize("NFKD", str(s or ""))
    return "".join(c for c in s if not unicodedata.combining(c)).upper().strip()


def _find_col(cols: List[str], *candidates: str) -> str:
    """
    Encontra coluna por nome (insensível a acento/case).
    Primeiro tenta match exato; depois substring.
    Candidatos vazios são ignorados.
    """
    nm = {_norm(c): c for c in cols}
    for cand in candidates:
        if not cand:
            continue
        cn = _norm(cand)
        if not cn:
            continue
        # Match exato
        if cn in nm:
            return nm[cn]
    # Segunda passagem: substring (só para candidatos com ≥ 4 chars)
    for cand in candidates:
        if not cand:
            continue
        cn = _norm(cand)
        if len(cn) < 4:
            continue
        for col_n, col_orig in nm.items():
            if cn in col_n:
                return col_orig
    return ""


def _parse_date(val: str) -> Optional[date]:
    """
    Tenta parsear data em vários formatos.
    Retorna None se não for uma data válida ou for 1899/1900.
    """
    v = _s(val)
    if not v:
        return None
    # Verifica se é apenas ano 1899 ou 1900
    if v.strip()[:4] in _DATE_1899_1900:
        return None
    for fmt in ("%d/%m/%Y", "%Y-%m-%d", "%d-%m-%Y", "%Y/%m/%d",
                "%d/%m/%y", "%m/%d/%Y", "%Y%m%d"):
        try:
            return date(*[int(x) for x in
                          (pd.to_datetime(v, format=fmt).timetuple()[:3])])
        except Exception:
            pass
    # Tenta pandas genérico
    try:
        dt = pd.to_datetime(v, dayfirst=True, errors="raise")
        if dt.year in (1899, 1900):
            return None
        return dt.date()
    except Exception:
        return None


def _extract_cn(text: str) -> Optional[str]:
    """Extrai CN (DDD de 2 dígitos) de uma string como DESIGNACAO."""
    v = _s(text)
    if not v:
        return None
    # Tenta número inteiro direto
    try:
        cn = str(int(float(v)))
        if cn in CN_TO_UF:
            return cn
    except (ValueError, TypeError):
        pass
    # Extrai primeiro número de 2 dígitos que é um CN válido
    for m in re.finditer(r"\b(\d{2})\b", v):
        cn = m.group(1)
        if cn in CN_TO_UF:
            return cn
    return None


def _build_arq3_rotulo_index(arq3_df: pd.DataFrame,
                              rotulos_col: str) -> Dict[str, Dict]:
    """
    Cria índice Rótulo de Linha → registro completo do Arq3.
    Cada rótulo aponta para a linha correspondente.
    """
    idx: Dict[str, Dict] = {}
    if arq3_df is None or arq3_df.empty or not rotulos_col:
        return idx
    for _, row in arq3_df.iterrows():
        rot = _s(row.get(rotulos_col, "")).upper().strip()
        if rot and rot not in idx:
            idx[rot] = {str(k): _s(v) for k, v in row.items()}
    return idx


def _build_arq3_central_index(arq3_df: pd.DataFrame,
                               central_col: str,
                               rede_filter: Optional[str] = None,
                               rede_col: Optional[str] = None) -> Dict[str, List[Dict]]:
    """
    Cria índice Central → lista de registros do Arq3 (para enriquecer Science).
    Opcionalmente filtra por REDE.
    """
    idx: Dict[str, List[Dict]] = {}
    if arq3_df is None or arq3_df.empty or not central_col:
        return idx
    for _, row in arq3_df.iterrows():
        if rede_filter and rede_col:
            rede = _s(row.get(rede_col, "")).upper()
            if rede_filter.upper() not in rede:
                continue
        cen = _s(row.get(central_col, "")).upper().strip()
        if cen:
            rec = {str(k): _s(v) for k, v in row.items()}
            idx.setdefault(cen, []).append(rec)
    return idx


def _extract_arq3_cols(arq3_df: pd.DataFrame, cfg: Dict) -> Dict[str, str]:
    """Detecta colunas do Arq3 via config ou auto."""
    cols = list(arq3_df.columns) if arq3_df is not None else []
    NONE = "(nenhuma)"

    def _pick(cfg_key, *subs):
        manual = cfg.get(cfg_key, "")
        if manual and manual != NONE and manual in cols:
            return manual
        return _find_col(cols, *subs)

    return {
        "rede":     _pick("arq3_rede_col",        "REDE", "Rede"),
        "central":  _pick("arq3_central_col",      "Central", "CENTRAL"),
        "uf":       _pick("arq3_uf_col",           "UF", "ESTADO", "Estado"),
        "cluster":  _pick("arq3_cluster_col",      "CLUSTER", "Cluster",
                          "CLÚSTER", "AGRUPAMENTO"),
        "tipo":     _pick("arq3_tipo_rota_col",    "Tipo de Rota",
                          "TIPO DE ROTA", "TIPO_ROTA"),
        "rotulos":  _pick("arq3_rotulos_col",      "Rótulos de Linha",
                          "RÓTULOS DE LINHA", "Rotulos de Linha",
                          "ROTULOS DE LINHA", "ROTULO", "LABEL_E"),
        "operadora":_pick("arq3_operadora_col",    "OPERADORA", "Operadora"),
        "den":      _pick("arq3_denominacao_col",  "Denominação",
                          "DENOMINAÇÃO", "Denominacao", "DENOMINACAO",
                          "Descrição", "DESCRICAO"),
    }


def _rec_to_output(arq3_rec: Dict, arq3_cols: Dict, source_rec: Dict = None) -> Dict:
    """Extrai campos de saída de um registro do Arq3."""
    def _g(key):
        col = arq3_cols.get(key, "")
        return _s(arq3_rec.get(col, "")) if col else ""
    return {
        "REDE":              _g("rede"),
        "UF":                _g("uf"),
        "CLUSTER":           _g("cluster"),
        "Tipo de Rota":      _g("tipo"),
        "Central":           _g("central"),
        "Rótulos de Linha":  _g("rotulos"),
        "OPERADORA":         _g("operadora"),
        "Denominação":       _g("den"),
    }


# ── PARTE 1 — Science ─────────────────────────────────────────────────────────

def process_science(
    sci_df: pd.DataFrame,
    arq3_df: Optional[pd.DataFrame],
    config: Optional[Dict] = None,
) -> Tuple[pd.DataFrame, Dict]:
    """
    Executa etapas 1-4 na base Science.
    Retorna (resultado_df, relatorio_dict).
    """
    cfg = config or {}
    today = date.today()
    inconsistencias: List[Dict] = []

    df = sci_df.copy()
    total_inicial = len(df)
    log.info("[Science] Início: %d linhas", total_inicial)

    # ── Detectar colunas relevantes ──────────────────────────────────────
    sci_cols = list(df.columns)
    col_central   = _find_col(sci_cols,
                               cfg.get("central_sci_col", ""),
                               "Central", "CENTRAL",
                               "Central Interna", "Central Origem")
    col_data_desv = _find_col(sci_cols,
                               cfg.get("data_desativ_sci_col", ""),
                               "data desativação", "Data Desativação",
                               "DATA DESATIVACAO", "DT_DESATIVACAO",
                               "data desativacao", "DATA_DESATIVACAO")
    col_id_rota   = _find_col(sci_cols,
                               cfg.get("id_rota_sci_col", ""),
                               "ID Rota", "ID_ROTA", "IDROTA",
                               "ID ROTA", "Id Rota")

    log.info("[Science] Colunas: central=%r data_desativ=%r id_rota=%r",
             col_central, col_data_desv, col_id_rota)

    if not col_central:
        log.error("[Science] Coluna 'Central' não encontrada. Colunas: %s", sci_cols)

    # ── Etapa 1: Filtro por Central (começa com M) ───────────────────────
    if col_central and col_central in df.columns:
        mask_m = df[col_central].astype(str).str.strip().str.upper().str.startswith("M")
        descartados = df[~mask_m]
        for _, row in descartados.iterrows():
            val = _s(row.get(col_central, ""))
            inconsistencias.append({
                "etapa": "1 - Filtro Central",
                "chave": val or "(vazio)",
                "motivo": f"Central '{val}' não começa com 'M'" if val
                          else "Central vazia",
            })
        df = df[mask_m].copy()
    else:
        for i in range(len(df)):
            inconsistencias.append({
                "etapa": "1 - Filtro Central",
                "chave": str(i),
                "motivo": "Coluna 'Central' não encontrada na planilha Science",
            })
        df = df.iloc[0:0].copy()  # zera se não tem a coluna

    total_apos_central = len(df)
    log.info("[Science] Após etapa 1 (Central M): %d → %d",
             total_inicial, total_apos_central)

    # ── Etapa 2: Filtro data desativação ────────────────────────────────
    if col_data_desv and col_data_desv in df.columns:
        keep_mask = []
        for _, row in df.iterrows():
            raw = _s(row.get(col_data_desv, ""))
            central_val = _s(row.get(col_central, "")) if col_central else ""

            if not raw:
                # vazio → manter (rota ativa)
                keep_mask.append(True)
                continue

            # 1899 / 1900 → manter
            if raw[:4] in _DATE_1899_1900:
                keep_mask.append(True)
                continue

            # Tenta parsear
            dt = _parse_date(raw)
            if dt is None:
                # Formato inválido → reportar mas manter
                inconsistencias.append({
                    "etapa": "2 - Data Desativação",
                    "chave": central_val or str(row.name),
                    "motivo": f"Formato de data não reconhecido: '{raw}' (mantido)",
                })
                keep_mask.append(True)
                continue

            if dt < today:
                # Data passada → remover
                inconsistencias.append({
                    "etapa": "2 - Data Desativação",
                    "chave": central_val or str(row.name),
                    "motivo": f"Rota desativada em {raw} (anterior a {today})",
                })
                keep_mask.append(False)
            else:
                keep_mask.append(True)

        df = df[keep_mask].copy()
    else:
        log.info("[Science] Coluna de data desativação não encontrada — etapa 2 ignorada")

    total_apos_data = len(df)
    log.info("[Science] Após etapa 2 (data desativação): %d", total_apos_data)

    # ── Etapa 3: Validação de rotas externas via Arq3 ────────────────────
    arq3_cols: Dict[str, str] = {}
    rotulo_index: Dict[str, Dict] = {}

    if arq3_df is not None and not arq3_df.empty:
        arq3_cols = _extract_arq3_cols(arq3_df, cfg)
        rot_col = arq3_cols.get("rotulos", "")
        rotulo_index = _build_arq3_rotulo_index(arq3_df, rot_col)
        log.info("[Science] Arq3 carregado: %d rótulos únicos (col=%r)",
                 len(rotulo_index), rot_col)
    else:
        log.warning("[Science] Arq3 não disponível — etapas 3 e 4 ignoradas")

    if not col_id_rota:
        log.warning("[Science] Coluna 'ID Rota' não encontrada — etapa 3 ignorada")

    matched_rows = []
    if rotulo_index and col_central and col_id_rota:
        for _, row in df.iterrows():
            central_val = _s(row.get(col_central, "")).strip().upper()
            id_rota_val = _s(row.get(col_id_rota,  "")).strip().upper()

            if not central_val or not id_rota_val:
                inconsistencias.append({
                    "etapa": "3 - Validação Rota Externa",
                    "chave": f"{central_val}_{id_rota_val}",
                    "motivo": "Central ou ID_Rota vazio — não foi possível construir chave",
                })
                continue

            chave = central_val[:7] + "_" + id_rota_val
            if chave in rotulo_index:
                matched_rows.append((row, chave, rotulo_index[chave]))
            else:
                inconsistencias.append({
                    "etapa": "3 - Validação Rota Externa",
                    "chave": chave,
                    "motivo": f"Chave '{chave}' não encontrada em Rótulos de Linha do Arq3",
                })

        df_matched = pd.DataFrame([r for r, _, _ in matched_rows],
                                  columns=df.columns) if matched_rows else df.iloc[0:0].copy()
    else:
        # Sem Arq3 ou sem ID Rota → passa todas as linhas (sem enriquecimento)
        df_matched = df.copy()
        matched_rows = [(row, "", {}) for _, row in df.iterrows()]

    total_apos_validacao = len(df_matched)
    log.info("[Science] Após etapa 3 (validação Arq3): %d", total_apos_validacao)

    # ── Etapa 4: Enriquecimento ──────────────────────────────────────────
    output_records = []
    total_enriquecido = 0

    if rotulo_index and col_central and col_id_rota:
        for row, chave, arq3_rec in matched_rows:
            out = _rec_to_output(arq3_rec, arq3_cols, row.to_dict())
            out["_source_tag"]   = "SCIENCE"
            out["_chave"]        = chave
            out["_arq3_match"]   = "True"
            if out.get("REDE") or out.get("UF") or out.get("CLUSTER"):
                total_enriquecido += 1
            output_records.append(out)
    else:
        # Sem enriquecimento: preenche o que puder
        for _, row in df_matched.iterrows():
            out = {col: "" for col in OUTPUT_COLUMNS}
            if col_central:
                out["Central"] = _s(row.get(col_central, "")).upper()
            out["REDE"]        = "VIVO-SMP"
            out["_source_tag"] = "SCIENCE"
            out["_chave"]      = ""
            out["_arq3_match"] = "False"
            output_records.append(out)

    result_df = pd.DataFrame(output_records)
    for col in OUTPUT_COLUMNS:
        if col not in result_df.columns:
            result_df[col] = ""

    report = {
        "total_inicial_science":       total_inicial,
        "total_apos_filtro_central":   total_apos_central,
        "total_apos_filtro_data":      total_apos_data,
        "total_apos_validacao_rotas":  total_apos_validacao,
        "total_enriquecido_arq3":      total_enriquecido,
    }

    log.info("[Science] Finalizado: %d linhas de saída, %d enriquecidas, "
             "%d inconsistências",
             len(result_df), total_enriquecido,
             sum(1 for x in inconsistencias if x["etapa"].startswith("3")))

    return result_df, report, inconsistencias


# ── PARTE 2 — Portal ──────────────────────────────────────────────────────────

def process_portal(
    por_df: pd.DataFrame,
    arq3_df: Optional[pd.DataFrame],
    config: Optional[Dict] = None,
) -> Tuple[pd.DataFrame, Dict, List]:
    """
    Executa etapas 5-9 na base Portal de Cadastros.
    Retorna (resultado_df, relatorio_dict, inconsistencias).
    """
    cfg = config or {}
    inconsistencias: List[Dict] = []

    df = por_df.copy()
    total_inicial = len(df)
    log.info("[Portal] Início: %d linhas", total_inicial)

    # ── Detectar colunas ─────────────────────────────────────────────────
    por_cols  = list(df.columns)
    col_cen   = _find_col(por_cols, cfg.get("central_portal_col", ""), "CENTRAL")
    col_le    = _find_col(por_cols, cfg.get("label_e_col",         ""), "LABEL_E")
    col_ls    = _find_col(por_cols, cfg.get("label_s_col",         ""), "LABEL_S")
    col_desig = _find_col(por_cols, cfg.get("designacao_col",      ""),
                          "DESIGNACAO", "DESIGNAÇÃO", "Designacao",
                          "TIPO_ROTA", "TIPO ROTA")
    col_emp   = _find_col(por_cols, "EMPRESA", "OPERADORA", "Operadora")
    col_tipo  = _find_col(por_cols, cfg.get("tipo_rota_portal_col", ""),
                          "TIPO_ROTA", "TIPO DE ROTA", "TIPO ROTA")

    log.info("[Portal] Colunas: central=%r label_e=%r label_s=%r designacao=%r",
             col_cen, col_le, col_ls, col_desig)

    # Carrega Arq3
    arq3_cols: Dict[str, str] = {}
    rotulo_index: Dict[str, Dict] = {}

    if arq3_df is not None and not arq3_df.empty:
        arq3_cols   = _extract_arq3_cols(arq3_df, cfg)
        rot_col     = arq3_cols.get("rotulos", "")
        rotulo_index = _build_arq3_rotulo_index(arq3_df, rot_col)
        log.info("[Portal] Arq3: %d rótulos (col=%r)", len(rotulo_index), rot_col)

    output_records = []
    novas_rotas = 0

    for row_idx, (_, row) in enumerate(df.iterrows()):
        row_id = str(row_idx + 1)
        row_d  = {c: _s(row.get(c, "")) for c in por_cols}

        # ── Etapa 5: Converter CENTRAL ────────────────────────────────
        central_raw = row_d.get(col_cen, "") if col_cen else ""
        central_upper = central_raw.strip().upper()
        central_conv  = CENTRAL_PORTAL_MAP.get(central_upper)

        if not central_conv:
            # Tenta case-insensitive parcial
            for k, v in CENTRAL_PORTAL_MAP.items():
                if _norm(central_upper) == _norm(k):
                    central_conv = v
                    break

        if not central_conv:
            inconsistencias.append({
                "etapa": "5 - Conversão CENTRAL",
                "chave": central_raw or f"linha {row_id}",
                "motivo": f"CENTRAL '{central_raw}' não está no mapa de conversão "
                          f"(valores aceitos: {list(CENTRAL_PORTAL_MAP.keys())})",
            })
            # Usa o valor original como fallback para não perder a linha
            central_conv = central_raw.strip().upper() or f"CENTRAL_{row_id}"

        # ── Etapa 6: Definir 6 chars do label ─────────────────────────
        le_val = row_d.get(col_le, "") if col_le else ""
        ls_val = row_d.get(col_ls, "") if col_ls else ""

        if len(le_val) > 2:
            label_used = le_val
            label_src  = "LABEL_E"
        elif len(ls_val) > 2:
            label_used = ls_val
            label_src  = "LABEL_S"
        else:
            inconsistencias.append({
                "etapa": "6 - Label insuficiente",
                "chave": central_conv or f"linha {row_id}",
                "motivo": f"LABEL_E='{le_val}' e LABEL_S='{ls_val}' "
                          "ambos com ≤ 2 caracteres",
            })
            label_used = (le_val or ls_val or f"L{row_id}").strip()
            label_src  = "FALLBACK"

        # ── Etapa 7: Construir Rótulo de Linha ────────────────────────
        rotulo = (central_conv[:7] + "_" + label_used[:6]).upper().strip()

        # ── Etapa 8: Buscar no Arq3 ───────────────────────────────────
        arq3_rec = rotulo_index.get(rotulo)

        if arq3_rec:
            out = _rec_to_output(arq3_rec, arq3_cols, row_d)
            out["_arq3_match"] = "True"
            out["_nova_rota"]  = "False"
        else:
            # ── Etapa 9: Criar nova rota ──────────────────────────────
            novas_rotas += 1

            # UF via CN extraído de DESIGNACAO
            desig_val = row_d.get(col_desig, "") if col_desig else ""
            cn = _extract_cn(desig_val)
            uf = CN_TO_UF.get(cn, "") if cn else ""

            if not uf:
                inconsistencias.append({
                    "etapa": "9 - Nova rota (UF via CN)",
                    "chave": rotulo,
                    "motivo": f"Rótulo '{rotulo}' não encontrado em Arq3. "
                              f"DESIGNACAO='{desig_val}' CN extraído='{cn or '?'}' "
                              f"UF não encontrada → inserida sem UF",
                })

            # Operadora e Denominação da própria linha do Portal
            operadora  = row_d.get(col_emp,   "") if col_emp  else ""
            tipo_rota  = row_d.get(col_tipo,  "") if col_tipo else ""
            denominacao = desig_val

            out = {
                "REDE":             "VIVO-STFC",
                "UF":               uf,
                "CLUSTER":          "",
                "Tipo de Rota":     tipo_rota.upper() or "EXTERNA",
                "Central":          central_conv[:7],
                "Rótulos de Linha": rotulo,
                "OPERADORA":        operadora.upper(),
                "Denominação":      denominacao,
            }
            out["_arq3_match"] = "False"
            out["_nova_rota"]  = "True"

        out["_source_tag"]  = "PORTAL"
        out["_chave"]       = rotulo
        out["_central_raw"] = central_raw
        out["_label_used"]  = label_used
        out["_label_src"]   = label_src

        output_records.append(out)

    result_df = pd.DataFrame(output_records)
    for col in OUTPUT_COLUMNS:
        if col not in result_df.columns:
            result_df[col] = ""

    total_processado = len(result_df)
    report = {
        "total_portal_processado":  total_processado,
        "total_novas_rotas_criadas": novas_rotas,
        "total_encontrado_arq3":    total_processado - novas_rotas,
    }

    log.info("[Portal] Finalizado: %d linhas, %d novas rotas, %d inconsistências",
             total_processado, novas_rotas, len(inconsistencias))

    return result_df, report, inconsistencias


# ── Pipeline completo ─────────────────────────────────────────────────────────

def run_pipeline(
    sci_df: pd.DataFrame,
    por_df: pd.DataFrame,
    arq3_df: Optional[pd.DataFrame],
    config: Optional[Dict] = None,
) -> Tuple[pd.DataFrame, Dict]:
    """
    Executa o pipeline completo Science (1-4) + Portal (5-9).
    Retorna (result_df, report_completo).
    """
    cfg = config or {}

    # Science
    sci_result, sci_report, sci_incons = process_science(sci_df, arq3_df, cfg)

    # Portal
    por_result, por_report, por_incons = process_portal(por_df, arq3_df, cfg)

    # Concatena
    all_frames = []
    if not sci_result.empty:
        all_frames.append(sci_result)
    if not por_result.empty:
        all_frames.append(por_result)

    if all_frames:
        result = pd.concat(all_frames, ignore_index=True)
    else:
        result = pd.DataFrame(columns=OUTPUT_COLUMNS)

    for col in OUTPUT_COLUMNS:
        if col not in result.columns:
            result[col] = ""

    # Todas as inconsistências
    all_incons = sci_incons + por_incons
    total_incons = len(all_incons)

    # Percentual de redução Science
    ti = sci_report["total_inicial_science"]
    tf = sci_report["total_apos_validacao_rotas"]
    reducao_pct = round((1 - tf / ti) * 100, 1) if ti > 0 else 0.0

    report = {
        # Science
        "total_inicial_science":       sci_report["total_inicial_science"],
        "total_apos_filtro_central":   sci_report["total_apos_filtro_central"],
        "total_apos_filtro_data":      sci_report["total_apos_filtro_data"],
        "total_apos_validacao_rotas":  sci_report["total_apos_validacao_rotas"],
        "total_enriquecido_arq3":      sci_report["total_enriquecido_arq3"],
        # Portal
        "total_portal_processado":     por_report["total_portal_processado"],
        "total_novas_rotas_criadas":   por_report["total_novas_rotas_criadas"],
        "total_encontrado_arq3_portal":por_report["total_encontrado_arq3"],
        # Geral
        "total_resultado_final":       len(result),
        "total_inconsistencias":       total_incons,
        "reducao_science_pct":         reducao_pct,
        # Breakdown
        "uf_missing":                  int(result["UF"].eq("").sum()),
        "cluster_missing":             int(result["CLUSTER"].eq("").sum()),
        "arq3_match_count":            int(result.get("_arq3_match",
                                           pd.Series()).eq("True").sum()),
        "source_breakdown":            result["_source_tag"].value_counts().to_dict()
                                       if "_source_tag" in result.columns else {},
        "rows_science":                len(sci_result),
        "rows_portal":                 len(por_result),
        "rows_merged":                 len(result),
        # Lista detalhada
        "_inconsistencias":            all_incons,
    }

    log.info(
        "[Pipeline] Concluído: Science=%d→%d Portal=%d→%d "
        "Total=%d Inconsistências=%d",
        sci_report["total_inicial_science"],
        sci_report["total_apos_validacao_rotas"],
        por_df.__len__(),
        por_report["total_portal_processado"],
        len(result),
        total_incons,
    )

    return result, report
