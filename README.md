"""
app.utils — utilitários compartilhados.
Expõe as funções mais usadas de encoding, IDs e logging.
"""
from app.utils.encoding import (
    fix_mojibake,
    normalize_column_label,
    normalize_column_key,
    normalize_columns,
    safe_filename,
)
from app.utils.ids import file_hash, new_uuid, version_tag
from app.utils.logging_utils import get_logger

__all__ = [
    "fix_mojibake",
    "normalize_column_label",
    "normalize_column_key",
    "normalize_columns",
    "safe_filename",
    "file_hash",
    "new_uuid",
    "version_tag",
    "get_logger",
]
