"""forms.py — Extração e validação de dados de formulários Flask."""
import json
import logging

from flask import request

from config import (
    ALLOWED_SCOPE_FLAGS,
    BOOLEAN_FIELDS,
    JSON_FIELDS,
    MAX_TABLE_ROWS,
    TEXT_FIELDS,
)
from helpers import parse_bool_field, truncate_json_list

logger = logging.getLogger(__name__)


# =============================================================================
# ESCOPO / FLAGS
# =============================================================================
def extract_scope_flags_from_request() -> str:
    """
    Lê flags de escopo do request.

    Tentativa 1: checkboxes com name="escopo_flags".
    Tentativa 2: hidden input "escopo_flags_json" atualizado pelo JavaScript.
    """
    raw = request.form.getlist("escopo_flags") or []

    if not raw:
        json_raw = request.form.get("escopo_flags_json", "[]")
        try:
            parsed = json.loads(json_raw) if isinstance(json_raw, str) else []
            if isinstance(parsed, list):
                raw = [str(x).strip() for x in parsed if str(x).strip()]
        except (json.JSONDecodeError, TypeError):
            raw = []

    valid = [f for f in raw if f in ALLOWED_SCOPE_FLAGS]
    return json.dumps(list(dict.fromkeys(valid)), ensure_ascii=False)


def _parse_json_dict(raw) -> str:
    if not raw:
        return "{}"
    try:
        parsed = json.loads(raw) if isinstance(raw, str) else raw
        if isinstance(parsed, dict):
            return json.dumps(parsed, ensure_ascii=False)
    except (json.JSONDecodeError, TypeError):
        pass
    return "{}"


# =============================================================================
# PAYLOAD COMPLETO
# =============================================================================
def extract_form_payload() -> dict:
    payload: dict = {}
    for f in TEXT_FIELDS:
        payload[f] = (request.form.get(f) or "").strip()
    for f in BOOLEAN_FIELDS:
        payload[f] = parse_bool_field(request.form.get(f))
    payload["escopo_flags_json"] = extract_scope_flags_from_request()
    for f in JSON_FIELDS:
        if f == "escopo_flags_json":
            continue
        raw = request.form.get(f, "")
        payload[f] = (
            _parse_json_dict(raw)
            if f == "engenharia_params_json"
            else truncate_json_list(raw, "[]")
        )
    return payload


def validate_table_rows(payload: dict) -> bool:
    """Trunca tabelas que excedam MAX_TABLE_ROWS. Retorna True se truncou."""
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
