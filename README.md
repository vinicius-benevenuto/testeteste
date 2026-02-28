"""
ui/grid.py
AG Grid com floatingFilter (caixas de filtro abaixo do cabeçalho).
Fallback para filtros nativos Streamlit se st-aggrid não estiver instalado.
"""
from __future__ import annotations
from typing import Dict, List, Optional
import pandas as pd
import streamlit as st

try:
    from st_aggrid import AgGrid, GridOptionsBuilder, GridUpdateMode, DataReturnMode, JsCode  # type: ignore
    _HAS_AGGRID = True
except ImportError:
    _HAS_AGGRID = False


# ── AG Grid — configuração completa com floating filters ─────────────────

def _build_grid_options(df: pd.DataFrame, height: int = 520) -> dict:
    """Constrói gridOptions com floating filters estilo Excel."""
    gb = GridOptionsBuilder.from_dataframe(df)

    # Coluna padrão: filtro + floating filter + ordenação
    gb.configure_default_column(
        filter=True,
        floatingFilter=True,          # ← caixa abaixo do cabeçalho
        sortable=True,
        resizable=True,
        editable=False,
        wrapText=False,
        autoHeight=False,
        suppressMenu=False,
        filterParams={
            "filterOptions": [
                "contains", "notContains",
                "equals", "notEqual",
                "startsWith", "endsWith",
                "blank", "notBlank",
            ],
            "defaultOption": "contains",
            "suppressAndOrCondition": False,
            "debounceMs": 200,
        },
    )

    # Colunas específicas com configuração de filtro por tipo
    for col in df.columns:
        col_lower = col.lower()
        if any(k in col_lower for k in ("data", "dt_", "ativacao")):
            gb.configure_column(col, type=["dateColumnFilter"],
                                filterParams={"comparator": "dateComparator"})
        elif col.startswith("_"):
            gb.configure_column(col, hide=True)  # oculta colunas auxiliares

    # Paginação
    gb.configure_pagination(paginationAutoPageSize=False, paginationPageSize=100)

    # Seleção múltipla
    gb.configure_selection(
        selection_mode="multiple",
        use_checkbox=True,
        header_checkbox=True,
    )

    # Sidebar com painéis de filtros e colunas
    gb.configure_side_bar(filters_panel=True, columns_panel=True)

    # Grid options adicionais
    gb.configure_grid_options(
        domLayout="normal",
        rowHeight=28,
        headerHeight=38,
        floatingFiltersHeight=38,     # ← altura da linha de filtros
        enableRangeSelection=True,
        copyHeadersToClipboard=True,
        clipboardDelimiter="\t",      # compatível com Excel
        suppressCopyRowsToClipboard=False,
        tooltipShowDelay=300,
        animateRows=True,
        rowBuffer=20,
        enableCellTextSelection=True,
    )

    return gb.build()


def show_aggrid(df: pd.DataFrame, key: str = "grid",
                height: int = 520) -> pd.DataFrame:
    """Exibe AG Grid com floating filters. Retorna DataFrame filtrado."""
    if df.empty:
        st.info("Nenhum dado para exibir.")
        return df

    grid_options = _build_grid_options(df, height)

    result = AgGrid(
        df,
        gridOptions=grid_options,
        update_mode=GridUpdateMode.FILTERING_CHANGED,
        data_return_mode=DataReturnMode.FILTERED_AND_SORTED,
        fit_columns_on_grid_load=False,
        height=height,
        key=key,
        allow_unsafe_jscode=True,
        enable_enterprise_modules=False,
        theme="streamlit",
    )

    filtered: pd.DataFrame = result["data"]
    return filtered if filtered is not None else df


# ── Fallback nativo ───────────────────────────────────────────────────────

def _native_grid(df: pd.DataFrame, key: str = "grid") -> pd.DataFrame:
    """Filtros nativos do Streamlit quando st-aggrid não está instalado."""
    display_cols = [c for c in df.columns if not c.startswith("_")]
    df_disp = df[display_cols].copy()

    st.markdown("**🔍 Filtros por coluna** (equivalente a floating filters):")

    n_per_row = 4
    groups = [display_cols[i:i+n_per_row]
              for i in range(0, len(display_cols), n_per_row)]
    filters: Dict[str, str] = {}
    for grp in groups:
        cols_ui = st.columns(len(grp))
        for i, col in enumerate(grp):
            with cols_ui[i]:
                v = st.text_input(col, key=f"{key}_f_{col}",
                                  placeholder="filtrar...",
                                  label_visibility="visible")
                filters[col] = v.strip()

    # Busca global
    qs = st.text_input("🔎 Busca global em todas as colunas",
                        key=f"{key}_global", placeholder="Pesquisa rápida...")

    filtered = df_disp.copy()
    for col, val in filters.items():
        if val:
            filtered = filtered[
                filtered[col].astype(str).str.contains(val, case=False, na=False)]
    if qs:
        mask = filtered.apply(
            lambda r: r.astype(str).str.contains(qs, case=False, na=False).any(), axis=1)
        filtered = filtered[mask]

    # Ordenação
    sort_col = st.selectbox("Ordenar por:", ["(nenhum)"] + display_cols,
                             key=f"{key}_sort")
    if sort_col != "(nenhum)":
        asc = st.radio("Direção:", ["↑ Crescente", "↓ Decrescente"],
                        horizontal=True, key=f"{key}_sortdir") == "↑ Crescente"
        filtered = filtered.sort_values(sort_col, ascending=asc)

    st.dataframe(filtered, use_container_width=True, height=520, hide_index=True)
    return filtered


# ── API pública ────────────────────────────────────────────────────────────

def show_grid(df: pd.DataFrame, key: str = "grid",
              title: str = "", height: int = 520) -> pd.DataFrame:
    """
    Exibe grid interativo. Usa AG Grid com floatingFilter se disponível,
    senão usa filtros nativos do Streamlit.
    Retorna o DataFrame após filtros (para exportação fiel ao que o usuário vê).
    """
    if title:
        st.subheader(title)

    total = len(df)
    st.caption(
        f"📊 {total:,} linhas · {len(df.columns)} colunas"
        + (" · **AG Grid**" if _HAS_AGGRID else " · filtros nativos")
    )

    if df.empty:
        st.info("Nenhum dado para exibir.")
        return df

    if _HAS_AGGRID:
        filtered = show_aggrid(df, key=key, height=height)
    else:
        st.warning(
            "⚠️ `streamlit-aggrid` não instalado. "
            "Instale com `pip install streamlit-aggrid` para filtros estilo Excel avançados."
        )
        filtered = _native_grid(df, key=key)

    if len(filtered) < total:
        st.caption(f"🔽 {len(filtered):,} linhas visíveis (de {total:,} totais)")

    return filtered
