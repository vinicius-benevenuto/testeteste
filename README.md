"""
utils/encoding.py
Corrige mojibake em labels de colunas. Dados brutos nunca são alterados.
"""
from __future__ import annotations
import re, unicodedata
from typing import Dict, List

_MOJIBAKE: Dict[str, str] = {
    "Ã§": "ç",
    "Ã£": "ã",
    "Ã³": "ó",
    "Ã©": "é",
    "Ã­": "í",
    "Ãº": "ú",
    "Ã¡": "á",
    "Ã ": "à",
    "Ã¢": "â",
    "Ãµ": "õ",
    "Ã‡": "Ç",
    "Ã‰": "É",
    "Ãƒ": "Ã",
    "Ã‚": "Â",
    "\xe3\x83": "ã",
}

def fix_mojibake(text: str) -> str:
    if not isinstance(text, str):
        return str(text)
    for bad, good in _MOJIBAKE.items():
        text = text.replace(bad, good)
    try:
        fixed = text.encode("latin-1").decode("utf-8")
        ctrl_orig = sum(1 for c in text if unicodedata.category(c).startswith("C"))
        ctrl_fix  = sum(1 for c in fixed if unicodedata.category(c).startswith("C"))
        if ctrl_fix <= ctrl_orig:
            return fixed
    except (UnicodeEncodeError, UnicodeDecodeError):
        pass
    return text

def normalize_column_label(col: str) -> str:
    col = fix_mojibake(str(col))
    return re.sub(r"\s+", " ", col).strip()

def normalize_column_key(col: str) -> str:
    col = normalize_column_label(col)
    col = unicodedata.normalize("NFKD", col)
    col = "".join(c for c in col if not unicodedata.combining(c))
    col = col.upper().strip()
    col = re.sub(r"[\s\-/\\()]+", "_", col)
    return re.sub(r"_+", "_", col).strip("_")

def normalize_columns(columns: List[str]) -> Dict[str, str]:
    return {col: normalize_column_label(col) for col in columns}

def detect_encoding_from_bytes(sample: bytes) -> str:
    try:
        import chardet
        return chardet.detect(sample).get("encoding") or "utf-8"
    except ImportError:
        try:
            sample.decode("utf-8"); return "utf-8"
        except UnicodeDecodeError:
            return "latin-1"

def safe_filename(name: str) -> str:
    return re.sub(r"[^\w.\-() ]", "_", name)[:255]
