"""utils/logging_utils.py — Logger estruturado."""
from __future__ import annotations
import json, logging, sys
from typing import Any, Dict, Optional

def get_logger(name: str = "data_merger") -> logging.Logger:
    logger = logging.getLogger(name)
    if not logger.handlers:
        h = logging.StreamHandler(sys.stdout)
        h.setFormatter(logging.Formatter(
            "%(asctime)s [%(levelname)s] %(name)s: %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S"))
        logger.addHandler(h)
        logger.setLevel(logging.DEBUG)
    return logger

def build_context(**kw: Any) -> str:
    return json.dumps(kw, default=str, ensure_ascii=False)
