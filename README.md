"""Testes — regras de mapeamento (Arquivo 3, COALESCE, OPERADORA)."""
import sys; sys.path.insert(0, ".")
import pytest
import pandas as pd
from app.core.map_rules import (
    build_ref_index, lookup_ref, derive_operadora,
    derive_rotulos, derive_tipo_rota, derive_central,
    build_output_row, coalesce,
)

@pytest.fixture
def ref_df():
    return pd.DataFrame({
        "REDE":             ["VIVO-SMP","VIVO-SMP","VIVO-STFC"],
        "UF":               ["SE","SP","MG"],
        "CLUSTER":          ["CL-SE-001","CL-SP-001","CL-MG-001"],
        "Tipo de Rota":     ["ITX","LOCAL","ITX"],
        "Central":          ["MBCAJUD","SAOPAULO01","BELOHZ01"],
        "Rótulos de Linha": ["MBCAJUD_L1 | MBCAJUD_L2","SPO01_L1 | SPO01_L2","BHZ_L1 | BHZ_L2"],
        "OPERADORA":        ["EMBRATEL","OI","CLARO"],
        "Denominação":      ["ROTA SE ITX","ROTA SP LOCAL","ROTA MG ITX"],
    })

@pytest.fixture
def ref_index(ref_df):
    return build_ref_index(ref_df)

class TestRefIndex:
    def test_lookup_by_central_tipo(self, ref_index):
        result = lookup_ref("MBCAJUD", "ITX", "", ref_index)
        assert result is not None
        assert result["CLUSTER"] == "CL-SE-001"

    def test_lookup_by_central_rotulos(self, ref_index):
        result = lookup_ref("SAOPAULO01", "", "SPO01_L1 | SPO01_L2", ref_index)
        assert result is not None
        assert result["UF"] == "SP"

    def test_lookup_case_insensitive(self, ref_index):
        result = lookup_ref("mbcajud", "itx", "", ref_index)
        assert result is not None

    def test_lookup_not_found_returns_none(self, ref_index):
        assert lookup_ref("INEXISTENTE", "XXX", "", ref_index) is None

    def test_cluster_from_arq3(self, ref_index):
        result = lookup_ref("BELOHZ01", "ITX", "", ref_index)
        assert result["CLUSTER"] == "CL-MG-001"

    def test_denominacao_from_arq3(self, ref_index):
        result = lookup_ref("MBCAJUD", "ITX", "", ref_index)
        assert result["Denominação"] == "ROTA SE ITX"

    def test_operadora_from_arq3(self, ref_index):
        result = lookup_ref("SAOPAULO01", "LOCAL", "", ref_index)
        assert result["OPERADORA"] == "OI"


class TestCoalesce:
    def test_first_non_empty(self):
        assert coalesce("", "", "found", "other") == "found"

    def test_all_empty_returns_empty(self):
        assert coalesce("", "", "") == ""

    def test_first_wins(self):
        assert coalesce("first", "second") == "first"

    def test_whitespace_ignored(self):
        assert coalesce("  ", "", "value") == "value"


class TestDeriveTipoRota:
    def test_portal_priority(self):
        val, src = derive_tipo_rota(
            {"TIPO_ROTA": "ITX"}, {"Sinalização da Rota": "LOCAL"}
        )
        assert val == "ITX" and src == "PORTAL"

    def test_falls_back_to_science(self):
        val, src = derive_tipo_rota(
            {"TIPO_ROTA": ""}, {"Sinalização da Rota": "LOCAL"}
        )
        assert val == "LOCAL" and src == "SCIENCE"

    def test_normalized_upper(self):
        val, _ = derive_tipo_rota({"TIPO_ROTA": "itx"}, {})
        assert val == "ITX"


class TestDeriveCentral:
    def test_portal_priority(self):
        val, src = derive_central({"CENTRAL": "MBCAJUD"}, {"Central Origem": "OTHER"})
        assert val == "MBCAJUD" and src == "PORTAL"

    def test_falls_back_to_science(self):
        val, src = derive_central({"CENTRAL": ""}, {"Central Origem": "SAOPAULO01"})
        assert val == "SAOPAULO01" and src == "SCIENCE"


class TestDeriveRotulos:
    def test_concat_label_e_s(self):
        result = derive_rotulos({"LABEL_E": "L_E", "LABEL_S": "L_S"}, concat=True)
        assert "L_E" in result and "L_S" in result

    def test_label_e_only(self):
        result = derive_rotulos({"LABEL_E": "L_E", "LABEL_S": ""}, concat=True)
        assert result == "L_E"

    def test_custom_separator(self):
        result = derive_rotulos({"LABEL_E": "A", "LABEL_S": "B"}, sep=" // ", concat=True)
        assert " // " in result

    def test_no_concat(self):
        result = derive_rotulos({"LABEL_E": "A", "LABEL_S": "B"}, concat=False)
        assert result == "A"


class TestDeriveOperadora:
    def test_arq3_priority(self):
        result = derive_operadora("ARQ3_OP", {"EMPRESA": "PORTAL_OP"}, {"Operadora Origem": "SCI_OP"})
        assert result == "ARQ3_OP"

    def test_portal_fallback(self):
        result = derive_operadora("", {"EMPRESA": "PORTAL_OP"}, {"Operadora Origem": "SCI_OP"})
        assert result == "PORTAL_OP"

    def test_science_fallback(self):
        result = derive_operadora("", {"EMPRESA": ""}, {"Operadora Origem": "SCI_OP"})
        assert result == "SCI_OP"

    def test_normalized_upper(self):
        result = derive_operadora("embratel", {}, {})
        assert result == "EMBRATEL"


class TestBuildOutputRow:
    def test_full_row_all_cols_present(self, ref_index):
        sci = {"Central Origem": "MBCAJUD", "Sinalização da Rota": "ITX",
               "Operadora Origem": "VIVO", "Descrição": "Rota A", "CNL": "79001"}
        por = {"CENTRAL": "MBCAJUD", "TIPO_ROTA": "ITX", "EMPRESA": "EMBRATEL",
               "LABEL_E": "LBL_E", "LABEL_S": "LBL_S", "DESIGNACAO": "Des Portal",
               "CNL_PPI": "79001"}
        from app.core.map_rules import OUTPUT_COLUMNS
        row = build_output_row(sci, por, "BOTH", ref_index, {"79001": "SE"}, {})
        for col in OUTPUT_COLUMNS:
            assert col in row, f"Coluna ausente: {col}"

    def test_rede_both_is_smp(self, ref_index):
        row = build_output_row({}, {}, "BOTH", ref_index, {}, {})
        assert row["REDE"] == "VIVO-SMP"

    def test_rede_portal_is_stfc(self, ref_index):
        row = build_output_row({}, {}, "PORTAL", ref_index, {}, {})
        assert row["REDE"] == "VIVO-STFC"

    def test_uf_from_uf_map(self, ref_index):
        sci = {"CNL": "79001"}
        row = build_output_row(sci, {}, "SCIENCE", ref_index, {"79001": "SE"}, {})
        assert row["UF"] == "SE"

    def test_cluster_from_arq3(self, ref_index):
        sci = {"Central Origem": "MBCAJUD", "Sinalização da Rota": "ITX"}
        por = {"CENTRAL": "MBCAJUD", "TIPO_ROTA": "ITX"}
        row = build_output_row(sci, por, "BOTH", ref_index, {}, {})
        assert row["CLUSTER"] == "CL-SE-001"

    def test_cluster_empty_when_no_arq3_match(self, ref_index):
        sci = {"Central Origem": "INEXISTENTE", "Sinalização da Rota": "XXX"}
        row = build_output_row(sci, {}, "SCIENCE", {}, {}, {})
        assert row["CLUSTER"] == ""
