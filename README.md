"""
app.db — camada de persistência.
Expõe os modelos, a sessão e o repositório para uso externo.
"""
from app.db.models import (
    ColumnRename,
    Import,
    LogEntry,
    MappingVersion,
    MergedRow,
    RawPortal,
    RawScience,
)
from app.db.repository import Repository
from app.db.session import get_session, init_db, session_scope

__all__ = [
    # modelos
    "Import",
    "ColumnRename",
    "RawScience",
    "RawPortal",
    "MappingVersion",
    "MergedRow",
    "LogEntry",
    # sessão
    "init_db",
    "get_session",
    "session_scope",
    # repositório
    "Repository",
]
