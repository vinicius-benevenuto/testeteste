"""db/repository.py — Toda operação no banco passa por aqui."""
from __future__ import annotations
import json
from pathlib import Path
from typing import Dict, List, Optional

import pandas as pd
from sqlalchemy import text
from sqlalchemy.orm import Session

from app.db.models import (
    ColumnRename, CnlTable, CnToUf, Import, LogEntry,
    MappingVersion, MergedRow, RawPortal, RawScience, RefRotas,
)
from app.utils.ids import new_uuid
from app.utils.logging_utils import get_logger

log = get_logger(__name__)
_CHUNK = 500


class Repository:
    def __init__(self, session: Session) -> None:
        self.session = session

    # ── Logs ─────────────────────────────────────────────────────────────

    def add_log(self, level: str, message: str, ctx: Optional[Dict] = None) -> None:
        self.session.add(LogEntry(
            level=level.upper(), message=message,
            context_json=json.dumps(ctx or {}, default=str)))
        try: self.session.flush()
        except Exception: pass

    def get_logs(self, limit: int = 500) -> List[Dict]:
        rows = (self.session.query(LogEntry)
                .order_by(LogEntry.timestamp.desc()).limit(limit).all())
        return [{"id": r.id,
                 "timestamp": r.timestamp.isoformat() if r.timestamp else "",
                 "level": r.level, "message": r.message} for r in rows]

    # ── Seeds CNL ─────────────────────────────────────────────────────────

    def load_cnl_seeds(self, sql_path: str) -> int:
        """Executa seeds/cnl.sql para popular a tabela cnl."""
        sql = Path(sql_path).read_text(encoding="utf-8")
        count = 0
        for stmt in sql.split(";"):
            stmt = stmt.strip()
            if not stmt: continue
            try:
                self.session.execute(text(stmt))
                count += 1
            except Exception as e:
                log.warning("CNL seed skip: %s", e)
        self.session.flush()
        log.info("CNL seeds: %d statements executados", count)
        return count

    def load_cn_to_uf_csv(self, csv_path: str) -> int:
        """Carrega seeds/cn_to_uf.csv para popular cn_to_uf."""
        content = Path(csv_path).read_text(encoding="utf-8").strip()
        if not content:
            log.warning("cn_to_uf.csv está vazio, ignorando.")
            return 0
        import io as _io
        df = pd.read_csv(_io.StringIO(content), dtype=str).fillna("")
        # Aceita cabeçalhos: CN,UF ou cn,uf
        df.columns = [c.strip().upper() for c in df.columns]
        if "CN" not in df.columns or "UF" not in df.columns:
            raise ValueError("cn_to_uf.csv deve ter colunas CN e UF")
        count = 0
        for _, row in df.iterrows():
            cn = str(row["CN"]).strip()
            uf = str(row["UF"]).strip().upper()
            if not cn or not uf: continue
            existing = self.session.query(CnToUf).filter(CnToUf.CN == cn).first()
            if existing:
                existing.UF = uf
            else:
                self.session.add(CnToUf(CN=cn, UF=uf))
            count += 1
        self.session.flush()
        log.info("cn_to_uf: %d registros carregados", count)
        return count

    def get_uf_for_cnl(self, cod_cnl: str) -> Optional[str]:
        """Resolve COD_CNL → CN → UF via JOINs nas tabelas de referência."""
        if not cod_cnl: return None
        cod_cnl = cod_cnl.strip()
        row = (
            self.session.query(CnToUf.UF)
            .join(CnlTable, CnlTable.CN == CnToUf.CN)
            .filter(CnlTable.COD_CNL == cod_cnl)
            .first()
        )
        if row: return row.UF
        # Tenta sem zeros à esquerda
        cod_stripped = cod_cnl.lstrip("0")
        row2 = (
            self.session.query(CnToUf.UF)
            .join(CnlTable, CnlTable.CN == CnToUf.CN)
            .filter(CnlTable.COD_CNL == cod_stripped)
            .first()
        )
        return row2.UF if row2 else None

    def resolve_ufs_batch(self, cod_cnls: List[str]) -> Dict[str, str]:
        """Resolve uma lista de COD_CNL para UF em batch.
        Normaliza sufixo .0 (floats do Excel) antes do lookup.
        """
        from app.utils.cnl_utils import clean_cnl
        unique = list(set(clean_cnl(c) for c in cod_cnls if clean_cnl(c)))
        result: Dict[str, str] = {}
        for cod in unique:
            uf = self.get_uf_for_cnl(cod)
            if uf:
                result[cod] = uf
                result[cod.lstrip("0")] = uf
        return result


    def get_cn_to_uf_map(self) -> Dict[str, str]:
        """Retorna dict CN→UF para lookup direto (sem passar pelo CNL)."""
        rows = self.session.query(CnToUf.CN, CnToUf.UF).all()
        result = {}
        for cn, uf in rows:
            cn_s = str(cn).strip()
            result[cn_s] = uf
            # Também indexa com zero-padding para facilitar comparação
            try:
                result[str(int(cn_s))] = uf
            except (ValueError, TypeError):
                pass
        return result

    def cnl_count(self) -> int:
        return self.session.query(CnlTable).count()

    def cn_to_uf_count(self) -> int:
        return self.session.query(CnToUf).count()

    # ── Importações ───────────────────────────────────────────────────────

    def save_import(self, source: str, filename: str, sheet: Optional[str],
                    file_hash: str, df: pd.DataFrame,
                    column_map: Dict[str, str]) -> str:
        import_id = new_uuid()
        self.session.add(Import(
            id=import_id, source=source, filename=filename,
            sheet=sheet, file_hash=file_hash,
            rows=len(df), cols=len(df.columns),
        ))
        for orig, norm in column_map.items():
            self.session.add(ColumnRename(
                import_id=import_id, original_name=orig, normalized_name=norm))

        RawModel = {"SCIENCE": RawScience, "PORTAL": RawPortal}.get(source)
        if RawModel:
            records = df.to_dict(orient="records")
            for i in range(0, len(records), _CHUNK):
                for j, row in enumerate(records[i:i+_CHUNK]):
                    self.session.add(RawModel(
                        import_id=import_id, row_num=i+j,
                        data_json=json.dumps(row, ensure_ascii=False, default=str)))
                self.session.flush()

        log.info("Import salvo: %s %s rows=%d", source, import_id[:8], len(df))
        return import_id

    def save_arq3(self, import_id: str, df: pd.DataFrame,
                  col_map: Dict[str, str]) -> int:
        """Persiste Arquivo 3 em ref_rotas."""
        # Mapeamento flexível de nomes de coluna
        def _find(possible: List[str]) -> str:
            for p in possible:
                for c in df.columns:
                    if c.strip().upper() == p.upper():
                        return c
            return ""

        rede_col = _find(["REDE"])
        uf_col   = _find(["UF"])
        cl_col   = _find(["CLUSTER"])
        tr_col   = _find(["TIPO DE ROTA", "TIPO_DE_ROTA", "TIPO_ROTA"])
        ce_col   = _find(["CENTRAL"])
        rl_col   = _find(["RÓTULOS DE LINHA", "ROTULOS DE LINHA", "ROTULOS_DE_LINHA", "LABEL_E"])
        op_col   = _find(["OPERADORA"])
        dn_col   = _find(["DENOMINAÇÃO", "DENOMINACAO"])

        output_cols = [rede_col, uf_col, cl_col, tr_col, ce_col, rl_col, op_col, dn_col]
        extra_cols  = [c for c in df.columns if c not in output_cols]

        count = 0
        for i in range(0, len(df), _CHUNK):
            batch = df.iloc[i:i+_CHUNK]
            for j, (_, row) in enumerate(batch.iterrows()):
                extra = {c: str(row.get(c, "")) for c in extra_cols}
                self.session.add(RefRotas(
                    import_id=import_id, row_num=i+j,
                    REDE             = str(row.get(rede_col, "") or ""),
                    UF               = str(row.get(uf_col, "") or ""),
                    CLUSTER          = str(row.get(cl_col, "") or ""),
                    Tipo_de_Rota     = str(row.get(tr_col, "") or ""),
                    Central          = str(row.get(ce_col, "") or "").strip().upper(),
                    Rotulos_de_Linha = str(row.get(rl_col, "") or ""),
                    OPERADORA        = str(row.get(op_col, "") or ""),
                    Denominacao      = str(row.get(dn_col, "") or ""),
                    extra_json       = json.dumps(extra, ensure_ascii=False),
                ))
                count += 1
            self.session.flush()
        log.info("RefRotas salvo: %d linhas", count)
        return count

    def list_imports(self, source: Optional[str] = None) -> List[Dict]:
        q = self.session.query(Import)
        if source: q = q.filter(Import.source == source)
        return [{"id": r.id, "source": r.source, "filename": r.filename,
                 "sheet": r.sheet, "rows": r.rows, "cols": r.cols,
                 "created_at": r.created_at.isoformat() if r.created_at else ""}
                for r in q.order_by(Import.created_at.desc()).all()]

    def load_raw_df(self, import_id: str, source: str) -> pd.DataFrame:
        RawModel = {"SCIENCE": RawScience, "PORTAL": RawPortal}.get(source.upper())
        if not RawModel: return pd.DataFrame()
        rows = (self.session.query(RawModel)
                .filter(RawModel.import_id == import_id)
                .order_by(RawModel.row_num).all())
        if not rows: return pd.DataFrame()
        return pd.DataFrame([json.loads(r.data_json) for r in rows])

    def load_ref_rotas_df(self, import_id: Optional[str] = None) -> pd.DataFrame:
        """Carrega ref_rotas. Se import_id=None, carrega o mais recente."""
        q = self.session.query(RefRotas)
        if import_id:
            q = q.filter(RefRotas.import_id == import_id)
        else:
            latest_imp = (self.session.query(Import)
                          .filter(Import.source == "ARQ3")
                          .order_by(Import.created_at.desc()).first())
            if latest_imp:
                q = q.filter(RefRotas.import_id == latest_imp.id)
        rows = q.order_by(RefRotas.row_num).all()
        if not rows: return pd.DataFrame()
        return pd.DataFrame([{
            "REDE": r.REDE, "UF": r.UF, "CLUSTER": r.CLUSTER,
            "Tipo de Rota": r.Tipo_de_Rota, "Central": r.Central,
            "Rótulos de Linha": r.Rotulos_de_Linha,
            "OPERADORA": r.OPERADORA, "Denominação": r.Denominacao,
        } for r in rows])

    # ── Merge ─────────────────────────────────────────────────────────────

    def save_merge_version(self, tag: str, sci_id: Optional[str],
                           por_id: Optional[str], arq3_id: Optional[str],
                           mapping: Dict, join_keys: List[str], join_type: str,
                           fuzzy_threshold: int, merged_df: pd.DataFrame,
                           rows_sci: int, rows_por: int) -> str:
        version_id = new_uuid()
        self.session.add(MappingVersion(
            id=version_id, tag=tag,
            science_import_id=sci_id, portal_import_id=por_id,
            arq3_import_id=arq3_id,
            mapping_json=json.dumps(mapping, ensure_ascii=False, default=str),
            join_keys_json=json.dumps(join_keys),
            join_type=join_type, fuzzy_threshold=fuzzy_threshold,
            rows_science=rows_sci, rows_portal=rows_por,
            rows_merged=len(merged_df),
        ))
        _COL = {
            "REDE": "REDE", "UF": "UF", "CLUSTER": "CLUSTER",
            "Tipo de Rota": "Tipo_de_Rota", "Central": "Central",
            "Rótulos de Linha": "Rotulos_de_Linha",
            "OPERADORA": "OPERADORA", "Denominação": "Denominacao",
        }
        for i, row in enumerate(merged_df.to_dict(orient="records")):
            mr = MergedRow(
                version_id=version_id, row_id=i,
                source_tag=str(row.get("_source_tag", "")),
                source_keys_json=json.dumps(row, ensure_ascii=False, default=str),
            )
            for out, db in _COL.items():
                setattr(mr, db, str(row.get(out, "") or ""))
            self.session.add(mr)
            if i % _CHUNK == 0: self.session.flush()
        self.session.flush()
        log.info("Merge salvo: version_id=%s rows=%d", version_id[:8], len(merged_df))
        return version_id

    def list_versions(self) -> List[Dict]:
        rows = (self.session.query(MappingVersion)
                .order_by(MappingVersion.created_at.desc()).all())
        return [{"id": r.id, "tag": r.tag, "rows_merged": r.rows_merged,
                 "rows_science": r.rows_science, "rows_portal": r.rows_portal,
                 "join_type": r.join_type,
                 "created_at": r.created_at.isoformat() if r.created_at else ""}
                for r in rows]

    def load_merged_df(self, version_id: Optional[str] = None) -> pd.DataFrame:
        """Carrega resultado de uma versão. Se None, carrega a mais recente."""
        if version_id is None:
            v = (self.session.query(MappingVersion)
                 .filter(MappingVersion.is_active == True)
                 .order_by(MappingVersion.created_at.desc()).first())
            if v is None: return pd.DataFrame()
            version_id = v.id
        rows = (self.session.query(MergedRow)
                .filter(MergedRow.version_id == version_id)
                .order_by(MergedRow.row_id).all())
        if not rows: return pd.DataFrame()
        return pd.DataFrame([{
            "REDE": r.REDE, "UF": r.UF, "CLUSTER": r.CLUSTER,
            "Tipo de Rota": r.Tipo_de_Rota, "Central": r.Central,
            "Rótulos de Linha": r.Rotulos_de_Linha,
            "OPERADORA": r.OPERADORA, "Denominação": r.Denominacao,
            "_source_tag": r.source_tag,
        } for r in rows])
