"""Data Merger v2 — pacote principal."""
__version__ = "2.0.0"

def init_db(db_path=None):
    from app.db.session import init_db as _init_db
    return _init_db(db_path)
