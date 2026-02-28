"""Testes — persistência SQLite, versões e auditoria."""
import sys, json; sys.path.insert(0, ".")
import pytest, pandas as pd
from app.db.session import init_db, session_scope
from app.db.repository import Repository

@pytest.fixture(autouse=True)
def db(tmp_path):
    init_db(str(tmp_path / "test.db"))

def _sci():
    return pd.DataFrame({
        "Central Interna": ["CA","CB"], "Tipo da Rota": ["ITX","LOCAL"],
        "Operadora Origem": ["E1","OI"], "CNL": ["79001","11001"],
    })

def _por():
    return pd.DataFrame({
        "CENTRAL": ["CA"], "TIPO_ROTA": ["ITX"],
        "EMPRESA": ["E1"], "CNL_PPI": ["79001"],
    })

def _merged():
    return pd.DataFrame({
        "REDE": ["VIVO-SMP"], "UF": ["SE"], "CLUSTER": ["CL-SE"],
        "Tipo de Rota": ["ITX"], "Central": ["CA"],
        "Rótulos de Linha": ["L1"], "OPERADORA": ["E1"],
        "Denominação": ["Rota A"], "_source_tag": ["BOTH"],
    })

class TestImportPersistence:
    def test_save_science_import(self):
        with session_scope() as s:
            r = Repository(s)
            iid = r.save_import("SCIENCE","sci.xlsx",None,"hash123",_sci(),{})
        assert len(iid) == 36

    def test_raw_science_preserved(self):
        with session_scope() as s:
            r = Repository(s)
            iid = r.save_import("SCIENCE","sci.xlsx",None,"h1",_sci(),{})
        with session_scope() as s:
            r = Repository(s)
            df = r.load_raw_df(iid,"SCIENCE")
        assert len(df) == 2

    def test_save_portal_import(self):
        with session_scope() as s:
            r = Repository(s)
            iid = r.save_import("PORTAL","por.xlsx",None,"h2",_por(),{})
        assert iid

    def test_list_imports_returns_all(self):
        with session_scope() as s:
            r = Repository(s)
            r.save_import("SCIENCE","s.xlsx",None,"h1",_sci(),{})
            r.save_import("PORTAL","p.xlsx",None,"h2",_por(),{})
        with session_scope() as s:
            r = Repository(s)
            imps = r.list_imports()
        assert len(imps) == 2

    def test_raw_100_percent(self):
        """Garante que todas as linhas brutas são persistidas."""
        big = pd.DataFrame({"col": [str(i) for i in range(200)]})
        with session_scope() as s:
            r = Repository(s)
            iid = r.save_import("SCIENCE","big.xlsx",None,"hbig",big,{})
        with session_scope() as s:
            r = Repository(s)
            df = r.load_raw_df(iid,"SCIENCE")
        assert len(df) == 200


class TestMergeVersioning:
    def test_save_merge_version(self):
        with session_scope() as s:
            r = Repository(s)
            sci_id = r.save_import("SCIENCE","s.xlsx",None,"h1",_sci(),{})
            por_id = r.save_import("PORTAL","p.xlsx",None,"h2",_por(),{})
            vid = r.save_merge_version(
                "tag_test", sci_id, por_id, None,
                {}, [], "outer", 90, _merged(), 2, 1
            )
        assert len(vid) == 36

    def test_list_versions(self):
        with session_scope() as s:
            r = Repository(s)
            sci_id = r.save_import("SCIENCE","s.xlsx",None,"h1",_sci(),{})
            por_id = r.save_import("PORTAL","p.xlsx",None,"h2",_por(),{})
            r.save_merge_version("v1",sci_id,por_id,None,{},[],
                                  "outer",90,_merged(),2,1)
        with session_scope() as s:
            r = Repository(s)
            versions = r.list_versions()
        assert len(versions) == 1
        assert versions[0]["rows_merged"] == 1

    def test_load_merged_latest(self):
        with session_scope() as s:
            r = Repository(s)
            sci_id = r.save_import("SCIENCE","s.xlsx",None,"h1",_sci(),{})
            por_id = r.save_import("PORTAL","p.xlsx",None,"h2",_por(),{})
            r.save_merge_version("v1",sci_id,por_id,None,{},[],
                                  "outer",90,_merged(),2,1)
        with session_scope() as s:
            r = Repository(s)
            df = r.load_merged_df()
        assert len(df) == 1

    def test_multiple_versions_preserved(self):
        """Versões antigas nunca são deletadas."""
        with session_scope() as s:
            r = Repository(s)
            sci_id = r.save_import("SCIENCE","s.xlsx",None,"h1",_sci(),{})
            por_id = r.save_import("PORTAL","p.xlsx",None,"h2",_por(),{})
            for i in range(3):
                r.save_merge_version(f"v{i}",sci_id,por_id,None,{},[],
                                      "outer",90,_merged(),2,1)
        with session_scope() as s:
            r = Repository(s)
            assert len(r.list_versions()) == 3


class TestLogs:
    def test_add_and_get_log(self):
        with session_scope() as s:
            r = Repository(s)
            r.add_log("INFO","Teste de log",{"key":"val"})
        with session_scope() as s:
            r = Repository(s)
            logs = r.get_logs(10)
        assert any("Teste de log" in l["message"] for l in logs)

    def test_log_levels(self):
        with session_scope() as s:
            r = Repository(s)
            for lvl in ["INFO","WARNING","ERROR","DEBUG"]:
                r.add_log(lvl, f"log {lvl}")
        with session_scope() as s:
            r = Repository(s)
            logs = r.get_logs(10)
        levels = {l["level"] for l in logs}
        assert "INFO" in levels and "ERROR" in levels
