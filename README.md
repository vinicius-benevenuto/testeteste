"""Testes para core/map_rules.py"""
import pytest
import sys; sys.path.insert(0, ".")

from app.core.map_rules import (
    OUTPUT_COLUMNS, apply_rule, suggest_mappings, validate_mapping,
)


SCIENCE_COLS = [
    "COD_CCC Origem", "ID Rota", "Data Ativação", "Tipo da Rota",
    "Operadora Origem", "Operadora destino", "Central Interna",
    "Descrição", "Sequencial", "OPC", "DPC",
]
PORTAL_COLS = [
    "SOLICITACAO", "TIPO", "CENTRAL", "BILHETADOR", "TIPO_ROTA",
    "LABEL_E", "LABEL_S", "EMPRESA", "DESIGNACAO", "SENTIDO",
]


class TestSuggestMappings:
    def test_returns_all_output_columns(self):
        mapping = suggest_mappings(SCIENCE_COLS, PORTAL_COLS)
        for col in OUTPUT_COLUMNS:
            assert col in mapping, f"Missing output column: {col}"

    def test_tipo_de_rota_maps_science_first(self):
        mapping = suggest_mappings(SCIENCE_COLS, PORTAL_COLS)
        rule = mapping["Tipo de Rota"]
        assert rule["type"] in ("coalesce", "concat")
        sources = rule.get("sources", [])
        if sources:
            # Ciência deve vir antes do portal
            sci_idx = next((i for i, s in enumerate(sources) if s["table"] == "science"), 999)
            por_idx = next((i for i, s in enumerate(sources) if s["table"] == "portal"), 999)
            assert sci_idx <= por_idx

    def test_operadora_mapped(self):
        mapping = suggest_mappings(SCIENCE_COLS, PORTAL_COLS)
        rule = mapping["OPERADORA"]
        assert rule["type"] in ("coalesce", "literal")

    def test_rotulos_concat(self):
        mapping = suggest_mappings(SCIENCE_COLS, PORTAL_COLS)
        rule = mapping.get("Rótulos de Linha", {})
        # Deve ser concat quando LABEL_E e LABEL_S presentes
        if rule.get("type") == "concat":
            assert "sources" in rule


class TestApplyRule:
    def setup_method(self):
        self.sci = {
            "Tipo da Rota": "ITX",
            "Central Interna": "MBCAJUD",
            "Operadora Origem": "EMBRATEL",
            "Descrição": "ROTA VIVO ITX",
        }
        self.por = {
            "TIPO_ROTA": "LOCAL",
            "CENTRAL": "OUTRO",
            "EMPRESA": "OI",
            "LABEL_E": "MBCAJUD_TCE1C9",
            "LABEL_S": "OUTRO_LABEL",
            "DESIGNACAO": "ROTA PORTAL",
        }

    def test_coalesce_science_first(self):
        rule = {"type": "coalesce", "sources": [
            {"table": "science", "col": "Tipo da Rota"},
            {"table": "portal", "col": "TIPO_ROTA"},
        ]}
        result = apply_rule(rule, self.sci, self.por)
        assert result == "ITX"

    def test_coalesce_falls_back_to_portal(self):
        rule = {"type": "coalesce", "sources": [
            {"table": "science", "col": "COLUNA_INEXISTENTE"},
            {"table": "portal", "col": "TIPO_ROTA"},
        ]}
        result = apply_rule(rule, self.sci, self.por)
        assert result == "LOCAL"

    def test_concat_labels(self):
        rule = {"type": "concat", "sources": [
            {"table": "portal", "col": "LABEL_E"},
            {"table": "portal", "col": "LABEL_S"},
        ], "separator": " | "}
        result = apply_rule(rule, self.sci, self.por)
        assert "MBCAJUD_TCE1C9" in result
        assert " | " in result

    def test_literal_value(self):
        rule = {"type": "literal", "value": "VIVO-SMP"}
        result = apply_rule(rule, self.sci, self.por)
        assert result == "VIVO-SMP"

    def test_coalesce_empty_falls_through(self):
        sci_empty = {**self.sci, "Tipo da Rota": ""}
        rule = {"type": "coalesce", "sources": [
            {"table": "science", "col": "Tipo da Rota"},
            {"table": "portal", "col": "TIPO_ROTA"},
        ]}
        result = apply_rule(rule, sci_empty, self.por)
        assert result == "LOCAL"


class TestValidateMapping:
    def test_warns_missing_column(self):
        mapping = {"Tipo de Rota": {"type": "coalesce", "sources": [
            {"table": "science", "col": "COLUNA_FANTASMA"}
        ]}}
        warns = validate_mapping(mapping, SCIENCE_COLS, PORTAL_COLS)
        assert len(warns) > 0
        assert any("COLUNA_FANTASMA" in w for w in warns)

    def test_no_warnings_for_valid_mapping(self):
        mapping = {"Tipo de Rota": {"type": "coalesce", "sources": [
            {"table": "science", "col": "Tipo da Rota"},
            {"table": "portal", "col": "TIPO_ROTA"},
        ]}}
        warns = validate_mapping(mapping, SCIENCE_COLS, PORTAL_COLS)
        assert len(warns) == 0

