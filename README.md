"""io/readers.py — Leitura multi-formato: xlsx, xls, csv, parquet."""
from __future__ import annotations
import io
from pathlib import Path
from typing import Dict, List, Optional, Tuple, Union

import pandas as pd
from app.utils.encoding import detect_encoding
from app.utils.logging_utils import get_logger

log = get_logger(__name__)
FileInput = Union[str, Path, bytes, io.BytesIO]

def _to_bytesio(src: FileInput) -> Tuple[io.BytesIO, str]:
    if isinstance(src, (str, Path)):
        p = Path(src); return io.BytesIO(p.read_bytes()), p.name
    if isinstance(src, bytes):
        return io.BytesIO(src), "upload"
    if isinstance(src, io.BytesIO):
        src.seek(0); return src, "upload"
    raise TypeError(f"Tipo não suportado: {type(src)}")

def list_sheets(src: FileInput) -> List[str]:
    buf, name = _to_bytesio(src)
    ext = Path(name).suffix.lower()
    if ext not in (".xlsx", ".xls"): return []
    return pd.ExcelFile(buf, engine="openpyxl" if ext == ".xlsx" else "xlrd").sheet_names

def read_file(src: FileInput, filename: str = "",
              sheet: Optional[Union[str, int]] = 0) -> pd.DataFrame:
    """Lê qualquer formato suportado. Tudo como string, preserva dados brutos."""
    buf, auto = _to_bytesio(src)
    name = filename or auto
    ext  = Path(name).suffix.lower()

    if ext in (".xlsx", ".xls"):
        engine = "openpyxl" if ext == ".xlsx" else "xlrd"
        # sheet=None retornaria dict; garantimos sempre um único sheet
        sheet_name = sheet if sheet is not None else 0
        result = pd.read_excel(buf, sheet_name=sheet_name, engine=engine,
                               dtype=str, keep_default_na=False)
        # Segurança extra: se ainda vier dict, pega a primeira aba
        if isinstance(result, dict):
            result = next(iter(result.values()))
        df = result.fillna("")
        # Garante que strings "nan"/"None" viram "" em colunas object
        for col in df.select_dtypes(include="object").columns:
            df[col] = df[col].replace({"nan": "", "None": "", "NaN": "", "none": "", "NULL": ""})
        log.info("%s lido: %d × %d (sheet=%s)", name, len(df), len(df.columns), sheet_name)
        return df

    if ext == ".csv":
        raw = buf.read()
        enc = detect_encoding(raw[:65_536])
        for e in [enc, "utf-8", "latin-1"]:
            try:
                df = pd.read_csv(io.BytesIO(raw), sep=None, engine="python",
                                 encoding=e, dtype=str, keep_default_na=False,
                                 on_bad_lines="skip")
                df = df.fillna("")
                log.info("CSV lido: %d × %d (enc=%s)", len(df), len(df.columns), e)
                return df
            except Exception:
                continue
        raise ValueError("Não foi possível ler o CSV.")

    if ext == ".parquet":
        df = pd.read_parquet(buf).astype(str).fillna("")
        log.info("Parquet lido: %d × %d", len(df), len(df.columns))
        return df

    log.warning("Extensão '%s' desconhecida — tentando CSV.", ext)
    buf.seek(0)
    return read_file(buf, filename=name.replace(ext, ".csv"), sheet=sheet)

def read_file_metadata(src: FileInput, filename: str = "") -> Dict:
    buf, auto = _to_bytesio(src)
    name = filename or auto
    ext  = Path(name).suffix.lower()
    sheets = []
    if ext in (".xlsx", ".xls"):
        buf.seek(0); sheets = list_sheets(buf)
    return {"filename": name, "ext": ext, "sheets": sheets}
