# =============================================================================
# app.py — VIVOHUB: Sistema de Gestão de Projetos Técnicos de Interligação
# =============================================================================
#
# Aplicação Flask para gerenciar formulários PTI (Projeto Técnico de
# Interligação) entre operadoras de telecomunicações e a VIVO.
#
# Arquitetura:
#   ┌─────────────┐     ┌──────────────┐     ┌──────────┐     ┌──────────┐
#   │  Flask Web   │────▶│  SQLite DB   │     │  Excel   │     │  SBC CSV │
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
import csv
import json
import sqlite3
import click
import tempfile
import logging
import time
import glob
from pathlib import Path
from functools import wraps
from datetime import datetime
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

# Diretório dos CSVs de SBC (override via SBC_CSV_DIR env var)
DEFAULT_SBC_CSV_DIR = r"C:\Users\40418843\Desktop\SBC_Reports"

# Cache TTL para dados de SBC (5 minutos)
SBC_CACHE_TTL_SECONDS = 300

# Thresholds de saúde do SBC (baseado em CPU AVG %)
SBC_HEALTH_THRESHOLDS = {
    "disponivel": (0, 40),     # Verde — muita folga, ideal para uso
    "moderado":   (41, 60),    # Amarelo — aceitável
    "atencao":    (61, 80),    # Laranja — quente, evitar se possível
    "critico":    (81, 100),   # Vermelho — lotado, não recomendado
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
    """Uma única medição de um SBC (uma linha do CSV/XLSX)."""
    semana: str
    dia: str
    regional: str
    uf: str
    sbc: str
    modelo: str
    cpu_pct: float
    cpu_max: float
    cpu_avg: float
    status: str
    servico: str


@dataclass
class SBCAnalysisResult:
    """Resultado da análise de um SBC individual."""
    nome: str
    modelo: str
    servicos: list
    regional: str
    uf: str
    cpu_avg: float
    cpu_media: float
    cpu_max: int
    cpu_min: int
    cpu_ultima: float
    total_medicoes: int
    status_csv: str
    saude: str
    score: int
    recomendado: bool
    motivo: str
    dia_mais_recente: str
    semana: str


@dataclass
class SBCSuggestionResponse:
    """Resposta completa da API de sugestão de SBC."""
    cn: str
    uf: str
    cidade: str
    regional: str
    csv_source: str
    csv_modificado_em: str
    data_medicao: str
    total_sbcs: int
    sbcs: list
    fallback_usado: bool
    fallback_origem: str
    mensagem: str


# =============================================================================
# SBC — PROCESSADOR DE DADOS (XLSX / CSV)
# =============================================================================
class SBCDataProcessor:
    """
    Lê, limpa e cacheia dados de SBC a partir de planilhas XLSX ou CSVs.
    """

    COLUMN_MAP = {
        "SEMANA": "semana", "ANO SEMANA": "semana",
        "DIA": "dia", "REGIONAL": "regional", "UF": "uf",
        "SBC": "sbc", "MODELO": "modelo",
        "% CPU": "cpu_pct", "%CPU": "cpu_pct", "CPU": "cpu_pct",
        "CPU MAX": "cpu_max",
        "CPU AVG": "cpu_avg", "CPU AVERAGE": "cpu_avg",
        "CPU MEDIA": "cpu_avg", "CPU MEDIO": "cpu_avg", "% CPU AVG": "cpu_avg",
        "ST": "status", "STATUS": "status",
        "SERVIÇO": "servico", "SERVICO": "servico",
    }

    def __init__(self, csv_dir: str, cache_ttl: int = SBC_CACHE_TTL_SECONDS):
        self.csv_dir = csv_dir
        self.cache_ttl = cache_ttl
        self._cache_data: list[SBCMeasurement] = []
        self._cache_aggregated: list[dict] = []
        self._cache_timestamp: float = 0.0
        self._cache_file_path: str = ""
        self._cache_file_mtime: float = 0.0

    def find_latest_file(self) -> Optional[str]:
        if not os.path.isdir(self.csv_dir):
            logger.warning(f"Diretório de SBC não encontrado: {self.csv_dir}")
            return None
        data_files = []
        for pattern in ("*.xlsx", "*.XLSX", "*.xls", "*.XLS", "*.csv", "*.CSV"):
            data_files.extend(glob.glob(os.path.join(self.csv_dir, pattern)))
        if not data_files:
            logger.warning(f"Nenhum arquivo de dados encontrado em: {self.csv_dir}")
            return None
        data_files.sort(key=lambda f: os.path.getmtime(f), reverse=True)
        latest = data_files[0]
        logger.info(f"Arquivo SBC mais recente: {os.path.basename(latest)} "
                     f"({len(data_files)} arquivos no diretório)")
        return latest

    def _normalize_column_name(self, raw_name: str) -> str:
        clean = raw_name.strip().upper()
        return self.COLUMN_MAP.get(clean, clean.lower().replace(" ", "_"))

    def _parse_float(self, value: str, default: float = 0.0) -> float:
        try:
            cleaned = str(value).strip().replace(",", ".")
            return float(cleaned) if cleaned else default
        except (ValueError, TypeError):
            return default

    def _read_file(self, filepath: str) -> tuple[list[SBCMeasurement], list[dict]]:
        ext = os.path.splitext(filepath)[1].lower()
        if ext in (".xlsx", ".xls"):
            return self._read_xlsx(filepath)
        return self._read_csv_file(filepath)

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
                        regional=normalized_row.get("regional", "").strip().upper(),
                        uf=uf, sbc=sbc_name,
                        modelo=normalized_row.get("modelo", ""),
                        cpu_pct=self._parse_float(normalized_row.get("cpu_pct", "0")),
                        cpu_max=self._parse_float(normalized_row.get("cpu_max", "0")),
                        cpu_avg=self._parse_float(normalized_row.get("cpu_avg", "0")),
                        status=normalized_row.get("status", "Normal").strip(),
                        servico=normalized_row.get("servico", "").strip(),
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

    def _read_csv_file(self, filepath: str) -> tuple[list[SBCMeasurement], list[dict]]:
        measurements = []
        aggregated = []
        try:
            with open(filepath, "r", encoding="utf-8") as f:
                reader = csv.DictReader(f, delimiter=",")
                if not reader.fieldnames:
                    logger.error(f"CSV sem cabeçalho: {filepath}")
                    return [], []
                col_mapping = {}
                for raw_col in reader.fieldnames:
                    col_mapping[raw_col] = self._normalize_column_name(raw_col)
                for row_num, row in enumerate(reader, start=2):
                    normalized_row = {}
                    for raw_key, value in row.items():
                        norm_key = col_mapping.get(raw_key, raw_key)
                        normalized_row[norm_key] = (value or "").strip()
                    sbc_name = normalized_row.get("sbc", "").strip()
                    uf = normalized_row.get("uf", "").strip().upper()
                    if sbc_name and uf:
                        measurement = SBCMeasurement(
                            semana=normalized_row.get("semana", ""),
                            dia=normalized_row.get("dia", ""),
                            regional=normalized_row.get("regional", "").strip().upper(),
                            uf=uf, sbc=sbc_name,
                            modelo=normalized_row.get("modelo", ""),
                            cpu_pct=self._parse_float(normalized_row.get("cpu_pct", "0")),
                            cpu_max=self._parse_float(normalized_row.get("cpu_max", "0")),
                            cpu_avg=self._parse_float(normalized_row.get("cpu_avg", "0")),
                            status=normalized_row.get("status", "Normal").strip(),
                            servico=normalized_row.get("servico", "").strip(),
                        )
                        measurements.append(measurement)
                    else:
                        aggregated.append({
                            "row_num": row_num, "raw_data": normalized_row,
                            "nota": "Linha sem SBC/UF identificado (dado agregado ou total)",
                        })
            logger.info(f"CSV lido: {len(measurements)} medições, {len(aggregated)} agregadas")
        except FileNotFoundError:
            logger.error(f"CSV não encontrado: {filepath}")
        except UnicodeDecodeError:
            logger.error(f"Erro de encoding no CSV: {filepath}. Esperado UTF-8.")
        except Exception as e:
            logger.error(f"Erro ao ler CSV {filepath}: {e}")
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

    def get_csv_info(self) -> dict:
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
    """Motor de análise e recomendação de SBCs."""

    STATUS_SCORES = {
        "Normal": 0, "Atenção": -15, "Atencao": -15,
        "Crítico": -30, "Critico": -30,
    }

    def __init__(self, processor: SBCDataProcessor):
        self.processor = processor

    def _classify_health(self, cpu_avg: float) -> str:
        for level, (low, high) in SBC_HEALTH_THRESHOLDS.items():
            if low <= cpu_avg <= high:
                return level
        return "critico" if cpu_avg > 100 else "disponivel"

    def _worst_status(self, statuses: list[str]) -> str:
        severity = {"Normal": 0, "Atenção": 1, "Atencao": 1, "Crítico": 2, "Critico": 2}
        worst = "Normal"
        worst_score = 0
        for s in statuses:
            s_score = severity.get(s.strip(), 0)
            if s_score > worst_score:
                worst_score = s_score
                worst = s.strip()
        return worst

    def _calculate_score(self, cpu_avg, cpu_max, status, num_servicos, total_medicoes):
        base = max(0, 100 - cpu_avg)
        bonus = 0
        if status == "Normal": bonus += 5
        if num_servicos > 1: bonus += 3
        if total_medicoes >= 5: bonus += 2
        penalty = abs(self.STATUS_SCORES.get(status, 0))
        if cpu_max > 85: penalty += 5
        return max(0, min(100, int(base + bonus - penalty)))

    def _generate_reason(self, result: SBCAnalysisResult) -> str:
        parts = []
        labels = {"disponivel": "capacidade ampla", "moderado": "ocupação moderada",
                  "atencao": "ocupação alta, monitorar", "critico": "capacidade crítica"}
        parts.append(f"CPU avg {result.cpu_avg}% — {labels.get(result.saude, '')}")
        if result.total_medicoes > 1:
            parts.append(f"{result.total_medicoes} medições analisadas")
        if len(result.servicos) > 1:
            parts.append(f"Serviços: {', '.join(result.servicos)}")
        if result.status_csv != "Normal":
            parts.append(f"Status CSV: {result.status_csv}")
        return " | ".join(parts)

    def _aggregate_sbc_measurements(self, measurements):
        groups: dict[str, list[SBCMeasurement]] = defaultdict(list)
        for m in measurements:
            groups[m.sbc].append(m)
        results = []
        for sbc_name, sbc_measurements in groups.items():
            cpu_avg_values = [m.cpu_avg for m in sbc_measurements]
            cpu_pct_values = [m.cpu_pct for m in sbc_measurements]
            cpu_max_values = [m.cpu_max for m in sbc_measurements]
            has_avg_data = any(v > 0 for v in cpu_avg_values)
            primary_values = cpu_avg_values if has_avg_data else cpu_pct_values
            cpu_avg_media = sum(primary_values) / len(primary_values) if primary_values else 0
            cpu_pct_media = sum(cpu_pct_values) / len(cpu_pct_values) if cpu_pct_values else 0
            cpu_max = max(cpu_max_values) if cpu_max_values else 0
            cpu_min = min(primary_values) if primary_values else 0
            cpu_ultima = primary_values[-1] if primary_values else 0
            all_statuses = [m.status for m in sbc_measurements]
            worst_status = self._worst_status(all_statuses)
            servicos = list(dict.fromkeys(m.servico for m in sbc_measurements if m.servico))
            modelo = sbc_measurements[0].modelo
            regional = sbc_measurements[0].regional
            uf = sbc_measurements[0].uf
            semana = sbc_measurements[0].semana
            dias = [m.dia for m in sbc_measurements if m.dia]
            dia_recente = dias[-1] if dias else ""
            saude = self._classify_health(cpu_avg_media)
            score = self._calculate_score(cpu_avg_media, cpu_max, worst_status, len(servicos), len(sbc_measurements))
            result = SBCAnalysisResult(
                nome=sbc_name, modelo=modelo, servicos=servicos,
                regional=regional, uf=uf,
                cpu_avg=round(cpu_avg_media, 1), cpu_media=round(cpu_pct_media, 1),
                cpu_max=int(cpu_max), cpu_min=int(cpu_min),
                cpu_ultima=round(cpu_ultima, 1), total_medicoes=len(sbc_measurements),
                status_csv=worst_status, saude=saude, score=score,
                recomendado=False, motivo="",
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
        cn = str(cn).strip().zfill(2)
        meta = self._resolve_uf_from_cn(cn)
        if not meta:
            return SBCSuggestionResponse(
                cn=cn, uf="", cidade="", regional="",
                csv_source="", csv_modificado_em="", data_medicao="",
                total_sbcs=0, sbcs=[], fallback_usado=False, fallback_origem="",
                mensagem=f"CN {cn} não encontrado no cadastro de códigos nacionais.",
            )
        cidade, uf = meta
        regional = UF_TO_REGIONAL.get(uf, "")
        measurements, _ = self.processor.get_data()
        csv_info = self.processor.get_csv_info()
        if not measurements:
            return SBCSuggestionResponse(
                cn=cn, uf=uf, cidade=cidade, regional=regional,
                csv_source=csv_info.get("filename", ""),
                csv_modificado_em=csv_info.get("modified_at", ""),
                data_medicao="", total_sbcs=0, sbcs=[],
                fallback_usado=False, fallback_origem="",
                mensagem="Nenhum dado de SBC disponível. Verifique o diretório de CSVs.",
            )
        uf_measurements = [m for m in measurements if m.uf == uf]
        fallback_usado = False
        fallback_origem = ""
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
                cn=cn, uf=uf, cidade=cidade, regional=regional,
                csv_source=csv_info.get("filename", ""),
                csv_modificado_em=csv_info.get("modified_at", ""),
                data_medicao="", total_sbcs=0, sbcs=[],
                fallback_usado=False, fallback_origem="",
                mensagem=f"Nenhum SBC encontrado para {uf} ({cidade}), vizinhos ou regional {regional}.",
            )
        results = self._aggregate_sbc_measurements(uf_measurements)
        data_medicao = results[0].dia_mais_recente if results else ""
        msg_parts = [f"{len(results)} SBC(s) encontrado(s) para {uf} ({cidade})"]
        if fallback_usado:
            msg_parts.append(f"Usando dados de {fallback_origem}")
        return SBCSuggestionResponse(
            cn=cn, uf=uf, cidade=cidade, regional=regional,
            csv_source=csv_info.get("filename", ""),
            csv_modificado_em=csv_info.get("modified_at", ""),
            data_medicao=data_medicao,
            total_sbcs=len(results),
            sbcs=[asdict(r) for r in results],
            fallback_usado=fallback_usado,
            fallback_origem=fallback_origem,
            mensagem=". ".join(msg_parts),
        )

    def get_overview(self) -> dict:
        measurements, aggregated = self.processor.get_data()
        csv_info = self.processor.get_csv_info()
        if not measurements:
            return {"csv_info": csv_info, "total_sbcs": 0, "por_regional": {},
                    "por_saude": {}, "por_status_csv": {}, "dados_agregados_csv": len(aggregated)}
        all_results = self._aggregate_sbc_measurements(measurements)
        por_regional = defaultdict(int)
        por_saude = defaultdict(int)
        por_status = defaultdict(int)
        for r in all_results:
            por_regional[r.regional] += 1
            por_saude[r.saude] += 1
            por_status[r.status_csv] += 1
        return {
            "csv_info": csv_info, "total_sbcs": len(all_results),
            "por_regional": dict(por_regional), "por_saude": dict(por_saude),
            "por_status_csv": dict(por_status), "dados_agregados_csv": len(aggregated),
            "sbcs": [asdict(r) for r in all_results],
        }

    def health_check(self) -> dict:
        csv_info = self.processor.get_csv_info()
        csv_path = self.processor.find_latest_file()
        return {
            "csv_dir_exists": os.path.isdir(self.processor.csv_dir),
            "csv_dir": self.processor.csv_dir,
            "latest_csv": os.path.basename(csv_path) if csv_path else None,
            "csv_info": csv_info,
            "status": "ok" if csv_path else "no_csv_found",
        }


# =============================================================================
# FLASK APP FACTORY
# =============================================================================
def create_app() -> Flask:
    app = Flask(__name__, instance_relative_config=True)
    app.config["SECRET_KEY"] = os.environ.get("SECRET_KEY", "dev-secret-change-me")
    app.config["DATABASE"] = os.path.join(app.instance_path, "vivohub.db")
    app.config["ADMIN_CODE"] = os.environ.get("ADMIN_CODE", DEFAULT_ADMIN_CODE)
    app.config["SBC_CSV_DIR"] = os.environ.get("SBC_CSV_DIR", DEFAULT_SBC_CSV_DIR)
    Path(app.instance_path).mkdir(parents=True, exist_ok=True)
    export_dir = os.path.join(app.instance_path, "exports")
    Path(export_dir).mkdir(parents=True, exist_ok=True)
    app.config["EXPORT_DIR"] = export_dir
    sbc_processor = SBCDataProcessor(app.config["SBC_CSV_DIR"])
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
        self.font_check = Font(name="Calibri", size=11, bold=True, color="FF0000")
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

    def __init__(self, form_row):
        if not OPENPYXL_AVAILABLE: raise RuntimeError("openpyxl não instalado.")
        if not PIL_AVAILABLE: raise RuntimeError("Pillow não instalado.")
        self.form = form_row
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

        # CNs e Áreas Locais únicos (derivados de dados_vivo_json)
        self.cns_unicos = self._unique_field(self.vivo_rows, "cn")
        self.areas_locais = self._unique_field(self.vivo_rows, "cidade")

        # Se não tem CNs no vivo, tenta pegar do operadora
        if not self.cns_unicos:
            self.cns_unicos = self._unique_field(self.op_rows, "cn")
        if not self.areas_locais:
            self.areas_locais = self._unique_field(self.op_rows, "cidade")

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
        raw = row_get(self.form, "dados_operadora_json", "[]")
        try:
            d = json.loads(raw) if isinstance(raw, str) else raw
            if not isinstance(d, list): return []
            r = []
            for i in d:
                if not isinstance(i, dict): continue
                c = {k: str(i.get(k,"")).strip() for k in ("ref","cn","cnl","cod_cni","status_link")}
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
                c.alignment=s.align_wrap if col in (7,9) else (s.align_left if col in (5,6) else s.align_center)
        ws.freeze_panes="C7"; self._ps(ws,ls=True)

    # =====================================================================
    # ABA: DIAGRAMA DE INTERLIGAÇÃO
    # FIX: Tráfego (flags de escopo) e Mask preenchidos corretamente
    #      em AMBAS as pontas (A=VIVO, B=Operadora).
    # =====================================================================
    def _b_diagram(self):
        s=self.s; ws=self.wb.create_sheet(title="Diagrama de Interligação"); cfg=self._diagram_cfg(); n=self.nome
        num_blocks = max(1, len(self.vivo_rows))
        block_width = 12
        total_cols = 2 + (num_blocks * block_width)
        col_widths = {1: 2.0}
        for b in range(num_blocks):
            base = 3 + (b * block_width)
            for i in range(block_width):
                col_widths[base + i] = 12.0
        col_widths[total_cols + 1] = 2.0
        self._cw(ws, col_widths)

        last_col = 2 + (num_blocks * block_width)
        self._bh(ws, 2, 3, last_col)
        for r_row, t, b in [
            (4, f"Diagrama de Interligação entre a VIVO e a {n}", True),
            (6, "2.2.1 DIAGRAMAÇÃO DO PROJETO SIP", True),
            (7, "2.2.1.1 Anúncio de Redes pelos 2 Links SPC", True),
            (8, f"A {n} abordará os endereços da VIVO conforme abaixo.", False),
            (9, f"A VIVO abordará os endereços da {n} conforme abaixo.", False),
            (11, "2.2.1.2 Parâmetros de Configuração do Link", True),
        ]:
            ws.merge_cells(start_row=r_row, start_column=3, end_row=r_row, end_column=last_col)
            c = ws.cell(row=r_row, column=3, value=t)
            c.font = Font(name="Calibri", size=10 if not b else 11, bold=b)
            c.alignment = s.align_left if b else s.align_wrap
            ws.row_dimensions[r_row].height = 22.0 if b else 25.0

        # ASN (global)
        for o, l, v in [(0, "ASN VIVO", "10429 (Público)"), (1, f"ASN {n}", self.asn_op)]:
            r_row = 12 + o
            ws.merge_cells(start_row=r_row, start_column=3, end_row=r_row, end_column=5)
            ws.cell(row=r_row, column=3, value=l).font = Font(name="Calibri", size=10, bold=True)
            ws.cell(row=r_row, column=3).alignment = s.align_center
            ws.merge_cells(start_row=r_row, start_column=6, end_row=r_row, end_column=8)
            ws.cell(row=r_row, column=6, value=v).font = s.font_body
            ws.cell(row=r_row, column=6).alignment = s.align_center

        # -----------------------------------------------------------------
        # FLAGS DE ESCOPO = itens de tráfego que preenchem ambas as pontas
        # FIX: Usa TODOS os flags (sem limite de 3). Se nenhum flag
        # selecionado, mantém pelo menos 3 linhas vazias.
        # -----------------------------------------------------------------
        traffic_items = [t for t in self.traffic if t]
        num_traffic = max(len(traffic_items), 3)

        # ===== BLOCOS DE DIAGRAMA (um por linha do formulário) =====
        for b_idx in range(num_blocks):
            base_col = 3 + (b_idx * block_width)
            vivo_row = self.vivo_rows[b_idx] if b_idx < len(self.vivo_rows) else {}
            op_row = self._find_matching_op_row(vivo_row.get("cn",""), b_idx)

            cn_val = vivo_row.get("cn", "")
            mask_val = vivo_row.get("mask", "")
            endereco_vivo = vivo_row.get("endereco_link", "")
            endereco_op = op_row.get("endereco_link", "") if op_row else ""
            faixa_ip_op = op_row.get("faixa_ip", "") if op_row else ""

            mid = base_col + 5
            end = base_col + block_width - 1

            # Linha 15: CN e VRF
            ws.merge_cells(start_row=15, start_column=base_col, end_row=15, end_column=base_col+2)
            c = ws.cell(row=15, column=base_col, value=f"CN {cn_val}" if cn_val else "CN")
            c.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c.alignment = s.align_center; c.fill = s.fill_light
            ws.merge_cells(start_row=15, start_column=base_col+3, end_row=15, end_column=end)
            c2 = ws.cell(row=15, column=base_col+3, value="VRF: __________________________")
            c2.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c2.alignment = s.align_center; c2.fill = s.fill_light

            # Linha 17-18: Ponta A / Ponta B
            for sc, ec, l, v in [
                (base_col, base_col+4, "Ponta A", "VIVO"),
                (mid+1, end, "Ponta B", n)
            ]:
                ws.merge_cells(start_row=17, start_column=sc, end_row=17, end_column=ec)
                ws.cell(row=17, column=sc, value=l).font = Font(name="Calibri", size=10, bold=True)
                ws.cell(row=17, column=sc).alignment = s.align_center
                ws.merge_cells(start_row=18, start_column=sc, end_row=18, end_column=ec)
                ws.cell(row=18, column=sc, value=v).font = Font(name="Calibri", size=10, bold=True)
                ws.cell(row=18, column=sc).alignment = s.align_center

            # Linha 20: Cabeçalhos Tráfego / Endereço IP / NET MASK (Ponta A e B)
            for lc in (base_col, mid+1):
                for i, h in enumerate(["Tráfego", "Endereço IP", "NET MASK"]):
                    c = ws.cell(row=20, column=lc+i, value=h)
                    c.font = s.font_subheader; c.alignment = s.align_center; c.fill = s.fill_light
            ws.row_dimensions[20].height = 25.0

            # ---------------------------------------------------------
            # Linhas 21+: DADOS DE TRÁFEGO
            #
            # FIX PRINCIPAL:
            #  - Coluna "Tráfego": recebe os flags de escopo selecionados
            #    (LC, LD15+CNG, etc.) em AMBAS as pontas.
            #  - Coluna "Endereço IP": IP VIVO na Ponta A (1ª linha),
            #    Faixa IP Operadora na Ponta B (1ª linha).
            #  - Coluna "NET MASK": preenchida em AMBAS as pontas (antes
            #    a Ponta B ficava sempre vazia).
            # ---------------------------------------------------------
            for j in range(num_traffic):
                r_row = 21 + j
                ws.row_dimensions[r_row].height = 22.0
                f = s.alt_fill(j)
                traf_val = traffic_items[j] if j < len(traffic_items) else ""

                # --- PONTA A (VIVO) ---
                # Tráfego: flag de escopo
                cell_a_traf = ws.cell(row=r_row, column=base_col, value=traf_val)
                cell_a_traf.fill = f; cell_a_traf.font = s.font_small; cell_a_traf.alignment = s.align_center

                # Endereço IP: apenas na primeira linha
                cell_a_ip = ws.cell(row=r_row, column=base_col+1, value=endereco_vivo if j == 0 else "")
                cell_a_ip.fill = f; cell_a_ip.font = s.font_small; cell_a_ip.alignment = s.align_center

                # NET MASK: em todas as linhas (mesmo subnet)
                cell_a_mask = ws.cell(row=r_row, column=base_col+2, value=mask_val if j == 0 else "")
                cell_a_mask.fill = f; cell_a_mask.font = s.font_small; cell_a_mask.alignment = s.align_center

                # --- PONTA B (OPERADORA) ---
                # Tráfego: MESMO flag de escopo que Ponta A
                cell_b_traf = ws.cell(row=r_row, column=mid+1, value=traf_val)
                cell_b_traf.fill = f; cell_b_traf.font = s.font_small; cell_b_traf.alignment = s.align_center

                # Endereço IP: faixa IP da operadora, apenas na primeira linha
                cell_b_ip = ws.cell(row=r_row, column=mid+2, value=faixa_ip_op if j == 0 else "")
                cell_b_ip.fill = f; cell_b_ip.font = s.font_small; cell_b_ip.alignment = s.align_center

                # NET MASK: FIX — antes era "" sempre. Agora preenche com
                # a mesma mask (interconexão usa mesmo subnet).
                cell_b_mask = ws.cell(row=r_row, column=mid+3, value=mask_val if j == 0 else "")
                cell_b_mask.fill = f; cell_b_mask.font = s.font_small; cell_b_mask.alignment = s.align_center

            # Linha de endereço abaixo da tabela
            end_row = 21 + num_traffic + 1
            for sc, ec, t in [
                (base_col, base_col+4, f"Endereço (VIVO): {endereco_vivo}"),
                (mid+1, end, f"Endereço ({n}): {endereco_op}")
            ]:
                ws.merge_cells(start_row=end_row, start_column=sc, end_row=end_row, end_column=ec)
                ws.cell(row=end_row, column=sc, value=t).font = s.font_small
                ws.cell(row=end_row, column=sc).alignment = s.align_left

        # Imagem do diagrama (abaixo de todos os blocos)
        img_row = 21 + num_traffic + 4
        if not os.path.exists(cfg["img"]): raise FileNotFoundError(f"Imagem não encontrada: {cfg['img']}")
        el=[{"text":"Roteador OP","xy_pct":cfg["r1"],"font_size_pct":cfg["rfp"],"stroke_width_pct":cfg["rsp"]},
            {"text":"Roteador OP","xy_pct":cfg["r2"],"font_size_pct":cfg["rfp"],"stroke_width_pct":cfg["rsp"]},
            {"text":"Link resp Vivo","xy_pct":cfg["lvxy"],"font_size_pct":cfg["lfsz"],"stroke_width_pct":0.006},
            {"text":f"Link resp {n}","xy_pct":cfg["loxy"],"font_size_pct":cfg["lfsz"],"stroke_width_pct":0.006},
            {"text":"RAC/RAV/HL4","xy_pct":(0.33,0.70),"font_size_pct":0.010,"stroke_width_pct":0.006},
            {"text":"RAC/RAV/HL4","xy_pct":(0.34,0.60),"font_size_pct":0.010,"stroke_width_pct":0.006}]
        ann=render_labels_on_image(image_path=cfg["img"],vivo_text="VIVO",operator_text=n,vivo_xy_pct=cfg["vxy"],operator_xy_pct=cfg["oxy"],font_path=cfg["fp"],font_size_pct=cfg["fsz"],stroke_width_pct=0.009,extra_labels=el)
        xi=XLImage(ann)
        try:
            with PILImage.open(ann) as src: ow,oh=src.size
        except: ow,oh=800,300
        fw,fh=fit_size_keep_aspect(ow,oh,cfg["bw"],cfg["bh"]); xi.width,xi.height=fw,fh
        ws.add_image(xi,f"{get_column_letter(3+cfg['aco'])}{img_row+cfg['aro']}")
        self._ps(ws,ls=True)

    def _find_matching_op_row(self, cn, index):
        """Busca linha da operadora que corresponde ao CN ou ao índice da linha VIVO."""
        if cn:
            for op in self.op_rows:
                if op.get("cn","") == cn:
                    return op
        if index < len(self.op_rows):
            return self.op_rows[index]
        return None

    # =====================================================================
    # ABA: ENCAMINHAMENTO (inalterada)
    # =====================================================================
    def _b_routing(self):
        s=self.s; ws=self.wb.create_sheet(title="Encaminhamento"); n=self.nome
        self._cw(ws,{1:2,2:15,3:10,4:15,5:10,6:18,7:15,8:18,9:12,10:8,11:8,12:8,13:8,14:8,15:8,16:12,17:18,18:12,19:12,20:15,21:12,22:10,23:12,24:10,25:2})
        self._bh(ws,2,2,24); self._st(ws,4,2,24,"2.3. CARACTERÍSTICAS DO PROJETO DE INTERLIGAÇÃO E DO PLANO DE ENCAMINHAMENTO")
        h1=6; ws.row_dimensions[h1].height=30.0
        for sc,ec,t,v in [(2,2,"ÁREA LOCAL",True),(3,6,"LOCALIZAÇÃO",False),(7,9,"DADOS DA ROTA",False),(10,11,"BANDA",False),(12,13,"CANAIS",False),(14,15,"CAPS",False),(16,16,"ATIVAÇÃO",True),(17,18,"ENCAMINHAMENTO",False),(19,20,"SINALIZAÇÃO",False),(21,24,"ENDEREÇO IP",False)]:
            if v: ws.merge_cells(start_row=h1,start_column=sc,end_row=h1+2,end_column=ec)
            else: ws.merge_cells(start_row=h1,start_column=sc,end_row=h1,end_column=ec)
            c=ws.cell(row=h1,column=sc,value=t); c.font=s.font_subheader; c.alignment=s.align_center_wrap; c.fill=s.fill_secondary if v else s.fill_light; c.border=s.box_border
        h2=h1+1; ws.row_dimensions[h2].height=25.0
        for col,t in [(3,"CN"),(4,"POI/PPI VIVO"),(5,"CN"),(6,f"POI/PPI {n.upper()}"),(7,"PONTA A VIVO"),(8,f"PONTA B {n.upper()}"),(9,"TIPO DE TRÁFEGO"),(10,"EXIST."),(11,"PLAN."),(12,"EXIST."),(13,"PLAN."),(14,"EXIST."),(15,"PLAN."),(17,"DE A > B\n(FORMATO DE ENTREGA)"),(18,"DE B > A"),(19,"CODEC"),(20,"OBSERVAÇÃO")]:
            ws.merge_cells(start_row=h2,start_column=col,end_row=h2+1,end_column=col); c=ws.cell(row=h2,column=col,value=t); c.font=Font(name="Calibri",size=7 if col==17 else 8,bold=True,color="FFFFFF"); c.alignment=s.align_center_wrap; c.fill=s.fill_accent; c.border=s.box_border
        for sc,ec,t in [(21,22,"VIVO"),(23,24,n.upper())]:
            ws.merge_cells(start_row=h2,start_column=sc,end_row=h2,end_column=ec); c=ws.cell(row=h2,column=sc,value=t); c.font=s.font_micro; c.alignment=s.align_center; c.fill=s.fill_accent; c.border=s.box_border
        h3=h2+1; ws.row_dimensions[h3].height=25.0
        for col,t in [(21,"IP ADDRESS"),(22,"NETMASK"),(23,"IP ADDRESS"),(24,"NETMASK")]:
            c=ws.cell(row=h3,column=col,value=t); c.font=Font(name="Calibri",size=7,bold=True,color="FFFFFF"); c.alignment=s.align_center; c.fill=s.fill_light; c.border=s.box_border
        ds=h3+1
        for i in range(15):
            r=ds+i; ws.row_dimensions[r].height=22.0; f=s.alt_fill(i)
            for col in range(2,25): c=ws.cell(row=r,column=col,value=""); c.font=s.font_small; c.alignment=s.align_center; c.border=s.box_border; c.fill=f
        self._ca(ws,1,1,1,30); self._ca(ws,3,3,1,30); self._ca(ws,5,5,1,30)
        for col in (1,25):
            for r in range(1,ds+20): ws.cell(row=r,column=col).fill=s.fill_white; ws.cell(row=r,column=col).border=s.no_border
        ws.freeze_panes=f"B{ds}"; self._ps(ws,ls=True)

    def _b_conc(self):
        s=self.s; ws=self.wb.create_sheet(title="Concentração")
        self._cw(ws,{1:2,2:2,3:8,4:8,5:12,6:12,7:25})
        ws.merge_cells(start_row=2,start_column=3,end_row=2,end_column=7); c=ws.cell(row=2,column=3,value="2.5 - Concentração"); c.font=Font(name="Calibri",size=11,bold=True,color="FFFFFF"); c.fill=s.fill_primary; ws.row_dimensions[2].height=22.0
        for r,t,it in [(3,"A 'Operadora' informou interesse em realizar a concentração de ALs conforme detalhado na tabela abaixo:",False),(4,'Caso necessite de incluir mais ALs, Favor inserir linhas adicionais de acordo com reunião que foi acordado o tema (coluna B - "Ref").',True)]:
            ws.merge_cells(start_row=r,start_column=3,end_row=r,end_column=7); c=ws.cell(row=r,column=3,value=t); c.font=Font(name="Calibri",size=9 if it else 10,italic=it); c.alignment=s.align_wrap; c.fill=s.fill_background
        hr=6
        for t,col in [("Ref",3),("CN",4),("CNL",5),("Cod. CnI",6),("Status Link",7)]:
            c=ws.cell(row=hr,column=col,value=t); c.font=Font(name="Calibri",size=10,bold=True,color="FFFFFF"); c.alignment=s.align_center; c.fill=s.fill_secondary
        ws.row_dimensions[hr].height=25.0
        for i in range(10):
            r=hr+1+i; ws.row_dimensions[r].height=22.0; f=s.alt_fill(i); d=self.conc[i] if i<len(self.conc) else {}
            for col,k in [(3,"ref"),(4,"cn"),(5,"cnl"),(6,"cod_cni"),(7,"status_link")]:
                c=ws.cell(row=r,column=col,value=d.get(k,"")); c.font=s.font_body; c.fill=f; c.alignment=s.align_center if col!=7 else s.align_left
        ws.freeze_panes=f"C{hr+1}"; self._ps(ws)

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
        for i in range(5):
            r=ds+i; ws.row_dimensions[r].height=20.0; f=s.alt_fill(i)
            for col in range(2,11): c=ws.cell(row=r,column=col,value=""); c.font=s.font_small; c.alignment=s.align_center; c.border=s.box_border; c.fill=f
        ps=ds+6
        for i,p in enumerate(["RN1","EOT Local","EOT LDN/LDI","CSP","Código Especial:","CNG:","RN2"]):
            r=ps+i; ws.row_dimensions[r].height=20.0; f=s.alt_fill(i)
            ws.cell(row=r,column=2,value=p).font=Font(name="Calibri",size=9,bold=True); ws.cell(row=r,column=2).border=s.box_border; ws.cell(row=r,column=2).fill=f
            ws.cell(row=r,column=3,value="").border=s.box_border; ws.cell(row=r,column=3).fill=f
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
        s=self.s; ws=self.wb.create_sheet(title="Dados MTL")
        self._cw(ws,{1:2,2:6,3:8,4:10,5:12,6:15,7:15,8:20,9:15,10:18,11:18,12:20}); self._bh(ws,2,2,11); self._st(ws,4,2,11,"2.7. Dados de MTL")
        ws.merge_cells(start_row=5,start_column=2,end_row=5,end_column=11); ws.cell(row=5,column=2,value="Tabela de dados MTL para interligação entre VIVO e operadora.").font=s.font_body; ws.cell(row=5,column=2).fill=s.fill_background
        ws.merge_cells(start_row=6,start_column=2,end_row=6,end_column=11); ws.cell(row=6,column=2,value="Preencher com os dados técnicos dos circuitos de interligação.").font=Font(name="Calibri",size=9,italic=True); ws.cell(row=6,column=2).fill=s.fill_background
        hr=8
        for col,t in [(2,"Ref"),(3,"Cn"),(4,"ID"),(5,"MTL"),(6,"PONTA VIVO"),(7,"PONTA IMPERIAL"),(8,"DESIGNADOR DO CIRCUITO"),(9,"PROVEDOR"),(10,"Posição Física VIVO"),(11,"Posição Física OPERADORA"),(12,"OBSERVAÇÕES")]:
            c=ws.cell(row=hr,column=col,value=t); c.font=s.font_subheader; c.alignment=s.align_center; c.fill=s.fill_secondary; c.border=s.box_border
        ws.row_dimensions[hr].height=25.0
        for i in range(10):
            r=hr+1+i; ws.row_dimensions[r].height=20.0; f=s.alt_fill(i); ws.cell(row=r,column=2,value=i+1)
            for col in range(2,13):
                c=ws.cell(row=r,column=col);
                if col!=2: c.value=""
                c.font=s.font_small; c.border=s.box_border; c.fill=f; c.alignment=s.align_left if col==12 else s.align_center
        ws.freeze_panes=f"B{hr+1}"; self._ps(ws,ls=True)

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
                    c=ws.cell(row=r,column=ci,value=v); c.font=s.font_check if ci==5 and v=="X" else s.font_small
                    c.alignment=s.align_left if ci==2 else s.align_center; c.fill=f; c.border=s.box_border
                cur[0]+=1
        def _sec(title,data,ch=False):
            _h(title)
            if ch: _ch()
            _r(data); cur[0]+=1

        ws.merge_cells(start_row=cur[0],start_column=2,end_row=cur[0],end_column=6)
        c=ws.cell(row=cur[0],column=2,value="Interconexão Nacional"); c.font=Font(name="Calibri",size=11,bold=True,color="FFFFFF"); c.alignment=s.align_center; c.fill=s.fill_primary; c.border=s.box_border; cur[0]+=1
        _sec("Informação do Gateway",[("Fabricante / Fornecedor","sipwise","Mandatório","X",""),("Modelo - Versão","Elementos Rede Fixa, Fixa I e Móvel","Mandatório","X","")],ch=True)
        _sec("Tipo de Serviço",[("Voz","SIM","Mandatório","X",""),("Fax","SIM","Mandatório","X",""),("DTMF","SIM","Mandatório","X","")])
        _sec("Protocolo",[("Tipo de Protocolo","SIP-I (Q.1912.5)","Mandatório","X","")])
        _sec("Atributos SIP",[("SIP Version","SIP v2.1","Mandatório","X",""),("Protocolo de Transporte","UDP","Mandatório","X",""),("Endereço IP Gateway SIP","Gateway de Gateway - NNI","Mandatório","X",""),("IP ADDR do RTP","IP ADDR do RTP","Mandatório","X",""),("FW ou SBC antes do Gateway","Mandatório","Mandatório","X",""),("Envia DOMAIN na URI","P-Asserted-Identity e Diversion","Mandatório","X",""),("Envia DOMAIN IP","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP para Controle","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP para Controle de QoS","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Chamada","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Sessão","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Recursos","Mandatório","Mandatório","X",""),("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Política","Mandatório","Mandatório","X","")])
        _sec("Codec Rede Móvel",[("AMR encaminhamento","","Mandatório","X",""),("2ª opção G.711a 20ms","","Mandatório","X","Não aceita codec g.711u")])
        _sec("Codec Rede Fixa",[("G.711a 20ms","","Mandatório","X",""),("2ª opção G.729a 20ms","","Mandatório","X","Não aceita codec g.711u")])
        _sec("DTMF",[("1ª Opção: RFC 2833 (Outband) com valor payload = 100","","Mandatório","X",""),("Todos os payloads de DTMF definidos no padrão (96-125) podem ser usados","","Mandatório","X","")])
        _sec("FAX",[("Protocolo T.38 suportado","","Mandatório","X","")])
        _sec("POS",[("Detecção do UPSEEED","Utilização do g711a em re-INVITE","Mandatório","X","Dados necessários no SDP"),("Detecção do UPSEEED","Utilização do g711u em re-INVITE","Mandatório","X","Dados necessários no SDP")])
        _sec("Encaminhamento Enfrante",[("P-ASSERTED Identity (RFC 3325)","SIM - Fomento E.164","Mandatório","X",""),("Nature of address of Calling","SUB, NAT, UNKW","Mandatório","X",""),("B-NUMBER (Called Number)","E.164 - Padrão Roteamento","Mandatório","X",""),("Nature of address","SUB, NAT, UNKW","Mandatório","X","")])
        _sec("Encaminhamento Sainte",[("A-NUMBER (Calling Number)","E.164 - Nacional","Mandatório","X",""),("P-Asserted Identity (RFC 3325)","SIM - Formato E.164","Mandatório","X",""),("Nature of address of Calling","SUB, NAT, UNKW","Mandatório","X",""),("B-NUMBER (Called Number)","E.164 - Padrão Roteamento","Mandatório","X",""),("Nature of address","SUB, NAT, UNKW","Mandatório","X","")])
        _sec("Principais RFC's que devem estar habilitadas e/ou suportadas",[("3261 - SIP Base","","","",""),("3311 - UPDATE (PRACK)","","","",""),("3264/4028 - Offer/Answer - SDP","","","",""),("2833 - DTMF - RTP","","","",""),("RFC 3398 - SIP-I interworking","","","",""),("RFC 4033/4035 - DNSSEC","","","",""),("RFC 4566 - SDP","","","",""),("RFC 3606 - Early Media & Tone Generation","","","","SIP 180Ring (puro), o Proxy deverá gerar o RBT localmente")])
        _sec("Método para alteração dos dados do SDP",[("Método Update","","Mandatório","","Antes do Atendimento"),("Método Update","","Mandatório","","Após atendimento")])
        _sec("Negociação de Codec",[("É mandatório a utilização de re-invites pelo Origem para definição de codecs.","","Mandatório","X",""),("O ptime no SDP answer deve ser múltiplo inteiro de 20 para codecs móveis; 3GPP 26.114","","Mandatório","X",""),("Pacotes suportados às marcações USR, DSCP 46 ou EF para pacotes RTP","","Mandatório","X","")])
        self._ps(ws,ls=True)

    def build(self):
        """Constrói o Workbook completo com todas as abas do PTI."""
        logger.info(f"Gerando PTI Excel para '{self.nome}'...")
        self._b_index(); self._b_versions(); self._b_diagram(); self._b_routing()
        self._b_conc(); self._b_prefixes(); self._b_mtl(); self._b_params()
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
            db.execute("UPDATE atacado_forms SET engenharia_params_json=?,responsavel_engenharia=?,updated_at=? WHERE id=?",(eng_json,resp_eng,datetime.utcnow().isoformat(timespec="seconds"),form_id)); db.commit()
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
        wb=PTIWorkbookBuilder(form).build()
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
        if not cn:
            return jsonify({"error": "Parâmetro 'cn' é obrigatório."}), 400
        analyzer: SBCAnalyzer = app.config["SBC_ANALYZER"]
        result = analyzer.suggest_for_cn(cn)
        return jsonify(asdict(result) if hasattr(result, '__dataclass_fields__') else {
            "cn": result.cn, "uf": result.uf, "cidade": result.cidade,
            "regional": result.regional, "csv_source": result.csv_source,
            "csv_modificado_em": result.csv_modificado_em,
            "data_medicao": result.data_medicao, "total_sbcs": result.total_sbcs,
            "sbcs": result.sbcs, "fallback_usado": result.fallback_usado,
            "fallback_origem": result.fallback_origem, "mensagem": result.mensagem,
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
            "csv_info": analyzer.processor.get_csv_info(),
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
        click.echo(f"Diretório: {health['csv_dir']} ({'OK' if health['csv_dir_exists'] else 'NÃO ENCONTRADO'})")
        click.echo(f"CSV mais recente: {health.get('latest_csv', 'Nenhum')}")
        if health['csv_info'].get('filename'):
            info = health['csv_info']
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
        click.echo(f"  CSV: {result.csv_source} ({result.csv_modificado_em})")
        click.echo(f"  Medição: {result.data_medicao}")
        if result.fallback_usado:
            click.echo(f"  ⚠️  Fallback: {result.fallback_origem}")
        click.echo(f"{'='*60}")
        click.echo(f"  {result.mensagem}")
        click.echo(f"{'='*60}\n")
        if not result.sbcs:
            click.echo("  Nenhum SBC encontrado."); return
        for i, sbc in enumerate(result.sbcs):
            icon = "✅" if sbc.get("recomendado") else ("⚠️" if sbc.get("saude") in ("atencao","critico") else "  ")
            click.echo(f"  {icon} #{i+1} {sbc['nome']}")
            click.echo(f"     Modelo: {sbc['modelo']} | Serviços: {', '.join(sbc.get('servicos',[]))}")
            click.echo(f"     CPU avg: {sbc['cpu_avg']}% | máx {sbc['cpu_max']}% | mín {sbc['cpu_min']}% | última {sbc['cpu_ultima']}%")
            click.echo(f"     Medições: {sbc['total_medicoes']} | Status CSV: {sbc['status_csv']} | Saúde: {sbc['saude']}")
            click.echo(f"     Score: {sbc['score']}/100 {'🏆 RECOMENDADO' if sbc.get('recomendado') else ''}")
            click.echo(f"     Motivo: {sbc['motivo']}"); click.echo()


# =============================================================================
# ENTRYPOINT
# =============================================================================
app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
