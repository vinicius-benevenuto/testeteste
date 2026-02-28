"""utils/ids.py — UUIDs, hash SHA-256, version tags."""
from __future__ import annotations
import hashlib, uuid
from datetime import datetime, timezone

def new_uuid() -> str: return str(uuid.uuid4())
def file_hash(b: bytes) -> str: return hashlib.sha256(b).hexdigest()
def version_tag() -> str: return datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S")
def short_id() -> str: return uuid.uuid4().hex[:8]
