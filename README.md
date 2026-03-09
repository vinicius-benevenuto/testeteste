"""db/models.py — Modelos SQLAlchemy. Nunca deletar dados."""
from __future__ import annotations
import json
from datetime import datetime, timezone
from sqlalchemy import (Boolean, Column, DateTime, ForeignKey,
                        Integer, String, Text)
from sqlalchemy.orm import DeclarativeBase, relationship

def _now() -> datetime:
    return datetime.now(timezone.utc)

class Base(DeclarativeBase):
    pass

# ── Importações ───────────────────────────────────────────────────────────

class Import(Base):
    __tablename__ = "imports"
    id         = Column(String(36), primary_key=True)
    source     = Column(String(20), nullable=False)   # SCIENCE|PORTAL|ARQ3
    filename   = Column(String(255), nullable=False)
    sheet      = Column(String(100))
    file_hash  = Column(String(64), nullable=False)
    rows       = Column(Integer, nullable=False)
    cols       = Column(Integer, nullable=False)
    created_at = Column(DateTime, default=_now, nullable=False)
    is_active  = Column(Boolean, default=True)

    column_renames = relationship("ColumnRename", back_populates="imp",
                                  cascade="all, delete-orphan")
    raw_science    = relationship("RawScience",   back_populates="imp",
                                  cascade="all, delete-orphan")
    raw_portal     = relationship("RawPortal",    back_populates="imp",
                                  cascade="all, delete-orphan")
    ref_rotas      = relationship("RefRotas",     back_populates="imp",
                                  cascade="all, delete-orphan")

class ColumnRename(Base):
    __tablename__ = "column_renames"
    id              = Column(Integer, primary_key=True, autoincrement=True)
    import_id       = Column(String(36), ForeignKey("imports.id"), nullable=False)
    original_name   = Column(String(255), nullable=False)
    normalized_name = Column(String(255), nullable=False)
    imp = relationship("Import", back_populates="column_renames")

# ── Dados brutos ──────────────────────────────────────────────────────────

class RawScience(Base):
    __tablename__ = "raw_science"
    id        = Column(Integer, primary_key=True, autoincrement=True)
    import_id = Column(String(36), ForeignKey("imports.id"), nullable=False)
    row_num   = Column(Integer, nullable=False)
    data_json = Column(Text, nullable=False)
    created_at = Column(DateTime, default=_now)
    imp = relationship("Import", back_populates="raw_science")

    @property
    def data(self) -> dict:
        return json.loads(self.data_json)

class RawPortal(Base):
    __tablename__ = "raw_portal"
    id        = Column(Integer, primary_key=True, autoincrement=True)
    import_id = Column(String(36), ForeignKey("imports.id"), nullable=False)
    row_num   = Column(Integer, nullable=False)
    data_json = Column(Text, nullable=False)
    created_at = Column(DateTime, default=_now)
    imp = relationship("Import", back_populates="raw_portal")

    @property
    def data(self) -> dict:
        return json.loads(self.data_json)

class RefRotas(Base):
    """Arquivo 3 — tabela de referência para CLUSTER/Rótulos/Denominação."""
    __tablename__ = "ref_rotas"
    id               = Column(Integer, primary_key=True, autoincrement=True)
    import_id        = Column(String(36), ForeignKey("imports.id"), nullable=False)
    row_num          = Column(Integer, nullable=False)
    REDE             = Column(Text)
    UF               = Column(Text)
    CLUSTER          = Column(Text)
    Tipo_de_Rota     = Column(Text)
    Central          = Column(Text)
    Rotulos_de_Linha = Column(Text)
    OPERADORA        = Column(Text)
    Denominacao      = Column(Text)
    extra_json       = Column(Text, default="{}")  # demais colunas
    created_at       = Column(DateTime, default=_now)
    imp = relationship("Import", back_populates="ref_rotas")

# ── Tabelas de referência CNL ─────────────────────────────────────────────

class CnlTable(Base):
    """COD_CNL → CN (código numérico do estado)."""
    __tablename__ = "cnl"
    COD_CNL = Column(String(20), primary_key=True)
    CN      = Column(String(10), nullable=False)

class CnToUf(Base):
    """CN → UF (sigla do estado)."""
    __tablename__ = "cn_to_uf"
    CN = Column(String(10), primary_key=True)
    UF = Column(String(2), nullable=False)

# ── Versões de merge ──────────────────────────────────────────────────────

class MappingVersion(Base):
    __tablename__ = "mapping_versions"
    id                 = Column(String(36), primary_key=True)
    tag                = Column(String(50), nullable=False)
    science_import_id  = Column(String(36), ForeignKey("imports.id"))
    portal_import_id   = Column(String(36), ForeignKey("imports.id"))
    arq3_import_id     = Column(String(36), ForeignKey("imports.id"))
    mapping_json       = Column(Text, nullable=False)
    join_keys_json     = Column(Text, nullable=False)
    join_type          = Column(String(20), default="outer")
    fuzzy_threshold    = Column(Integer, default=90)
    rows_science       = Column(Integer, default=0)
    rows_portal        = Column(Integer, default=0)
    rows_merged        = Column(Integer, default=0)
    created_at         = Column(DateTime, default=_now, nullable=False)
    is_active          = Column(Boolean, default=True)

    merged_rows = relationship("MergedRow", back_populates="version",
                               cascade="all, delete-orphan")

class MergedRow(Base):
    __tablename__ = "merged_results"
    id               = Column(Integer, primary_key=True, autoincrement=True)
    version_id       = Column(String(36), ForeignKey("mapping_versions.id"), nullable=False)
    row_id           = Column(Integer, nullable=False)
    REDE             = Column(Text)
    UF               = Column(Text)
    CLUSTER          = Column(Text)
    Tipo_de_Rota     = Column(Text)
    Central          = Column(Text)
    Rotulos_de_Linha = Column(Text)
    OPERADORA        = Column(Text)
    Denominacao      = Column(Text)
    source_tag       = Column(Text)   # SCIENCE | PORTAL | BOTH
    source_keys_json = Column(Text)
    created_at       = Column(DateTime, default=_now)
    version = relationship("MappingVersion", back_populates="merged_rows")

class LogEntry(Base):
    __tablename__ = "logs"
    id           = Column(Integer, primary_key=True, autoincrement=True)
    timestamp    = Column(DateTime, default=_now, nullable=False)
    level        = Column(String(10), nullable=False)
    message      = Column(Text, nullable=False)
    context_json = Column(Text, default="{}")


# ── Tabela Final (schema de negócio + deduplicação por hash_rota) ─────────

class TabelaFinal(Base):
    """Tabela de rotas consolidadas. hash_rota garante unicidade."""
    __tablename__ = "tabela_final"

    id               = Column(Integer, primary_key=True, autoincrement=True)
    hash_rota        = Column(String(40), unique=True, nullable=False, index=True)
    rede             = Column(Text)
    uf               = Column(Text)
    cluster          = Column(Text)
    tipo_de_rota     = Column(Text)
    central          = Column(Text)
    rotulos_de_linha = Column(Text)
    operadora        = Column(Text)
    denominacao      = Column(Text)
    created_at       = Column(DateTime, default=_now)

