"""Testes — leitura multi-formato."""
import sys, io; sys.path.insert(0, ".")
import pytest, pandas as pd
from app.io.readers import read_file, list_sheets

def _make_xlsx(data: dict) -> io.BytesIO:
    buf = io.BytesIO()
    pd.DataFrame(data).to_excel(buf, index=False, engine="openpyxl")
    buf.seek(0)
    return buf

class TestReaders:
    def test_csv_comma(self):
        buf = io.BytesIO(b"col1,col2\nA,1\nB,2\n")
        df = read_file(buf, filename="test.csv")
        assert len(df) == 2 and list(df["col1"]) == ["A","B"]

    def test_csv_semicolon(self):
        buf = io.BytesIO(b"col1;col2\nX;10\nY;20\n")
        df = read_file(buf, filename="test.csv")
        assert len(df) == 2

    def test_xlsx_read(self):
        buf = _make_xlsx({"a":[1,2],"b":["x","y"]})
        df = read_file(buf, filename="test.xlsx")
        assert len(df) == 2

    def test_all_strings(self):
        buf = _make_xlsx({"num":[1,2],"text":["a","b"]})
        df = read_file(buf, filename="test.xlsx")
        assert df["num"].dtype == object  # tudo string

    def test_100_pct_rows(self):
        data = {"val": [str(i) for i in range(150)]}
        buf = _make_xlsx(data)
        df = read_file(buf, filename="test.xlsx")
        assert len(df) == 150

    def test_list_sheets_xlsx(self):
        buf = io.BytesIO()
        with pd.ExcelWriter(buf, engine="openpyxl") as w:
            pd.DataFrame({"a":[1]}).to_excel(w,sheet_name="Aba1",index=False)
            pd.DataFrame({"b":[2]}).to_excel(w,sheet_name="Aba2",index=False)
        buf.seek(0)
        sheets = list_sheets(buf)
        assert "Aba1" in sheets and "Aba2" in sheets

    def test_list_sheets_csv_empty(self):
        buf = io.BytesIO(b"a,b\n1,2\n")
        assert list_sheets(buf) == []

    def test_fillna_empty_string(self):
        buf = _make_xlsx({"a":["","x"],"b":["y",""]})
        df = read_file(buf, filename="test.xlsx")
        assert "" in df["a"].values

    def test_encoding_latin1(self):
        csv_lat = "col\nSão Paulo\nRio de Janeiro\n".encode("latin-1")
        buf = io.BytesIO(csv_lat)
        df = read_file(buf, filename="test.csv")
        assert len(df) == 2
