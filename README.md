"""
core/map_rules.py
Lógica de mapeamento de colunas: sugestões automáticas, COALESCE, precedência.
"""
from __future__ import annotations
import json
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional

import pandas as pd

from app.utils.encoding import normalize_column_key
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

# ── Colunas de saída alvo ─────────────────────────────────────────────────
OUTPUT_COLUMNS = [
    "REDE", "UF", "CLUSTER", "Tipo de Rota",
    "Central", "Rótulos de Linha", "OPERADORA", "Denominação",
]

# ── Sinônimos para sugestão automática ───────────────────────────────────
_SYNONYMS: Dict[str, List[str]] = {
    "REDE":             ["rede", "network"],
    "UF":               ["uf", "estado", "state", "sg_uf"],
    "CLUSTER":          ["cluster", "grupo", "region"],
    "Tipo de Rota":     ["tipo_da_rota", "tipo_rota", "tipo de rota", "type_route",
                         "TIPO_ROTA", "TIPO DA ROTA"],
    "Central":          ["central_interna", "central interna", "central", "CENTRAL",
                         "central_origem"],
    "Rótulos de Linha": ["label_e", "label_s", "rotulo", "label", "LABEL_E", "LABEL_S"],
    "OPERADORA":        ["operadora_origem", "operadora destino", "empresa", "EMPRESA",
                         "operadora", "OPERADORA", "OP_DESTINO"],
    "Denominação":      ["descricao", "descricao_rota", "description", "DESIGNACAO",
                         "designacao", "denominacao"],
}

# ── Defaults de mapeamento ────────────────────────────────────────────────
DEFAULT_MAPPING: Dict[str, Dict] = {
    "REDE":             {"type": "literal", "value": ""},
    "UF":               {"type": "literal", "value": ""},
    "CLUSTER":          {"type": "literal", "value": ""},
    "Tipo de Rota":     {"type": "coalesce",
                         "sources": [
                             {"table": "science", "col": "Tipo da Rota"},
                             {"table": "portal",  "col": "TIPO_ROTA"},
                         ]},
    "Central":          {"type": "coalesce",
                         "sources": [
                             {"table": "science", "col": "Central Interna"},
                             {"table": "portal",  "col": "CENTRAL"},
                         ]},
    "Rótulos de Linha": {"type": "concat",
                         "sources": [
                             {"table": "portal", "col": "LABEL_E"},
                             {"table": "portal", "col": "LABEL_S"},
                         ],
                         "separator": " | "},
    "OPERADORA":        {"type": "coalesce",
                         "sources": [
                             {"table": "science", "col": "Operadora Origem"},
                             {"table": "science", "col": "Operadora destino"},
                             {"table": "portal",  "col": "EMPRESA"},
                         ]},
    "Denominação":      {"type": "coalesce",
                         "sources": [
                             {"table": "science", "col": "Descrição"},
                             {"table": "portal",  "col": "DESIGNACAO"},
                         ]},
}


def suggest_mappings(science_cols: List[str], portal_cols: List[str]) -> Dict[str, Dict]:
    """
    Sugere mapeamentos automáticos baseados em nomes similares e sinônimos.
    Retorna dict {output_col: rule_dict}.
    """
    all_cols = {
        "science": {normalize_column_key(c): c for c in science_cols},
        "portal":  {normalize_column_key(c): c for c in portal_cols},
    }

    suggestions: Dict[str, Dict] = {}
    for out_col in OUTPUT_COLUMNS:
        synonyms = [normalize_column_key(s) for s in _SYNONYMS.get(out_col, [])]
        synonyms.append(normalize_column_key(out_col))

        found: List[Dict] = []
        for table, key_map in all_cols.items():
            for syn in synonyms:
                if syn in key_map:
                    found.append({"table": table, "col": key_map[syn]})
                    break  # pega o primeiro match por tabela

        default = DEFAULT_MAPPING.get(out_col, {"type": "literal", "value": ""})

        if found:
            if out_col == "Rótulos de Linha":
                # Concat especial se LABEL_E e LABEL_S presentes
                portal_labels = [f for f in found if f["table"] == "portal"]
                if len(portal_labels) >= 1:
                    suggestions[out_col] = {
                        "type": "concat",
                        "sources": portal_labels[:2],
                        "separator": " | ",
                    }
                else:
                    suggestions[out_col] = {"type": "coalesce", "sources": found}
            else:
                suggestions[out_col] = {"type": "coalesce", "sources": found}
        else:
            suggestions[out_col] = default

    return suggestions


def apply_rule(rule: Dict, science_row: Dict, portal_row: Dict,
               literal_defaults: Optional[Dict] = None) -> str:
    """
    Aplica uma regra de mapeamento a uma linha combinada.

    Tipos de regra:
    - literal: valor fixo
    - coalesce: primeiro não-vazio na ordem de precedência
    - concat: concatenação com separador
    - reference: lookup em tabela de referência (resolvido antes)
    """
    defaults = literal_defaults or {}

    rtype = rule.get("type", "literal")

    if rtype == "literal":
        return str(rule.get("value", defaults.get(rule.get("target", ""), "")) or "")

    if rtype == "coalesce":
        for src in rule.get("sources", []):
            row = science_row if src["table"] == "science" else portal_row
            val = str(row.get(src["col"], "") or "").strip()
            if val:
                return val
        return ""

    if rtype == "concat":
        parts = []
        for src in rule.get("sources", []):
            row = science_row if src["table"] == "science" else portal_row
            val = str(row.get(src["col"], "") or "").strip()
            if val:
                parts.append(val)
        sep = rule.get("separator", " | ")
        return sep.join(parts)

    return ""


def validate_mapping(mapping: Dict, science_cols: List[str],
                     portal_cols: List[str]) -> List[str]:
    """
    Valida o mapeamento contra as colunas disponíveis.
    Retorna lista de avisos/erros.
    """
    warnings: List[str] = []
    sci_set = set(science_cols)
    por_set = set(portal_cols)

    for out_col, rule in mapping.items():
        rtype = rule.get("type", "literal")
        if rtype in ("coalesce", "concat"):
            for src in rule.get("sources", []):
                col = src.get("col", "")
                table = src.get("table", "")
                cols_set = sci_set if table == "science" else por_set
                if col and col not in cols_set:
                    warnings.append(
                        f"⚠️ '{out_col}': coluna '{col}' não encontrada em {table}.")
        elif rtype == "reference":
            if not rule.get("ref_col"):
                warnings.append(f"⚠️ '{out_col}': tabela de referência sem coluna-chave.")

    return warnings
