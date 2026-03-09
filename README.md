"""
core/analytics.py
=================
Pré-calcula frequências e KPIs para todas as colunas.
Cache em st.session_state para dashboard instantâneo.
"""
from __future__ import annotations
import pandas as pd
from app.core.hash_utils import BUSINESS_COLS


_TOP_N = 15


def _norm_series(s: pd.Series) -> pd.Series:
    return s.fillna("").astype(str).str.strip().replace("", "(Sem valor)")


def freq_table(df: pd.DataFrame, col: str, top_n: int = _TOP_N) -> pd.DataFrame:
    s     = _norm_series(df[col])
    vc    = s.value_counts()
    total = len(s)
    if len(vc) > top_n:
        top   = vc.iloc[:top_n]
        other = vc.iloc[top_n:].sum()
        vc    = pd.concat([top, pd.Series({"Outros": other})])
    return pd.DataFrame({
        "Categoria":   vc.index,
        "Quantidade":  vc.values,
        "Percentual":  (vc.values / total * 100).round(1),
    }).reset_index(drop=True)


def kpis(df: pd.DataFrame, col: str) -> dict:
    s     = _norm_series(df[col])
    total = len(s)
    nulls = (s == "(Sem valor)").sum()
    real  = s[s != "(Sem valor)"]
    vc    = real.value_counts()
    return {
        "n":        total,
        "pct_null": round(nulls / total * 100, 1) if total else 0,
        "unique":   real.nunique(),
        "pct_top1": round(vc.iloc[0] / total * 100, 1) if len(vc) else 0,
        "pct_top3": round(vc.iloc[:3].sum() / total * 100, 1) if len(vc) else 0,
    }


def precompute_all(df: pd.DataFrame) -> dict:
    """Retorna dict col → {kpis, freq} para todas as colunas de negócio."""
    cache = {}
    for col in BUSINESS_COLS:
        if col in df.columns:
            cache[col] = {
                "kpis": kpis(df, col),
                "freq": freq_table(df, col),
            }
    return cache
