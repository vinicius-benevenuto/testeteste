"""
io/readers.py
Leitura de arquivos multi-formato. Preserva 100% das colunas e linhas originais.
Toda normalização de labels é feita em core/normalize.py.
"""
from __future__ import annotations
import io
from pathlib import Path
from typing import Dict, List, Optional, Tuple, Union

import pandas as pd

from app.utils.encoding import detect_encoding_from_bytes
from app.utils.logging_utils import get_logger

log = get_logger(__name__)
FileInput = Union[str, Path, bytes, io.BytesIO]


def _to_bytesio(source: FileInput) -> Tuple[io.BytesIO, str]:
    if isinstance(source, (str, Path)):
        p = Path(source); return io.BytesIO(p.read_bytes()), p.name
    if isinstance(source, bytes):
        return io.BytesIO(source), "upload"
    if isinstance(source, io.BytesIO):
        source.seek(0); return source, "upload"
    raise TypeError(f"Tipo não suportado: {type(source)}")


def list_sheets(source: FileInput) -> List[str]:
    """Retorna planilhas de XLSX/XLS."""
    buf, name = _to_bytesio(source)
    ext = Path(name).suffix.lower()
    if ext not in (".xlsx", ".xls"):
        return []
    engine = "openpyxl" if ext == ".xlsx" else "xlrd"
    xl = pd.ExcelFile(buf, engine=engine)
    return xl.sheet_names


def _read_excel(buf: io.BytesIO, sheet: Optional[Union[str, int]] = 0,
                ext: str = ".xlsx") -> pd.DataFrame:
    engine = "openpyxl" if ext == ".xlsx" else "xlrd"
    df = pd.read_excel(buf, sheet_name=sheet, engine=engine,
                       dtype=str, keep_default_na=False)
    df = df.fillna("")
    log.info("XLSX lido: %d × %d (sheet=%s)", len(df), len(df.columns), sheet)
    return df


def _read_csv(buf: io.BytesIO) -> pd.DataFrame:
    raw = buf.read()
    encoding = detect_encoding_from_bytes(raw[:65_536])
    for enc in [encoding, "utf-8", "latin-1"]:
        try:
            df = pd.read_csv(io.BytesIO(raw), sep=None, engine="python",
                             encoding=enc, dtype=str, keep_default_na=False,
                             on_bad_lines="skip")
            df = df.fillna("")
            log.info("CSV lido: %d × %d (enc=%s)", len(df), len(df.columns), enc)
            return df
        except (UnicodeDecodeError, Exception):
            continue
    raise ValueError("Não foi possível ler o CSV com nenhum encoding.")


def _read_parquet(buf: io.BytesIO) -> pd.DataFrame:
    df = pd.read_parquet(buf)
    df = df.astype(str).fillna("")
    log.info("Parquet lido: %d × %d", len(df), len(df.columns))
    return df


def read_file(source: FileInput, filename: str = "",
              sheet: Optional[Union[str, int]] = 0) -> pd.DataFrame:
    """
    Lê arquivo em qualquer formato suportado. Todos os valores como string.
    Suporta: .xlsx, .xls, .csv, .parquet
    """
    buf, auto_name = _to_bytesio(source)
    name = filename or auto_name
    ext = Path(name).suffix.lower()

    if ext in (".xlsx", ".xls"):
        return _read_excel(buf, sheet=sheet, ext=ext)
    if ext == ".csv":
        return _read_csv(buf)
    if ext == ".parquet":
        return _read_parquet(buf)
    # Fallback CSV
    log.warning("Extensão '%s' desconhecida — tentando CSV.", ext)
    return _read_csv(buf)


def read_file_metadata(source: FileInput, filename: str = "") -> Dict:
    """Retorna metadados sem carregar todo o arquivo."""
    buf, auto_name = _to_bytesio(source)
    name = filename or auto_name
    ext = Path(name).suffix.lower()
    sheets: List[str] = []
    if ext in (".xlsx", ".xls"):
        buf.seek(0); sheets = list_sheets(buf)
    return {"filename": name, "ext": ext, "sheets": sheets}
