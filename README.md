"""Testes de integração para db/repository.py + models."""
import os, sys, tempfile, pytest
sys.path.insert(0, ".")

import pandas as pd
from app.db.session import init_db, session_scope
from app.db.repository import Repository


@pytest.fixture(scope="function")
def tmp_db(tmp_path):
    db_file = str(tmp_path / "test.db")
    init_db(db_file)
    return db_file


@pytest.fixture
def repo(tmp_db):
    with session_scope() as session:
        yield Repository(session)


def _sci_df():
    return pd.DataFrame({
        "Tipo da Rota": ["ITX", "LOCAL"],
        "Central Interna": ["MBCAJUD", "SAO001"],
        "Operadora Origem": ["EMBRATEL", "OI"],
    })


def _por_df():
    return pd.DataFrame({
        "CENTRAL": ["MBCAJUD"],
        "EMPRESA": ["TIM"],
        "LABEL_E": ["LABEL_X"],
    })


def _merged_df():
    return pd.DataFrame({
        "REDE": ["VIVO", "VIVO"],
        "UF": ["SP", "SP"],
        "CLUSTER": ["C1", "C1"],
        "Tipo de Rota": ["ITX", "LOCAL"],
        "Central": ["MBCAJUD", "SAO001"],
        "Rótulos de Linha": ["LABEL_X", ""],
        "OPERADORA": ["EMBRATEL", "OI"],
        "Denominação": ["Rota A", "Rota B"],
    })


class TestImportPersistence:
    def test_save_and_list_science(self, tmp_db):
        init_db(tmp_db)
        with session_scope() as s:
            r = Repository(s)
            iid = r.save_import("science", "test.xlsx", "Sheet1",
                                 "hash123", _sci_df(), {"a": "a"})
            s.commit()
        with session_scope() as s:
            r = Repository(s)
            imports = r.list_imports(source="science")
        assert len(imports) >= 1
        assert imports[0]["source"] == "science"
        assert imports[0]["rows"] == 2

    def test_raw_rows_preserved(self, tmp_db):
        """100% das linhas devem estar no banco após importação."""
        init_db(tmp_db)
        sci = _sci_df()
        with session_scope() as s:
            r = Repository(s)
            iid = r.save_import("science", "test.xlsx", None,
                                 "hash_sci", sci, {})
            s.commit()
        with session_scope() as s:
            r = Repository(s)
            loaded = r.load_raw_df(iid, "science")
        assert len(loaded) == len(sci), "Linhas perdidas na persistência!"
        assert set(sci.columns) == set(loaded.columns)

    def test_portal_raw_preserved(self, tmp_db):
        init_db(tmp_db)
        por = _por_df()
        with session_scope() as s:
            r = Repository(s)
            iid = r.save_import("portal", "portal.xlsx", None,
                                 "hash_por", por, {})
            s.commit()
        with session_scope() as s:
            r = Repository(s)
            loaded = r.load_raw_df(iid, "portal")
        assert len(loaded) == len(por)


class TestMergePersistence:
    def test_save_and_load_merge(self, tmp_db):
        init_db(tmp_db)
        merged = _merged_df()
        with session_scope() as s:
            r = Repository(s)
            vid = r.save_merge_version(
                "20240101_120000", None, None,
                {"REDE": {"type": "literal", "value": "VIVO"}},
                ["Central"], "outer", 90,
                merged, 2, 1,
            )
            s.commit()
        with session_scope() as s:
            r = Repository(s)
            loaded = r.load_merged_df(vid)
        assert len(loaded) == len(merged)
        assert list(loaded.columns) == [
            "REDE", "UF", "CLUSTER", "Tipo de Rota", "Central",
            "Rótulos de Linha", "OPERADORA", "Denominação",
        ]

    def test_version_listed(self, tmp_db):
        init_db(tmp_db)
        with session_scope() as s:
            r = Repository(s)
            vid = r.save_merge_version(
                "tag_test", None, None, {}, [], "outer", 90,
                _merged_df(), 2, 1,
            )
            s.commit()
        with session_scope() as s:
            r = Repository(s)
            versions = r.list_versions()
        assert any(v["id"] == vid for v in versions)

    def test_no_data_deleted(self, tmp_db):
        """Verificação de que nenhum dado some após múltiplas operações."""
        init_db(tmp_db)
        with session_scope() as s:
            r = Repository(s)
            id1 = r.save_import("science", "f1.xlsx", None, "h1", _sci_df(), {})
            id2 = r.save_import("portal",  "f2.xlsx", None, "h2", _por_df(), {})
            s.commit()
        with session_scope() as s:
            r = Repository(s)
            imports = r.list_imports()
        assert len(imports) >= 2
        # Verifica que linhas ainda existem
        with session_scope() as s:
            r = Repository(s)
            sci_loaded = r.load_raw_df(id1, "science")
            por_loaded = r.load_raw_df(id2, "portal")
        assert len(sci_loaded) == len(_sci_df())
        assert len(por_loaded) == len(_por_df())


class TestLogs:
    def test_add_and_get_logs(self, tmp_db):
        init_db(tmp_db)
        with session_scope() as s:
            r = Repository(s)
            r.add_log("INFO", "Teste de log", {"ctx": "pytest"})
            s.commit()
        with session_scope() as s:
            r = Repository(s)
            logs = r.get_logs()
        assert len(logs) >= 1
        assert any("Teste de log" in lg["message"] for lg in logs)
