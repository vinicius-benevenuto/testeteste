"""helpers.py — Funções utilitárias, context processors e template filters."""
import json
import logging
from datetime import datetime
from typing import Optional

from flask import Flask

from config import MAX_TABLE_ROWS
from db import get_db

logger = logging.getLogger(__name__)


# =============================================================================
# UTILITÁRIOS GERAIS
# =============================================================================
def row_get(row, key: str, default=""):
    """Acessa uma linha de DB com fallback seguro."""
    try:
        v = row[key]
        return default if v is None else v
    except (KeyError, IndexError):
        return default


def safe_filename(name: str) -> str:
    """Remove caracteres inválidos de um nome de arquivo."""
    name = (name or "").strip().replace(" ", "_")
    for ch in ("..", "/", "\\", ":", "*", "?", '"', "<", ">", "|"):
        name = name.replace(ch, "_")
    return name or "Operadora"


def parse_db_datetime(value) -> Optional[datetime]:
    """Converte string ISO ou similar para datetime."""
    if not value:
        return None
    try:
        return datetime.fromisoformat(str(value).replace("T", " "))
    except (ValueError, TypeError):
        return None


def truncate_json_list(raw, default: str = "[]") -> str:
    """Limita uma lista JSON a MAX_TABLE_ROWS itens."""
    if not raw:
        return default
    try:
        parsed = json.loads(raw) if isinstance(raw, str) else raw
        if not isinstance(parsed, list):
            return default
        return json.dumps(parsed[:MAX_TABLE_ROWS], ensure_ascii=False)
    except (json.JSONDecodeError, TypeError):
        return default


def parse_bool_field(value) -> int:
    return 1 if str(value).lower() in ("on", "1", "true", "sim") else 0


# =============================================================================
# CONTEXT PROCESSORS E TEMPLATE FILTERS
# =============================================================================
def _register_context_processors(app: Flask) -> None:
    @app.context_processor
    def inject_cn_codes():
        db = get_db()
        rows = db.execute(
            "SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf "
            "FROM cns WHERE ativo = 1 ORDER BY codigo ASC"
        ).fetchall()
        return {
            "CN_CODES": [r["codigo"] for r in rows],
            "CN_FULL": [
                {"codigo": r["codigo"], "nome": r["nome"], "uf": r["uf"]}
                for r in rows
            ],
        }


def _register_template_filters(app: Flask) -> None:
    @app.template_filter("date_br")
    def date_br_filter(value) -> str:
        if not value:
            return ""
        if hasattr(value, "strftime"):
            try:
                return value.strftime("%d/%m/%Y %H:%M")
            except (ValueError, OSError):
                return str(value)
        dt = parse_db_datetime(value)
        return dt.strftime("%d/%m/%Y %H:%M") if dt else str(value)
