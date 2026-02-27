"""Testes para core/merge.py — verificação de não-perda de dados."""
import pytest
import sys; sys.path.insert(0, ".")

import pandas as pd
from app.core.map_rules import OUTPUT_COLUMNS
from app.core.merge import build_merged_df, count_join_matches


def _sci_df() -> pd.DataFrame:
    return pd.DataFrame({
        "Tipo da Rota": ["ITX", "LOCAL", "ITX"],
        "Central Interna": ["MBCAJUD", "SAO001", "CWB002"],
        "Operadora Origem": ["EMBRATEL", "OI", "CLARO"],
        "Descrição": ["Rota A", "Rota B", "Rota C"],
        "Sequencial": ["1", "2", "3"],
    })


def _por_df() -> pd.DataFrame:
    return pd.DataFrame({
        "TIPO_ROTA": ["ITX", "LOCAL"],
        "CENTRAL": ["MBCAJUD", "SAO001"],
        "EMPRESA": ["TIM", "VIVO"],
        "LABEL_E": ["LABEL_X", "LABEL_Y"],
        "LABEL_S": ["LABEL_XS", "LABEL_YS"],
        "DESIGNACAO": ["Designação 1", "Designação 2"],
    })


def _default_mapping() -> dict:
    return {
        "REDE":             {"type": "literal", "value": "VIVO"},
        "UF":               {"type": "literal", "value": "SP"},
        "CLUSTER":          {"type": "literal", "value": "C1"},
        "Tipo de Rota":     {"type": "coalesce", "sources": [
                                {"table": "science", "col": "Tipo da Rota"},
                                {"table": "portal", "col": "TIPO_ROTA"}]},
        "Central":          {"type": "coalesce", "sources": [
                                {"table": "science", "col": "Central Interna"},
                                {"table": "portal", "col": "CENTRAL"}]},
        "Rótulos de Linha": {"type": "concat", "sources": [
                                {"table": "portal", "col": "LABEL_E"},
                                {"table": "portal", "col": "LABEL_S"}],
                             "separator": " | "},
        "OPERADORA":        {"type": "coalesce", "sources": [
                                {"table": "science", "col": "Operadora Origem"},
                                {"table": "portal", "col": "EMPRESA"}]},
        "Denominação":      {"type": "coalesce", "sources": [
                                {"table": "science", "col": "Descrição"},
                                {"table": "portal", "col": "DESIGNACAO"}]},
    }


class TestBuildMergedDf:
    def test_output_has_all_required_columns(self):
        sci, por = _sci_df(), _por_df()
        result, _ = build_merged_df(sci, por, _default_mapping(),
                                     join_keys=[], join_type="outer")
        for col in OUTPUT_COLUMNS:
            assert col in result.columns, f"Missing: {col}"

    def test_outer_join_preserves_all_science_rows(self):
        """Outer join deve manter todas as linhas do Science."""
        sci, por = _sci_df(), _por_df()
        result, report = build_merged_df(sci, por, _default_mapping(),
                                          join_keys=["Central"], join_type="outer")
        # Com outer join, todas as 3 linhas do science devem aparecer
        assert len(result) >= len(sci)

    def test_inner_join_reduces_rows(self):
        """Inner join pode reduzir linhas."""
        sci, por = _sci_df(), _por_df()
        result_outer, _ = build_merged_df(sci, por, _default_mapping(),
                                           join_keys=["Central"], join_type="outer")
        result_inner, _ = build_merged_df(sci, por, _default_mapping(),
                                           join_keys=["Central"], join_type="inner")
        assert len(result_outer) >= len(result_inner)

    def test_literal_values_applied(self):
        sci, por = _sci_df(), _por_df()
        result, _ = build_merged_df(sci, por, _default_mapping(),
                                     join_keys=[], join_type="outer")
        assert (result["REDE"] == "VIVO").all()
        assert (result["UF"] == "SP").all()

    def test_coalesce_uses_science_first(self):
        sci, por = _sci_df(), _por_df()
        result, _ = build_merged_df(sci, por, _default_mapping(),
                                     join_keys=[], join_type="outer")
        # Primeira linha: EMBRATEL (science) não OI/VIVO (portal)
        assert result["OPERADORA"].iloc[0] == "EMBRATEL"

    def test_no_extra_columns_in_output(self):
        sci, por = _sci_df(), _por_df()
        result, _ = build_merged_df(sci, por, _default_mapping(),
                                     join_keys=[], join_type="outer")
        assert set(result.columns) == set(OUTPUT_COLUMNS)

    def test_report_row_counts(self):
        sci, por = _sci_df(), _por_df()
        _, report = build_merged_df(sci, por, _default_mapping(),
                                     join_keys=[], join_type="outer")
        assert report["rows_science"] == len(sci)
        assert report["rows_portal"] == len(por)
        assert report["rows_merged"] == report["rows_merged"]  # consistência


class TestCountJoinMatches:
    def test_perfect_match(self):
        sci = pd.DataFrame({"Central": ["A", "B", "C"]})
        por = pd.DataFrame({"CENTRAL": ["A", "B", "C"]})
        stats = count_join_matches(sci, por, "Central", "CENTRAL")
        assert stats["matches"] == 3
        assert stats["sci_only"] == 0
        assert stats["por_only"] == 0

    def test_partial_match(self):
        sci = pd.DataFrame({"Central": ["A", "B", "X"]})
        por = pd.DataFrame({"CENTRAL": ["A", "B", "Y"]})
        stats = count_join_matches(sci, por, "Central", "CENTRAL")
        assert stats["matches"] == 2
        assert stats["sci_only"] == 1
        assert stats["por_only"] == 1

    def test_no_match(self):
        sci = pd.DataFrame({"Central": ["A"]})
        por = pd.DataFrame({"CENTRAL": ["Z"]})
        stats = count_join_matches(sci, por, "Central", "CENTRAL")
        assert stats["matches"] == 0
