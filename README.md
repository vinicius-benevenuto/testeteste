"""Testes — merge Science + Portal + Arquivo 3."""
import sys; sys.path.insert(0, ".")
import pytest, pandas as pd
from app.core.merge import build_merged_df, count_join_matches
from app.core.map_rules import OUTPUT_COLUMNS, build_ref_index

@pytest.fixture
def sci():
    return pd.DataFrame({
        "Central Interna":     ["MBCAJUD","SAOPAULO01","BELOHZ01"],
        "Tipo da Rota":        ["ITX","LOCAL","ITX"],
        "Sinalização da Rota": ["SS7","R2","SS7"],
        "Operadora Origem":    ["EMBRATEL","OI","CLARO"],
        "Descrição":           ["Rota A","Rota B","Rota C"],
        "CNL":                 ["79001","11001","31001"],
        "Central Origem":      ["MBCAJUD","SAOPAULO01","BELOHZ01"],
    })

@pytest.fixture
def por():
    return pd.DataFrame({
        "CENTRAL":   ["MBCAJUD","SAOPAULO01"],
        "TIPO_ROTA": ["ITX","LOCAL"],
        "EMPRESA":   ["EMBRATEL","OI"],
        "LABEL_E":   ["MBCAJUD_L1","SPO01_L1"],
        "LABEL_S":   ["MBCAJUD_L2","SPO01_L2"],
        "DESIGNACAO":["Des A","Des B"],
        "CNL_PPI":   ["79001","11001"],
    })

@pytest.fixture
def ref():
    return pd.DataFrame({
        "REDE":             ["VIVO-SMP","VIVO-SMP"],
        "UF":               ["SE","SP"],
        "CLUSTER":          ["CL-SE","CL-SP"],
        "Tipo de Rota":     ["ITX","LOCAL"],
        "Central":          ["MBCAJUD","SAOPAULO01"],
        "Rótulos de Linha": ["MBCAJUD_L1 | MBCAJUD_L2","SPO01_L1 | SPO01_L2"],
        "OPERADORA":        ["EMBRATEL","OI"],
        "Denominação":      ["ROTA SE","ROTA SP"],
    })

@pytest.fixture
def uf_map():
    return {"79001":"SE","11001":"SP","31001":"MG"}

class TestMerge:
    def test_output_columns_present(self, sci, por, ref, uf_map):
        result, _ = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        for col in OUTPUT_COLUMNS:
            assert col in result.columns, f"Coluna ausente: {col}"

    def test_outer_join_preserves_all_science_rows(self, sci, por, ref, uf_map):
        result, _ = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        assert len(result) >= len(sci)

    def test_outer_join_preserves_all_portal_rows(self, sci, por, ref, uf_map):
        result, _ = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        assert len(result) >= len(por)

    def test_rede_smp_when_science_contributes(self, sci, por, ref, uf_map):
        result, _ = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        both_rows = result[result["_source_tag"].isin(["SCIENCE","BOTH"])]
        assert (both_rows["REDE"] == "VIVO-SMP").all()

    def test_uf_resolved_from_map(self, sci, por, ref, uf_map):
        result, _ = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        se_rows = result[result["Central"] == "MBCAJUD"]
        assert (se_rows["UF"] == "SE").any()

    def test_cluster_from_arq3(self, sci, por, ref, uf_map):
        result, _ = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        row = result[result["Central"] == "MBCAJUD"]
        assert (row["CLUSTER"] == "CL-SE").any()

    def test_no_data_loss_positional(self, sci, por, ref, uf_map):
        """Sem chaves: concatenação posicional, nenhuma linha some."""
        result, _ = build_merged_df(sci, por, ref, uf_map, [], [], "outer")
        assert len(result) == max(len(sci), len(por))

    def test_inner_join_smaller(self, sci, por, ref, uf_map):
        result_outer, _ = build_merged_df(sci, por, ref, uf_map,
                                           ["Central Interna"],["CENTRAL"],"outer")
        result_inner, _ = build_merged_df(sci, por, ref, uf_map,
                                           ["Central Interna"],["CENTRAL"],"inner")
        assert len(result_inner) <= len(result_outer)

    def test_source_tag_column_present(self, sci, por, ref, uf_map):
        result, _ = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        assert "_source_tag" in result.columns

    def test_report_contains_metrics(self, sci, por, ref, uf_map):
        _, report = build_merged_df(sci, por, ref, uf_map,
                                     ["Central Interna"],["CENTRAL"],"outer")
        assert "rows_science" in report
        assert "rows_portal"  in report
        assert "rows_merged"  in report
        assert report["rows_science"] == len(sci)
        assert report["rows_portal"]  == len(por)


class TestCountJoinMatches:
    def test_exact_matches(self, sci, por):
        stats = count_join_matches(sci, por, "Central Interna", "CENTRAL")
        assert stats["matches"] == 2

    def test_sci_only(self, sci, por):
        stats = count_join_matches(sci, por, "Central Interna", "CENTRAL")
        assert stats["sci_only"] == 1

    def test_invalid_col_returns_error(self, sci, por):
        stats = count_join_matches(sci, por, "INEXISTENTE", "CENTRAL")
        assert "error" in stats
