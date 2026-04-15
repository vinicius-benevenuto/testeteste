"""db.py — Conexão, schema e seeds do banco SQLite."""
import sqlite3
import logging
from flask import Flask, g

from config import _CN_SEED_RAW, CN_METADATA

logger = logging.getLogger(__name__)


# =============================================================================
# CONEXÃO
# =============================================================================
def get_db() -> sqlite3.Connection:
    if "db" not in g:
        from flask import current_app
        g.db = sqlite3.connect(current_app.config["DATABASE"])
        g.db.row_factory = sqlite3.Row
        g.db.execute("PRAGMA foreign_keys = ON;")
    return g.db


def _register_db_hooks(app: Flask) -> None:
    @app.teardown_appcontext
    def close_db(exception=None):
        db = g.pop("db", None)
        if db is not None:
            db.close()

    @app.before_request
    def ensure_schema():
        _init_db()


# =============================================================================
# SCHEMA E MIGRAÇÕES
# =============================================================================
def _init_db() -> None:
    db = get_db()
    db.executescript("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            role TEXT NOT NULL CHECK (role IN ('engenharia', 'atacado')),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        CREATE TABLE IF NOT EXISTS atacado_forms (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            owner_id INTEGER NOT NULL,
            status TEXT DEFAULT 'rascunho',
            nome_operadora TEXT, rn1 TEXT,
            csp INTEGER DEFAULT 0, servicos_especiais INTEGER DEFAULT 0,
            cng INTEGER DEFAULT 0, scm INTEGER DEFAULT 0, av INTEGER DEFAULT 0,
            atendimento TEXT, redes TEXT,
            qual TEXT, tmr TEXT,
            responsavel_operadora TEXT, responsavel_vivo TEXT,
            sbc_ativo INTEGER DEFAULT 0, ip_reservado INTEGER DEFAULT 0,
            vivo_reserva INTEGER DEFAULT 0, asn TEXT,
            escopo_text TEXT, escopo_flags_json TEXT, dados_vivo_json TEXT,
            dados_operadora_json TEXT,
            operadora_ciente INTEGER DEFAULT 0, responsavel_infra TEXT,
            lcr_nacional INTEGER DEFAULT 0, white_list INTEGER DEFAULT 0,
            prefixos_liberados_abr INTEGER DEFAULT 0,
            premissas_ok INTEGER DEFAULT 0, aprovado_por TEXT,
            engenharia_params_json TEXT,
            responsavel_atacado TEXT, responsavel_engenharia TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(owner_id) REFERENCES users(id)
        );
        CREATE TABLE IF NOT EXISTS exports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            form_id INTEGER NOT NULL,
            filename TEXT NOT NULL, filepath TEXT NOT NULL,
            size_bytes INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(form_id) REFERENCES atacado_forms(id)
        );
    """)
    for table, col, coldef in [
        ("atacado_forms", "owner_id",               "INTEGER NOT NULL DEFAULT 0"),
        ("atacado_forms", "engenharia_params_json",  "TEXT"),
        ("atacado_forms", "dados_vivo_json",         "TEXT"),
        ("atacado_forms", "dados_operadora_json",    "TEXT"),
        ("atacado_forms", "escopo_text",             "TEXT"),
        ("atacado_forms", "escopo_flags_json",       "TEXT"),
        ("atacado_forms", "responsavel_atacado",     "TEXT"),
        ("atacado_forms", "responsavel_engenharia",  "TEXT"),
        ("atacado_forms", "scm",                     "INTEGER DEFAULT 0"),
        ("atacado_forms", "av",                      "INTEGER DEFAULT 0"),
    ]:
        _ensure_column(db, table, col, coldef)
    db.commit()
    _seed_cns(db)
    _apply_cn_metadata(db)
    from siprouter_sp import init_siprouter_sp
    init_siprouter_sp(db)


def _ensure_column(
    db: sqlite3.Connection, table: str, column: str, coldef: str
) -> None:
    existing = {
        r["name"]
        for r in db.execute(f"PRAGMA table_info({table});").fetchall()
    }
    if column not in existing:
        db.execute(f"ALTER TABLE {table} ADD COLUMN {column} {coldef};")


# =============================================================================
# SEEDS
# =============================================================================
def _parse_cn_seed(raw: str) -> list[str]:
    seen: set[str] = set()
    result: list[str] = []
    for tok in raw.split():
        code = tok.strip().zfill(2)
        if code.isdigit() and code not in seen:
            seen.add(code)
            result.append(code)
    return result


def _seed_cns(db: sqlite3.Connection) -> None:
    db.execute("""
        CREATE TABLE IF NOT EXISTS cns (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            codigo TEXT UNIQUE NOT NULL, nome TEXT, uf TEXT,
            ativo INTEGER NOT NULL DEFAULT 1
        )
    """)
    if db.execute("SELECT COUNT(*) AS c FROM cns").fetchone()["c"] == 0:
        codes = _parse_cn_seed(_CN_SEED_RAW)
        db.executemany(
            "INSERT OR IGNORE INTO cns (codigo, ativo) VALUES (?, 1)",
            [(c,) for c in codes],
        )
        db.commit()


def _apply_cn_metadata(db: sqlite3.Connection) -> None:
    for code, (nome, uf) in CN_METADATA.items():
        db.execute(
            """
            INSERT INTO cns (codigo, nome, uf, ativo) VALUES (?, ?, ?, 1)
            ON CONFLICT(codigo) DO UPDATE
                SET nome=excluded.nome, uf=excluded.uf, ativo=1
            """,
            (code, nome, uf),
        )
    db.commit()
