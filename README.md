"""
ui/grid.py
Grid interativo com filtros por coluna estilo Excel.
Implementado com Streamlit nativo + session_state para compatibilidade máxima.
Se st-aggrid estiver disponível, usa ele; caso contrário, usa fallback nativo.
"""
from __future__ import annotations
from typing import Dict, List, Optional, Tuple

import pandas as pd
import streamlit as st

# Tenta importar st-aggrid; usa fallback se indisponível
try:
    from st_aggrid import AgGrid, GridOptionsBuilder, GridUpdateMode, DataReturnMode  # type: ignore
    _HAS_AGGRID = True
except ImportError:
    _HAS_AGGRID = False


def _native_filter_grid(df: pd.DataFrame, key_prefix: str = "grid") -> pd.DataFrame:
    """
    Grid com filtros nativos do Streamlit.
    Cada coluna tem um input de busca livre (texto).
    Suporta operadores: contém, começa com, igual, não vazio.
    """
    st.markdown("**🔍 Filtros por coluna** — Digite para filtrar cada campo:")

    cols = list(df.columns)
    n_filter_cols = min(4, len(cols))  # máximo 4 filtros por linha
    filter_values: Dict[str, str] = {}

    groups = [cols[i:i + n_filter_cols] for i in range(0, len(cols), n_filter_cols)]
    for group in groups:
        ui_cols = st.columns(len(group))
        for i, col in enumerate(group):
            with ui_cols[i]:
                val = st.text_input(
                    col,
                    key=f"{key_prefix}_filt_{col}",
                    placeholder="filtrar...",
                    label_visibility="visible",
                )
                filter_values[col] = val.strip()

    # Busca global
    global_search = st.text_input(
        "🔎 Busca em todas as colunas",
        key=f"{key_prefix}_global",
        placeholder="Pesquisa rápida...",
    )

    # Aplicar filtros
    filtered = df.copy()
    for col, val in filter_values.items():
        if val:
            filtered = filtered[
                filtered[col].astype(str).str.contains(val, case=False, na=False)
            ]

    if global_search:
        mask = filtered.apply(
            lambda row: row.astype(str).str.contains(global_search, case=False, na=False).any(),
            axis=1,
        )
        filtered = filtered[mask]

    return filtered


def _aggrid_filter_grid(df: pd.DataFrame, key_prefix: str = "grid") -> pd.DataFrame:
    """Grid com st-aggrid — filtros estilo Excel completos."""
    gb = GridOptionsBuilder.from_dataframe(df)
    gb.configure_default_column(
        filter=True, sortable=True, resizable=True,
        editable=False, groupable=True,
        filterParams={"filterOptions": ["contains", "equals", "startsWith",
                                        "endsWith", "notContains"],
                      "suppressAndOrCondition": False},
    )
    gb.configure_selection(selection_mode="multiple", use_checkbox=True)
    gb.configure_pagination(paginationAutoPageSize=False, paginationPageSize=100)
    gb.configure_side_bar(filters_panel=True, columns_panel=True)
    gb.configure_grid_options(
        domLayout="normal",
        rowHeight=28,
        headerHeight=40,
        suppressMovableColumns=False,
        enableRangeSelection=True,
    )

    grid_options = gb.build()
    result = AgGrid(
        df,
        gridOptions=grid_options,
        update_mode=GridUpdateMode.FILTERING_CHANGED,
        data_return_mode=DataReturnMode.FILTERED_AND_SORTED,
        fit_columns_on_grid_load=False,
        height=500,
        key=f"{key_prefix}_aggrid",
        allow_unsafe_jscode=False,
    )
    return result["data"]


def show_grid(df: pd.DataFrame, key_prefix: str = "grid",
              title: str = "") -> pd.DataFrame:
    """
    Exibe grid com filtros. Usa st-aggrid se disponível, senão fallback nativo.
    Retorna o DataFrame filtrado/selecionado.
    """
    if title:
        st.subheader(title)

    total_rows = len(df)
    st.caption(f"📊 {total_rows:,} linhas · {len(df.columns)} colunas")

    if df.empty:
        st.info("Nenhum dado para exibir.")
        return df

    if _HAS_AGGRID:
        filtered = _aggrid_filter_grid(df, key_prefix)
    else:
        filtered = _native_filter_grid(df, key_prefix)
        # Ordenação nativa
        sort_col = st.selectbox(
            "Ordenar por:", ["(sem ordenação)"] + list(df.columns),
            key=f"{key_prefix}_sort_col",
        )
        if sort_col != "(sem ordenação)":
            asc = st.radio("Direção:", ["Crescente", "Decrescente"],
                           horizontal=True, key=f"{key_prefix}_sort_dir")
            filtered = filtered.sort_values(sort_col, ascending=(asc == "Crescente"))

        st.dataframe(
            filtered,
            use_container_width=True,
            height=520,
            hide_index=True,
        )

    filtered_rows = len(filtered)
    if filtered_rows < total_rows:
        st.caption(f"🔽 {filtered_rows:,} linhas após filtros (de {total_rows:,})")

    return filtered
