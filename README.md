"""
utils/cnl_utils.py
Funções utilitárias para normalização de valores CNL e CN.
Centraliza toda a lógica de limpeza em um único lugar.
"""
from __future__ import annotations
import re
from typing import Optional


def clean_cnl(v) -> str:
    """
    Normaliza um valor CNL/COD_CNL:
    - Remove NaN, None, strings vazias
    - Remove sufixo '.0' de floats lidos do Excel  (41406.0 → '41406')
    - Remove zeros à esquerda                       ('041406' → '41406')
    - Retorna string limpa ou ''
    """
    if v is None:
        return ""
    s = str(v).strip()
    if s.lower() in ("nan", "none", "null", "<na>", "n/a", "", "0"):
        return ""
    # Remove sufixo .0 e .00 etc.
    s = re.sub(r"\.0+$", "", s)
    # Remove zeros à esquerda (mantém pelo menos 1 dígito)
    s = s.lstrip("0") or s
    return s


def clean_cn(v) -> str:
    """
    Normaliza um valor CN (DDD):
    - Remove .0 de floats
    - Remove zeros à esquerda
    - Retorna string ou ''
    """
    if v is None:
        return ""
    s = str(v).strip()
    if s.lower() in ("nan", "none", "null", "<na>", "n/a", ""):
        return ""
    s = re.sub(r"\.0+$", "", s)
    s = s.lstrip("0") or s
    return s


def normalize_cnl_series(series) -> "list[str]":
    """Normaliza uma pd.Series de CNL para lista de strings limpas únicas."""
    return [clean_cnl(v) for v in series if clean_cnl(v)]
