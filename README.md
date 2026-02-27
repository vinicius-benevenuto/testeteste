"""utils/ids.py — Geração de IDs únicos e hash de arquivos."""
from __future__ import annotations
import hashlib, uuid
from datetime import datetime, timezone

def new_uuid() -> str:
    return str(uuid.uuid4())

def file_hash(content: bytes) -> str:
    return hashlib.sha256(content).hexdigest()

def version_tag() -> str:
    return datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S")

def short_id() -> str:
    return uuid.uuid4().hex[:8]
