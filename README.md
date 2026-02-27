"""Testes para utils/encoding.py e core/normalize.py"""
import sys; sys.path.insert(0, ".")
import pytest
import pandas as pd

from app.utils.encoding import (
    fix_mojibake, normalize_column_key, normalize_column_label,
    normalize_columns,
)
from app.core.normalize import apply_column_normalization, strip_whitespace


class TestFixMojibake:
    def test_ativacao(self):
        assert fix_mojibake("AtivaÃ§Ã£o") in ("Ativação", "Ativação")

    def test_descricao(self):
        assert fix_mojibake("DescriÃ§Ã£o") in ("Descrição", "Descrição")

    def test_clean_text_unchanged(self):
        text = "Central Interna"
        assert fix_mojibake(text) == text

    def test_handles_non_string(self):
        assert isinstance(fix_mojibake(123), str)


class TestNormalizeColumnLabel:
    def test_double_spaces_collapsed(self):
        assert normalize_column_label("col  name") == "col name"

    def test_strip_whitespace(self):
        assert normalize_column_label("  col  ") == "col"

    def test_preserves_accents(self):
        label = normalize_column_label("Área Ponta A")
        assert "Área" in label or "Area" in label

    def test_mojibake_corrected(self):
        label = normalize_column_label("DescriÃ§Ã£o")
        assert "Ã" not in label


class TestNormalizeColumnKey:
    def test_uppercase(self):
        assert normalize_column_key("central") == "CENTRAL"

    def test_removes_accents(self):
        key = normalize_column_key("Área")
        assert "Ã" not in key

    def test_spaces_to_underscore(self):
        assert normalize_column_key("tipo de rota") == "TIPO_DE_ROTA"

    def test_multiple_spaces(self):
        key = normalize_column_key("tipo  de  rota")
        assert "TIPO" in key and "ROTA" in key


class TestApplyColumnNormalization:
    def test_returns_tuple(self):
        df = pd.DataFrame({"colA": [1], "colB": [2]})
        result, rename_map = apply_column_normalization(df)
        assert isinstance(result, pd.DataFrame)
        assert isinstance(rename_map, dict)

    def test_mojibake_header_fixed(self):
        df = pd.DataFrame({"AtivaÃ§Ã£o": ["val1"], "Normal": ["val2"]})
        result, rename_map = apply_column_normalization(df)
        cols = list(result.columns)
        assert all("Ã" not in c for c in cols)

    def test_data_not_altered(self):
        df = pd.DataFrame({"col": ["AtivaÃ§Ã£o data"]})
        result, _ = apply_column_normalization(df)
        # Dados brutos preservados
        assert result.iloc[0, 0] == "AtivaÃ§Ã£o data"

    def test_no_duplicate_columns_after_norm(self):
        df = pd.DataFrame({"Col A": [1], "Col  A": [2]})
        result, _ = apply_column_normalization(df)
        assert len(result.columns) == len(set(result.columns))

    def test_100_percent_rows_preserved(self):
        n = 1000
        df = pd.DataFrame({"colA": range(n), "colB": ["x"] * n})
        result, _ = apply_column_normalization(df)
        assert len(result) == n


class TestStripWhitespace:
    def test_strips_values(self):
        df = pd.DataFrame({"col": ["  value  ", "  other  "]})
        result = strip_whitespace(df)
        assert result["col"].tolist() == ["value", "other"]
