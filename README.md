"""Testes para io/readers.py"""
import io
import pytest
import pandas as pd


def _make_csv(content: str) -> io.BytesIO:
    return io.BytesIO(content.encode("utf-8"))


def _make_xlsx(data: dict) -> io.BytesIO:
    df = pd.DataFrame(data)
    buf = io.BytesIO()
    df.to_excel(buf, index=False, engine="openpyxl")
    buf.seek(0)
    return buf


# Importação lazy para que o módulo funcione sem Streamlit instalado
def get_reader():
    import sys; sys.path.insert(0, ".")
    from app.io.readers import read_file
    return read_file


class TestReadCSV:
    def test_basic_csv(self):
        rf = get_reader()
        buf = _make_csv("col1,col2\nA,1\nB,2\n")
        df = rf(buf, filename="test.csv")
        assert len(df) == 2
        assert "col1" in df.columns
        assert df["col1"].tolist() == ["A", "B"]

    def test_semicolon_separator(self):
        rf = get_reader()
        buf = _make_csv("col1;col2\nX;10\nY;20\n")
        df = rf(buf, filename="test.csv")
        assert len(df) == 2

    def test_csv_preserves_all_rows(self):
        rf = get_reader()
        rows = 10_000
        lines = ["a,b"] + [f"val{i},{i}" for i in range(rows)]
        buf = _make_csv("\n".join(lines))
        df = rf(buf, filename="big.csv")
        assert len(df) == rows

    def test_csv_all_as_string(self):
        rf = get_reader()
        buf = _make_csv("num,text\n42,hello\n")
        df = rf(buf, filename="test.csv")
        assert df["num"].dtype == object  # string


class TestReadXLSX:
    def test_basic_xlsx(self):
        rf = get_reader()
        buf = _make_xlsx({"colA": ["x", "y"], "colB": [1, 2]})
        df = rf(buf, filename="test.xlsx")
        assert len(df) == 2
        assert "colA" in df.columns

    def test_xlsx_empty_as_string(self):
        rf = get_reader()
        buf = _make_xlsx({"col": ["a", None, "b"]})
        df = rf(buf, filename="test.xlsx")
        assert "" in df["col"].tolist() or df["col"].isna().sum() == 0


class TestListSheets:
    def test_list_sheets_xlsx(self):
        from app.io.readers import list_sheets
        import sys; sys.path.insert(0, ".")
        buf = _make_xlsx({"c": [1, 2]})
        sheets = list_sheets(buf)
        assert isinstance(sheets, list)
        assert len(sheets) >= 1

    def test_list_sheets_csv_empty(self):
        from app.io.readers import list_sheets
        buf = _make_csv("a,b\n1,2\n")
        sheets = list_sheets(buf)
        assert sheets == []
