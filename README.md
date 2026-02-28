"""io/writers.py — Exportação CSV/XLSX/JSON/log."""
from __future__ import annotations
import io, json
from typing import List
import pandas as pd
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

def to_csv_bytes(df: pd.DataFrame) -> bytes:
    buf = io.StringIO()
    df.to_csv(buf, index=False)
    result = buf.getvalue().encode("utf-8-sig")
    log.info("CSV: %d linhas, %d cols", len(df), len(df.columns))
    return result

def to_xlsx_bytes(df: pd.DataFrame, sheet: str = "Dados") -> bytes:
    buf = io.BytesIO()
    with pd.ExcelWriter(buf, engine="openpyxl") as w:
        df.to_excel(w, index=False, sheet_name=sheet)
        ws = w.sheets[sheet]
        for col_cells in ws.columns:
            mlen = max((len(str(c.value or "")) for c in col_cells), default=8)
            ws.column_dimensions[col_cells[0].column_letter].width = min(mlen + 2, 60)
        ws.freeze_panes = "A2"
    buf.seek(0)
    log.info("XLSX: %d linhas", len(df))
    return buf.read()

def to_mapping_json(mapping: dict) -> bytes:
    return json.dumps(mapping, ensure_ascii=False, indent=2).encode("utf-8")

def logs_to_text(logs: List[dict]) -> bytes:
    lines = [f"[{r.get('timestamp','')}] {r.get('level','')}: {r.get('message','')}"
             for r in logs]
    return "\n".join(lines).encode("utf-8")
