"""Testes — normalização de headers e dados."""
import sys; sys.path.insert(0, ".")
import pytest, pandas as pd
from app.utils.encoding import fix_mojibake, normalize_column_label, normalize_column_key
from app.core.normalize import apply_column_normalization, strip_whitespace

class TestFixMojibake:
    def test_ativacao(self):
        assert "ç" in fix_mojibake("AtivaÃ§Ã£o") or "Ativação" == fix_mojibake("AtivaÃ§Ã£o")

    def test_descricao(self):
        result = fix_mojibake("DescriÃ§Ã£o")
        assert "Ã" not in result

    def test_area(self):
        result = fix_mojibake("Ãrea")
        assert "Ã" not in result or result == "Área"

    def test_non_string_coerced(self):
        assert isinstance(fix_mojibake(123), str)


class TestNormalizeColumnLabel:
    def test_strip(self):
        assert normalize_column_label("  col  ") == "col"

    def test_double_space(self):
        assert normalize_column_label("col  name") == "col name"

    def test_mojibake_fixed(self):
        result = normalize_column_label("AtivaÃ§Ã£o")
        assert "Ã" not in result


class TestNormalizeColumnKey:
    def test_upper(self):
        assert normalize_column_key("tipo de rota") == "TIPO_DE_ROTA"

    def test_spaces_to_underscore(self):
        key = normalize_column_key("Tipo da Rota")
        assert " " not in key

    def test_accents_removed(self):
        key = normalize_column_key("Descrição")
        assert "ç" not in key and "Ã" not in key


class TestApplyColumnNormalization:
    def test_data_intact(self):
        df = pd.DataFrame({"AtivaÃ§Ã£o": ["valor_original"]})
        df_norm, _ = apply_column_normalization(df)
        assert df_norm.iloc[0, 0] == "valor_original"

    def test_no_mojibake_in_headers(self):
        df = pd.DataFrame({"DescriÃ§Ã£o": ["x"], "Ãrea": ["y"]})
        df_norm, _ = apply_column_normalization(df)
        for col in df_norm.columns:
            assert "Ã" not in col

    def test_rename_map_returned(self):
        df = pd.DataFrame({"AtivaÃ§Ã£o": ["x"]})
        _, rmap = apply_column_normalization(df)
        assert "AtivaÃ§Ã£o" in rmap

    def test_100_pct_rows_preserved(self):
        data = {"col": [str(i) for i in range(100)]}
        df = pd.DataFrame(data)
        df_norm, _ = apply_column_normalization(df)
        assert len(df_norm) == 100


class TestStripWhitespace:
    def test_strips_values(self):
        df = pd.DataFrame({"col": ["  value  ", "  other  "]})
        result = strip_whitespace(df)
        assert result["col"].tolist() == ["value", "other"]

    def test_data_intact(self):
        df = pd.DataFrame({"col": ["abc", "def"]})
        result = strip_whitespace(df)
        assert result["col"].tolist() == ["abc", "def"]

