"""
db/models.py
Modelos SQLAlchemy. Estratégia: NUNCA deletar — usar flags is_active e versionamento.
"""
from __future__ import annotations
import json
from datetime import datetime, timezone

from sqlalchemy import (
    Boolean, Column, DateTime, ForeignKey, Integer, String, Text,
)
from sqlalchemy.orm import DeclarativeBase, relationship


def _now() -> datetime:
    return datetime.now(timezone.utc)


class Base(DeclarativeBase):
    pass


class Import(Base):
    """Registro de cada arquivo importado."""
    __tablename__ = "imports"
    id         = Column(String(36), primary_key=True)
    source     = Column(String(20), nullable=False)   # "science" | "portal"
    filename   = Column(String(255), nullable=False)
    sheet      = Column(String(100))
    file_hash  = Column(String(64), nullable=False)
    rows       = Column(Integer, nullable=False)
    cols       = Column(Integer, nullable=False)
    created_at = Column(DateTime, default=_now, nullable=False)
    is_active  = Column(Boolean, default=True, nullable=False)

    column_renames = relationship("ColumnRename", back_populates="imp",
                                  cascade="all, delete-orphan")
    raw_science    = relationship("RawScience",   back_populates="imp",
                                  cascade="all, delete-orphan")
    raw_portal     = relationship("RawPortal",    back_populates="imp",
                                  cascade="all, delete-orphan")


class ColumnRename(Base):
    """Mapeamento original → normalizado por importação."""
    __tablename__ = "column_renames"
    id              = Column(Integer, primary_key=True, autoincrement=True)
    import_id       = Column(String(36), ForeignKey("imports.id"), nullable=False)
    original_name   = Column(String(255), nullable=False)
    normalized_name = Column(String(255), nullable=False)
    imp = relationship("Import", back_populates="column_renames")


class RawScience(Base):
    """Linhas brutas da tabela Science — preservação 100%."""
    __tablename__ = "raw_science"
    id        = Column(Integer, primary_key=True, autoincrement=True)
    import_id = Column(String(36), ForeignKey("imports.id"), nullable=False)
    row_num   = Column(Integer, nullable=False)
    data_json = Column(Text, nullable=False)
    imp = relationship("Import", back_populates="raw_science")

    @property
    def data(self) -> dict:
        return json.loads(self.data_json)


class RawPortal(Base):
    """Linhas brutas da tabela Portal — preservação 100%."""
    __tablename__ = "raw_portal"
    id        = Column(Integer, primary_key=True, autoincrement=True)
    import_id = Column(String(36), ForeignKey("imports.id"), nullable=False)
    row_num   = Column(Integer, nullable=False)
    data_json = Column(Text, nullable=False)
    imp = relationship("Import", back_populates="raw_portal")

    @property
    def data(self) -> dict:
        return json.loads(self.data_json)


class MappingVersion(Base):
    """Versão de um conjunto de regras de mapeamento + junção."""
    __tablename__ = "mapping_versions"
    id                  = Column(String(36), primary_key=True)
    tag                 = Column(String(50), nullable=False)
    science_import_id   = Column(String(36), ForeignKey("imports.id"))
    portal_import_id    = Column(String(36), ForeignKey("imports.id"))
    mapping_json        = Column(Text, nullable=False)
    join_keys_json      = Column(Text, nullable=False)
    join_type           = Column(String(20), default="outer")
    fuzzy_threshold     = Column(Integer, default=90)
    rows_science        = Column(Integer, default=0)
    rows_portal         = Column(Integer, default=0)
    rows_merged         = Column(Integer, default=0)
    created_at          = Column(DateTime, default=_now, nullable=False)
    is_active           = Column(Boolean, default=True, nullable=False)

    merged_rows = relationship("MergedRow", back_populates="version",
                               cascade="all, delete-orphan")


class MergedRow(Base):
    """Uma linha do resultado final do merge."""
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
    source_keys_json = Column(Text)
    version = relationship("MappingVersion", back_populates="merged_rows")


class LogEntry(Base):
    """Log persistido no banco."""
    __tablename__ = "logs"
    id           = Column(Integer, primary_key=True, autoincrement=True)
    timestamp    = Column(DateTime, default=_now, nullable=False)
    level        = Column(String(10), nullable=False)
    message      = Column(Text, nullable=False)
    context_json = Column(Text, default="{}")
