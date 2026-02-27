"""db/session.py — Gerenciamento de sessão SQLAlchemy."""
from __future__ import annotations
import os
from contextlib import contextmanager
from pathlib import Path
from typing import Generator

from sqlalchemy import create_engine, event
from sqlalchemy.orm import Session, sessionmaker

from app.db.models import Base

_DEFAULT_DB = Path("data") / "app.db"
_engine = None
_SessionLocal = None


def init_db(db_path: str | None = None) -> None:
    """Inicializa engine e cria tabelas se não existirem."""
    global _engine, _SessionLocal
    path = db_path or os.environ.get("DATABASE_PATH", str(_DEFAULT_DB))
    Path(path).parent.mkdir(parents=True, exist_ok=True)
    url = f"sqlite:///{path}"
    _engine = create_engine(url, connect_args={"check_same_thread": False}, echo=False)

    @event.listens_for(_engine, "connect")
    def _pragmas(conn, _):
        cur = conn.cursor()
        for pragma in ("PRAGMA journal_mode=WAL",
                       "PRAGMA foreign_keys=ON",
                       "PRAGMA synchronous=NORMAL"):
            cur.execute(pragma)
        cur.close()

    _SessionLocal = sessionmaker(bind=_engine, autocommit=False, autoflush=False)
    Base.metadata.create_all(_engine)


def get_session() -> Session:
    if _SessionLocal is None:
        init_db()
    return _SessionLocal()


@contextmanager
def session_scope() -> Generator[Session, None, None]:
    """Context manager transacional."""
    s = get_session()
    try:
        yield s; s.commit()
    except Exception:
        s.rollback(); raise
    finally:
        s.close()
