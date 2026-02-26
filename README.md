import sqlite3
from datetime import datetime
from typing import List

import click
from flask import current_app, g

sqlite3.register_adapter(datetime, lambda v: v.replace(microsecond=0).isoformat())
sqlite3.register_converter("timestamp", lambda v: datetime.fromisoformat(v.decode()))


def get_db() -> sqlite3.Connection:
    if "db" not in g:
        conn = sqlite3.connect(
            current_app.config["DATABASE"],
            detect_types=sqlite3.PARSE_DECLTYPES | sqlite3.PARSE_COLNAMES,
        )
        conn.row_factory = sqlite3.Row
        conn.text_factory = str
        conn.execute("PRAGMA foreign_keys = ON")
        conn.execute("PRAGMA journal_mode = WAL")
        conn.execute("PRAGMA synchronous = NORMAL")
        conn.execute("PRAGMA busy_timeout = 5000")
        g.db = conn
    return g.db


def get_table_columns(conn: sqlite3.Connection, table: str) -> List[str]:
    """
    Retorna os nomes das colunas de uma tabela como lista de strings puras.

    Usa um cursor isolado com row_factory=None para garantir que o resultado
    do PRAGMA seja sempre uma tupla Python simples, independente do row_factory
    configurado na conexão principal.
    Retorna lista vazia se a tabela não existir.
    """
    cur = conn.cursor()
    cur.row_factory = None
    try:
        cur.execute(f"PRAGMA table_info({table})")
        return [row[1] for row in cur.fetchall()]
    finally:
        cur.close()


def table_exists(conn: sqlite3.Connection, table: str) -> bool:
    cur = conn.cursor()
    cur.row_factory = None
    try:
        cur.execute(
            "SELECT 1 FROM sqlite_master WHERE type='table' AND name=?", (table,)
        )
        return cur.fetchone() is not None
    finally:
        cur.close()


def close_db(e=None):
    db = g.pop("db", None)
    if db is not None:
        db.close()


def init_db():
    db = get_db()
    with current_app.open_resource("schema.sql") as f:
        db.executescript(f.read().decode("utf-8"))


@click.command("init-db")
def init_db_command():
    """Inicializa o esquema do banco a partir de schema.sql."""
    init_db()
    click.echo("Banco de dados inicializado.")


def init_app(app):
    app.teardown_appcontext(close_db)
    app.cli.add_command(init_db_command)
