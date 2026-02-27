"""
app.io — leitura e escrita de arquivos.
Expõe as funções públicas de readers e writers.
"""
from app.io.readers import list_sheets, read_file, read_file_metadata
from app.io.writers import logs_to_text, to_csv_bytes, to_mapping_json, to_xlsx_bytes

__all__ = [
    # leitura
    "read_file",
    "read_file_metadata",
    "list_sheets",
    # escrita
    "to_csv_bytes",
    "to_xlsx_bytes",
    "to_mapping_json",
    "logs_to_text",
]
