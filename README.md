"""io/writers.py — Exportação para CSV e XLSX."""
from __future__ import annotations
import io
from typing import List

import pandas as pd

from app.utils.logging_utils import get_logger
log = get_logger(__name__)


def to_csv_bytes(df: pd.DataFrame, encoding: str = "utf-8-sig") -> bytes:
    """CSV com BOM UTF-8 para compatibilidade com Excel."""
    buf = io.StringIO()
    df.to_csv(buf, index=False)
    result = buf.getvalue().encode(encoding)
    log.info("CSV gerado: %d linhas, %d colunas", len(df), len(df.columns))
    return result


def to_xlsx_bytes(df: pd.DataFrame, sheet_name: str = "Dados") -> bytes:
    """XLSX com cabeçalho formatado e colunas auto-ajustadas."""
    buf = io.BytesIO()
    with pd.ExcelWriter(buf, engine="openpyxl") as writer:
        df.to_excel(writer, index=False, sheet_name=sheet_name)
        ws = writer.sheets[sheet_name]
        for col_cells in ws.columns:
            max_len = max(
                (len(str(cell.value or "")) for cell in col_cells), default=8)
            ws.column_dimensions[col_cells[0].column_letter].width = min(max_len + 2, 60)
        ws.freeze_panes = "A2"
    buf.seek(0)
    log.info("XLSX gerado: %d linhas", len(df))
    return buf.read()


def to_mapping_json(mapping: dict) -> bytes:
    """Serializa mapeamento para JSON."""
    import json
    return json.dumps(mapping, ensure_ascii=False, indent=2).encode("utf-8")


def logs_to_text(logs: List[dict]) -> bytes:
    lines = [f"[{r.get('timestamp','')}] {r.get('level','INFO')}: {r.get('message','')}"
             for r in logs]
    return "\n".join(lines).encode("utf-8")
