"""
ui/dashboard.py
===============
Renderiza o dashboard de uma coluna: KPIs + gráficos + tabela de frequência.
Usa apenas Streamlit nativo (sem plotly/altair) para máxima compatibilidade.
"""
from __future__ import annotations
import pandas as pd
import streamlit as st
from app.core.analytics import freq_table, kpis, precompute_all
from app.core.hash_utils import BUSINESS_COLS

_ACCENT = "#1B5FBF"
_OK     = "#16A34A"
_WARN   = "#D97706"
_FAINT  = "#9CA3AF"
_TEXT   = "#111827"
_SURFACE= "#FFFFFF"
_BORDER = "#E2E8F0"


def _kpi_row(kpi: dict, col: str) -> None:
    c1, c2, c3, c4, c5 = st.columns(5)
    c1.metric("Registros",  f"{kpi['n']:,}")
    c2.metric("% Nulos",    f"{kpi['pct_null']:.1f}%")
    c3.metric("Únicos",     f"{kpi['unique']:,}")
    c4.metric("% Top 1",    f"{kpi['pct_top1']:.1f}%")
    c5.metric("% Top 3",    f"{kpi['pct_top3']:.1f}%")


def _bar_chart(freq: pd.DataFrame, col: str) -> None:
    """Gráfico de barras horizontal nativo."""
    chart_df = freq.set_index("Categoria")[["Quantidade"]]
    st.bar_chart(chart_df, use_container_width=True, height=320, color=_ACCENT)


def _freq_table(freq: pd.DataFrame) -> None:
    """Tabela de frequência estilizada."""
    st.markdown(
        f'<div style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        f'text-transform:uppercase;color:{_FAINT};margin:12px 0 5px;">Frequências</div>',
        unsafe_allow_html=True,
    )
    display = freq.copy()
    display["Percentual"] = display["Percentual"].apply(lambda x: f"{x:.1f}%")
    display["Quantidade"] = display["Quantidade"].apply(lambda x: f"{int(x):,}")
    st.dataframe(display, use_container_width=True, hide_index=True,
                 column_config={
                     "Categoria":  st.column_config.TextColumn("Categoria",  width="large"),
                     "Quantidade": st.column_config.TextColumn("Quantidade", width="small"),
                     "Percentual": st.column_config.TextColumn("%",          width="small"),
                 })


def _col_insight(col: str, kpi: dict, freq: pd.DataFrame) -> str:
    top1 = freq.iloc[0]["Categoria"] if len(freq) else "—"
    top1_pct = freq.iloc[0]["Percentual"] if len(freq) else 0
    insights = {
        "REDE":             f"Distribuição VIVO-SMP vs. VIVO-STFC. Dominante: **{top1}** ({top1_pct:.1f}%).",
        "UF":               f"Ranking por estado. Maior concentração em **{top1}** ({top1_pct:.1f}%).",
        "CLUSTER":          f"{kpi['unique']} clusters distintos. **{kpi['pct_null']:.1f}%** sem valor.",
        "Tipo de Rota":     f"Mix INTERNA vs. ITX-SIP_AS. Tipo dominante: **{top1}** ({top1_pct:.1f}%).",
        "Central":          f"Top 3 centrais concentram **{kpi['pct_top3']:.1f}%** das rotas.",
        "Rótulos de Linha": f"{kpi['unique']:,} rótulos únicos. Top 1: **{top1}** ({top1_pct:.1f}%).",
        "OPERADORA":        f"Participação por operadora. Líder: **{top1}** ({top1_pct:.1f}%).",
        "Denominação":      f"{kpi['unique']:,} denominações distintas. **{kpi['pct_null']:.1f}%** sem valor.",
    }
    return insights.get(col, f"Top 1: **{top1}** ({top1_pct:.1f}%). Únicos: {kpi['unique']:,}.")


def render_column_dashboard(
    df: pd.DataFrame,
    col: str,
    cache: dict | None = None,
) -> None:
    """Renderiza o painel completo de uma coluna."""
    if col not in df.columns:
        st.warning(f"Coluna '{col}' não encontrada.")
        return

    # Usa cache pré-calculado se disponível
    if cache and col in cache:
        k  = cache[col]["kpis"]
        fr = cache[col]["freq"]
    else:
        k  = kpis(df, col)
        fr = freq_table(df, col)

    # Cabeçalho do dashboard
    st.markdown(
        f'<div style="background:#0F172A;border-left:4px solid {_ACCENT};'
        f'padding:10px 16px;margin-bottom:14px;">'
        f'<span style="font-size:15px;font-weight:700;color:#F1F5F9;">'
        f'Dashboard — {col}</span></div>',
        unsafe_allow_html=True,
    )

    # KPIs
    _kpi_row(k, col)

    # Insight automático
    insight = _col_insight(col, k, fr)
    st.markdown(
        f'<div style="background:{_SURFACE};border-left:3px solid {_ACCENT};'
        f'padding:9px 14px;margin:10px 0;font-size:12px;color:{_TEXT};">'
        f'{insight}</div>',
        unsafe_allow_html=True,
    )

    # Gráfico + Tabela lado a lado
    gc, tc = st.columns([3, 2], gap="medium")
    with gc:
        _bar_chart(fr, col)
    with tc:
        _freq_table(fr)
