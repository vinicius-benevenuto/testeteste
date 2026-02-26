import sqlite3
from datetime import datetime
import click
from flask import current_app, g
 
# Adapters/Converters para datetime ISO
sqlite3.register_adapter(datetime, lambda v: v.replace(microsecond=0).isoformat())
sqlite3.register_converter("timestamp", lambda v: datetime.fromisoformat(v.decode()))
 
def get_db():
    if 'db' not in g:
        g.db = sqlite3.connect(
            current_app.config['DATABASE'],
            detect_types=sqlite3.PARSE_DECLTYPES | sqlite3.PARSE_COLNAMES,
        )
        g.db.row_factory = sqlite3.Row
        g.db.text_factory = str
        # PRAGMAs essenciais
        g.db.execute("PRAGMA foreign_keys = ON")
        g.db.execute("PRAGMA journal_mode = WAL")
        g.db.execute("PRAGMA synchronous = NORMAL")
        g.db.execute("PRAGMA busy_timeout = 5000")
    return g.db
 
 
def close_db(e=None):
    db = g.pop('db', None)
    if db is not None:
        db.close()
 
 
def init_db():
    db = get_db()
    with current_app.open_resource('schema.sql') as f:
        db.executescript(f.read().decode('utf8'))
 
 
@click.command('init-db')
def init_db_command():
    """Initialize database schema from schema.sql."""
    init_db()
    click.echo('Initialized the database.')
 
 
def init_app(app):
    app.teardown_appcontext(close_db)
    app.cli.add_command(init_db_command)
 
