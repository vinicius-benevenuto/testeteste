"""
db/repository.py
Toda interação com o banco passa por aqui. Nenhum dado é deletado.
"""
from __future__ import annotations
import json
from typing import Any, Dict, List, Optional

import pandas as pd
from sqlalchemy.orm import Session

from app.db.models import (
    ColumnRename, Import, LogEntry, MappingVersion, MergedRow,
    RawPortal, RawScience,
)
from app.utils.ids import new_uuid
from app.utils.logging_utils import get_logger

log = get_logger(__name__)
_CHUNK = 500


class Repository:
    def __init__(self, session: Session) -> None:
        self.session = session

    # ── Logs ──────────────────────────────────────────────────────────────

    def add_log(self, level: str, message: str, context: Optional[Dict] = None) -> None:
        self.session.add(LogEntry(
            level=level.upper(),
            message=message,
            context_json=json.dumps(context or {}, default=str),
        ))
        try: self.session.flush()
        except Exception: pass

    def get_logs(self, limit: int = 500) -> List[Dict]:
        rows = (self.session.query(LogEntry)
                .order_by(LogEntry.timestamp.desc()).limit(limit).all())
        return [{"id": r.id,
                 "timestamp": r.timestamp.isoformat() if r.timestamp else "",
                 "level": r.level, "message": r.message,
                 "context": json.loads(r.context_json or "{}")} for r in rows]

    # ── Importações ────────────────────────────────────────────────────────

    def save_import(self, source: str, filename: str, sheet: Optional[str],
                    file_hash: str, df: pd.DataFrame,
                    column_map: Dict[str, str]) -> str:
        """Persiste importação + renames + linhas brutas. Retorna import_id."""
        import_id = new_uuid()
        self.session.add(Import(
            id=import_id, source=source, filename=filename, sheet=sheet,
            file_hash=file_hash, rows=len(df), cols=len(df.columns),
        ))
        for orig, norm in column_map.items():
            self.session.add(ColumnRename(
                import_id=import_id, original_name=orig, normalized_name=norm))

        RawModel = RawScience if source == "science" else RawPortal
        records = df.to_dict(orient="records")
        for i in range(0, len(records), _CHUNK):
            for j, row in enumerate(records[i:i + _CHUNK]):
                self.session.add(RawModel(
                    import_id=import_id, row_num=i + j,
                    data_json=json.dumps(row, ensure_ascii=False, default=str),
                ))
            self.session.flush()

        log.info("Import salvo: id=%s source=%s rows=%d", import_id, source, len(df))
        return import_id

    def list_imports(self, source: Optional[str] = None) -> List[Dict]:
        q = self.session.query(Import)
        if source:
            q = q.filter(Import.source == source)
        return [{"id": r.id, "source": r.source, "filename": r.filename,
                 "sheet": r.sheet, "rows": r.rows, "cols": r.cols,
                 "created_at": r.created_at.isoformat() if r.created_at else "",
                 "is_active": r.is_active} for r in q.order_by(Import.created_at.desc()).all()]

    def load_raw_df(self, import_id: str, source: str) -> pd.DataFrame:
        RawModel = RawScience if source == "science" else RawPortal
        rows = (self.session.query(RawModel)
                .filter(RawModel.import_id == import_id)
                .order_by(RawModel.row_num).all())
        if not rows: return pd.DataFrame()
        return pd.DataFrame([json.loads(r.data_json) for r in rows])

    def get_import(self, import_id: str) -> Optional[Dict]:
        r = self.session.query(Import).filter(Import.id == import_id).first()
        if not r: return None
        return {"id": r.id, "source": r.source, "filename": r.filename,
                "sheet": r.sheet, "rows": r.rows, "cols": r.cols}

    # ── Versões de merge ───────────────────────────────────────────────────

    def save_merge_version(self, tag: str, science_import_id: Optional[str],
                           portal_import_id: Optional[str], mapping: Dict,
                           join_keys: List[str], join_type: str,
                           fuzzy_threshold: int, merged_df: pd.DataFrame,
                           rows_science: int, rows_portal: int) -> str:
        version_id = new_uuid()
        self.session.add(MappingVersion(
            id=version_id, tag=tag,
            science_import_id=science_import_id,
            portal_import_id=portal_import_id,
            mapping_json=json.dumps(mapping, ensure_ascii=False, default=str),
            join_keys_json=json.dumps(join_keys),
            join_type=join_type, fuzzy_threshold=fuzzy_threshold,
            rows_science=rows_science, rows_portal=rows_portal,
            rows_merged=len(merged_df),
        ))

        _COL_MAP = {
            "REDE": "REDE", "UF": "UF", "CLUSTER": "CLUSTER",
            "Tipo de Rota": "Tipo_de_Rota", "Central": "Central",
            "Rótulos de Linha": "Rotulos_de_Linha",
            "OPERADORA": "OPERADORA", "Denominação": "Denominacao",
        }
        for i, row in enumerate(merged_df.to_dict(orient="records")):
            mr = MergedRow(version_id=version_id, row_id=i,
                           source_keys_json=json.dumps(row, ensure_ascii=False, default=str))
            for out_col, db_col in _COL_MAP.items():
                setattr(mr, db_col, str(row.get(out_col, "") or ""))
            self.session.add(mr)
            if i % _CHUNK == 0:
                self.session.flush()

        self.session.flush()
        log.info("Merge salvo: version_id=%s rows=%d", version_id, len(merged_df))
        return version_id

    def list_versions(self) -> List[Dict]:
        rows = (self.session.query(MappingVersion)
                .order_by(MappingVersion.created_at.desc()).all())
        return [{"id": r.id, "tag": r.tag, "rows_merged": r.rows_merged,
                 "rows_science": r.rows_science, "rows_portal": r.rows_portal,
                 "join_type": r.join_type,
                 "created_at": r.created_at.isoformat() if r.created_at else "",
                 "is_active": r.is_active} for r in rows]

    def load_merged_df(self, version_id: str) -> pd.DataFrame:
        rows = (self.session.query(MergedRow)
                .filter(MergedRow.version_id == version_id)
                .order_by(MergedRow.row_id).all())
        if not rows: return pd.DataFrame()
        return pd.DataFrame([{
            "REDE": r.REDE, "UF": r.UF, "CLUSTER": r.CLUSTER,
            "Tipo de Rota": r.Tipo_de_Rota, "Central": r.Central,
            "Rótulos de Linha": r.Rotulos_de_Linha,
            "OPERADORA": r.OPERADORA, "Denominação": r.Denominacao,
        } for r in rows])
