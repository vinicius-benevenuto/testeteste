"""Testes — derivação de UF via CNL→CN→UF."""
import sys, tempfile, os
sys.path.insert(0, ".")
import pytest
from app.db.session import init_db, session_scope
from app.db.repository import Repository

@pytest.fixture(scope="function")
def db(tmp_path):
    init_db(str(tmp_path / "test_uf.db"))
    return str(tmp_path / "test_uf.db")

def _seed(db_path):
    init_db(db_path)
    sql = """
        INSERT OR IGNORE INTO cnl (COD_CNL, CN) VALUES ('79001','79');
        INSERT OR IGNORE INTO cnl (COD_CNL, CN) VALUES ('11001','11');
        INSERT OR IGNORE INTO cnl (COD_CNL, CN) VALUES ('41406','41');
        INSERT OR IGNORE INTO cnl (COD_CNL, CN) VALUES ('00079','79');
    """
    csv_content = "CN,UF\n79,SE\n11,SP\n41,PR\n31,MG\n21,RJ\n"
    import tempfile, os
    with tempfile.NamedTemporaryFile(suffix=".sql", delete=False, mode="w") as f:
        f.write(sql); sql_path = f.name
    with tempfile.NamedTemporaryFile(suffix=".csv", delete=False, mode="w") as f:
        f.write(csv_content); csv_path = f.name
    try:
        with session_scope() as s:
            r = Repository(s)
            r.load_cnl_seeds(sql_path)
            r.load_cn_to_uf_csv(csv_path)
    finally:
        os.unlink(sql_path); os.unlink(csv_path)

class TestUfDerivation:
    def test_cnl_resolves_to_uf(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            assert r.get_uf_for_cnl("79001") == "SE"

    def test_cnl_sp(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            assert r.get_uf_for_cnl("11001") == "SP"

    def test_cnl_pr(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            assert r.get_uf_for_cnl("41406") == "PR"

    def test_unknown_cnl_returns_none(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            assert r.get_uf_for_cnl("99999") is None

    def test_empty_cnl_returns_none(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            assert r.get_uf_for_cnl("") is None

    def test_leading_zeros_fallback(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            # "00079" está mapeado no seed
            assert r.get_uf_for_cnl("00079") == "SE"

    def test_batch_resolve(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            result = r.resolve_ufs_batch(["79001","11001","41406","99999"])
        assert result.get("79001") == "SE"
        assert result.get("11001") == "SP"
        assert result.get("41406") == "PR"
        assert "99999" not in result

    def test_cnl_count_after_seed(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            assert r.cnl_count() >= 4

    def test_uf_count_after_seed(self, db):
        _seed(db)
        with session_scope() as s:
            r = Repository(s)
            assert r.cn_to_uf_count() >= 5
