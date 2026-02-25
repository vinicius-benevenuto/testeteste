# =============================================================================
# app.py — VIVOHUB: Sistema de Gestão de Projetos Técnicos de Interligação
# =============================================================================
#
# Aplicação Flask para gerenciar formulários PTI (Projeto Técnico de
# Interligação) entre operadoras de telecomunicações e a VIVO.
#
# Arquitetura:
#   ┌─────────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────┐
#   │  Flask Web   │────▶│  SQLite DB   │     │  Excel   │     │  SBC XLSX│
#   │  (Rotas)     │     │  (Dados)     │     │ (Export) │     │ (Análise)│
#   └──────┬──────┘     └──────────────┘     └────┬─────┘     └────┬─────┘
#          │                                       │                │
#          │            ┌──────────────┐           │                │
#          └───────────▶│  Pillow      │◀──────────┘                │
#          │            │  (Imagens)   │                            │
#          │            └──────────────┘                            │
#          │                                                       │
#          │            ┌──────────────┐                            │
#          └───────────▶│ SBCAnalyzer  │◀───────────────────────────┘
#                       │ (Inteligência│
#                       │  de SBCs)    │
#                       └──────────────┘
#
# Perfis de Usuário:
#   - Atacado:     Cria e gerencia formulários (CRUD completo)
#   - Engenharia:  Visualiza todos + edita Seção 9 + sugestão SBC
#   - Admin:       Registra novos usuários (protegido por código)
#
# -*- coding: utf-8 -*-
# =============================================================================

import os
import glob
import json
import sqlite3
import click
import tempfile
import logging
import time
from pathlib import Path
from functools import wraps
from datetime import datetime, timedelta
from dataclasses import dataclass, field, asdict
from io import BytesIO
from typing import Any, Optional
from collections import defaultdict

from flask import (
    Flask, render_template, request, redirect, url_for,
    session, flash, g, abort, send_file, jsonify,
)
from werkzeug.security import generate_password_hash, check_password_hash

# =============================================================================
# IMPORTAÇÕES OPCIONAIS (graceful degradation)
# =============================================================================
try:
    from openpyxl import Workbook
    from openpyxl.styles import Font, Alignment, PatternFill, Border, Side
    from openpyxl.utils import get_column_letter
    from openpyxl.drawing.image import Image as XLImage
    OPENPYXL_AVAILABLE = True
except ImportError:
    Workbook = None
    XLImage = None
    OPENPYXL_AVAILABLE = False

try:
    from PIL import Image as PILImage, ImageDraw, ImageFont
    PIL_AVAILABLE = True
except ImportError:
    PIL_AVAILABLE = False

# =============================================================================
# LOGGING
# =============================================================================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger("vivohub")

# =============================================================================
# CONSTANTES DA APLICAÇÃO
# =============================================================================
MAX_TABLE_ROWS = 10
DEFAULT_ADMIN_CODE = "v28B112004"
DEFAULT_DIAGRAM_IMAGE = (
    r"C:\Users\40418843\Desktop\VIVOHUB\templates\20251007_140942_0000.png"
)

# Diretório dos XLSX de SBC (override via SBC_DATA_DIR env var)
DEFAULT_SBC_DATA_DIR = r"C:\Users\40418843\Desktop\SBC_Reports"

# Cache TTL para dados de SBC (5 minutos)
SBC_CACHE_TTL_SECONDS = 300

# Mapeamento STATUS do XLSX → nível de saúde interno
# O STATUS já vem classificado pela equipe de rede no XLSX
SBC_STATUS_TO_HEALTH = {
    "normal":   "disponivel",   # Verde — folga, ideal para uso
    "atenção":  "moderado",     # Amarelo — aceitável com ressalvas
    "atencao":  "moderado",     # (sem acento)
    "crítico":  "critico",      # Vermelho — lotado, não recomendado
    "critico":  "critico",      # (sem acento)
}

# Score base por STATUS (0-100)
SBC_STATUS_BASE_SCORE = {
    "disponivel": 70,
    "moderado":   45,
    "critico":    15,
}

# Whitelist de flags do escopo
ALLOWED_SCOPE_FLAGS = frozenset({
    "LC", "LD15 + CNG", "LDS/CSP + CNG", "Transporte", "VC1", "Concentração"
})

BOOLEAN_FIELDS = (
    "csp", "servicos_especiais", "cng",
    "sbc_ativo", "ip_reservado", "vivo_reserva",
    "operadora_ciente", "lcr_nacional", "white_list",
    "prefixos_liberados_abr", "premissas_ok",
)

TEXT_FIELDS = (
    "nome_operadora", "rn1", "atendimento", "redes", "qual", "tmr",
    "responsavel_operadora", "responsavel_vivo", "asn",
    "responsavel_infra", "aprovado_por",
    "status", "escopo_text",
    "responsavel_atacado", "responsavel_engenharia",
)

JSON_FIELDS = (
    "escopo_flags_json", "dados_vivo_json",
    "dados_operadora_json", "engenharia_params_json",
)

# =============================================================================
# SEED DE CNs (Códigos Nacionais de telefonia)
# =============================================================================
_CN_SEED_RAW = """
68 82 97 92 96 77 75 74 73 71 88 85 61 27 28 64 62 61 98 99
34 37 31 35 32 38 33 67 66 65 91 94 93 83 81 87 86 89
43 44 45 46 41 24 22 21 84 69 95 51 53 54 55 47 48 49 79
18 14 15 16 13 19 17 11 12 63
"""

CN_METADATA = {
    "11": ("São Paulo", "SP"), "12": ("São José dos Campos", "SP"),
    "13": ("Santos", "SP"), "14": ("Bauru", "SP"),
    "15": ("Sorocaba", "SP"), "16": ("Ribeirão Preto", "SP"),
    "17": ("São José do Rio Preto", "SP"), "18": ("Presidente Prudente", "SP"),
    "19": ("Campinas", "SP"),
    "21": ("Rio de Janeiro", "RJ"), "22": ("Campos dos Goytacazes", "RJ"),
    "24": ("Volta Redonda", "RJ"), "27": ("Vitória", "ES"),
    "28": ("Cachoeiro de Itapemirim", "ES"),
    "31": ("Belo Horizonte", "MG"), "32": ("Juiz de Fora", "MG"),
    "33": ("Governador Valadares", "MG"), "34": ("Uberlândia", "MG"),
    "35": ("Poços de Caldas", "MG"), "37": ("Divinópolis", "MG"),
    "38": ("Montes Claros", "MG"),
    "41": ("Curitiba", "PR"), "42": ("Ponta Grossa", "PR"),
    "43": ("Londrina", "PR"), "44": ("Maringá", "PR"),
    "45": ("Foz do Iguaçu", "PR"), "46": ("Francisco Beltrão", "PR"),
    "47": ("Joinville", "SC"), "48": ("Florianópolis", "SC"),
    "49": ("Chapecó", "SC"),
    "51": ("Porto Alegre", "RS"), "53": ("Pelotas", "RS"),
    "54": ("Caxias do Sul", "RS"), "55": ("Santa Maria", "RS"),
    "61": ("Brasília", "DF"), "62": ("Goiânia", "GO"),
    "63": ("Palmas", "TO"), "64": ("Rio Verde", "GO"),
    "65": ("Cuiabá", "MT"), "66": ("Rondonópolis", "MT"),
    "67": ("Campo Grande", "MS"),
    "71": ("Salvador", "BA"), "73": ("Ilhéus", "BA"),
    "74": ("Juazeiro", "BA"), "75": ("Feira de Santana", "BA"),
    "77": ("Vitória da Conquista", "BA"),
    "79": ("Aracaju", "SE"), "81": ("Recife", "PE"),
    "82": ("Maceió", "AL"), "83": ("João Pessoa", "PB"),
    "84": ("Natal", "RN"), "85": ("Fortaleza", "CE"),
    "86": ("Teresina", "PI"), "87": ("Petrolina", "PE"),
    "88": ("Juazeiro do Norte", "CE"), "89": ("Picos", "PI"),
    "91": ("Belém", "PA"), "92": ("Manaus", "AM"),
    "93": ("Santarém", "PA"), "94": ("Marabá", "PA"),
    "95": ("Boa Vista", "RR"), "96": ("Macapá", "AP"),
    "97": ("Coari", "AM"),
    "98": ("São Luís", "MA"), "99": ("Imperatriz", "MA"),
    "68": ("Rio Branco", "AC"), "69": ("Porto Velho", "RO"),
}

# =============================================================================
# MAPEAMENTO UF → REGIONAL E VIZINHOS (para fallback inteligente de SBC)
# =============================================================================
UF_TO_REGIONAL = {
    "AC": "NORTE", "AM": "NORTE", "AP": "NORTE", "PA": "NORTE",
    "RO": "NORTE", "RR": "NORTE", "TO": "NORTE",
    "AL": "NORDESTE", "BA": "NORDESTE", "CE": "NORDESTE",
    "MA": "NORDESTE", "PB": "NORDESTE", "PE": "NORDESTE",
    "PI": "NORDESTE", "RN": "NORDESTE", "SE": "NORDESTE",
    "DF": "CENTRO-OESTE", "GO": "CENTRO-OESTE",
    "MS": "CENTRO-OESTE", "MT": "CENTRO-OESTE",
    "ES": "SUDESTE", "MG": "SUDESTE", "RJ": "SUDESTE", "SP": "SUDESTE",
    "PR": "SUL", "RS": "SUL", "SC": "SUL",
}

UF_NEIGHBORS = {
    "AC": ["RO", "AM"],
    "AL": ["PE", "SE", "BA"],
    "AM": ["PA", "RR", "AC", "RO", "MT"],
    "AP": ["PA"],
    "BA": ["SE", "AL", "PE", "PI", "MG", "GO", "TO", "MA"],
    "CE": ["RN", "PB", "PE", "PI"],
    "DF": ["GO", "MG"],
    "ES": ["MG", "RJ", "BA"],
    "GO": ["DF", "MG", "MS", "MT", "TO", "BA"],
    "MA": ["PI", "TO", "PA"],
    "MG": ["SP", "RJ", "ES", "BA", "GO", "DF", "MS"],
    "MS": ["PR", "SP", "MG", "GO", "MT"],
    "MT": ["MS", "GO", "TO", "PA", "AM", "RO"],
    "PA": ["MA", "TO", "MT", "AM", "AP", "RR"],
    "PB": ["PE", "RN", "CE"],
    "PE": ["PB", "AL", "BA", "CE", "PI"],
    "PI": ["MA", "CE", "PE", "BA", "TO"],
    "PR": ["SP", "SC", "MS"],
    "RJ": ["SP", "MG", "ES"],
    "RN": ["PB", "CE"],
    "RO": ["MT", "AM", "AC"],
    "RR": ["AM", "PA"],
    "RS": ["SC"],
    "SC": ["PR", "RS"],
    "SE": ["AL", "BA"],
    "SP": ["RJ", "MG", "PR", "MS"],
    "TO": ["MA", "PI", "BA", "GO", "MT", "PA"],
}

# =============================================================================
# SBC — MODELOS DE DADOS (Dataclasses)
# =============================================================================
@dataclass
class SBCMeasurement:
    """Uma única medição de um SBC (uma linha do XLSX)."""
    semana: str        # Y2025_W22
    dia: str           # 26/05/2025
    cidade: str        # RECIFE
    uf: str            # PE
    regional: str      # NORDESTE
    sbc: str           # PERCE_NGNTRO_SBC01
    caps: int          # 277
    status: str        # Atenção / Normal / Crítico
    modelo: str        # ORACLE (coluna MOD/ FORNEC)
    servico: str       # ITX
    responsavel: str   # (pode ser vazio)
    prazo: str         # (pode ser vazio)


@dataclass
class SBCAnalysisResult:
    """Resultado da análise de um SBC individual."""
    nome: str
    cidade: str
    uf: str
    regional: str
    modelo: str
    servicos: list
    caps_avg: float        # média CAPS no período
    caps_max: int          # CAPS máximo
    caps_min: int          # CAPS mínimo
    caps_ultimo: int       # CAPS do dia mais recente
    caps_tendencia: str    # "estavel" | "subindo" | "descendo"
    total_medicoes: int
    status_fonte: str        # pior status no período
    saude: str             # disponivel | moderado | critico
    score: int             # 0-100
    recomendado: bool
    motivo: str
    responsavel: str
    prazo: str
    dia_mais_recente: str
    semana: str


@dataclass
class SBCSuggestionResponse:
    """Resposta completa da API de sugestão de SBC."""
    cn: str
    uf: str
    cidade: str
    regional: str
    source_file: str
    source_modificado_em: str
    data_medicao: str
    total_sbcs: int
    sbcs: list
    fallback_usado: bool
    fallback_origem: str
    mensagem: str
    todos_criticos: bool = False
    total_disponiveis: int = 0
    total_criticos: int = 0


# =============================================================================
# SBC — PROCESSADOR DE DADOS (XLSX)
# =============================================================================
class SBCDataProcessor:
    """
    Lê, limpa e cacheia dados de SBC a partir de planilhas XLSX.
    Estrutura esperada do XLSX:
      SEMANA | DIA | CIDADE | UF | REGIONAL | SBC | CAPS | STATUS |
      MOD/ FORNEC | SERVIÇO | RESPONSÁVEL | PRAZO
    """

    COLUMN_MAP = {
        # Período
        "SEMANA": "semana", "ANO SEMANA": "semana",
        "DIA": "dia",
        # Localização
        "CIDADE": "cidade",
        "UF": "uf",
        "REGIONAL": "regional",
        # SBC
        "SBC": "sbc",
        "CAPS": "caps",
        # Status
        "ST": "status", "STATUS": "status",
        # Equipamento
        "MOD/ FORNEC": "modelo", "MOD/FORNEC": "modelo",
        "MODELO": "modelo", "FORNECEDOR": "modelo",
        "MOD / FORNEC": "modelo", "FABRICANTE": "modelo",
        # Serviço
        "SERVIÇO": "servico", "SERVICO": "servico",
        # Responsável / Prazo
        "RESPONSÁVEL": "responsavel", "RESPONSAVEL": "responsavel",
        "PRAZO": "prazo",
    }

    def __init__(self, data_dir: str, cache_ttl: int = SBC_CACHE_TTL_SECONDS):
        self.data_dir = data_dir
        self.cache_ttl = cache_ttl
        self._cache_data: list[SBCMeasurement] = []
        self._cache_aggregated: list[dict] = []
        self._cache_timestamp: float = 0.0
        self._cache_file_path: str = ""
        self._cache_file_mtime: float = 0.0

    def find_latest_file(self) -> Optional[str]:
        if not os.path.isdir(self.data_dir):
            logger.warning(f"Diretório de SBC não encontrado: {self.data_dir}")
            return None
        data_files = []
        for pattern in ("*.xlsx", "*.XLSX", "*.xls", "*.XLS"):
            data_files.extend(glob.glob(os.path.join(self.data_dir, pattern)))
        if not data_files:
            logger.warning(f"Nenhum arquivo XLSX encontrado em: {self.data_dir}")
            return None
        data_files.sort(key=lambda f: os.path.getmtime(f), reverse=True)
        latest = data_files[0]
        logger.info(f"Arquivo SBC mais recente: {os.path.basename(latest)} "
                     f"({len(data_files)} arquivos no diretório)")
        return latest

    def _normalize_column_name(self, raw_name: str) -> str:
        clean = raw_name.strip().upper()
        return self.COLUMN_MAP.get(clean, clean.lower().replace(" ", "_").replace("/", "_"))

    def _parse_int(self, value, default: int = 0) -> int:
        try:
            cleaned = str(value).strip().replace(",", ".").replace(" ", "")
            return int(float(cleaned)) if cleaned else default
        except (ValueError, TypeError):
            return default

    def _read_file(self, filepath: str) -> tuple[list[SBCMeasurement], list[dict]]:
        ext = os.path.splitext(filepath)[1].lower()
        if ext in (".xlsx", ".xls"):
            return self._read_xlsx(filepath)
        logger.warning(f"Formato não suportado: {ext}. Use XLSX.")
        return [], []

    def _read_xlsx(self, filepath: str) -> tuple[list[SBCMeasurement], list[dict]]:
        measurements = []
        aggregated = []
        if not OPENPYXL_AVAILABLE:
            logger.error("openpyxl não disponível — não é possível ler XLSX.")
            return [], []
        try:
            from openpyxl import load_workbook
            wb = load_workbook(filepath, read_only=True, data_only=True)
            ws = wb.active
            rows_iter = ws.iter_rows(values_only=True)
            try:
                header_raw = next(rows_iter)
            except StopIteration:
                logger.error(f"XLSX vazio: {filepath}")
                wb.close()
                return [], []
            headers = []
            for cell_val in header_raw:
                raw = str(cell_val).strip() if cell_val is not None else ""
                headers.append(self._normalize_column_name(raw))
            logger.info(f"Colunas detectadas: {headers}")

            # Validação: verificar se todas as 12 colunas obrigatórias estão presentes
            required = {"semana", "dia", "cidade", "uf", "regional", "sbc",
                        "caps", "status", "modelo", "servico", "responsavel", "prazo"}
            found = set(headers)
            missing = required - found
            if missing:
                logger.warning(f"Colunas ausentes no XLSX: {sorted(missing)}")

            for row_num, row_values in enumerate(rows_iter, start=2):
                normalized_row = {}
                for col_idx, cell_val in enumerate(row_values):
                    if col_idx < len(headers):
                        val = str(cell_val).strip() if cell_val is not None else ""
                        normalized_row[headers[col_idx]] = val
                sbc_name = normalized_row.get("sbc", "").strip()
                uf = normalized_row.get("uf", "").strip().upper()
                if sbc_name and uf:
                    measurement = SBCMeasurement(
                        semana=normalized_row.get("semana", ""),
                        dia=normalized_row.get("dia", ""),
                        cidade=normalized_row.get("cidade", "").strip().title(),
                        uf=uf, sbc=sbc_name,
                        regional=normalized_row.get("regional", "").strip().upper(),
                        caps=self._parse_int(normalized_row.get("caps", "0")),
                        status=normalized_row.get("status", "Normal").strip(),
                        modelo=normalized_row.get("modelo", "").strip(),
                        servico=normalized_row.get("servico", "").strip(),
                        responsavel=normalized_row.get("responsavel", "").strip(),
                        prazo=normalized_row.get("prazo", "").strip(),
                    )
                    measurements.append(measurement)
                else:
                    aggregated.append({
                        "row_num": row_num, "raw_data": normalized_row,
                        "nota": "Linha sem SBC/UF identificado (dado agregado ou total)",
                    })
            wb.close()
            logger.info(f"XLSX lido: {len(measurements)} medições, {len(aggregated)} agregadas")
        except FileNotFoundError:
            logger.error(f"XLSX não encontrado: {filepath}")
        except Exception as e:
            logger.error(f"Erro ao ler XLSX {filepath}: {e}")
        return measurements, aggregated

    def get_data(self, force_reload: bool = False) -> tuple[list[SBCMeasurement], list[dict]]:
        now = time.time()
        cache_expired = (now - self._cache_timestamp) > self.cache_ttl
        if force_reload or not self._cache_data or cache_expired:
            latest_file = self.find_latest_file()
            if not latest_file:
                return [], []
            current_mtime = os.path.getmtime(latest_file)
            if (latest_file == self._cache_file_path
                    and current_mtime == self._cache_file_mtime
                    and self._cache_data and not force_reload):
                self._cache_timestamp = now
                return self._cache_data, self._cache_aggregated
            logger.info(f"Carregando do disco: {os.path.basename(latest_file)}")
            measurements, aggregated = self._read_file(latest_file)
            self._cache_data = measurements
            self._cache_aggregated = aggregated
            self._cache_timestamp = now
            self._cache_file_path = latest_file
            self._cache_file_mtime = current_mtime
        return self._cache_data, self._cache_aggregated

    def get_file_info(self) -> dict:
        if self._cache_file_path and os.path.exists(self._cache_file_path):
            mtime = datetime.fromtimestamp(self._cache_file_mtime)
            return {
                "filename": os.path.basename(self._cache_file_path),
                "modified_at": mtime.strftime("%d/%m/%Y %H:%M"),
                "total_measurements": len(self._cache_data),
                "total_aggregated": len(self._cache_aggregated),
                "cache_age_seconds": int(time.time() - self._cache_timestamp),
            }
        return {"filename": None, "modified_at": None, "total_measurements": 0}


# =============================================================================
# SBC — ANALISADOR INTELIGENTE
# =============================================================================
class SBCAnalyzer:
    """
    Motor de análise e recomendação de SBCs.

    Lógica de saúde: STATUS do XLSX é a fonte primária (já classificado pela
    equipe de rede). CAPS é métrica secundária para tendência e desempate.

    Score (0-100):
      base = 70 (Normal) | 45 (Atenção) | 15 (Crítico)
      + bônus estabilidade CAPS (variação < 10% → +10)
      + bônus medições (>= 5 dias → +5)
      + bônus múltiplos serviços (+3)
      - penalidade tendência CAPS subindo (-10)
    """

    def __init__(self, processor: SBCDataProcessor):
        self.processor = processor

    def _classify_health(self, status: str) -> str:
        """Converte STATUS do XLSX para nível de saúde interno.

        Usa mapeamento direto primeiro, depois heurística por substring.
        """
        key = status.strip().lower()
        result = SBC_STATUS_TO_HEALTH.get(key)
        if result:
            return result
        # Heurística por substring para variações inesperadas
        if "crít" in key or "crit" in key:
            return "critico"
        if "aten" in key:
            return "moderado"
        if "norm" in key or "ok" in key or "disp" in key:
            return "disponivel"
        return "moderado"  # default seguro

    def _worst_status(self, statuses: list[str]) -> str:
        """Retorna o pior STATUS entre as medições (case-insensitive).

        Hierarquia: Normal (0) < Atenção (1) < Crítico (2).
        Normaliza variações de casing/acento do XLSX.
        """
        severity_map = {
            "normal": 0,
            "atenção": 1, "atencao": 1, "atencion": 1,
            "crítico": 2, "critico": 2,
        }
        # Mapeia de volta para label canônico usado pelo _classify_health
        canonical = {0: "Normal", 1: "Atenção", 2: "Crítico"}
        worst_score = -1
        for s in statuses:
            key = s.strip().lower()
            s_score = severity_map.get(key, -1)
            if s_score < 0:
                # Heurística: se contém "crít" ou "crit" → crítico;
                #             se contém "aten" → atenção; senão → normal
                if "crít" in key or "crit" in key:
                    s_score = 2
                elif "aten" in key:
                    s_score = 1
                else:
                    s_score = 0
            if s_score > worst_score:
                worst_score = s_score
        if worst_score < 0:
            worst_score = 0
        return canonical.get(worst_score, "Normal")

    def _calc_tendencia(self, caps_values: list[int]) -> str:
        """Analisa tendência de CAPS: estavel, subindo ou descendo."""
        if len(caps_values) < 2:
            return "estavel"
        first_half = caps_values[:len(caps_values)//2]
        second_half = caps_values[len(caps_values)//2:]
        avg_first = sum(first_half) / len(first_half) if first_half else 0
        avg_second = sum(second_half) / len(second_half) if second_half else 0
        if avg_first == 0:
            return "estavel"
        variation = (avg_second - avg_first) / avg_first
        if variation > 0.10:
            return "subindo"
        elif variation < -0.10:
            return "descendo"
        return "estavel"

    def _calculate_score(self, saude: str, tendencia: str,
                         caps_values: list[int], num_servicos: int,
                         total_medicoes: int) -> int:
        base = SBC_STATUS_BASE_SCORE.get(saude, 45)
        bonus = 0
        # Estabilidade: variação < 10% entre min e max
        if caps_values:
            mn, mx = min(caps_values), max(caps_values)
            avg = sum(caps_values) / len(caps_values)
            if avg > 0 and (mx - mn) / avg < 0.10:
                bonus += 10
        # Mais medições = mais confiança
        if total_medicoes >= 5:
            bonus += 5
        # Múltiplos serviços = mais versatilidade
        if num_servicos > 1:
            bonus += 3
        # Tendência de CAPS subindo = penalidade
        penalty = 0
        if tendencia == "subindo":
            penalty += 10
        return max(0, min(100, int(base + bonus - penalty)))

    def _generate_reason(self, result: SBCAnalysisResult) -> str:
        parts = []
        labels = {
            "disponivel": "capacidade disponível",
            "moderado": "ocupação moderada, monitorar",
            "critico": "capacidade crítica, evitar",
        }
        parts.append(f"Status XLSX: {result.status_fonte} — {labels.get(result.saude, result.saude)}")
        parts.append(f"CAPS avg {result.caps_avg} (mín {result.caps_min}, máx {result.caps_max}, último {result.caps_ultimo})")
        if result.caps_tendencia and result.caps_tendencia != "estavel":
            icon = "📈" if result.caps_tendencia == "subindo" else "📉"
            parts.append(f"Tendência: {icon} {result.caps_tendencia}")
        else:
            parts.append("Tendência: → estável")
        if result.total_medicoes > 1:
            parts.append(f"{result.total_medicoes} medições")
        if len(result.servicos) > 1:
            parts.append(f"Serviços: {', '.join(result.servicos)}")
        if result.cidade:
            parts.append(f"Local: {result.cidade}/{result.uf}")
        return " | ".join(parts)

    def _aggregate_sbc_measurements(self, measurements):
        groups: dict[str, list[SBCMeasurement]] = defaultdict(list)
        for m in measurements:
            groups[m.sbc].append(m)
        results = []
        for sbc_name, sbc_measurements in groups.items():
            caps_values = [m.caps for m in sbc_measurements]
            caps_avg = sum(caps_values) / len(caps_values) if caps_values else 0
            caps_max = max(caps_values) if caps_values else 0
            caps_min = min(caps_values) if caps_values else 0
            caps_ultimo = caps_values[-1] if caps_values else 0
            caps_tendencia = self._calc_tendencia(caps_values)
            all_statuses = [m.status for m in sbc_measurements]
            worst_status = self._worst_status(all_statuses)
            saude = self._classify_health(worst_status)
            servicos = list(dict.fromkeys(m.servico for m in sbc_measurements if m.servico))
            modelo = sbc_measurements[0].modelo
            cidade = sbc_measurements[0].cidade
            regional = sbc_measurements[0].regional
            uf = sbc_measurements[0].uf
            semana = sbc_measurements[0].semana
            responsavel = next((m.responsavel for m in sbc_measurements if m.responsavel), "")
            prazo = next((m.prazo for m in sbc_measurements if m.prazo), "")
            dias = [m.dia for m in sbc_measurements if m.dia]
            dia_recente = dias[-1] if dias else ""
            score = self._calculate_score(saude, caps_tendencia, caps_values,
                                           len(servicos), len(sbc_measurements))
            result = SBCAnalysisResult(
                nome=sbc_name, cidade=cidade, uf=uf, regional=regional,
                modelo=modelo, servicos=servicos,
                caps_avg=round(caps_avg, 1), caps_max=caps_max,
                caps_min=caps_min, caps_ultimo=caps_ultimo,
                caps_tendencia=caps_tendencia,
                total_medicoes=len(sbc_measurements),
                status_fonte=worst_status, saude=saude, score=score,
                recomendado=False, motivo="",
                responsavel=responsavel, prazo=prazo,
                dia_mais_recente=dia_recente, semana=semana,
            )
            results.append(result)
        results.sort(key=lambda r: r.score, reverse=True)
        for i, r in enumerate(results):
            r.recomendado = (i == 0)
            r.motivo = self._generate_reason(r)
        return results

    def _resolve_uf_from_cn(self, cn: str) -> Optional[tuple[str, str]]:
        cn = str(cn).strip().zfill(2)
        meta = CN_METADATA.get(cn)
        return meta if meta else None

    def suggest_for_cn(self, cn: str) -> SBCSuggestionResponse:
        """Sugere SBCs a partir de CN, resolvendo para UF e delegando."""
        cn = str(cn).strip().zfill(2)
        meta = self._resolve_uf_from_cn(cn)
        if not meta:
            return SBCSuggestionResponse(
                cn=cn, uf="", cidade="", regional="",
                source_file="", source_modificado_em="", data_medicao="",
                total_sbcs=0, sbcs=[], fallback_usado=False, fallback_origem="",
                mensagem=f"CN {cn} não encontrado no cadastro de códigos nacionais.",
            )
        cidade, uf = meta
        result = self.suggest_for_uf(uf)
        result.cn = cn
        result.cidade = cidade
        return result

    def suggest_for_uf(self, uf: str) -> SBCSuggestionResponse:
        """
        Busca TODOS os SBCs pela UF — sempre retorna todos, inclusive críticos.

        Ordenação: não-críticos primeiro (score desc), depois críticos (score desc).
        O frontend se encarrega de renderizar críticos com destaque visual em vermelho.
        """
        uf = uf.strip().upper()
        if len(uf) != 2:
            return SBCSuggestionResponse(
                cn="", uf=uf, cidade="", regional="",
                source_file="", source_modificado_em="", data_medicao="",
                total_sbcs=0, sbcs=[], fallback_usado=False, fallback_origem="",
                mensagem=f"UF inválida: '{uf}'. Use 2 letras (ex.: SP, RJ).",
            )

        regional = UF_TO_REGIONAL.get(uf, "")
        cidade = ""
        cn_found = ""
        for cn_code, (cn_cidade, cn_uf) in CN_METADATA.items():
            if cn_uf == uf:
                cidade = cn_cidade
                cn_found = cn_code
                break
        if not cidade:
            cidade = uf

        measurements, _ = self.processor.get_data()
        file_info = self.processor.get_file_info()
        if not measurements:
            return SBCSuggestionResponse(
                cn=cn_found, uf=uf, cidade=cidade, regional=regional,
                source_file=file_info.get("filename", ""),
                source_modificado_em=file_info.get("modified_at", ""),
                data_medicao="", total_sbcs=0, sbcs=[],
                fallback_usado=False, fallback_origem="",
                mensagem="Nenhum dado de SBC disponível. Verifique o diretório de XLSX.",
            )

        # --- Filtrar SBCs do UF ---
        uf_measurements = [m for m in measurements if m.uf == uf]
        fallback_usado = False
        fallback_origem = ""

        # --- Fallback se não há SBCs para o UF ---
        if not uf_measurements:
            neighbors = UF_NEIGHBORS.get(uf, [])
            for neighbor_uf in neighbors:
                uf_measurements = [m for m in measurements if m.uf == neighbor_uf]
                if uf_measurements:
                    fallback_usado = True
                    fallback_origem = f"UF vizinha: {neighbor_uf}"
                    break
        if not uf_measurements and regional:
            regional_ufs = [u for u, r in UF_TO_REGIONAL.items() if r == regional]
            uf_measurements = [m for m in measurements if m.uf in regional_ufs]
            if uf_measurements:
                fallback_usado = True
                fallback_origem = f"Regional: {regional}"
        if not uf_measurements:
            return SBCSuggestionResponse(
                cn=cn_found, uf=uf, cidade=cidade, regional=regional,
                source_file=file_info.get("filename", ""),
                source_modificado_em=file_info.get("modified_at", ""),
                data_medicao="", total_sbcs=0, sbcs=[],
                fallback_usado=False, fallback_origem="",
                mensagem=f"Nenhum SBC encontrado para {uf}, vizinhos ou regional {regional}.",
            )

        # --- Agregar e classificar TODOS os SBCs ---
        all_results = self._aggregate_sbc_measurements(uf_measurements)
        nao_criticos = [r for r in all_results if r.saude != "critico"]
        criticos = [r for r in all_results if r.saude == "critico"]

        # Ordenar: não-críticos primeiro (score desc), depois críticos (score desc)
        nao_criticos.sort(key=lambda r: r.score, reverse=True)
        criticos.sort(key=lambda r: r.score, reverse=True)

        # Marcar recomendado: melhor não-crítico, ou melhor crítico se todos são críticos
        ordered = nao_criticos + criticos
        for i, r in enumerate(ordered):
            r.recomendado = (i == 0)
            r.motivo = self._generate_reason(r)

        data_medicao = all_results[0].dia_mais_recente if all_results else ""
        n_disp = len(nao_criticos)
        n_crit = len(criticos)
        todos_crit = (n_disp == 0 and n_crit > 0)

        # Montar mensagem descritiva
        msg_parts = [f"{len(ordered)} SBC(s) encontrado(s) para {uf} ({cidade})"]
        if n_disp > 0:
            msg_parts.append(f"{n_disp} disponível(is)")
        if n_crit > 0:
            msg_parts.append(f"{n_crit} crítico(s)")
        if fallback_usado:
            msg_parts.append(f"Dados de {fallback_origem}")

        return SBCSuggestionResponse(
            cn=cn_found, uf=uf, cidade=cidade, regional=regional,
            source_file=file_info.get("filename", ""),
            source_modificado_em=file_info.get("modified_at", ""),
            data_medicao=data_medicao,
            total_sbcs=len(ordered),
            sbcs=[asdict(r) for r in ordered],
            fallback_usado=fallback_usado,
            fallback_origem=fallback_origem,
            todos_criticos=todos_crit,
            total_disponiveis=n_disp,
            total_criticos=n_crit,
            mensagem=" · ".join(msg_parts),
        )

    def get_overview(self) -> dict:
        measurements, aggregated = self.processor.get_data()
        file_info = self.processor.get_file_info()
        if not measurements:
            return {"file_info": file_info, "total_sbcs": 0, "por_regional": {},
                    "por_saude": {}, "por_status_fonte": {}, "dados_agregados": len(aggregated)}
        all_results = self._aggregate_sbc_measurements(measurements)
        por_regional = defaultdict(int)
        por_saude = defaultdict(int)
        por_status = defaultdict(int)
        for r in all_results:
            por_regional[r.regional] += 1
            por_saude[r.saude] += 1
            por_status[r.status_fonte] += 1
        return {
            "file_info": file_info, "total_sbcs": len(all_results),
            "por_regional": dict(por_regional), "por_saude": dict(por_saude),
            "por_status_fonte": dict(por_status), "dados_agregados": len(aggregated),
            "sbcs": [asdict(r) for r in all_results],
        }

    def health_check(self) -> dict:
        file_info = self.processor.get_file_info()
        file_path = self.processor.find_latest_file()
        return {
            "data_dir_exists": os.path.isdir(self.processor.data_dir),
            "data_dir": self.processor.data_dir,
            "latest_file": os.path.basename(file_path) if file_path else None,
            "file_info": file_info,
            "status": "ok" if file_path else "no_file_found",
        }


# =============================================================================
# FLASK APP FACTORY
# =============================================================================
def create_app() -> Flask:
    app = Flask(__name__, instance_relative_config=True)
    app.config["SECRET_KEY"] = os.environ.get("SECRET_KEY", "dev-secret-change-me")
    app.config["DATABASE"] = os.path.join(app.instance_path, "vivohub.db")
    app.config["ADMIN_CODE"] = os.environ.get("ADMIN_CODE", DEFAULT_ADMIN_CODE)
    app.config["SBC_DATA_DIR"] = os.environ.get("SBC_DATA_DIR", DEFAULT_SBC_DATA_DIR)
    Path(app.instance_path).mkdir(parents=True, exist_ok=True)
    export_dir = os.path.join(app.instance_path, "exports")
    Path(export_dir).mkdir(parents=True, exist_ok=True)
    app.config["EXPORT_DIR"] = export_dir
    sbc_processor = SBCDataProcessor(app.config["SBC_DATA_DIR"])
    sbc_analyzer = SBCAnalyzer(sbc_processor)
    app.config["SBC_ANALYZER"] = sbc_analyzer
    _register_db_hooks(app)
    _register_security_headers(app)
    _register_context_processors(app)
    _register_template_filters(app)
    _register_routes(app)
    _register_cli_commands(app)
    return app


# =============================================================================
# BANCO DE DADOS
# =============================================================================
def get_db() -> sqlite3.Connection:
    if "db" not in g:
        from flask import current_app
        g.db = sqlite3.connect(current_app.config["DATABASE"])
        g.db.row_factory = sqlite3.Row
        g.db.execute("PRAGMA foreign_keys = ON;")
    return g.db


def _register_db_hooks(app):
    @app.teardown_appcontext
    def close_db(exception=None):
        db = g.pop("db", None)
        if db is not None:
            db.close()

    @app.before_request
    def ensure_schema():
        _init_db()


def _init_db():
    db = get_db()
    db.executescript("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            role TEXT NOT NULL CHECK (role IN ('engenharia', 'atacado')),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        CREATE TABLE IF NOT EXISTS atacado_forms (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            owner_id INTEGER NOT NULL,
            status TEXT DEFAULT 'rascunho',
            nome_operadora TEXT, rn1 TEXT,
            csp INTEGER DEFAULT 0, servicos_especiais INTEGER DEFAULT 0,
            cng INTEGER DEFAULT 0, atendimento TEXT, redes TEXT,
            qual TEXT, tmr TEXT,
            responsavel_operadora TEXT, responsavel_vivo TEXT,
            sbc_ativo INTEGER DEFAULT 0, ip_reservado INTEGER DEFAULT 0,
            vivo_reserva INTEGER DEFAULT 0, asn TEXT,
            escopo_text TEXT, escopo_flags_json TEXT, dados_vivo_json TEXT,
            dados_operadora_json TEXT,
            operadora_ciente INTEGER DEFAULT 0, responsavel_infra TEXT,
            lcr_nacional INTEGER DEFAULT 0, white_list INTEGER DEFAULT 0,
            prefixos_liberados_abr INTEGER DEFAULT 0,
            premissas_ok INTEGER DEFAULT 0, aprovado_por TEXT,
            engenharia_params_json TEXT,
            responsavel_atacado TEXT, responsavel_engenharia TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(owner_id) REFERENCES users(id)
        );
        CREATE TABLE IF NOT EXISTS exports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            form_id INTEGER NOT NULL,
            filename TEXT NOT NULL, filepath TEXT NOT NULL,
            size_bytes INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(form_id) REFERENCES atacado_forms(id)
        );
    """)
    for table, col, coldef in [
        ("atacado_forms", "owner_id", "INTEGER NOT NULL DEFAULT 0"),
        ("atacado_forms", "engenharia_params_json", "TEXT"),
        ("atacado_forms", "dados_vivo_json", "TEXT"),
        ("atacado_forms", "dados_operadora_json", "TEXT"),
        ("atacado_forms", "escopo_text", "TEXT"),
        ("atacado_forms", "escopo_flags_json", "TEXT"),
        ("atacado_forms", "responsavel_atacado", "TEXT"),
        ("atacado_forms", "responsavel_engenharia", "TEXT"),
    ]:
        _ensure_column(db, table, col, coldef)
    db.commit()
    _seed_cns(db)
    _apply_cn_metadata(db)


def _ensure_column(db, table, column, coldef):
    existing = {r["name"] for r in db.execute(f"PRAGMA table_info({table});").fetchall()}
    if column not in existing:
        db.execute(f"ALTER TABLE {table} ADD COLUMN {column} {coldef};")


def _parse_cn_seed(raw):
    seen, result = set(), []
    for tok in raw.split():
        code = tok.strip().zfill(2)
        if code.isdigit() and code not in seen:
            seen.add(code)
            result.append(code)
    return result


def _seed_cns(db):
    db.execute("""
        CREATE TABLE IF NOT EXISTS cns (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            codigo TEXT UNIQUE NOT NULL, nome TEXT, uf TEXT,
            ativo INTEGER NOT NULL DEFAULT 1
        )
    """)
    if db.execute("SELECT COUNT(*) AS c FROM cns").fetchone()["c"] == 0:
        codes = _parse_cn_seed(_CN_SEED_RAW)
        db.executemany("INSERT OR IGNORE INTO cns (codigo, ativo) VALUES (?, 1)", [(c,) for c in codes])
        db.commit()


def _apply_cn_metadata(db):
    for code, (nome, uf) in CN_METADATA.items():
        db.execute("""
            INSERT INTO cns (codigo, nome, uf, ativo) VALUES (?, ?, ?, 1)
            ON CONFLICT(codigo) DO UPDATE SET nome=excluded.nome, uf=excluded.uf, ativo=1
        """, (code, nome, uf))
    db.commit()


# =============================================================================
# SEGURANÇA
# =============================================================================
def _register_security_headers(app):
    @app.after_request
    def add_headers(resp):
        resp.headers.setdefault("X-Content-Type-Options", "nosniff")
        resp.headers.setdefault("X-Frame-Options", "SAMEORIGIN")
        resp.headers.setdefault("X-XSS-Protection", "0")
        return resp


def login_required(view):
    @wraps(view)
    def wrapped(*a, **kw):
        if "user_id" not in session:
            return redirect(url_for("login"))
        return view(*a, **kw)
    return wrapped


def role_required(required_role):
    def decorator(view):
        @wraps(view)
        def wrapped(*a, **kw):
            if "user_id" not in session:
                return redirect(url_for("login"))
            if session.get("role") != required_role:
                flash("Acesso negado para esta área.", "danger")
                return redirect(url_for(f"central_{session.get('role')}"))
            return view(*a, **kw)
        return wrapped
    return decorator


def admin_required(view):
    @wraps(view)
    def wrapped(*a, **kw):
        if not session.get("is_admin"):
            flash("Área restrita ao administrador.", "danger")
            return redirect(url_for("admin_login"))
        return view(*a, **kw)
    return wrapped


# =============================================================================
# HELPERS
# =============================================================================
def row_get(row, key, default=""):
    try:
        v = row[key]
        return default if v is None else v
    except (KeyError, IndexError):
        return default


def safe_filename(name):
    name = (name or "").strip().replace(" ", "_")
    for ch in ("..", "/", "\\", ":", "*", "?", '"', "<", ">", "|"):
        name = name.replace(ch, "_")
    return name or "Operadora"


def parse_db_datetime(value):
    if not value:
        return None
    try:
        return datetime.fromisoformat(str(value).replace("T", " "))
    except (ValueError, TypeError):
        return None


def truncate_json_list(raw, default="[]"):
    if not raw:
        return default
    try:
        parsed = json.loads(raw) if isinstance(raw, str) else raw
        if not isinstance(parsed, list):
            return default
        return json.dumps(parsed[:MAX_TABLE_ROWS], ensure_ascii=False)
    except (json.JSONDecodeError, TypeError):
        return default


def parse_bool_field(value):
    return 1 if str(value).lower() in ("on", "1", "true", "sim") else 0


# =============================================================================
# PROCESSAMENTO DE FORMULÁRIO
# =============================================================================
def extract_scope_flags_from_request():
    """
    Extrai flags de escopo do request.

    FIX CRÍTICO: O código original lia request.form.getlist("escopo_flags"),
    mas os checkboxes no template NÃO têm name="escopo_flags" — apenas
    data-flag. O JS atualiza o hidden input "escopo_flags_json" com um JSON
    string. Esta função agora lê corretamente do hidden input como fallback.
    """
    # Tentativa 1: checkboxes com name="escopo_flags" (caso existam no futuro)
    raw = request.form.getlist("escopo_flags") or []

    # Tentativa 2 (PRINCIPAL): lê do hidden input "escopo_flags_json"
    # que é atualizado pelo JavaScript no frontend
    if not raw:
        json_raw = request.form.get("escopo_flags_json", "[]")
        try:
            parsed = json.loads(json_raw) if isinstance(json_raw, str) else []
            if isinstance(parsed, list):
                raw = [str(x).strip() for x in parsed if str(x).strip()]
            else:
                raw = []
        except (json.JSONDecodeError, TypeError):
            raw = []

    valid = [f for f in raw if f in ALLOWED_SCOPE_FLAGS]
    return json.dumps(list(dict.fromkeys(valid)), ensure_ascii=False)


def _parse_json_dict(raw):
    if not raw:
        return "{}"
    try:
        parsed = json.loads(raw) if isinstance(raw, str) else raw
        if isinstance(parsed, dict):
            return json.dumps(parsed, ensure_ascii=False)
    except (json.JSONDecodeError, TypeError):
        pass
    return "{}"


def extract_form_payload():
    payload = {}
    for field in TEXT_FIELDS:
        payload[field] = (request.form.get(field) or "").strip()
    for field in BOOLEAN_FIELDS:
        payload[field] = parse_bool_field(request.form.get(field))
    payload["escopo_flags_json"] = extract_scope_flags_from_request()
    for field in JSON_FIELDS:
        if field == "escopo_flags_json":
            continue
        raw = request.form.get(field, "")
        payload[field] = _parse_json_dict(raw) if field == "engenharia_params_json" else truncate_json_list(raw, "[]")
    return payload


def validate_table_rows(payload):
    truncated = False
    for key in ("dados_vivo_json", "dados_operadora_json"):
        try:
            rows = json.loads(payload[key])
            if isinstance(rows, list) and len(rows) > MAX_TABLE_ROWS:
                payload[key] = json.dumps(rows[:MAX_TABLE_ROWS], ensure_ascii=False)
                truncated = True
        except (json.JSONDecodeError, TypeError, KeyError):
            payload[key] = "[]"
    return truncated


# =============================================================================
# CONTEXT PROCESSORS E TEMPLATE FILTERS
# =============================================================================
def _register_context_processors(app):
    @app.context_processor
    def inject_cn_codes():
        db = get_db()
        rows = db.execute(
            "SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf "
            "FROM cns WHERE ativo = 1 ORDER BY codigo ASC"
        ).fetchall()
        return {
            "CN_CODES": [r["codigo"] for r in rows],
            "CN_FULL": [{"codigo": r["codigo"], "nome": r["nome"], "uf": r["uf"]} for r in rows],
        }


def _register_template_filters(app):
    @app.template_filter("date_br")
    def date_br_filter(value):
        if not value:
            return ""
        if hasattr(value, "strftime"):
            try:
                return value.strftime("%d/%m/%Y %H:%M")
            except (ValueError, OSError):
                return str(value)
        dt = parse_db_datetime(value)
        return dt.strftime("%d/%m/%Y %H:%M") if dt else str(value)


# =============================================================================
# QUERIES
# =============================================================================
def build_list_query(base_sql, *, search_term=None, status_filter=None,
                     owner_id=None, sort_key="-created_at"):
    conditions, params = [], []
    if owner_id is not None:
        conditions.append("owner_id = ?"); params.append(owner_id)
    if search_term:
        conditions.append("nome_operadora LIKE ?"); params.append(f"%{search_term}%")
    if status_filter:
        conditions.append("LOWER(COALESCE(status, '')) = LOWER(?)"); params.append(status_filter)
    if conditions:
        base_sql += " WHERE " + " AND ".join(conditions)
    sort_map = {
        "created_at": "created_at ASC", "-created_at": "created_at DESC",
        "nome_operadora": "nome_operadora COLLATE NOCASE ASC",
        "-nome_operadora": "nome_operadora COLLATE NOCASE DESC",
        "id": "id ASC", "-id": "id DESC",
    }
    base_sql += f" ORDER BY {sort_map.get(sort_key, 'id DESC')}"
    return base_sql, params


def get_status_counters(db, extra_where="", extra_params=()):
    connector = "AND" if "WHERE" in extra_where else "WHERE"
    def _count(cond=""):
        sql = f"SELECT COUNT(*) AS c FROM atacado_forms {extra_where}"
        if cond: sql += f" {connector} {cond}"
        return db.execute(sql, extra_params).fetchone()["c"]
    return {
        "total": _count(), "rascunho": _count("LOWER(status) = 'rascunho'"),
        "enviado": _count("LOWER(status) = 'enviado'"),
        "em_revisao": _count("LOWER(status) = 'em revisão'"),
        "aprovado": _count("LOWER(status) = 'aprovado'"),
    }


# =============================================================================
# PROCESSAMENTO DE IMAGENS (Pillow)
# =============================================================================
def parse_xy_percent(raw, default):
    if not raw: return default
    try:
        parts = [p.strip() for p in str(raw).split(",")]
        if len(parts) != 2: return default
        return (max(0.0, min(1.0, float(parts[0]))), max(0.0, min(1.0, float(parts[1]))))
    except (ValueError, TypeError): return default


def fit_size_keep_aspect(ow, oh, mw, mh):
    if ow <= 0 or oh <= 0: return mw, mh
    s = min(mw / ow, mh / oh)
    return int(ow * s), int(oh * s)


def _find_system_font(size_px, preferred=None):
    if not PIL_AVAILABLE: return None
    if preferred:
        try: return ImageFont.truetype(preferred, size_px)
        except (OSError, IOError): pass
    for p in ["C:\\Windows\\Fonts\\arial.ttf",
              "/System/Library/Fonts/Supplemental/Arial.ttf",
              "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf"]:
        try: return ImageFont.truetype(p, size_px)
        except (OSError, IOError): continue
    return ImageFont.load_default()


def render_labels_on_image(
    image_path, vivo_text, operator_text, *,
    vivo_xy_pct=(0.18, 0.55), operator_xy_pct=(0.80, 0.55),
    font_path=None, font_size_pct=0.06,
    fill=(255,255,255,255), stroke_fill=(0,0,0,255),
    stroke_width_pct=0.012, extra_labels=None,
) -> str:
    if not PIL_AVAILABLE:
        raise RuntimeError("Pillow não instalado.")
    img = PILImage.open(image_path).convert("RGBA")
    W, H = img.size
    draw = ImageDraw.Draw(img)
    def _draw(text, xy, sz, *, lf=fill, ls=stroke_fill, lw=stroke_width_pct, anch="mm"):
        if not text: return
        x, y = int(W * xy[0]), int(H * xy[1])
        font = _find_system_font(max(12, int(H * sz)), font_path)
        draw.text((x, y), text, font=font, fill=lf,
                  stroke_width=max(0, int(H * lw)), stroke_fill=ls, anchor=anch)
    _draw(vivo_text, vivo_xy_pct, font_size_pct)
    _draw(operator_text, operator_xy_pct, font_size_pct)
    if extra_labels:
        for lbl in extra_labels:
            _draw(lbl.get("text",""), lbl.get("xy_pct",(0.5,0.5)),
                  float(lbl.get("font_size_pct", font_size_pct)),
                  lf=lbl.get("fill",fill), ls=lbl.get("stroke_fill",stroke_fill),
                  lw=float(lbl.get("stroke_width_pct",stroke_width_pct)),
                  anch=lbl.get("anchor","mm"))
    fd, tmp = tempfile.mkstemp(prefix="vivohub_diagrama_", suffix=".png")
    os.close(fd)
    img.save(tmp, format="PNG")
    return tmp


# =============================================================================
# EXCEL — Estilos e Builder
# =============================================================================
class ExcelStylePalette:
    """Paleta centralizada de estilos para Excel."""
    def __init__(self):
        self.fill_primary = PatternFill("solid", fgColor="4A148C")
        self.fill_secondary = PatternFill("solid", fgColor="6A1B9A")
        self.fill_light = PatternFill("solid", fgColor="7B1FA2")
        self.fill_accent = PatternFill("solid", fgColor="8E24AA")
        self.fill_neutral = PatternFill("solid", fgColor="E1BEE7")
        self.fill_background = PatternFill("solid", fgColor="F3E5F5")
        self.fill_white = PatternFill("solid", fgColor="FFFFFF")
        self.font_title = Font(name="Calibri", size=14, bold=True, color="FFFFFF")
        self.font_subtitle = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
        self.font_brand = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
        self.font_header = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
        self.font_subheader = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
        self.font_body = Font(name="Calibri", size=10)
        self.font_small = Font(name="Calibri", size=9)
        self.font_micro = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        self.font_check = Font(name="Calibri", size=11, bold=True, color="008000")
        thin = Side(style="thin", color="DDDDDD")
        self.box_border = Border(left=thin, right=thin, top=thin, bottom=thin)
        self.no_border = Border()
        self.align_center = Alignment(horizontal="center", vertical="center")
        self.align_left = Alignment(horizontal="left", vertical="center")
        self.align_wrap = Alignment(horizontal="left", vertical="center", wrap_text=True)
        self.align_center_wrap = Alignment(horizontal="center", vertical="center", wrap_text=True)

    def alt_fill(self, idx):
        return self.fill_background if idx % 2 == 0 else self.fill_white


class PTIWorkbookBuilder:
    """Builder para Workbook Excel do PTI."""

    def __init__(self, form_row, sbc_analyzer=None):
        if not OPENPYXL_AVAILABLE: raise RuntimeError("openpyxl não instalado.")
        if not PIL_AVAILABLE: raise RuntimeError("Pillow não instalado.")
        self.form = form_row
        self.sbc_analyzer = sbc_analyzer  # SBCAnalyzer instance (optional)
        self.s = ExcelStylePalette()
        self.wb = Workbook()
        self.nome = (row_get(form_row, "nome_operadora") or "Operadora").strip()
        self.resp_atk = (row_get(form_row, "responsavel_atacado") or row_get(form_row, "responsavel_vivo") or "").strip()
        self.resp_eng = (row_get(form_row, "responsavel_engenharia","") or "").strip()
        self.asn_op = (row_get(form_row, "asn","") or "").strip()
        cdt = parse_db_datetime(row_get(form_row, "created_at",""))
        self.created_str = cdt.strftime("%d/%m/%Y") if cdt else ""

        # -----------------------------------------------------------------
        # FIX: Extrair TODOS os flags de escopo (sem truncar em 3)
        # -----------------------------------------------------------------
        self.traffic = self._extract_traffic()

        self.conc = self._extract_conc()

        # Texto livre do escopo
        self.escopo_text = (row_get(form_row, "escopo_text","") or "").strip()

        # -----------------------------------------------------------------
        # FIX: Escopo completo para aba Versões = texto + flags
        # -----------------------------------------------------------------
        if self.traffic:
            flags_str = " | ".join(self.traffic)
            self.escopo_completo = f"{self.escopo_text} [{flags_str}]" if self.escopo_text else flags_str
        else:
            self.escopo_completo = self.escopo_text

        # Dados das tabelas do formulário
        self.vivo_rows = self._extract_vivo_rows()
        self.op_rows = self._extract_op_rows()

        # CNs e Áreas Locais únicos (combinando vivo + operadora)
        cns_vivo = self._unique_field(self.vivo_rows, "cn")
        cns_op = self._unique_field(self.op_rows, "cn")
        self.cns_unicos = list(dict.fromkeys(cns_vivo + cns_op))  # merge preservando ordem

        areas_vivo = self._unique_field(self.vivo_rows, "cidade")
        areas_op = self._unique_field(self.op_rows, "cidade")
        self.areas_locais = list(dict.fromkeys(areas_vivo + areas_op))  # merge preservando ordem

    def _extract_vivo_rows(self):
        raw = row_get(self.form, "dados_vivo_json", "[]")
        try:
            d = json.loads(raw) if isinstance(raw, str) else raw
            if not isinstance(d, list): return []
            rows = []
            for item in d:
                if not isinstance(item, dict): continue
                row = {k: str(item.get(k,"")).strip() for k in (
                    "ref","data","escopo","localidade","cn","sbc","mask",
                    "endereco_link","cidade","uf","lat","long"
                )}
                if any(row.values()): rows.append(row)
            return rows[:MAX_TABLE_ROWS]
        except: return []

    def _extract_op_rows(self):
        raw = row_get(self.form, "dados_operadora_json", "[]")
        try:
            d = json.loads(raw) if isinstance(raw, str) else raw
            if not isinstance(d, list): return []
            rows = []
            for item in d:
                if not isinstance(item, dict): continue
                row = {k: str(item.get(k,"")).strip() for k in (
                    "ref","localidade","eto_lc","eot_ld","cn","sbc","faixa_ip",
                    "concentracao","endereco_link","cidade","uf","lat","long"
                )}
                if any(row.values()): rows.append(row)
            return rows[:MAX_TABLE_ROWS]
        except: return []

    def _unique_field(self, rows, field):
        seen = []
        for r in rows:
            v = r.get(field, "").strip()
            if v and v not in seen: seen.append(v)
        return seen

    def _extract_traffic(self):
        """
        Extrai flags de escopo (tráfego) do formulário.

        FIX: O código original fazia padding para 3 e truncava em [:3].
        Agora retorna TODOS os flags selecionados, sem limite.
        Esses flags populam a coluna "Tráfego" nas tabelas Ponta A e Ponta B.
        """
        raw = row_get(self.form, "escopo_flags_json", "[]")
        try:
            p = json.loads(raw) if isinstance(raw, str) else raw
            items = [str(x).strip() for x in (p if isinstance(p, list) else []) if str(x).strip()]
        except:
            items = []
        return items  # Retorna TODOS — sem padding nem truncamento

    def _extract_conc(self):
        """
        Extrai dados de concentração do JSON da operadora.
        Campos expandidos para aba Concentração enriquecida:
          ref, cn, cnl, cod_cni, status_link, localidade, cidade, uf,
          concentracao, sbc, faixa_ip, endereco_link, eto_lc, eot_ld
        """
        raw = row_get(self.form, "dados_operadora_json", "[]")
        try:
            d = json.loads(raw) if isinstance(raw, str) else raw
            if not isinstance(d, list): return []
            r = []
            for i in d:
                if not isinstance(i, dict): continue
                c = {k: str(i.get(k, "")).strip() for k in (
                    "ref", "cn", "cnl", "cod_cni", "status_link",
                    "localidade", "cidade", "uf", "concentracao",
                    "sbc", "faixa_ip", "endereco_link", "eto_lc", "eot_ld",
                )}
                if any(c.values()): r.append(c)
            return r[:MAX_TABLE_ROWS]
        except: return []

    def _diagram_cfg(self):
        f = self.form
        def _fl(k,d):
            try: return float(row_get(f,k,d))
            except: return d
        def _it(k,d):
            try: return int(float(row_get(f,k,d)))
            except: return d
        ns = str(row_get(f,"roteador_op_no_stroke","")).lower() in ("1","true","yes","sim")
        rs = 0.0 if ns else 0.009
        ov = row_get(f,"roteador_op_stroke_width_pct",None)
        if ov not in (None,""):
            try: rs = float(ov)
            except: pass
        return {
            "img": row_get(f,"diagram_image_path") or DEFAULT_DIAGRAM_IMAGE,
            "vxy": parse_xy_percent(row_get(f,"vivo_label_xy_pct",""),(0.16,0.48)),
            "oxy": parse_xy_percent(row_get(f,"operadora_label_xy_pct",""),(0.83,0.44)),
            "lvxy": parse_xy_percent(row_get(f,"link_vivo_label_xy_pct",""),(0.49,0.53)),
            "loxy": parse_xy_percent(row_get(f,"link_op_label_xy_pct",""),(0.49,0.35)),
            "fsz": _fl("diagram_font_size_pct",0.040),
            "lfsz": _fl("link_labels_font_size_pct",0.030),
            "fp": row_get(f,"diagram_font_path",None),
            "r1": parse_xy_percent(row_get(f,"roteador_op1_label_xy_pct",""),(0.64,0.58)),
            "r2": parse_xy_percent(row_get(f,"roteador_op2_label_xy_pct",""),(0.64,0.39)),
            "rfp": _fl("roteador_op_font_size_pct",0.010), "rsp": rs,
            "bw": _it("diagram_max_w",1900), "bh": _it("diagram_max_h",650),
            "aro": _it("diagram_anchor_row_offset",-3), "aco": _it("diagram_anchor_col_offset",1),
        }

    def _cw(self, ws, w):
        for c,v in w.items(): ws.column_dimensions[get_column_letter(c)].width = v
    def _bh(self, ws, r, sc, ec):
        ws.merge_cells(start_row=r,start_column=sc,end_row=r,end_column=ec)
        c=ws.cell(row=r,column=sc,value=f"VIVO — {self.nome}")
        c.font=self.s.font_title; c.alignment=self.s.align_center; c.fill=self.s.fill_primary
        ws.row_dimensions[r].height=24.0
    def _st(self, ws, r, sc, ec, t):
        ws.merge_cells(start_row=r,start_column=sc,end_row=r,end_column=ec)
        c=ws.cell(row=r,column=sc,value=t)
        c.font=self.s.font_subtitle; c.alignment=self.s.align_center; c.fill=self.s.fill_secondary
        ws.row_dimensions[r].height=25.0
    def _ps(self, ws, *, ls=False):
        ws.sheet_view.showGridLines=False; ws.page_setup.paperSize=ws.PAPERSIZE_A4
        ws.page_setup.orientation=ws.ORIENTATION_LANDSCAPE if ls else ws.ORIENTATION_PORTRAIT
        ws.print_options.horizontalCentered=True
        ws.page_margins.left=ws.page_margins.right=0.4; ws.page_margins.top=ws.page_margins.bottom=0.5
    def _ca(self, ws, r1, r2, c1, c2):
        for r in range(r1,r2+1):
            for c in range(c1,c2+1):
                ws.cell(row=r,column=c).fill=self.s.fill_white; ws.cell(row=r,column=c).border=self.s.no_border

    # =====================================================================
    # ABA: ÍNDICE
    # =====================================================================
    def _b_index(self):
        s=self.s; ws=self.wb.active; ws.title="Índice"
        for c in range(1,10): ws.column_dimensions[get_column_letter(c)].width=12.0
        ws.column_dimensions["C"].width=8.0; ws.column_dimensions["D"].width=45.0
        self._bh(ws,2,3,8); ws.row_dimensions[2].height=28.0
        ws.merge_cells(start_row=4,start_column=3,end_row=4,end_column=8)
        c=ws.cell(row=4,column=3,value="ANEXO 3: PROJETO TÉCNICO"); c.font=s.font_subtitle; c.alignment=s.align_center; c.fill=s.fill_secondary
        ws.merge_cells(start_row=5,start_column=3,end_row=5,end_column=8)
        c=ws.cell(row=5,column=3,value="PROJETO DE INTERLIGAÇÃO PARA ENCAMINHAMENTO DA TERMINAÇÃO DE CHAMADAS DE VOZ")
        c.font=Font(name="Calibri",size=11,bold=True,color="FFFFFF"); c.alignment=s.align_center; c.fill=s.fill_secondary; ws.row_dimensions[5].height=30.0
        for i,t in enumerate(["1. Versões","2. Projeto de Interligação","2.2. Diagrama de Interligação",
            "2.3. Características do Projeto de Interligação e do Plano de Encaminhamento","2.4. Plano de Contingência",
            "2.5. Concentração","2.6. Plan NUM_Operadora","2.7. Dados de MTL","2.8. SE REG III (Vivo STFC Concessionária)","2.9. Parâmetros de Programação"]):
            r=7+i; ws.merge_cells(start_row=r,start_column=3,end_row=r,end_column=8)
            c=ws.cell(row=r,column=3,value=t); c.font=Font(name="Calibri",size=11); c.alignment=s.align_left; c.fill=s.alt_fill(i); ws.row_dimensions[r].height=22.0
        ws.freeze_panes="C7"; self._ps(ws)

    # =====================================================================
    # ABA: VERSÕES — FIX: usa escopo_completo (texto + flags)
    # =====================================================================
    def _b_versions(self):
        s=self.s; ws=self.wb.create_sheet(title="Versões")
        self._cw(ws,{1:2,2:2,3:8,4:12,5:28,6:28,7:28,8:10,9:18,10:10,11:2})
        self._bh(ws,2,3,10); self._st(ws,4,3,10,"CONTROLE DE VERSÕES DO PTI")
        for i,t in enumerate(["Versão","Data","Responsável Eng de ITX","Responsável Gestão de ITX","Escopo","CN","ÁREAS LOCAIS","ATA"]):
            c=ws.cell(row=6,column=3+i,value=t); c.font=Font(name="Calibri",size=9,bold=True,color="FFFFFF"); c.alignment=s.align_center; c.fill=s.fill_light
        ws.row_dimensions[6].height=25.0

        # -----------------------------------------------------------------
        # FIX: Dados da primeira versão puxados corretamente do formulário.
        # Col 7 (Escopo) agora inclui texto livre + flags selecionados.
        # Col 8 (CN) e Col 9 (Áreas Locais) agora também buscam de op_rows
        # quando vivo_rows está vazio.
        # -----------------------------------------------------------------
        fd = {
            4: self.created_str,
            5: self.resp_eng,
            6: self.resp_atk,
            7: self.escopo_completo,   # ← FIX: era self.escopo_text
            8: ", ".join(self.cns_unicos) if self.cns_unicos else "",
            9: ", ".join(self.areas_locais) if self.areas_locais else "",
        }
        for i in range(10):
            r=7+i; ws.row_dimensions[r].height=22.0; f=s.alt_fill(i)
            ws.cell(row=r,column=3,value=i+1).fill=f; ws.cell(row=r,column=3).font=s.font_body; ws.cell(row=r,column=3).alignment=s.align_center
            for col in range(4,11):
                c=ws.cell(row=r,column=col); c.value=fd.get(col,"") if i==0 else ""; c.font=s.font_body; c.fill=f
                c.alignment=Alignment(horizontal="center", vertical="center", wrap_text=True) if col in (7,9) else s.align_center
        ws.freeze_panes="C7"
        # Ocultar linhas sem conteúdo (apenas linhas de dados, row 7+)
        for i in range(10):
            r = 7 + i
            has_content = False
            for col in range(3, 11):
                val = ws.cell(row=r, column=col).value
                if val is not None and str(val).strip():
                    has_content = True
                    break
            if not has_content:
                ws.row_dimensions[r].hidden = True
        self._ps(ws,ls=True)

    # =====================================================================
    # ABA: DIAGRAMA DE INTERLIGAÇÃO
    # FIX: Tráfego (flags de escopo) e Mask preenchidos corretamente
    #      em AMBAS as pontas (A=VIVO, B=Operadora).
    # =====================================================================
    def _b_diagram(self):
        s=self.s; ws=self.wb.create_sheet(title="Diagrama de Interligação"); cfg=self._diagram_cfg(); n=self.nome

        # -----------------------------------------------------------------
        # FONTE DE DADOS: usa vivo_rows se existem, senão op_rows
        # Cada linha = 1 diagrama completo, empilhados verticalmente
        # -----------------------------------------------------------------
        source_rows = self.vivo_rows if self.vivo_rows else self.op_rows
        num_blocks = max(1, len(source_rows))
        is_vivo_source = bool(self.vivo_rows)

        # Layout fixo em colunas 3-14 (todos os blocos usam mesmas colunas)
        LC = 3   # left col start
        RC = 14  # right col end
        MID = 9  # ponta B starts here

        col_widths = {1: 2.0, 2: 2.0, 15: 2.0}
        for c in range(LC, RC + 1):
            col_widths[c] = 12.0
        self._cw(ws, col_widths)

        # ===== CABEÇALHO GLOBAL (linhas 2-13) =====
        self._bh(ws, 2, LC, RC)
        for r_row, t, bold in [
            (4, f"Diagrama de Interligação entre a VIVO e a {n}", True),
            (6, "2.2.1 DIAGRAMAÇÃO DO PROJETO SIP", True),
            (7, "2.2.1.1 Anúncio de Redes pelos 2 Links SPC", True),
            (8, f"A {n} abordará os endereços da VIVO conforme abaixo.", False),
            (9, f"A VIVO abordará os endereços da {n} conforme abaixo.", False),
            (11, "2.2.1.2 Parâmetros de Configuração do Link", True),
        ]:
            ws.merge_cells(start_row=r_row, start_column=LC, end_row=r_row, end_column=RC)
            c = ws.cell(row=r_row, column=LC, value=t)
            c.font = Font(name="Calibri", size=11 if bold else 10, bold=bold)
            c.alignment = s.align_left if bold else s.align_wrap
            ws.row_dimensions[r_row].height = 22.0 if bold else 25.0

        # ASN (global)
        for o, label, val in [(0, "ASN VIVO", "10429 (Público)"), (1, f"ASN {n}", self.asn_op)]:
            r_row = 12 + o
            ws.merge_cells(start_row=r_row, start_column=LC, end_row=r_row, end_column=LC+2)
            ws.cell(row=r_row, column=LC, value=label).font = Font(name="Calibri", size=10, bold=True)
            ws.cell(row=r_row, column=LC).alignment = s.align_center
            ws.merge_cells(start_row=r_row, start_column=LC+3, end_row=r_row, end_column=LC+5)
            ws.cell(row=r_row, column=LC+3, value=val).font = s.font_body
            ws.cell(row=r_row, column=LC+3).alignment = s.align_center

        # FLAGS DE ESCOPO = tráfego para ambas as pontas
        traffic_items = [t for t in self.traffic if t]
        num_traffic = max(len(traffic_items), 3)

        # ===== BLOCOS VERTICAIS (1 por linha do formulário) =====
        cursor = 15  # linha inicial do primeiro bloco

        for b_idx in range(num_blocks):
            src_row = source_rows[b_idx] if b_idx < len(source_rows) else {}

            # Resolver dados cruzando vivo ↔ operadora
            if is_vivo_source:
                vivo_row = src_row
                op_row = self._find_matching_op_row(src_row.get("cn",""), b_idx)
            else:
                op_row = src_row
                vivo_row = self._find_matching_vivo_row(src_row.get("cn",""), b_idx)

            # Extrair valores com fallback entre as tabelas
            cn_val = vivo_row.get("cn","") or (op_row.get("cn","") if op_row else "")
            mask_val = vivo_row.get("mask","") or ""
            endereco_vivo = vivo_row.get("endereco_link","") or ""
            sbc_vivo = vivo_row.get("sbc","") or ""
            endereco_op = (op_row.get("endereco_link","") if op_row else "") or ""
            faixa_ip_op = (op_row.get("faixa_ip","") if op_row else "") or ""
            sbc_op = (op_row.get("sbc","") if op_row else "") or ""

            # Resolver cidade/UF para o CN
            cn_meta = CN_METADATA.get(cn_val.zfill(2), ("","")) if cn_val else ("","")
            cn_cidade, cn_uf = cn_meta

            # --- Linha 1 do bloco: CN + Cidade/UF + VRF ---
            r = cursor
            cn_label = f"CN {cn_val}"
            if cn_cidade:
                cn_label += f" — {cn_cidade}/{cn_uf}"
            ws.merge_cells(start_row=r, start_column=LC, end_row=r, end_column=LC+4)
            c = ws.cell(row=r, column=LC, value=cn_label if cn_val else "CN ___")
            c.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c.alignment = s.align_center; c.fill = s.fill_light
            ws.merge_cells(start_row=r, start_column=MID, end_row=r, end_column=RC)
            c2 = ws.cell(row=r, column=MID, value="VRF: __________________________")
            c2.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c2.alignment = s.align_center; c2.fill = s.fill_light
            ws.row_dimensions[r].height = 25.0
            # Limpar gap entre CN label e VRF
            for gc in range(LC+5, MID):
                ws.cell(row=r, column=gc).fill = s.fill_white
                ws.cell(row=r, column=gc).border = s.no_border

            # --- Linha 2: separador ---
            cursor += 2

            # --- Linhas Ponta A / Ponta B ---
            r = cursor
            for sc, ec, label, valor in [
                (LC, LC+4, "Ponta A", "VIVO"),
                (MID, RC, "Ponta B", n)
            ]:
                ws.merge_cells(start_row=r, start_column=sc, end_row=r, end_column=ec)
                ws.cell(row=r, column=sc, value=label).font = Font(name="Calibri", size=10, bold=True)
                ws.cell(row=r, column=sc).alignment = s.align_center
                ws.cell(row=r, column=sc).fill = s.fill_neutral
                ws.merge_cells(start_row=r+1, start_column=sc, end_row=r+1, end_column=ec)
                ws.cell(row=r+1, column=sc, value=valor).font = Font(name="Calibri", size=10, bold=True)
                ws.cell(row=r+1, column=sc).alignment = s.align_center
            ws.row_dimensions[r].height = 22.0
            ws.row_dimensions[r+1].height = 22.0
            # Limpar gap entre Ponta A e Ponta B
            for rr in (r, r+1):
                for gc in range(LC+5, MID):
                    ws.cell(row=rr, column=gc).fill = s.fill_white
                    ws.cell(row=rr, column=gc).border = s.no_border
            cursor += 3

            # --- SBC de cada ponta (alinhado com coluna inicial da tabela) ---
            r = cursor
            ws.merge_cells(start_row=r, start_column=LC, end_row=r, end_column=LC+2)
            ws.cell(row=r, column=LC, value=f"SBC: {sbc_vivo}" if sbc_vivo else "SBC: ___").font = Font(name="Calibri", size=9, bold=True)
            ws.cell(row=r, column=LC).alignment = s.align_center
            ws.merge_cells(start_row=r, start_column=MID, end_row=r, end_column=MID+2)
            ws.cell(row=r, column=MID, value=f"SBC: {sbc_op}" if sbc_op else "SBC: ___").font = Font(name="Calibri", size=9, bold=True)
            ws.cell(row=r, column=MID).alignment = s.align_center
            ws.row_dimensions[r].height = 20.0
            # Limpar gap entre SBC labels
            for gc in range(LC+3, MID):
                ws.cell(row=r, column=gc).fill = s.fill_white
                ws.cell(row=r, column=gc).border = s.no_border
            cursor += 1

            # --- Cabeçalhos: Tráfego | Endereço IP | NET MASK (ambas pontas) ---
            r = cursor
            for lc_start in (LC, MID):
                for i, h in enumerate(["Tráfego", "Endereço IP", "NET MASK"]):
                    c = ws.cell(row=r, column=lc_start+i, value=h)
                    c.font = s.font_subheader; c.alignment = s.align_center; c.fill = s.fill_light
                    c.border = s.box_border
            ws.row_dimensions[r].height = 25.0
            # Limpar gap entre cabeçalhos das duas tabelas
            for gc in range(LC+3, MID):
                ws.cell(row=r, column=gc).fill = s.fill_white
                ws.cell(row=r, column=gc).border = s.no_border
            cursor += 1

            # --- DADOS DE TRÁFEGO (apenas linhas com conteúdo visíveis) ---
            num_visible = len(traffic_items)  # linhas com dados reais
            data_start_row = cursor
            for j in range(num_traffic):
                r = cursor + j
                ws.row_dimensions[r].height = 22.0
                f = s.alt_fill(j)
                traf_val = traffic_items[j] if j < len(traffic_items) else ""

                # Ocultar linhas sem conteúdo de tráfego
                if j >= num_visible:
                    ws.row_dimensions[r].hidden = True

                # PONTA A (VIVO): Tráfego | IP (só 1ª linha) | MASK (individual, será mesclado)
                ws.cell(row=r, column=LC, value=traf_val).font = s.font_small
                ws.cell(row=r, column=LC).fill = f; ws.cell(row=r, column=LC).alignment = s.align_center; ws.cell(row=r, column=LC).border = s.box_border
                ws.cell(row=r, column=LC+1, value=endereco_vivo if j == 0 else "").font = s.font_small
                ws.cell(row=r, column=LC+1).fill = f; ws.cell(row=r, column=LC+1).alignment = s.align_center; ws.cell(row=r, column=LC+1).border = s.box_border
                ws.cell(row=r, column=LC+2, value="" if num_visible > 1 else mask_val).font = s.font_small
                ws.cell(row=r, column=LC+2).fill = f; ws.cell(row=r, column=LC+2).alignment = s.align_center; ws.cell(row=r, column=LC+2).border = s.box_border

                # Colunas intermediárias (gap limpo — sem ligação visual)
                for gc in range(LC+3, MID):
                    ws.cell(row=r, column=gc).fill = s.fill_white
                    ws.cell(row=r, column=gc).border = s.no_border

                # PONTA B (OPERADORA): Tráfego | IP (só 1ª linha) | MASK (individual)
                ws.cell(row=r, column=MID, value=traf_val).font = s.font_small
                ws.cell(row=r, column=MID).fill = f; ws.cell(row=r, column=MID).alignment = s.align_center; ws.cell(row=r, column=MID).border = s.box_border
                ws.cell(row=r, column=MID+1, value=faixa_ip_op if j == 0 else "").font = s.font_small
                ws.cell(row=r, column=MID+1).fill = f; ws.cell(row=r, column=MID+1).alignment = s.align_center; ws.cell(row=r, column=MID+1).border = s.box_border
                ws.cell(row=r, column=MID+2, value="" if num_visible > 1 else mask_val).font = s.font_small
                ws.cell(row=r, column=MID+2).fill = f; ws.cell(row=r, column=MID+2).alignment = s.align_center; ws.cell(row=r, column=MID+2).border = s.box_border

            # --- Merge NET MASK verticalmente (somente linhas visíveis) ---
            if num_visible > 1:
                last_vis = data_start_row + num_visible - 1
                # Ponta A — NET MASK merge
                ws.merge_cells(start_row=data_start_row, start_column=LC+2,
                               end_row=last_vis, end_column=LC+2)
                mc_a = ws.cell(row=data_start_row, column=LC+2, value=mask_val)
                mc_a.font = s.font_small; mc_a.alignment = s.align_center; mc_a.border = s.box_border
                # Ponta B — NET MASK merge
                ws.merge_cells(start_row=data_start_row, start_column=MID+2,
                               end_row=last_vis, end_column=MID+2)
                mc_b = ws.cell(row=data_start_row, column=MID+2, value=mask_val)
                mc_b.font = s.font_small; mc_b.alignment = s.align_center; mc_b.border = s.box_border

            cursor += num_traffic

            # --- Resumo de endereços abaixo da tabela ---
            r = cursor + 1
            ws.merge_cells(start_row=r, start_column=LC, end_row=r, end_column=LC+4)
            ws.cell(row=r, column=LC, value=f"Endereço (VIVO): {endereco_vivo}").font = s.font_small
            ws.cell(row=r, column=LC).alignment = s.align_left
            ws.merge_cells(start_row=r, start_column=MID, end_row=r, end_column=RC)
            ws.cell(row=r, column=MID, value=f"Endereço ({n}): {endereco_op}").font = s.font_small
            ws.cell(row=r, column=MID).alignment = s.align_left
            # Limpar gap entre resumos
            for gc in range(LC+5, MID):
                ws.cell(row=r, column=gc).fill = s.fill_white
                ws.cell(row=r, column=gc).border = s.no_border
            cursor = r + 2

            # =============================================================
            # DIAGRAMA POR BLOCO — Cada linha gera seu próprio diagrama
            # com dados específicos (SBC, IP, CN) renderizados na imagem.
            # =============================================================
            if os.path.exists(cfg["img"]):
                # Deslocar imagem para baixo — espaço limpo entre tabelas e imagem
                cursor += 4  # 4 linhas extras de espaçamento
                img_anchor_row = cursor

                # Labels padrão (presentes em todos os diagramas)
                row_labels = [
                    {"text": "Roteador OP", "xy_pct": cfg["r1"],
                     "font_size_pct": cfg["rfp"], "stroke_width_pct": cfg["rsp"]},
                    {"text": "Roteador OP", "xy_pct": cfg["r2"],
                     "font_size_pct": cfg["rfp"], "stroke_width_pct": cfg["rsp"]},
                    {"text": "Link resp Vivo", "xy_pct": cfg["lvxy"],
                     "font_size_pct": cfg["lfsz"], "stroke_width_pct": 0.006},
                    {"text": f"Link resp {n}", "xy_pct": cfg["loxy"],
                     "font_size_pct": cfg["lfsz"], "stroke_width_pct": 0.006},
                    {"text": "RAC/RAV/HL4", "xy_pct": (0.33, 0.70),
                     "font_size_pct": 0.010, "stroke_width_pct": 0.006},
                    {"text": "RAC/RAV/HL4", "xy_pct": (0.34, 0.60),
                     "font_size_pct": 0.010, "stroke_width_pct": 0.006},
                ]

                # Labels dinâmicos PER-ROW: SBC, IP, CN/Cidade
                if sbc_vivo:
                    row_labels.append({
                        "text": sbc_vivo, "xy_pct": (0.16, 0.62),
                        "font_size_pct": 0.025, "stroke_width_pct": 0.006,
                    })
                if sbc_op:
                    row_labels.append({
                        "text": sbc_op, "xy_pct": (0.83, 0.56),
                        "font_size_pct": 0.025, "stroke_width_pct": 0.006,
                    })
                if endereco_vivo:
                    row_labels.append({
                        "text": endereco_vivo, "xy_pct": (0.16, 0.72),
                        "font_size_pct": 0.020, "stroke_width_pct": 0.005,
                    })
                if faixa_ip_op or endereco_op:
                    ip_text = faixa_ip_op or endereco_op
                    row_labels.append({
                        "text": ip_text, "xy_pct": (0.83, 0.66),
                        "font_size_pct": 0.020, "stroke_width_pct": 0.005,
                    })
                if cn_val:
                    cn_img_text = f"CN {cn_val}"
                    if cn_cidade:
                        cn_img_text += f" — {cn_cidade}/{cn_uf}"
                    row_labels.append({
                        "text": cn_img_text, "xy_pct": (0.50, 0.08),
                        "font_size_pct": 0.028, "stroke_width_pct": 0.007,
                    })

                try:
                    ann = render_labels_on_image(
                        image_path=cfg["img"],
                        vivo_text="VIVO",
                        operator_text=n,
                        vivo_xy_pct=cfg["vxy"],
                        operator_xy_pct=cfg["oxy"],
                        font_path=cfg["fp"],
                        font_size_pct=cfg["fsz"],
                        stroke_width_pct=0.009,
                        extra_labels=row_labels,
                    )
                    xi = XLImage(ann)
                    try:
                        with PILImage.open(ann) as src:
                            ow, oh = src.size
                    except Exception:
                        ow, oh = 800, 300
                    fw, fh = fit_size_keep_aspect(ow, oh, cfg["bw"], cfg["bh"])
                    xi.width, xi.height = fw, fh
                    anchor_col = get_column_letter(LC + cfg["aco"])
                    anchor_row = max(1, img_anchor_row + cfg["aro"])
                    ws.add_image(xi, f"{anchor_col}{anchor_row}")

                    # Reservar linhas para a altura da imagem (~15px por row)
                    estimated_img_rows = max(22, int(fh / 15) + 2)
                    cursor = img_anchor_row + estimated_img_rows + 2
                except Exception as e:
                    logger.warning(f"Erro ao gerar imagem do diagrama (bloco {b_idx+1}): {e}")
                    cursor += 3
            else:
                # Sem imagem base — apenas log, sem crash
                if b_idx == 0:
                    logger.warning(f"Imagem base do diagrama não encontrada: {cfg['img']}")
                cursor += 3

        self._ps(ws, ls=True)

    def _find_matching_op_row(self, cn, index):
        """Busca linha da operadora que corresponde ao CN ou ao índice."""
        if cn:
            for op in self.op_rows:
                if op.get("cn","") == cn:
                    return op
        if index < len(self.op_rows):
            return self.op_rows[index]
        return None

    def _find_matching_vivo_row(self, cn, index):
        """Busca linha da VIVO que corresponde ao CN ou ao índice."""
        if cn:
            for vr in self.vivo_rows:
                if vr.get("cn","") == cn:
                    return vr
        if index < len(self.vivo_rows):
            return self.vivo_rows[index]
        return {}

    # =====================================================================
    # MAPEAMENTO: Tipo de Tráfego → Encaminhamento / Sinalização / CODEC
    # =====================================================================
    def _resolve_traffic_mapping(self, traf_type, operadora, cn_vivo, cn_op):
        """
        Mapeia um tipo de tráfego para os campos:
          - enc_ab  (DE A > B / Formato de Entrega)  → col 17
          - enc_ba  (DE B > A)                        → col 18
          - codec                                     → col 19

        Substituições aplicadas:
          <Operadora> → nome da operadora
          XY          → código CSP extraído do tipo (ex: "15" de "CSP 15")
          CN          → cn_vivo ou cn_op conforme contexto
        """
        t = traf_type.strip()
        t_upper = t.upper()
        op = operadora or "Operadora"
        cv = cn_vivo or "CN"
        co = cn_op or "CN"

        # Extrair CSP (XY) do tipo de tráfego, se presente
        import re as _re
        csp_match = _re.search(r'CSP\s*(\d{2})', t, _re.IGNORECASE)
        # Também extrair de "Rota LD XY"
        rota_match = _re.search(r'Rota\s+LD\s*(\d{2})', t, _re.IGNORECASE)
        xy = csp_match.group(1) if csp_match else (rota_match.group(1) if rota_match else "XY")
        # Detectar se é "s/ CSP" (sem CSP)
        sem_csp = bool(_re.search(r's/\s*CSP', t, _re.IGNORECASE))

        # --- Tabela de mapeamento (ordem de matching) ---

        # 1) LC+TR LC - AL SPO
        if "LC+TR" in t_upper or ("LC" in t_upper and "TR" in t_upper and "AL" in t_upper):
            return {
                "enc_ab": f"LC: (9090) PREF-MCDU {op} do CN {cv}",
                "enc_ba": f"LC: (9090) PREF-MCDU VIVO do CN {cv}",
                "codec":  "G711 / G729",
            }

        # 2) CSP 15 + CNG VIVO-STFC - SPO - CN 11
        if "CSP" in t_upper and xy == "15" and "CNG" in t_upper and "VIVO" in t_upper:
            return {
                "enc_ab": f"LD: (9)01511 PREF-MCDU - NUM B do CN {cv}",
                "enc_ba": (
                    f"LD: (9) 015 {co} PREF-MCDU / LDI: 0015 (***)\n"
                    f"CNG VIVO: 08XX 03XX 05XX 09XX (10-11 dig) (***)\n"
                    f"SE: 10315/10615"
                ),
                "codec": "G711 / G729",
            }

        # 3) CSP XY + CNG - SPO - CN (genérico, não CSP 15 VIVO, não "s/ CSP")
        if "CSP" in t_upper and "CNG" in t_upper and "TRANSP" not in t_upper and not sem_csp:
            return {
                "enc_ab": (
                    f"LD: (9)0 {xy} {co} PREF-MCDU / LDI: 00 {xy} (***)\n"
                    f"CNG VIVO: 08XX 03XX 05XX 09XX (10-11 dig) (***)\n"
                    f"SE: 103{xy} / 106{xy}"
                ),
                "enc_ba": f"LD: (9)0+{xy}+{co} PREF-MCDU - NUM B do CN {cv}",
                "codec": "G711 / G729",
            }

        # 4) LD s/ CSP + CNG - SPO - CN
        if "LD" in t_upper and "S/" in t_upper and "CSP" in t_upper and "TRANSP" not in t_upper:
            return {
                "enc_ab": "CNG: 08XX 03XX 05XX 09XX (10-11 dig)",
                "enc_ba": f"LD: (9)0+{co} PREF-MCDU - NUM B do CN {cv}",
                "codec": "G711 / G729",
            }

        # 5) Transp CSP XY + CNG
        if "TRANSP" in t_upper and "CSP" in t_upper and "CNG" in t_upper:
            return {
                "enc_ab": "CNG: 08XX 03XX 05XX 09XX (10-11 dig) (**)",
                "enc_ba": f"LD: (9)0+{xy}+{co} PREF-MCDU - No. B diferente do CN {cv}",
                "codec": "G711 / G729",
            }

        # 6) Transp LD s/ CSP + CNG
        if "TRANSP" in t_upper and "LD" in t_upper and "S/" in t_upper:
            return {
                "enc_ab": "CNG: 08XX 03XX 05XX 09XX (10-11 dig) (**)",
                "enc_ba": f"LD: (9)0+{co} PREF-MCDU No. B diferente do CN {cv}",
                "codec": "G711 / G729",
            }

        # 7) VC1 - CN XX - Rota LD XY (precisa vir antes do VC1 genérico)
        if "VC1" in t_upper and "ROTA" in t_upper and "LD" in t_upper:
            return {
                "enc_ab": f"LD: (9) 0 {xy} {cv} PREF-MCDU do CN {cv} No de B VIVO",
                "enc_ba": f"LD: (9) 0 {xy} {co} PREF-MCDU - CN{cv}  No de B VIVO",
                "codec": "G-711 / AMR",
            }

        # 8) VC1 - CN XX (genérico)
        if "VC1" in t_upper:
            return {
                "enc_ab": f"LC: (9) 0{cv} PREF-MCDU - CN {cv}",
                "enc_ba": f"LC: (9) 0{co} PREF-MCDU - CN{cv} - SE: 1058",
                "codec": "G-711 / AMR",
            }

        # Fallback: tipo não mapeado → retorna vazio
        return {"enc_ab": "", "enc_ba": "", "codec": ""}

    # =====================================================================
    # SBC RESOLVER — Busca o melhor SBC para um UF/Cidade via XLSX data
    # =====================================================================
    def _resolve_sbc_for_uf(self, uf, cidade=""):
        """
        Busca o SBC mais adequado para um UF a partir dos dados XLSX.

        Lógica:
          1. Se cidade informada → busca SBC da mesma cidade
          2. Senão → SBC mais recorrente (mais medições) para o UF
          3. Fallback: vizinhos → regional

        Retorna dict com {nome, cidade, uf, caps_avg, status, modelo}
        ou None se nenhum SBC encontrado.
        """
        if not self.sbc_analyzer or not uf:
            return None
        uf = uf.strip().upper()
        try:
            measurements, _ = self.sbc_analyzer.processor.get_data()
        except Exception:
            return None
        if not measurements:
            return None

        # Filtrar medições por UF
        uf_m = [m for m in measurements if m.uf == uf]

        # Fallback: vizinhos
        fallback_label = ""
        if not uf_m:
            for n_uf in UF_NEIGHBORS.get(uf, []):
                uf_m = [m for m in measurements if m.uf == n_uf]
                if uf_m:
                    fallback_label = f"(vizinho: {n_uf})"
                    break
        # Fallback: regional
        if not uf_m:
            regional = UF_TO_REGIONAL.get(uf, "")
            if regional:
                regional_ufs = [u for u, r in UF_TO_REGIONAL.items() if r == regional]
                uf_m = [m for m in measurements if m.uf in regional_ufs]
                if uf_m:
                    fallback_label = f"(regional: {regional})"

        if not uf_m:
            return None

        # Agrupar por nome do SBC
        groups = defaultdict(list)
        for m in uf_m:
            groups[m.sbc].append(m)

        # Se cidade informada, priorizar SBC da mesma cidade
        cidade_norm = cidade.strip().upper() if cidade else ""
        if cidade_norm:
            for sbc_name, ms in groups.items():
                for m in ms:
                    if m.cidade.strip().upper() == cidade_norm:
                        caps_vals = [x.caps for x in ms]
                        return {
                            "nome": sbc_name,
                            "cidade": ms[0].cidade,
                            "uf": ms[0].uf,
                            "caps_avg": round(sum(caps_vals) / len(caps_vals), 1),
                            "status": ms[0].status,
                            "modelo": ms[0].modelo,
                            "total_medicoes": len(ms),
                            "fallback": fallback_label,
                        }

        # Senão: SBC mais recorrente (mais medições)
        best_name = max(groups, key=lambda k: len(groups[k]))
        best_ms = groups[best_name]
        caps_vals = [x.caps for x in best_ms]
        return {
            "nome": best_name,
            "cidade": best_ms[0].cidade,
            "uf": best_ms[0].uf,
            "caps_avg": round(sum(caps_vals) / len(caps_vals), 1),
            "status": best_ms[0].status,
            "modelo": best_ms[0].modelo,
            "total_medicoes": len(best_ms),
            "fallback": fallback_label,
        }

    # =====================================================================
    # ABA: ENCAMINHAMENTO — 1 tabela completa por linha do formulário
    #
    # Regras implementadas:
    #   - Cada linha do form → 1 tabela idêntica ao modelo padrão
    #   - CN único por coluna (merge vertical, centralizado)
    #   - SBC preenchido obrigatoriamente via lookup UF→XLSX
    #   - IP da Vivo incluído
    #   - Células com auto-adjust de largura
    #   - Alerta de campos ausentes
    # =====================================================================
    def _b_routing(self):
        s = self.s
        ws = self.wb.create_sheet(title="Encaminhamento")
        n = self.nome

        # ===== AUTO-ADJUST: larguras base (serão atualizadas no final) =====
        col_widths = {
            1: 2, 2: 16, 3: 10, 4: 16, 5: 10, 6: 18,
            7: 16, 8: 18, 9: 14, 10: 8, 11: 8, 12: 8,
            13: 8, 14: 8, 15: 8, 16: 12, 17: 18, 18: 12,
            19: 12, 20: 16, 21: 14, 22: 12, 23: 14, 24: 12, 25: 2,
        }
        # Track max widths for auto-adjust
        max_widths = {c: w for c, w in col_widths.items()}

        def _track_width(col, value):
            """Atualiza largura máxima da coluna com base no conteúdo."""
            if value:
                char_len = len(str(value)) * 1.15 + 2
                if char_len > max_widths.get(col, 0):
                    max_widths[col] = min(char_len, 45)  # cap em 45

        # ===== CABEÇALHO GLOBAL =====
        self._bh(ws, 2, 2, 24)
        self._st(ws, 4, 2, 24,
                 "2.3. CARACTERÍSTICAS DO PROJETO DE INTERLIGAÇÃO "
                 "E DO PLANO DE ENCAMINHAMENTO")

        # ===== FONTE DE DADOS =====
        source_rows = self.vivo_rows if self.vivo_rows else self.op_rows
        num_blocks = max(1, len(source_rows))
        is_vivo_source = bool(self.vivo_rows)
        traffic_items = [t for t in self.traffic if t]
        traffic_str = " / ".join(traffic_items) or ""

        cursor = 6  # primeira linha de conteúdo

        for b_idx in range(num_blocks):
            src_row = source_rows[b_idx] if b_idx < len(source_rows) else {}

            # ----- Resolver dados cruzando vivo ↔ operadora -----
            if is_vivo_source:
                vivo_row = src_row
                op_row = self._find_matching_op_row(src_row.get("cn", ""), b_idx)
            else:
                op_row = src_row
                vivo_row = self._find_matching_vivo_row(src_row.get("cn", ""), b_idx)
            if op_row is None:
                op_row = {}

            # ----- Extrair valores -----
            cn_vivo = vivo_row.get("cn", "") or ""
            cn_op = (op_row.get("cn", "") if op_row else "") or cn_vivo
            localidade_vivo = vivo_row.get("localidade", "") or ""
            localidade_op = (op_row.get("localidade", "") if op_row else "") or ""
            sbc_vivo = vivo_row.get("sbc", "") or ""
            sbc_op = (op_row.get("sbc", "") if op_row else "") or ""
            endereco_vivo = vivo_row.get("endereco_link", "") or ""
            mask_vivo = vivo_row.get("mask", "") or ""
            faixa_ip_op = (op_row.get("faixa_ip", "") if op_row else "") or ""
            endereco_op = (op_row.get("endereco_link", "") if op_row else "") or ""
            ip_op = faixa_ip_op or endereco_op
            mask_op = mask_vivo

            # Resolver cidade / Área Local
            cidade_vivo = vivo_row.get("cidade", "") or ""
            cidade_op = (op_row.get("cidade", "") if op_row else "") or ""
            area_local = cidade_vivo or cidade_op
            uf_val = vivo_row.get("uf", "") or (op_row.get("uf", "") if op_row else "")
            if not area_local and cn_vivo:
                cn_meta = CN_METADATA.get(cn_vivo.zfill(2))
                if cn_meta:
                    area_local = cn_meta[0]
                    if not uf_val:
                        uf_val = cn_meta[1]

            # =============================================================
            # SBC OBRIGATÓRIO VIA LOOKUP UF → XLSX
            # Se o formulário não informou SBC, buscar nos dados XLSX
            # =============================================================
            sbc_vivo_note = ""
            if not sbc_vivo and uf_val:
                resolved = self._resolve_sbc_for_uf(uf_val, cidade_vivo or area_local)
                if resolved:
                    sbc_vivo = resolved["nome"]
                    sbc_vivo_note = (
                        f"Auto: {resolved['nome']} — "
                        f"{resolved['cidade']}/{resolved['uf']} "
                        f"(CAPS avg {resolved['caps_avg']}, "
                        f"{resolved['total_medicoes']} medições) "
                        f"{resolved.get('fallback', '')}"
                    ).strip()
                else:
                    sbc_vivo_note = "Nenhum SBC disponível para o UF informado"

            sbc_op_note = ""
            if not sbc_op and uf_val:
                resolved = self._resolve_sbc_for_uf(uf_val, cidade_op or area_local)
                if resolved:
                    sbc_op = resolved["nome"]
                    sbc_op_note = f"Auto: {resolved['nome']} {resolved.get('fallback', '')}"

            # ----- SEPARADOR DO BLOCO -----
            r = cursor
            block_title = f"CN {cn_vivo}" if cn_vivo else f"Bloco {b_idx + 1}"
            if area_local:
                block_title += f" — {area_local}"
            if uf_val:
                block_title += f" ({uf_val})"
            ws.merge_cells(start_row=r, start_column=2, end_row=r, end_column=24)
            c = ws.cell(row=r, column=2, value=block_title)
            c.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
            c.alignment = s.align_center
            c.fill = s.fill_primary
            c.border = s.box_border
            ws.row_dimensions[r].height = 26.0
            cursor = r + 1

            # =============================================================
            # CABEÇALHO NÍVEL 1 (3 linhas de header)
            # =============================================================
            h1 = cursor
            ws.row_dimensions[h1].height = 30.0
            for sc, ec, t, vertical in [
                (2, 2, "ÁREA LOCAL", True),
                (3, 6, "LOCALIZAÇÃO", False),
                (7, 9, "DADOS DA ROTA", False),
                (10, 11, "BANDA", False),
                (12, 13, "CANAIS", False),
                (14, 15, "CAPS", False),
                (16, 16, "ATIVAÇÃO", True),
                (17, 18, "ENCAMINHAMENTO", False),
                (19, 20, "SINALIZAÇÃO", False),
                (21, 24, "ENDEREÇO IP", False),
            ]:
                if vertical:
                    ws.merge_cells(start_row=h1, start_column=sc,
                                   end_row=h1 + 2, end_column=ec)
                else:
                    ws.merge_cells(start_row=h1, start_column=sc,
                                   end_row=h1, end_column=ec)
                c = ws.cell(row=h1, column=sc, value=t)
                c.font = s.font_subheader
                c.alignment = s.align_center_wrap
                c.fill = s.fill_secondary if vertical else s.fill_light
                c.border = s.box_border

            # CABEÇALHO NÍVEL 2
            h2 = h1 + 1
            ws.row_dimensions[h2].height = 25.0
            for col, t in [
                (3, "CN"), (4, "POI/PPI VIVO"),
                (5, "CN"), (6, f"POI/PPI {n.upper()}"),
                (7, "PONTA A VIVO"), (8, f"PONTA B {n.upper()}"),
                (9, "TIPO DE TRÁFEGO"),
                (10, "EXIST."), (11, "PLAN."),
                (12, "EXIST."), (13, "PLAN."),
                (14, "EXIST."), (15, "PLAN."),
                (17, "DE A > B\n(FORMATO DE ENTREGA)"), (18, "DE B > A"),
                (19, "CODEC"), (20, "OBSERVAÇÃO"),
            ]:
                ws.merge_cells(start_row=h2, start_column=col,
                               end_row=h2 + 1, end_column=col)
                c = ws.cell(row=h2, column=col, value=t)
                c.font = Font(name="Calibri",
                              size=7 if col == 17 else 8,
                              bold=True, color="FFFFFF")
                c.alignment = s.align_center_wrap
                c.fill = s.fill_accent
                c.border = s.box_border

            for sc, ec, t in [(21, 22, "VIVO"), (23, 24, n.upper())]:
                ws.merge_cells(start_row=h2, start_column=sc,
                               end_row=h2, end_column=ec)
                c = ws.cell(row=h2, column=sc, value=t)
                c.font = s.font_micro
                c.alignment = s.align_center
                c.fill = s.fill_accent
                c.border = s.box_border

            # CABEÇALHO NÍVEL 3: IP ADDRESS / NETMASK
            h3 = h2 + 1
            ws.row_dimensions[h3].height = 25.0
            for col, t in [(21, "IP ADDRESS"), (22, "NETMASK"),
                           (23, "IP ADDRESS"), (24, "NETMASK")]:
                c = ws.cell(row=h3, column=col, value=t)
                c.font = Font(name="Calibri", size=7, bold=True, color="FFFFFF")
                c.alignment = s.align_center
                c.fill = s.fill_light
                c.border = s.box_border

            # =============================================================
            # LINHAS DE DADOS (DATA_ROWS linhas)
            #
            # COLUNAS COM VALOR ÚNICO (merge vertical, centralizado):
            #   Col 2  = ÁREA LOCAL
            #   Col 3  = CN VIVO
            #   Col 4  = POI/PPI VIVO
            #   Col 5  = CN OPERADORA
            #   Col 6  = POI/PPI OPERADORA
            #   Col 7  = PONTA A (SBC VIVO)
            #   Col 8  = PONTA B (SBC OPERADORA)
            #   Col 16 = ATIVAÇÃO (data atual + 90 dias)
            #   Col 21 = IP ADDRESS VIVO
            #   Col 22 = NETMASK VIVO
            #   Col 23 = IP ADDRESS OPERADORA
            #   Col 24 = NETMASK OPERADORA
            #
            # COLUNA SEM MERGE (1 valor por linha, altura proporcional):
            #   Col 9  = TIPO DE TRÁFEGO
            #
            # COLUNAS LIVRES (vazias para preenchimento manual):
            #   Col 10-15, 17-20
            # =============================================================
            ds = h3 + 1  # primeira linha de dados

            # ATIVAÇÃO: data real DD/MM/AAAA + texto literal " + 90 dias"
            ativacao_str = datetime.now().strftime("%d/%m/%Y") + " + 90 dias"

            # DATA_ROWS = exatamente o número de tipos de tráfego (mín 1)
            num_traffic = len(traffic_items) if traffic_items else 1
            DATA_ROWS = max(num_traffic, 1)

            # Altura total fixa da área de dados; distribuída proporcionalmente
            TOTAL_DATA_HEIGHT = 110.0  # px (equivale a ~5 linhas de 22px)
            row_height = max(18.0, round(TOTAL_DATA_HEIGHT / DATA_ROWS, 1))

            # Alignment reutilizável: centro vertical + wrap
            align_merged = Alignment(
                horizontal="center", vertical="center", wrap_text=True
            )
            align_data_wrap = Alignment(
                horizontal="center", vertical="center", wrap_text=True
            )

            # -----------------------------------------------------------
            # PRÉ-CALCULAR MAPEAMENTO: tráfego → enc_ab / enc_ba / codec
            # -----------------------------------------------------------
            traffic_mappings = []
            for t_item in (traffic_items if traffic_items else [""]):
                mapping = self._resolve_traffic_mapping(
                    t_item, n, cn_vivo, cn_op
                )
                traffic_mappings.append(mapping)

            # -----------------------------------------------------------
            # MERGE VERTICAL: todas as colunas com valor único
            # -----------------------------------------------------------
            merged_cols = {
                2:  (area_local,      Font(name="Calibri", size=10, bold=True)),
                3:  (cn_vivo,         Font(name="Calibri", size=10, bold=True)),
                4:  (localidade_vivo, s.font_small),
                5:  (cn_op,           Font(name="Calibri", size=10, bold=True)),
                6:  (localidade_op,   s.font_small),
                7:  (sbc_vivo,        s.font_small),
                8:  (sbc_op,          s.font_small),
                16: (ativacao_str,    s.font_small),
                21: (endereco_vivo,   s.font_small),
                22: (mask_vivo,       s.font_small),
                23: (ip_op,           s.font_small),
                24: (mask_op,         s.font_small),
            }

            for col_num, (val, fnt) in merged_cols.items():
                ws.merge_cells(
                    start_row=ds, start_column=col_num,
                    end_row=ds + DATA_ROWS - 1, end_column=col_num,
                )
                c = ws.cell(row=ds, column=col_num, value=val)
                c.font = fnt
                c.alignment = align_merged
                c.fill = s.alt_fill(0)
                c.border = s.box_border
                _track_width(col_num, val)

            # -----------------------------------------------------------
            # LINHAS DE DADOS: tráfego (1 por linha) + colunas livres
            #
            # Altura proporcional: se 3 tipos de tráfego, cada linha
            # fica mais alta (~36px) para preencher a tabela visualmente.
            # -----------------------------------------------------------
            merged_col_set = set(merged_cols.keys())

            for i in range(DATA_ROWS):
                r = ds + i
                ws.row_dimensions[r].height = row_height
                f = s.alt_fill(i)
                traf_val = traffic_items[i] if i < len(traffic_items) else ""
                traf_map = traffic_mappings[i] if i < len(traffic_mappings) else {}

                # Colunas NÃO merged: renderizar individualmente
                for col in range(2, 25):
                    if col in merged_col_set:
                        # Apenas aplicar borda + fill nas linhas internas do merge
                        ws.cell(row=r, column=col).border = s.box_border
                        ws.cell(row=r, column=col).fill = f
                        continue

                    # Determinar valor da célula
                    if col == 9:
                        val = traf_val  # TRÁFEGO: 1 por linha (sem merge)
                    elif col == 17:
                        val = traf_map.get("enc_ab", "")  # DE A > B
                    elif col == 18:
                        val = traf_map.get("enc_ba", "")  # DE B > A
                    elif col == 19:
                        val = traf_map.get("codec", "")   # CODEC
                    elif col == 20:
                        val = "SIP-I"  # OBSERVAÇÃO: sempre SIP-I
                    else:
                        val = ""  # Colunas livres (BANDA, CANAIS, CAPS)

                    c = ws.cell(row=r, column=col, value=val)
                    c.font = s.font_small
                    c.alignment = align_data_wrap
                    c.border = s.box_border
                    c.fill = f

                    if i == 0:
                        _track_width(col, val)

            # ----- AVISO DE DADOS AUSENTES -----
            missing = []
            if not cn_vivo:
                missing.append("CN VIVO")
            if not sbc_vivo:
                missing.append("SBC (nenhum encontrado para UF)")
            if not area_local:
                missing.append("ÁREA LOCAL")
            if not endereco_vivo:
                missing.append("IP DA VIVO")
            if not uf_val:
                missing.append("UF")

            if missing:
                note_row = ds + DATA_ROWS
                ws.merge_cells(start_row=note_row, start_column=2,
                               end_row=note_row, end_column=24)
                note_text = f"⚠ Campos não preenchidos: {', '.join(missing)}"
                c = ws.cell(row=note_row, column=2, value=note_text)
                c.font = Font(name="Calibri", size=8, italic=True, color="FF6600")
                c.alignment = s.align_left
                ws.row_dimensions[note_row].height = 18.0
                cursor = note_row + 2
            else:
                cursor = ds + DATA_ROWS + 1

        # ===== AUTO-ADJUST LARGURAS =====
        for col_num, width in max_widths.items():
            ws.column_dimensions[get_column_letter(col_num)].width = max(
                col_widths.get(col_num, 8), width
            )

        # ===== LIMPEZA DE MARGENS LATERAIS =====
        for col in (1, 25):
            for r in range(1, cursor + 5):
                ws.cell(row=r, column=col).fill = s.fill_white
                ws.cell(row=r, column=col).border = s.no_border

        ws.freeze_panes = "C6"
        self._ps(ws, ls=True)

    def _b_conc(self):
        """
        Aba Concentração — 5 colunas: Ref | CN | CNL | Cod. CnI | Status Link
        Apenas linhas com dados reais (sem linhas vazias extras).
        """
        s = self.s
        ws = self.wb.create_sheet(title="Concentração")
        n = self.nome

        # --- Larguras ---
        col_w = {1: 2, 2: 8, 3: 8, 4: 12, 5: 12, 6: 25}
        max_w = dict(col_w)
        last_col = 6  # cols 2-6 (B-F): margem em 1 e 7

        def _tw(col, val):
            if val:
                cw = len(str(val)) * 1.15 + 2
                if cw > max_w.get(col, 0):
                    max_w[col] = min(cw, 35)

        align_wrap = Alignment(horizontal="center", vertical="center",
                               wrap_text=True)

        # ===== BANNER + TÍTULO =====
        self._bh(ws, 2, 2, last_col)
        self._st(ws, 4, 2, last_col, "2.5 - Concentração")

        # ===== TEXTOS INTRODUTÓRIOS =====
        for row, text, italic in [
            (5, f"A '{n.upper()}' informou interesse em realizar a "
                f"concentração de ALs conforme detalhado na tabela abaixo:",
             False),
            (6, 'Caso necessite de incluir mais ALs, Favor inserir linhas '
                'adicionais de acordo com reunião que foi acordado o tema '
                '(coluna B - "Ref").', True),
        ]:
            ws.merge_cells(start_row=row, start_column=2,
                           end_row=row, end_column=last_col)
            c = ws.cell(row=row, column=2, value=text)
            c.font = Font(name="Calibri", size=9 if italic else 10,
                          italic=italic)
            c.alignment = Alignment(horizontal="left", vertical="center",
                                    wrap_text=True)
            c.fill = s.fill_background
            ws.row_dimensions[row].height = 28.0 if italic else 22.0

        # ===== CABEÇALHO =====
        hr = 8
        ws.row_dimensions[hr].height = 25.0
        headers = [
            (2, "Ref"), (3, "CN"), (4, "CNL"),
            (5, "Cod. CnI"), (6, "Status Link"),
        ]
        for col, txt in headers:
            c = ws.cell(row=hr, column=col, value=txt)
            c.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c.alignment = align_wrap
            c.fill = s.fill_secondary
            c.border = s.box_border

        # ===== DADOS — apenas linhas com conteúdo =====
        conc_rows = [d for d in self.conc if any(d.values())] if self.conc else []
        ds = hr + 1

        for i, d in enumerate(conc_rows):
            r = ds + i
            ws.row_dimensions[r].height = 22.0
            f = s.alt_fill(i)

            row_vals = {
                2: d.get("ref", ""),
                3: d.get("cn", ""),
                4: d.get("cnl", ""),
                5: d.get("cod_cni", ""),
                6: d.get("status_link", ""),
            }

            for col_num in range(2, last_col + 1):
                val = row_vals.get(col_num, "")
                c = ws.cell(row=r, column=col_num, value=val)
                c.font = s.font_body
                c.fill = f
                c.border = s.box_border
                c.alignment = Alignment(
                    horizontal="left" if col_num == 6 else "center",
                    vertical="center", wrap_text=True,
                )
                _tw(col_num, val)

        # ===== AUTO-ADJUST LARGURAS =====
        for col_num, w in max_w.items():
            ws.column_dimensions[get_column_letter(col_num)].width = max(
                col_w.get(col_num, 8), w
            )

        # ===== MARGENS =====
        last_row = ds + len(conc_rows)
        for col in (1, last_col + 1):
            for r in range(1, last_row + 3):
                ws.cell(row=r, column=col).fill = s.fill_white
                ws.cell(row=r, column=col).border = s.no_border

        ws.freeze_panes = f"C{ds}"
        self._ps(ws)

    def _b_prefixes(self):
        s=self.s; ws=self.wb.create_sheet(title="Plan Num_Oper")
        self._cw(ws,{1:2,2:4,3:12,4:12,5:12,6:6,7:10,8:10,9:8,10:15}); self._bh(ws,2,2,10); self._st(ws,4,2,10,f"2.6. Tabela de Prefixos da {self.nome.upper()}")
        ws.merge_cells(start_row=5,start_column=2,end_row=5,end_column=10); ws.cell(row=5,column=2,value='OBS: Caso necessite incluir mais faixas, inserir linhas adicionais (coluna B - "Ref").').font=Font(name="Calibri",size=9,italic=True); ws.cell(row=5,column=2).alignment=s.align_wrap; ws.cell(row=5,column=2).fill=s.fill_background
        h1=7; ws.row_dimensions[h1].height=25.0
        for sc,ec,t,v in [(2,2,"Ref",True),(3,3,"Município",True),(4,4,"ÁREA LOCAL",True),(5,5,"CÓDIGO CNL",True),(6,6,"CN",True),(7,8,"SERIE AUTORIZADA",False),(9,9,"EOT LOCAL",True),(10,10,"SNOA STFC / SMP",True)]:
            if v: ws.merge_cells(start_row=h1,start_column=sc,end_row=h1+1,end_column=ec)
            else: ws.merge_cells(start_row=h1,start_column=sc,end_row=h1,end_column=ec)
            c=ws.cell(row=h1,column=sc,value=t); c.font=s.font_subheader; c.alignment=s.align_center; c.fill=s.fill_secondary; c.border=s.box_border
        h2=h1+1; ws.row_dimensions[h2].height=30.0
        for col,t in [(7,"INCLUÍNOS\nN8 N7 N6 N5"),(8,"FINAL\nN4 N3 N2 N1")]:
            c=ws.cell(row=h2,column=col,value=t); c.font=s.font_micro; c.alignment=s.align_center_wrap; c.fill=s.fill_accent; c.border=s.box_border
        ds=h2+1
        # 5 linhas de dados vazias — só a 1ª visível, demais ocultas
        for i in range(5):
            r=ds+i; ws.row_dimensions[r].height=20.0; f=s.alt_fill(i)
            for col in range(2,11): c=ws.cell(row=r,column=col,value=""); c.font=s.font_small; c.alignment=s.align_center; c.border=s.box_border; c.fill=f
            if i > 0:
                ws.row_dimensions[r].hidden = True

        # ===== BLOCO RN1/EOT/CSP/CNG/RN2 — títulos padronizados =====
        ps=ds+6
        title_font = Font(name="Calibri", size=9, bold=True)
        title_align = Alignment(horizontal="left", vertical="center")
        resp_align = Alignment(horizontal="left", vertical="center", wrap_text=True)
        TITLE_H = 22.0  # altura fixa e igual para todos

        for i,p in enumerate(["RN1","EOT Local","EOT LDN/LDI","CSP","Código Especial:","CNG:","RN2"]):
            r=ps+i; ws.row_dimensions[r].height=TITLE_H; f=s.alt_fill(i)
            # Título (col 2) — formatação padronizada idêntica em todos
            ct = ws.cell(row=r, column=2, value=p)
            ct.font = title_font; ct.alignment = title_align
            ct.border = s.box_border; ct.fill = f
            # Resposta (col 3) — adaptável, wrap_text
            cr_cell = ws.cell(row=r, column=3, value="")
            cr_cell.font = s.font_small; cr_cell.alignment = resp_align
            cr_cell.border = s.box_border; cr_cell.fill = f
            # Bordas restantes (4-10)
            for col in range(4, 11):
                cx = ws.cell(row=r, column=col, value="")
                cx.border = s.box_border; cx.fill = f
        for o,t in [(0,"Operadora de transporte nas localidades onde não existir rota direta"),(1,"Transbordo nas localidades onde existe rota direta:")]:
            ws.merge_cells(start_row=ps+o,start_column=5,end_row=ps+o,end_column=10); ws.cell(row=ps+o,column=5,value=t).font=Font(name="Calibri",size=9,bold=True)
        tr=ps+7+1
        for sc,ec,t in [(2,3,"UF"),(4,7,"SERIE AUTORIZADA")]:
            ws.merge_cells(start_row=tr,start_column=sc,end_row=tr,end_column=ec); c=ws.cell(row=tr,column=sc,value=t); c.font=s.font_subheader; c.alignment=s.align_center; c.fill=s.fill_secondary; c.border=s.box_border
        sr=tr+1
        for sc,ec in [(2,3),(4,5),(6,7)]:
            ws.merge_cells(start_row=sr,start_column=sc,end_row=sr,end_column=ec); ws.cell(row=sr,column=sc,value="").border=s.box_border; ws.cell(row=sr,column=sc).fill=s.fill_background
        cr=sr+2
        for o,t in [(0,"A programação de CNG ocorre pelo processo de portabilidade intrínseca - via ABR"),(1,"O encaminhamento do CNG será programado nas localidades onde o ponto de entrega for informado")]:
            ws.merge_cells(start_row=cr+o,start_column=2,end_row=cr+o,end_column=10); ws.cell(row=cr+o,column=2,value=t).font=s.font_small
        sm=cr+3; ws.merge_cells(start_row=sm,start_column=2,end_row=sm,end_column=10); ws.cell(row=sm,column=2,value="2.12 Tabela de Prefixos Vivo SMP").font=Font(name="Calibri",size=10,bold=True)
        ws.merge_cells(start_row=sm+1,start_column=2,end_row=sm+1,end_column=10); ws.cell(row=sm+1,column=2,value="Os prefixos da VIVO SMP, poderão ser obtidos acessando o site www.telefonica.net.br/sp/transfer/").font=s.font_small
        ws.freeze_panes=f"B{h1}"; self._ps(ws,ls=True)

    def _b_mtl(self):
        """
        Aba Dados MTL — 2 linhas por referência.

        Para cada ref (par vivo_row + op_row):
          Linha 1: MTL = nome operadora, CN = cn_vivo
          Linha 2: MTL = "VIVO",          CN = cn_op (ou mesmo)
        Ambas linhas: PONTA VIVO = endereco_vivo, PONTA OP = endereco_op
        Demais colunas em branco.
        """
        s = self.s
        ws = self.wb.create_sheet(title="Dados MTL")
        n = self.nome

        # Larguras
        col_w = {1: 2, 2: 6, 3: 8, 4: 10, 5: 12, 6: 18, 7: 18,
                 8: 20, 9: 15, 10: 18, 11: 18, 12: 20}
        self._cw(ws, col_w)
        self._bh(ws, 2, 2, 12)
        self._st(ws, 4, 2, 12, "2.7. Dados de MTL")

        # Texto intro
        ws.merge_cells(start_row=5, start_column=2, end_row=5, end_column=12)
        c = ws.cell(row=5, column=2,
                    value="Tabela de dados MTL para interligação entre "
                          "VIVO e operadora.")
        c.font = s.font_body
        c.fill = s.fill_background
        c.alignment = Alignment(horizontal="left", vertical="center",
                                wrap_text=True)

        ws.merge_cells(start_row=6, start_column=2, end_row=6, end_column=12)
        c = ws.cell(row=6, column=2,
                    value="Preencher com os dados técnicos dos circuitos "
                          "de interligação.")
        c.font = Font(name="Calibri", size=9, italic=True)
        c.fill = s.fill_background
        c.alignment = Alignment(horizontal="left", vertical="center",
                                wrap_text=True)

        # Cabeçalho
        hr = 8
        headers = [
            (2, "Ref"), (3, "Cn"), (4, "ID"), (5, "MTL"),
            (6, "PONTA VIVO"), (7, f"PONTA {n.upper()}"),
            (8, "DESIGNADOR DO CIRCUITO"), (9, "PROVEDOR"),
            (10, "BGP VIVO"), (11, f"BGP {n.upper()}"),
            (12, "OBSERVAÇÕES"),
        ]
        ws.row_dimensions[hr].height = 25.0
        for col, txt in headers:
            c = ws.cell(row=hr, column=col, value=txt)
            c.font = s.font_subheader
            c.alignment = Alignment(horizontal="center", vertical="center",
                                    wrap_text=True)
            c.fill = s.fill_secondary
            c.border = s.box_border

        # ===== DADOS: 2 linhas por referência =====
        align_c = Alignment(horizontal="center", vertical="center",
                            wrap_text=True)
        align_l = Alignment(horizontal="left", vertical="center",
                            wrap_text=True)

        # Determinar número de referências (maior entre vivo e op)
        num_refs = max(len(self.vivo_rows), len(self.op_rows), 1)
        ds = hr + 1
        row_idx = 0

        for ref_i in range(num_refs):
            vr = self.vivo_rows[ref_i] if ref_i < len(self.vivo_rows) else {}
            opr = self.op_rows[ref_i] if ref_i < len(self.op_rows) else {}

            # Extrair dados
            cn_vivo = vr.get("cn", "").strip()
            cn_op = opr.get("cn", "").strip()
            endereco_vivo = vr.get("endereco_link", "").strip()
            endereco_op = opr.get("endereco_link", "").strip()
            ref_val = ref_i + 1

            # --- Linha 1: MTL = Operadora ---
            r1 = ds + row_idx
            ws.row_dimensions[r1].height = 22.0
            f1 = s.alt_fill(row_idx)
            line1 = {
                3: cn_vivo, 4: "", 5: n.upper(),
                6: endereco_vivo, 7: endereco_op,
                8: "", 9: "", 10: "", 11: "", 12: "",
            }
            for col_num, val in line1.items():
                c = ws.cell(row=r1, column=col_num, value=val)
                c.font = s.font_small
                c.alignment = align_l if col_num in (6, 7, 12) else align_c
                c.border = s.box_border
                c.fill = f1
            row_idx += 1

            # --- Linha 2: MTL = VIVO ---
            r2 = ds + row_idx
            ws.row_dimensions[r2].height = 22.0
            f2 = s.alt_fill(row_idx)
            line2 = {
                3: cn_op or cn_vivo, 4: "", 5: "VIVO",
                6: endereco_vivo, 7: endereco_op,
                8: "", 9: "", 10: "", 11: "", 12: "",
            }
            for col_num, val in line2.items():
                c = ws.cell(row=r2, column=col_num, value=val)
                c.font = s.font_small
                c.alignment = align_l if col_num in (6, 7, 12) else align_c
                c.border = s.box_border
                c.fill = f2
            row_idx += 1

            # --- Merge vertical na coluna Ref (col 2) cobrindo as 2 linhas ---
            ws.merge_cells(start_row=r1, start_column=2,
                           end_row=r2, end_column=2)
            c = ws.cell(row=r1, column=2, value=ref_val)
            c.font = s.font_small
            c.alignment = align_c
            c.border = s.box_border
            c.fill = f1

        ws.freeze_panes = f"B{ds}"
        self._ps(ws, ls=True)

    def _b_params(self):
        s=self.s; ws=self.wb.create_sheet(title="Parâmetros de Programação"); n=self.nome
        self._cw(ws,{1:2,2:50,3:35,4:15,5:12,6:35}); self._bh(ws,2,2,6); self._st(ws,4,2,6,"2.9 - Parâmetros de Programação")
        ws.merge_cells(start_row=5,start_column=2,end_row=5,end_column=6)
        ws.cell(row=5,column=2,value=f"Relação dos parâmetros de configuração de rotas SIP entre a TELEFÔNICA-VIVO e a Operadora {n.upper()}.").font=s.font_body; ws.cell(row=5,column=2).fill=s.fill_background
        cur=[7]
        def _h(t):
            r=cur[0]; ws.merge_cells(start_row=r,start_column=2,end_row=r,end_column=6); c=ws.cell(row=r,column=2,value=t); c.font=Font(name="Calibri",size=11,bold=True,color="FFFFFF"); c.alignment=s.align_center; c.fill=s.fill_light; c.border=s.box_border; ws.row_dimensions[r].height=20.0; cur[0]+=1
        def _ch():
            r=cur[0]
            for ci,t in enumerate(["Parâmetro",f"TELEFÔNICA VIVO <> {n.upper()}","Categoria","Cumpre?","Observação"],2):
                c=ws.cell(row=r,column=ci,value=t); c.font=s.font_subheader; c.alignment=s.align_center; c.fill=s.fill_secondary; c.border=s.box_border
            ws.row_dimensions[r].height=25.0; cur[0]+=1
        def _r(data):
            for i,rd in enumerate(data):
                r=cur[0]; f=s.alt_fill(i)
                for ci,v in enumerate(rd,2):
                    c=ws.cell(row=r,column=ci,value=v); c.font=s.font_check if ci==5 and v=="✔" else s.font_small
                    c.alignment=s.align_left if ci==2 else s.align_center; c.fill=f; c.border=s.box_border
                cur[0]+=1
        def _sec(title,data,ch=False):
            _h(title)
            if ch: _ch()
            _r(data); cur[0]+=1

        ws.merge_cells(start_row=cur[0],start_column=2,end_row=cur[0],end_column=6)
        c=ws.cell(row=cur[0],column=2,value="Interconexão Nacional"); c.font=Font(name="Calibri",size=11,bold=True,color="FFFFFF"); c.alignment=s.align_center; c.fill=s.fill_primary; c.border=s.box_border; cur[0]+=1
        _sec("Informação do Gateway",[("Fabricante / Fornecedor","sipwise","Mandatório","✔",""),("Modelo - Versão","Elementos Rede Fixa, Fixa I e Móvel","Mandatório","✔","")],ch=True)
        _sec("Tipo de Serviço",[("Voz","SIM","Mandatório","✔",""),("Fax","SIM","Mandatório","✔",""),("DTMF","SIM","Mandatório","✔","")])
        _sec("Protocolo",[("Tipo de Protocolo","SIP-I (Q.1912.5)","Mandatório","✔","")])
        _sec("Atributos SIP",[("SIP Version","SIP v2.1","Mandatório","✔",""),("Protocolo de Transporte","UDP","Mandatório","✔",""),("Endereço IP Gateway SIP","Gateway de Gateway - NNI","Mandatório","✔",""),("IP ADDR do RTP","IP ADDR do RTP","Mandatório","✔",""),("FW ou SBC antes do Gateway","Mandatório","Mandatório","✔",""),("Envia DOMAIN na URI","P-Asserted-Identity e Diversion","Mandatório","✔",""),("Envia DOMAIN IP","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP para Controle","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP para Controle de QoS","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Chamada","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Sessão","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Recursos","Mandatório","Mandatório","✔",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Política","Mandatório","Mandatório","✔","")])
        _sec("Codec Rede Móvel",[("AMR encaminhamento","","Mandatório","✔",""),("2ª opção G.711a 20ms","","Mandatório","✔","Não aceita codec g.711u")])
        _sec("Codec Rede Fixa",[("G.711a 20ms","","Mandatório","✔",""),("2ª opção G.729a 20ms","","Mandatório","✔","Não aceita codec g.711u")])
        _sec("DTMF",[("1ª Opção: RFC 2833 (Outband) com valor payload = 100","","Mandatório","✔",""),("Todos os payloads de DTMF definidos no padrão (96-125) podem ser usados","","Mandatório","✔","")])
        _sec("FAX",[("Protocolo T.38 suportado","","Mandatório","✔","")])
        _sec("POS",[("Detecção do UPSEEED","Utilização do g711a em re-INVITE","Mandatório","✔","Dados necessários no SDP"),("Detecção do UPSEEED","Utilização do g711u em re-INVITE","Mandatório","✔","Dados necessários no SDP")])
        _sec("Encaminhamento Enfrante",[("P-ASSERTED Identity (RFC 3325)","SIM - Fomento E.164","Mandatório","✔",""),("Nature of address of Calling","SUB, NAT, UNKW","Mandatório","✔",""),("B-NUMBER (Called Number)","E.164 - Padrão Roteamento","Mandatório","✔",""),("Nature of address","SUB, NAT, UNKW","Mandatório","✔","")])
        _sec("Encaminhamento Sainte",[("A-NUMBER (Calling Number)","E.164 - Nacional","Mandatório","✔",""),("P-Asserted Identity (RFC 3325)","SIM - Formato E.164","Mandatório","✔",""),("Nature of address of Calling","SUB, NAT, UNKW","Mandatório","✔",""),("B-NUMBER (Called Number)","E.164 - Padrão Roteamento","Mandatório","✔",""),("Nature of address","SUB, NAT, UNKW","Mandatório","✔","")])
        _sec("Principais RFC's que devem estar habilitadas e/ou suportadas",[("3261 - SIP Base","","","",""),("3311 - UPDATE (PRACK)","","","",""),("3264/4028 - Offer/Answer - SDP","","","",""),("2833 - DTMF - RTP","","","",""),("RFC 3398 - SIP-I interworking","","","",""),("RFC 4033/4035 - DNSSEC","","","",""),("RFC 4566 - SDP","","","",""),("RFC 3606 - Early Media & Tone Generation","","","","SIP 180Ring (puro), o Proxy deverá gerar o RBT localmente")])
        _sec("Método para alteração dos dados do SDP",[("Método Update","","Mandatório","","Antes do Atendimento"),("Método Update","","Mandatório","","Após atendimento")])
        _sec("Negociação de Codec",[("É mandatório a utilização de re-invites pelo Origem para definição de codecs.","","Mandatório","✔",""),("O ptime no SDP answer deve ser múltiplo inteiro de 20 para codecs móveis; 3GPP 26.114","","Mandatório","✔",""),("Pacotes suportados às marcações USR, DSCP 46 ou EF para pacotes RTP","","Mandatório","✔","")])
        self._ps(ws,ls=True)

    def build(self):
        """Constrói o Workbook completo com todas as abas do PTI."""
        logger.info(f"Gerando PTI Excel para '{self.nome}'...")
        self._b_index(); self._b_versions(); self._b_diagram(); self._b_routing()
        self._b_prefixes(); self._b_mtl(); self._b_params()
        self.wb.properties.title=f"PTI — Completo — {self.nome}"
        self.wb.properties.subject="Projeto Técnico de Interligação"
        self.wb.properties.creator="VIVOHUB"
        self.wb.active=0
        logger.info("PTI Excel gerado com sucesso.")
        return self.wb


# =============================================================================
# REGISTRO DE ROTAS HTTP
# =============================================================================
def _register_routes(app):

    @app.get("/")
    def index():
        if "user_id" in session: return redirect(url_for(f"central_{session['role']}"))
        return redirect(url_for("login"))

    @app.get("/login")
    def login(): return render_template("login.html")

    @app.post("/login")
    def login_post():
        email=(request.form.get("email") or "").strip().lower(); password=request.form.get("password","")
        db=get_db(); user=db.execute("SELECT * FROM users WHERE email = ?",(email,)).fetchone()
        if not user or not check_password_hash(user["password_hash"],password):
            flash("Credenciais inválidas.","danger"); return redirect(url_for("login"))
        session.clear(); session["user_id"]=user["id"]; session["email"]=user["email"]; session["role"]=user["role"]
        return redirect(url_for(f"central_{user['role']}"))

    @app.get("/logout")
    def logout(): session.clear(); flash("Sessão encerrada.","info"); return redirect(url_for("login"))

    @app.get("/admin_login")
    def admin_login(): return render_template("admin_login.html")

    @app.post("/admin_login")
    def admin_login_post():
        if request.form.get("code","")==app.config["ADMIN_CODE"]:
            session["is_admin"]=True; flash("Login de administrador efetuado.","success"); return redirect(url_for("register"))
        flash("Código de administrador inválido.","danger"); return redirect(url_for("admin_login"))

    @app.get("/register")
    @admin_required
    def register(): return render_template("register.html")

    @app.post("/register")
    @admin_required
    def register_post():
        email=(request.form.get("email") or "").strip().lower(); password=request.form.get("password",""); role=(request.form.get("role") or "").strip().lower()
        if not email or not password or role not in ("engenharia","atacado"):
            flash("Preencha todos os campos corretamente.","danger"); return redirect(url_for("register"))
        db=get_db()
        try:
            db.execute("INSERT INTO users (email, password_hash, role) VALUES (?, ?, ?)",(email,generate_password_hash(password),role)); db.commit()
            flash("Usuário criado com sucesso.","success")
        except sqlite3.IntegrityError: flash("Este e-mail já está cadastrado.","warning")
        return redirect(url_for("register"))

    @app.get("/central_atacado")
    @login_required
    @role_required("atacado")
    def central_atacado(): return render_template("central_atacado.html")

    @app.get("/central_engenharia")
    @login_required
    @role_required("engenharia")
    def central_engenharia(): return render_template("central_engenharia.html")

    @app.get("/atacado_formularios")
    @login_required
    @role_required("atacado")
    def atacado_form_list():
        db=get_db(); q=(request.args.get("q") or "").strip(); status=(request.args.get("status") or "").strip(); sort=(request.args.get("sort") or "-created_at").strip()
        sql,params=build_list_query("SELECT * FROM atacado_forms",search_term=q or None,status_filter=status or None,owner_id=session["user_id"],sort_key=sort)
        return render_template("atacado_formularios.html",forms=db.execute(sql,params).fetchall(),counters=get_status_counters(db,"WHERE owner_id = ?",(session["user_id"],)))

    @app.get("/atacado_formularios/new")
    @login_required
    @role_required("atacado")
    def atacado_form_new():
        db=get_db(); last=db.execute("SELECT responsavel_atacado FROM atacado_forms WHERE owner_id = ? AND COALESCE(responsavel_atacado,'')<>'' ORDER BY created_at DESC LIMIT 1",(session["user_id"],)).fetchone()
        preset=(last["responsavel_atacado"] if last else "") or session.get("email","").split("@")[0].replace("."," ").title()
        return render_template("formulario_atacado.html",form=None,preset_responsavel_atacado=preset)

    @app.post("/atacado_formularios/new")
    @login_required
    @role_required("atacado")
    def atacado_form_create():
        payload=extract_form_payload(); payload["owner_id"]=session["user_id"]; payload["status"]=(payload.get("status") or "rascunho").lower(); payload["engenharia_params_json"]="{}"
        truncated=validate_table_rows(payload)
        if payload["status"]=="enviado" and not (payload.get("responsavel_atacado") or "").strip():
            flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.',"warning"); return redirect(url_for("atacado_form_new"))
        cols=["owner_id"]+list(TEXT_FIELDS)+list(BOOLEAN_FIELDS)+["escopo_flags_json","dados_vivo_json","dados_operadora_json","engenharia_params_json"]
        db=get_db(); db.execute(f"INSERT INTO atacado_forms ({','.join(cols)}) VALUES ({','.join(['?']*len(cols))})",[payload.get(c) for c in cols]); db.commit()
        flash("Formulário criado." + (f" Tabelas limitadas a {MAX_TABLE_ROWS} linhas." if truncated else ""),"success"); return redirect(url_for("atacado_form_list"))

    @app.get("/atacado_formularios/<int:form_id>")
    @login_required
    @role_required("atacado")
    def atacado_form_edit(form_id):
        db=get_db(); form=db.execute("SELECT * FROM atacado_forms WHERE id = ? AND owner_id = ?",(form_id,session["user_id"])).fetchone()
        if not form: abort(404)
        return render_template("formulario_atacado.html",form=form)

    @app.post("/atacado_formularios/<int:form_id>")
    @login_required
    @role_required("atacado")
    def atacado_form_update(form_id):
        db=get_db()
        if not db.execute("SELECT id FROM atacado_forms WHERE id = ? AND owner_id = ?",(form_id,session["user_id"])).fetchone(): abort(404)
        payload=extract_form_payload(); payload["updated_at"]=datetime.utcnow().isoformat(timespec="seconds"); truncated=validate_table_rows(payload)
        if (payload.get("status") or "").lower()=="enviado" and not (payload.get("responsavel_atacado") or "").strip():
            flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.',"warning"); return redirect(url_for("atacado_form_edit",form_id=form_id))
        fields=list(TEXT_FIELDS)+list(BOOLEAN_FIELDS)+["escopo_flags_json","dados_vivo_json","dados_operadora_json"]
        params=[payload.get(f) for f in fields]+[payload["updated_at"],form_id,session["user_id"]]
        db.execute(f"UPDATE atacado_forms SET {', '.join([f'{f} = ?' for f in fields]+['updated_at = ?'])} WHERE id = ? AND owner_id = ?",params); db.commit()
        flash("Formulário atualizado." + (f" Tabelas limitadas a {MAX_TABLE_ROWS} linhas." if truncated else ""),"success"); return redirect(url_for("atacado_form_list"))

    @app.post("/atacado_formularios/<int:form_id>/delete")
    @login_required
    @role_required("atacado")
    def atacado_form_delete(form_id):
        db=get_db(); db.execute("DELETE FROM atacado_forms WHERE id = ? AND owner_id = ?",(form_id,session["user_id"])); db.commit()
        flash("Formulário excluído.","info"); return redirect(url_for("atacado_form_list"))

    @app.get("/engenharia_formularios")
    @login_required
    @role_required("engenharia")
    def engenharia_form_list():
        db=get_db(); q=(request.args.get("q") or "").strip(); status=(request.args.get("status") or "").strip(); sort=(request.args.get("sort") or "-created_at").strip()
        sql,params=build_list_query("SELECT * FROM atacado_forms",search_term=q or None,status_filter=status or None,sort_key=sort)
        forms=db.execute(sql,params).fetchall(); counters=get_status_counters(db)
        ff=request.args.get("form")
        if ff and ff.isdigit():
            exports=db.execute("SELECT e.id,e.form_id,e.filename,e.size_bytes,e.created_at,f.nome_operadora FROM exports e JOIN atacado_forms f ON f.id=e.form_id WHERE e.form_id=? ORDER BY e.created_at DESC LIMIT 100",(int(ff),)).fetchall()
        else:
            exports=db.execute("SELECT e.id,e.form_id,e.filename,e.size_bytes,e.created_at,f.nome_operadora FROM exports e JOIN atacado_forms f ON f.id=e.form_id ORDER BY e.created_at DESC LIMIT 100").fetchall()
        return render_template("engenharia_formularios.html",forms=forms,counters=counters,exports=exports,show_files=request.args.get("show_files")=="1",form_filter=ff)

    @app.route("/engenharia_formularios/<int:form_id>",methods=["GET","POST"])
    @login_required
    @role_required("engenharia")
    def engenharia_form_view(form_id):
        db=get_db(); form=db.execute("SELECT * FROM atacado_forms WHERE id = ?",(form_id,)).fetchone()
        if not form: abort(404)
        if request.method=="POST":
            resp_eng=(request.form.get("responsavel_engenharia") or "").strip()
            if not resp_eng:
                flash('Informe o "Responsável Eng de ITX" para salvar.',"warning"); return redirect(url_for("engenharia_form_view",form_id=form_id))
            eng_json=_parse_json_dict(request.form.get("engenharia_params_json","") or "{}")
            # FIX: Salvar também dados_vivo_json (antes era perdido no POST)
            vivo_json=truncate_json_list(request.form.get("dados_vivo_json","[]"), "[]")
            db.execute("UPDATE atacado_forms SET engenharia_params_json=?,responsavel_engenharia=?,dados_vivo_json=?,updated_at=? WHERE id=?",(eng_json,resp_eng,vivo_json,datetime.utcnow().isoformat(timespec="seconds"),form_id)); db.commit()
            flash("Validação da Engenharia salva.","success"); return redirect(url_for("engenharia_form_list",show_files="1",form=form_id))
        return render_template("formulario_atacado.html",form=form,readonly=True)

    @app.get("/formularios/<int:form_id>/excel_index")
    @login_required
    def exportar_form_excel_index(form_id):
        if not OPENPYXL_AVAILABLE:
            flash("openpyxl não instalado.","warning"); return redirect(url_for("index"))
        db=get_db(); form=db.execute("SELECT * FROM atacado_forms WHERE id = ?",(form_id,)).fetchone()
        if not form: abort(404)
        if session.get("role")=="atacado" and form["owner_id"]!=session.get("user_id"): abort(403)
        wb=PTIWorkbookBuilder(form, sbc_analyzer=app.config.get("SBC_ANALYZER")).build()
        buf=BytesIO(); wb.save(buf); buf.seek(0)
        nome=safe_filename(form["nome_operadora"] or "Operadora")
        return send_file(buf,mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",as_attachment=True,download_name=f"PTI_Completo_{nome}_ID{form['id']}.xlsx")

    @app.post("/engenharia_formularios/<int:form_id>/generate_excel")
    @login_required
    @role_required("engenharia")
    def engenharia_generate_excel(form_id):
        if not OPENPYXL_AVAILABLE:
            flash("openpyxl não instalado.","warning"); return redirect(url_for("engenharia_form_list"))
        db=get_db(); form=db.execute("SELECT id, nome_operadora FROM atacado_forms WHERE id = ?",(form_id,)).fetchone()
        if not form: abort(404)
        wb=Workbook(); wb.active.title="Índice"
        ts=datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        fname=f"PTI_Indice_{safe_filename(form['nome_operadora'] or 'Operadora')}_ID{form_id}_{ts}.xlsx"
        fpath=os.path.join(app.config["EXPORT_DIR"],fname)
        wb.save(fpath)
        db.execute("INSERT INTO exports (form_id, filename, filepath, size_bytes) VALUES (?, ?, ?, ?)",(form_id,fname,fpath,os.path.getsize(fpath))); db.commit()
        flash(f"Excel gerado: {fname}","success"); return redirect(url_for("engenharia_form_list",show_files="1",form=form_id))

    @app.get("/engenharia_exports/<int:export_id>/download")
    @login_required
    @role_required("engenharia")
    def engenharia_export_download(export_id):
        db=get_db(); row=db.execute("SELECT filename, filepath FROM exports WHERE id = ?",(export_id,)).fetchone()
        if not row: abort(404)
        if not os.path.exists(row["filepath"]):
            flash("Arquivo não encontrado.","warning"); return redirect(url_for("engenharia_form_list",show_files="1"))
        return send_file(row["filepath"],mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",as_attachment=True,download_name=row["filename"])

    @app.post("/engenharia_exports/<int:export_id>/delete")
    @login_required
    @role_required("engenharia")
    def engenharia_export_delete(export_id):
        db=get_db(); row=db.execute("SELECT filepath FROM exports WHERE id = ?",(export_id,)).fetchone()
        if not row: abort(404)
        try:
            if os.path.exists(row["filepath"]): os.remove(row["filepath"])
        except OSError: pass
        db.execute("DELETE FROM exports WHERE id = ?",(export_id,)); db.commit()
        flash("Arquivo removido.","info"); return redirect(url_for("engenharia_form_list",show_files="1"))

    # =====================================================================
    # API — SBC
    # =====================================================================
    @app.get("/api/sbc/suggest")
    @login_required
    @role_required("engenharia")
    def api_sbc_suggest():
        cn = request.args.get("cn", "").strip()
        uf = request.args.get("uf", "").strip().upper()
        analyzer: SBCAnalyzer = app.config["SBC_ANALYZER"]
        if cn:
            result = analyzer.suggest_for_cn(cn)
        elif uf:
            result = analyzer.suggest_for_uf(uf)
        else:
            return jsonify({"error": "Parâmetro 'cn' ou 'uf' é obrigatório."}), 400
        return jsonify(asdict(result) if hasattr(result, '__dataclass_fields__') else {
            "cn": result.cn, "uf": result.uf, "cidade": result.cidade,
            "regional": result.regional, "source_file": result.source_file,
            "source_modificado_em": result.source_modificado_em,
            "data_medicao": result.data_medicao, "total_sbcs": result.total_sbcs,
            "sbcs": result.sbcs, "fallback_usado": result.fallback_usado,
            "fallback_origem": result.fallback_origem, "mensagem": result.mensagem,
            "todos_criticos": result.todos_criticos,
            "total_disponiveis": result.total_disponiveis,
            "total_criticos": result.total_criticos,
        })

    @app.get("/api/sbc/overview")
    @login_required
    @role_required("engenharia")
    def api_sbc_overview():
        analyzer: SBCAnalyzer = app.config["SBC_ANALYZER"]
        return jsonify(analyzer.get_overview())

    @app.get("/api/sbc/health")
    @login_required
    @role_required("engenharia")
    def api_sbc_health():
        analyzer: SBCAnalyzer = app.config["SBC_ANALYZER"]
        return jsonify(analyzer.health_check())

    @app.get("/api/sbc/reload")
    @login_required
    @role_required("engenharia")
    def api_sbc_reload():
        analyzer: SBCAnalyzer = app.config["SBC_ANALYZER"]
        measurements, aggregated = analyzer.processor.get_data(force_reload=True)
        return jsonify({
            "status": "reloaded", "measurements": len(measurements),
            "aggregated_rows": len(aggregated),
            "file_info": analyzer.processor.get_file_info(),
        })

    @app.get("/api/cns")
    @login_required
    def api_cns():
        db=get_db(); rows=db.execute("SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf FROM cns WHERE ativo=1 ORDER BY codigo ASC").fetchall()
        return jsonify([{"codigo":r["codigo"],"nome":r["nome"],"uf":r["uf"]} for r in rows])


# =============================================================================
# CLI COMMANDS
# =============================================================================
def _register_cli_commands(app):
    @app.cli.command("create-user")
    @click.argument("email")
    @click.argument("password")
    @click.argument("role")
    def create_user_cmd(email, password, role):
        role=role.strip().lower()
        if role not in ("engenharia","atacado"):
            click.echo("Role inválida. Use: engenharia ou atacado."); return
        db=get_db()
        try:
            db.execute("INSERT INTO users (email, password_hash, role) VALUES (?, ?, ?)",(email.strip().lower(),generate_password_hash(password),role)); db.commit()
            click.echo("Usuário criado com sucesso.")
        except sqlite3.IntegrityError: click.echo("E-mail já cadastrado.")

    @app.cli.command("seed-cns")
    def seed_cns_cmd():
        db=get_db(); _seed_cns(db); _apply_cn_metadata(db)
        click.echo("Seed de CNs aplicado/atualizado.")

    @app.cli.command("sbc-check")
    def sbc_check_cmd():
        analyzer: SBCAnalyzer = app.config["SBC_ANALYZER"]
        health = analyzer.health_check()
        click.echo(f"Diretório: {health['data_dir']} ({'OK' if health['data_dir_exists'] else 'NÃO ENCONTRADO'})")
        click.echo(f"XLSX mais recente: {health.get('latest_file', 'Nenhum')}")
        if health['file_info'].get('filename'):
            info = health['file_info']
            click.echo(f"  Medições: {info['total_measurements']}")
            click.echo(f"  Linhas agregadas: {info['total_aggregated']}")
            click.echo(f"  Modificado: {info['modified_at']}")

    @app.cli.command("sbc-suggest")
    @click.argument("cn")
    def sbc_suggest_cmd(cn):
        analyzer: SBCAnalyzer = app.config["SBC_ANALYZER"]
        result = analyzer.suggest_for_cn(cn)
        click.echo(f"\n{'='*60}")
        click.echo(f"  SBC Suggestion para CN {result.cn} — {result.cidade} ({result.uf})")
        click.echo(f"  Regional: {result.regional}")
        click.echo(f"  Fonte: {result.source_file} ({result.source_modificado_em})")
        click.echo(f"  Medição: {result.data_medicao}")
        if result.fallback_usado:
            click.echo(f"  ⚠️  Fallback: {result.fallback_origem}")
        click.echo(f"{'='*60}")
        click.echo(f"  {result.mensagem}")
        click.echo(f"{'='*60}\n")
        if not result.sbcs:
            click.echo("  Nenhum SBC encontrado."); return
        for i, sbc in enumerate(result.sbcs):
            icon = "✅" if sbc.get("recomendado") else ("⚠️" if sbc.get("saude") in ("moderado","critico") else "  ")
            click.echo(f"  {icon} #{i+1} {sbc['nome']} — {sbc.get('cidade','')} ({sbc['uf']})")
            click.echo(f"     Modelo: {sbc['modelo']} | Serviços: {', '.join(sbc.get('servicos',[]))}")
            click.echo(f"     CAPS avg: {sbc['caps_avg']} | máx {sbc['caps_max']} | mín {sbc['caps_min']} | último {sbc['caps_ultimo']}")
            click.echo(f"     Tendência: {sbc['caps_tendencia']} | Medições: {sbc['total_medicoes']} | Status: {sbc['status_fonte']} | Saúde: {sbc['saude']}")
            click.echo(f"     Score: {sbc['score']}/100 {'🏆 RECOMENDADO' if sbc.get('recomendado') else ''}")
            if sbc.get('responsavel'): click.echo(f"     Responsável: {sbc['responsavel']}")
            if sbc.get('prazo'): click.echo(f"     Prazo: {sbc['prazo']}")
            click.echo(f"     Motivo: {sbc['motivo']}"); click.echo()


# =============================================================================
# ENTRYPOINT
# =============================================================================
app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
