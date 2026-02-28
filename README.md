"""db/session.py — Engine, init e session_scope."""
from __future__ import annotations
import os
from contextlib import contextmanager
from pathlib import Path
from typing import Generator

from sqlalchemy import create_engine, event, text
from sqlalchemy.orm import Session, sessionmaker

from app.db.models import Base

_DEFAULT_DB = Path("data") / "app.db"
_engine = None
_SessionLocal = None


def init_db(db_path: str | None = None) -> None:
    global _engine, _SessionLocal
    path = db_path or os.environ.get("DATABASE_PATH", str(_DEFAULT_DB))
    Path(path).parent.mkdir(parents=True, exist_ok=True)
    _engine = create_engine(
        f"sqlite:///{path}",
        connect_args={"check_same_thread": False},
        echo=False,
    )

    @event.listens_for(_engine, "connect")
    def _pragmas(conn, _):
        cur = conn.cursor()
        for p in ("PRAGMA journal_mode=WAL",
                  "PRAGMA foreign_keys=ON",
                  "PRAGMA synchronous=NORMAL"):
            cur.execute(p)
        cur.close()

    _SessionLocal = sessionmaker(bind=_engine, autocommit=False, autoflush=False)
    Base.metadata.create_all(_engine)
    _create_views(_engine)


def _create_views(engine) -> None:
    """Cria ou recria as views de leitura rápida."""
    views = {
        "v_merged_latest": """
            CREATE VIEW IF NOT EXISTS v_merged_latest AS
            SELECT mr.*
            FROM merged_results mr
            JOIN mapping_versions mv ON mr.version_id = mv.id
            WHERE mv.id = (
                SELECT id FROM mapping_versions
                WHERE is_active = 1
                ORDER BY created_at DESC LIMIT 1
            )
        """,
        "v_quality_pending": """
            CREATE VIEW IF NOT EXISTS v_quality_pending AS
            SELECT mr.*
            FROM merged_results mr
            JOIN mapping_versions mv ON mr.version_id = mv.id
            WHERE mv.id = (
                SELECT id FROM mapping_versions
                WHERE is_active = 1
                ORDER BY created_at DESC LIMIT 1
            )
            AND (
                mr.UF IS NULL OR mr.UF = '' OR
                mr.CLUSTER IS NULL OR mr.CLUSTER = '' OR
                mr.Denominacao IS NULL OR mr.Denominacao = ''
            )
        """,
    }
    with engine.connect() as conn:
        for name, ddl in views.items():
            conn.execute(text(ddl))
        conn.commit()


def get_session() -> Session:
    if _SessionLocal is None:
        init_db()
    return _SessionLocal()


@contextmanager
def session_scope() -> Generator[Session, None, None]:
    s = get_session()
    try:
        yield s; s.commit()
    except Exception:
        s.rollback(); raise
    finally:
        s.close()

