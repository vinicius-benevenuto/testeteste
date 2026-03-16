"""
app.py — ITX Analisys 1.0
==========================
Correções definitivas:
  [1] CSS totalmente reescrito sem regra global de cor que quebre headings.
      h1-h6 no conteúdo claro = #1C2536. Custom headers escuros = inline style.
  [2] Botão colapso sidebar: "font-size:0" esconde texto Material Icons sem
      afetar o SVG (que ignora font-size). Sem `:has()` — compatível com todos
      os browsers.
  [3] Filtros estilo Excel: painel com 2 linhas × 4 colunas de multiselect,
      totalmente incremental — cada dropdown reflete somente valores presentes
      após os outros filtros ativos.
  [4] Exports sempre abaixo da tabela e do dashboard.
  [5] Excel profissional via openpyxl puro.
  [6] PPTX via pptx_builder v3 (sem bugs de format code).
"""
from __future__ import annotations

import io as _io
import os
from datetime import datetime
from pathlib import Path

import pandas as pd
import streamlit as st

# ─── Configuração da página ────────────────────────────────────────────────────
st.set_page_config(
    page_title="ITX Analisys",
    layout="wide",
    initial_sidebar_state="expanded",
)

try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass

# ─── Banco ────────────────────────────────────────────────────────────────────
from app.db.session import init_db, session_scope
from app.db.repository import Repository
from app.db.tf_repository import TabelaFinalRepository

_DB = os.environ.get("DATABASE_PATH", str(Path("data") / "app.db"))
init_db(_DB)

def _auto_seeds() -> None:
    try:
        with session_scope() as s:
            repo = Repository(s)
            if repo.cnl_count() == 0:
                p = Path("seeds") / "cnl.sql"
                if p.exists() and p.stat().st_size:
                    repo.load_cnl_seeds(str(p))
            if repo.cn_to_uf_count() == 0:
                p = Path("seeds") / "cn_to_uf.csv"
                if p.exists() and p.stat().st_size:
                    repo.load_cn_to_uf_csv(str(p))
    except Exception:
        pass

_auto_seeds()

_CSS = """
<style>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@300;400;500;600;700&display=swap');

html, body,
.stApp,
[data-testid="stAppViewContainer"],
[data-testid="stMain"],
section[data-testid="stMain"],
.main, .block-container {
  background-color: #F8F9FA !important;
  font-family: 'IBM Plex Sans', 'Segoe UI', Arial, sans-serif !important;
}
.main .block-container {
  padding-top:    0 !important;
  padding-bottom: 2.5rem !important;
  max-width:      100% !important;
}

body, .main { color: #1C2536; }
h1, h2, h3, h4, h5, h6 { color: #1C2536; }
.main p,
[data-testid="stMarkdownContainer"] p { color: #374151; }

[data-testid="stSidebar"] > div:first-child,
[data-testid="stSidebarUserContent"] {
  background-color: #0F172A !important;
}
[data-testid="stSidebarUserContent"] p,
[data-testid="stSidebarUserContent"] span,
[data-testid="stSidebarUserContent"] label {
  color: #CBD5E1;
  font-family: 'IBM Plex Sans', sans-serif;
}
[data-testid="stSidebar"] hr {
  border-color: #1E293B !important;
  margin: 8px 0 !important;
}

[data-testid="stSidebarUserContent"] [data-testid="baseButton-secondary"] {
  background:     transparent !important;
  border:         none !important;
  border-left:    2px solid transparent !important;
  border-radius:  0 !important;
  color:          #94A3B8 !important;
  font-size:      12px !important;
  font-weight:    500 !important;
  text-align:     left !important;
  justify-content:flex-start !important;
  padding:        7px 12px !important;
  width:          100% !important;
  margin:         1px 0 !important;
  letter-spacing: normal !important;
  text-transform: none !important;
}
[data-testid="stSidebarUserContent"] [data-testid="baseButton-secondary"]:hover {
  background:  rgba(255,255,255,.06) !important;
  color:       #F1F5F9 !important;
  border-left: 2px solid #475569 !important;
}
[data-testid="stSidebarUserContent"] [data-testid="baseButton-secondary"]:last-of-type {
  background:     transparent !important;
  border:         1px solid #1E293B !important;
  border-radius:  3px !important;
  color:          #475569 !important;
  font-size:      10px !important;
  margin-top:     4px !important;
  text-align:     center !important;
  justify-content:center !important;
}
[data-testid="stSidebarUserContent"] button {
  background:     transparent !important;
  border:         1px solid #1E293B !important;
  color:          #475569 !important;
  font-size:      10px !important;
  font-weight:    500 !important;
  width:          100% !important;
  padding:        5px 10px !important;
  border-radius:  3px !important;
  text-transform: none !important;
  letter-spacing: normal !important;
}
[data-testid="stSidebarUserContent"] button:hover {
  border-color: #334155 !important;
  color:        #94A3B8 !important;
}

[data-testid="stSidebarCollapseButton"] {
  visibility: visible !important;
  display:    flex !important;
}
[data-testid="stSidebarCollapseButton"] button {
  display:         flex !important;
  align-items:     center !important;
  justify-content: center !important;
  visibility:      visible !important;
  font-size:       0 !important;
  background:      transparent !important;
  border:          none !important;
  border-radius:   4px !important;
  width:           32px !important;
  height:          32px !important;
  cursor:          pointer !important;
  padding:         0 !important;
}
[data-testid="stSidebarCollapseButton"] button:hover {
  background: rgba(255,255,255,.1) !important;
}
[data-testid="stSidebarCollapseButton"] svg {
  display:    block !important;
  visibility: visible !important;
  fill:       #64748B !important;
  width:      18px !important;
  height:     18px !important;
}
[data-testid="stSidebarCollapseButton"] button:hover svg {
  fill: #94A3B8 !important;
}
[data-testid="collapsedControl"] {
  display:    flex !important;
  visibility: visible !important;
}
[data-testid="collapsedControl"] button {
  display:     flex !important;
  visibility:  visible !important;
  cursor:      pointer !important;
  font-size:   0 !important;
}
[data-testid="collapsedControl"] svg {
  display:    block !important;
  visibility: visible !important;
}

[data-testid="metric-container"] {
  background:    #FFFFFF !important;
  border:        1px solid #E2E8F0 !important;
  border-left:   3px solid #1B5FBF !important;
  border-radius: 0 3px 3px 0 !important;
  padding:       10px 14px !important;
}
[data-testid="metric-container"] * { background: transparent !important; }
[data-testid="stMetricLabel"] > div {
  font-size:      9px !important;
  font-weight:    700 !important;
  letter-spacing: .08em !important;
  text-transform: uppercase !important;
  color:          #6B7280 !important;
}
[data-testid="stMetricValue"] > div {
  font-size:   22px !important;
  font-weight: 700 !important;
  color:       #1C2536 !important;
}

[data-testid="baseButton-primary"] {
  background-color: #1B5FBF !important;
  border:           none !important;
  border-radius:    3px !important;
  color:            #FFFFFF !important;
  font-size:        11px !important;
  font-weight:      700 !important;
  letter-spacing:   .04em !important;
  text-transform:   uppercase !important;
}
[data-testid="baseButton-primary"]:hover {
  background-color: #1549A0 !important;
}
[data-testid="baseButton-secondary"] {
  background:    #FFFFFF !important;
  border:        1px solid #D1D5DB !important;
  border-radius: 3px !important;
  color:         #374151 !important;
  font-size:     11px !important;
  font-weight:   500 !important;
}
[data-testid="baseButton-secondary"]:hover {
  border-color: #1B5FBF !important;
  color:        #1B5FBF !important;
  background:   #EFF6FF !important;
}
[data-testid="stDownloadButton"] button {
  background:    #FFFFFF !important;
  border:        1px solid #D1D5DB !important;
  border-radius: 3px !important;
  color:         #374151 !important;
  font-size:     11px !important;
  font-weight:   600 !important;
  padding:       7px 12px !important;
}
[data-testid="stDownloadButton"] button:hover {
  border-color: #1B5FBF !important;
  color:        #1B5FBF !important;
  background:   #EFF6FF !important;
}

[data-testid="stMultiSelect"] label {
  font-size:      9px !important;
  font-weight:    700 !important;
  letter-spacing: .08em !important;
  text-transform: uppercase !important;
  color:          #6B7280 !important;
  margin-bottom:  2px !important;
}
[data-baseweb="select"] > div {
  border-color:  #E2E8F0 !important;
  border-radius: 3px !important;
  min-height:    34px !important;
  background:    #FFFFFF !important;
  font-size:     12px !important;
}
[data-baseweb="select"] > div:focus-within {
  border-color: #1B5FBF !important;
  box-shadow:   0 0 0 2px rgba(27,95,191,.15) !important;
}
[data-baseweb="tag"] {
  background:    #EFF6FF !important;
  border-radius: 2px !important;
}
[data-baseweb="tag"] span { color: #1B5FBF !important; font-size: 10px !important; }
[data-testid="stMultiSelect"] input { font-size: 11px !important; }

[data-testid="stExpander"] {
  background:    #FFFFFF !important;
  border:        1px solid #E2E8F0 !important;
  border-radius: 4px !important;
  margin-bottom: 8px !important;
}
[data-testid="stExpander"] summary {
  background:    #F8F9FA !important;
  color:         #374151 !important;
  font-size:     12px !important;
  font-weight:   600 !important;
  padding:       10px 14px !important;
  border-radius: 4px !important;
}
[data-testid="stExpander"] summary:hover {
  background: #EFF6FF !important;
  color:      #1B5FBF !important;
}
[data-testid="stExpander"] summary [data-testid="stExpanderToggleIcon"] {
  display: none !important;
}
[data-testid="stExpanderDetails"] { padding: 12px 14px !important; }

input, [data-baseweb="input"] input {
  background:    #FFFFFF !important;
  color:         #1C2536 !important;
  border-color:  #D1D5DB !important;
  border-radius: 3px !important;
  font-size:     12px !important;
}
[data-testid="stTextInput"] label,
[data-testid="stFileUploader"] label {
  color:       #374151 !important;
  font-size:   11px !important;
  font-weight: 600 !important;
}

[data-testid="stAlert"] {
  border-radius:     0 3px 3px 0 !important;
  border-left-width: 3px !important;
  font-size:         12px !important;
}
[data-testid="stAlert"] p { color: inherit !important; }

[data-testid="stFileUploader"] section {
  background:    #FFFFFF !important;
  border:        1.5px dashed #D1D5DB !important;
  border-radius: 4px !important;
}
[data-testid="stFileUploader"] section:hover {
  border-color: #1B5FBF !important;
  background:   #F0F7FF !important;
}

[data-testid="stDataFrame"] {
  border:        1px solid #E2E8F0 !important;
  border-radius: 4px !important;
}

#MainMenu, footer { visibility: hidden; }
[data-testid="stToolbar"] { display: none !important; }
header { background: transparent !important; }
header [data-testid="stHeader"] a,
header [class*="viewerBadge"],
header [class*="reportStatus"] { display: none !important; }

[data-testid="stComponentContainer"] [data-testid="stFileUploaderDropzone"],
[data-testid="stComponentContainer"] [data-testid="stFileUploaderDropzoneInstructions"],
[data-testid="stComponentContainer"] [data-testid="stFileUploaderDropzoneInput"],
[data-testid="stComponentContainer"] section[data-testid="stFileUploaderDropzone"],
[data-testid="stComponentContainer"] > div > div > div[data-testid] {
  display: none !important;
}
[data-testid="stComponentContainer"] iframe {
  display: block !important;
  border:  none !important;
}
[data-testid="stComponentContainer"] {
  padding:    0 !important;
  border:     none !important;
  background: transparent !important;
  overflow:   hidden !important;
}
[data-testid="stFileUploader"] section {
  display: flex !important;
}

.filter-panel {
  background:    #FFFFFF;
  border:        1px solid #E2E8F0;
  border-radius: 6px;
  padding:       16px 18px 8px;
  margin-bottom: 14px;
}
.filter-header {
  display:        flex;
  align-items:    center;
  gap:            8px;
  margin-bottom:  12px;
  padding-bottom: 10px;
  border-bottom:  1px solid #F1F5F9;
}

*:focus-visible {
  outline:        2px solid #1B5FBF !important;
  outline-offset: 2px !important;
}
</style>
"""
st.markdown(_CSS, unsafe_allow_html=True)

# ─── Imports internos ─────────────────────────────────────────────────────────
from app.core.merge import build_merged_df
from app.core.normalize import (apply_column_normalization,
                                 infer_and_coerce_types, strip_whitespace)
from app.core.hash_utils import BUSINESS_COLS
from app.core.analytics import precompute_all
from app.io.readers import list_sheets, read_file
from app.ui.interactive_table import render_interactive_table
from app.ui.dashboard import render_column_dashboard
from app.export.pptx_builder import build_column_pptx, build_general_pptx
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

# ─── Helpers de UI ────────────────────────────────────────────────────────────

def _ss(k, default=None):
    return st.session_state.get(k, default)


def _alert(text: str, kind: str = "info") -> None:
    pal = {
        "info":  ("#EFF6FF", "#1B5FBF", "#1E3A5F"),
        "ok":    ("#F0FDF4", "#16A34A", "#14532D"),
        "warn":  ("#FFFBEB", "#D97706", "#78350F"),
        "error": ("#FEF2F2", "#DC2626", "#7F1D1D"),
    }
    bg, bd, fg = pal.get(kind, pal["info"])
    st.markdown(
        f'<div style="background:{bg};border-left:3px solid {bd};'
        f'padding:10px 14px;margin:8px 0;font-size:12px;color:{fg};'
        f'border-radius:0 3px 3px 0;">{text}</div>',
        unsafe_allow_html=True,
    )


def _divider():
    st.markdown(
        '<div style="height:1px;background:#E2E8F0;margin:16px 0;"></div>',
        unsafe_allow_html=True,
    )


def _label(text: str):
    st.markdown(
        f'<div style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        f'text-transform:uppercase;color:#9CA3AF;margin:12px 0 6px;">{text}</div>',
        unsafe_allow_html=True,
    )


def _page_header(title: str, subtitle: str = ""):
    st.markdown(
        f'<div style="background:#0F172A;border-bottom:2px solid #1B5FBF;'
        f'padding:16px 24px 14px;margin-bottom:22px;">'
        f'<span style="display:block;font-size:18px;font-weight:700;'
        f'color:#F1F5F9;line-height:1.3;margin-bottom:4px;">{title}</span>'
        f'<span style="display:block;font-size:10px;font-weight:600;'
        f'color:#475569;letter-spacing:.08em;text-transform:uppercase;">'
        f'{subtitle}</span>'
        f'</div>',
        unsafe_allow_html=True,
    )


def _section_title(text: str):
    st.markdown(
        f'<div style="background:#0F172A;border-left:4px solid #1B5FBF;'
        f'padding:10px 16px;margin:16px 0 12px;">'
        f'<span style="display:block;font-size:13px;font-weight:700;'
        f'color:#F1F5F9;">{text}</span>'
        f'</div>',
        unsafe_allow_html=True,
    )


# ─── Filtros estilo Excel ──────────────────────────────────────────────────────

def _apply_filters(df: pd.DataFrame, filters: dict) -> pd.DataFrame:
    for col, vals in filters.items():
        if vals and col in df.columns:
            mapped = ["" if v == "(Sem valor)" else v for v in vals]
            df = df[df[col].fillna("").astype(str).isin(mapped)]
    return df


def _render_excel_filters(df: pd.DataFrame) -> pd.DataFrame:
    if "excel_filters" not in st.session_state:
        st.session_state["excel_filters"] = {c: [] for c in BUSINESS_COLS}
    filters: dict = st.session_state["excel_filters"]

    _section_title("Filtros")
    for row_cols in (BUSINESS_COLS[:4], BUSINESS_COLS[4:]):
        grid = st.columns(4, gap="small")
        for i, col in enumerate(row_cols):
            with grid[i]:
                others = {k: v for k, v in filters.items() if k != col and v}
                df_ctx = _apply_filters(df, others)
                raw_vals = sorted(df_ctx[col].fillna("").astype(str).unique().tolist())
                options = (["(Sem valor)"] if "" in raw_vals else []) + [v for v in raw_vals if v]
                current = [v for v in filters.get(col, []) if v in options]
                filters[col] = st.multiselect(
                    col, options=options, default=current,
                    key=f"flt_{col}", placeholder="Todos",
                    label_visibility="visible",
                )

    df_filtered = _apply_filters(df, filters)
    n_active = sum(1 for v in filters.values() if v)

    sc1, sc2 = st.columns([8, 2])
    with sc1:
        showing, total = len(df_filtered), len(df)
        pct = round(showing / total * 100, 1) if total else 0.0
        color = "#DC2626" if showing == 0 else ("#16A34A" if showing < total else "#6B7280")
        st.markdown(
            f'<div style="font-size:11px;color:{color};padding:4px 0;">'
            f'<strong style="color:#1C2536">{showing:,}</strong> de '
            f'<strong style="color:#1C2536">{total:,}</strong> rotas ({pct:.1f}%)</div>',
            unsafe_allow_html=True,
        )
    with sc2:
        if n_active and st.button("Limpar", key="btn_clr_flt", use_container_width=True):
            st.session_state["excel_filters"] = {c: [] for c in BUSINESS_COLS}
            for k in list(st.session_state.keys()):
                if k.startswith("export_"):
                    st.session_state[k] = None
            st.rerun()

    # Invalida cache de exports quando filtros mudam
    flt_hash = str(sorted((k, tuple(v)) for k, v in filters.items() if v))
    if st.session_state.get("_flt_hash") != flt_hash:
        st.session_state["_flt_hash"] = flt_hash
        for k in list(st.session_state.keys()):
            if k.startswith("export_"):
                st.session_state[k] = None

    return df_filtered


# ─── Excel profissional ────────────────────────────────────────────────────────

def _make_excel_professional(df: pd.DataFrame) -> bytes:
    from openpyxl import Workbook
    from openpyxl.styles import Alignment, Border, Font, PatternFill, Side
    from openpyxl.utils import get_column_letter
    from openpyxl.formatting.rule import CellIsRule

    wb = Workbook()
    ws = wb.active
    ws.title = "Tabela Final"

    F_TITLE  = Font(name="Calibri", size=13, bold=True,  color="0F172A")
    F_HEADER = Font(name="Calibri", size=10, bold=True,  color="FFFFFF")
    F_CELL   = Font(name="Calibri", size=10, bold=False, color="111827")
    FILL_TITLE = PatternFill("solid", fgColor="EFF6FF")
    FILL_HEAD  = PatternFill("solid", fgColor="0F172A")
    FILL_EVEN  = PatternFill("solid", fgColor="F1F5F9")
    FILL_ODD   = PatternFill("solid", fgColor="FFFFFF")
    FILL_EMPTY = PatternFill("solid", fgColor="FFFBEB")
    TH  = Side(style="thin",   color="D1D5DB")
    ACC = Side(style="medium", color="1B5FBF")
    B_TITLE = Border(top=ACC, bottom=ACC, left=ACC, right=ACC)
    B_HEAD  = Border(top=ACC, bottom=ACC, left=TH,  right=TH)
    B_CELL  = Border(top=TH,  bottom=TH,  left=TH,  right=TH)

    n_cols   = len(df.columns)
    n_rows   = len(df)
    last_col = get_column_letter(n_cols)

    ws.merge_cells(f"A1:{last_col}1")
    tc = ws["A1"]
    tc.value = (
        f"ITX Analisys — Tabela de Rotas  ·  "
        f"{datetime.now().strftime('%d/%m/%Y  %H:%M')}  ·  "
        f"{n_rows:,} rotas"
    )
    tc.font      = F_TITLE
    tc.fill      = FILL_TITLE
    tc.border    = B_TITLE
    tc.alignment = Alignment(horizontal="center", vertical="center")
    ws.row_dimensions[1].height = 28

    for ci, col in enumerate(df.columns, 1):
        c = ws.cell(row=2, column=ci, value=col)
        c.font      = F_HEADER
        c.fill      = FILL_HEAD
        c.border    = B_HEAD
        c.alignment = Alignment(horizontal="center", vertical="center")
    ws.row_dimensions[2].height = 22

    for ri, (_, row) in enumerate(df.iterrows(), 3):
        fill = FILL_EVEN if ri % 2 == 0 else FILL_ODD
        for ci, col in enumerate(df.columns, 1):
            raw = row[col]
            try:
                val = "" if (
                    raw is None or
                    (not isinstance(raw, str) and pd.isna(raw))
                ) else str(raw)
            except Exception:
                val = "" if raw is None else str(raw)
            c = ws.cell(row=ri, column=ci, value=val)
            c.font      = F_CELL
            c.fill      = fill
            c.border    = B_CELL
            c.alignment = Alignment(horizontal="left", vertical="center")
        ws.row_dimensions[ri].height = 18

    ws.freeze_panes = "A3"
    for ci, col in enumerate(df.columns, 1):
        try:
            ml = max(len(str(col)), int(df[col].fillna("").astype(str).str.len().max() or 0))
        except Exception:
            ml = len(str(col))
        ws.column_dimensions[get_column_letter(ci)].width = min(55, max(10, int(ml * 1.12) + 2))

    ws.auto_filter.ref = f"A2:{last_col}2"
    ws.conditional_formatting.add(
        f"A3:{last_col}{n_rows + 2}",
        CellIsRule(operator="equal", formula=['""'], fill=FILL_EMPTY),
    )
    ws.print_area             = f"A1:{last_col}{n_rows + 2}"
    ws.page_setup.orientation = "landscape"
    ws.page_setup.fitToPage   = True
    ws.page_setup.fitToWidth  = 1

    buf = _io.BytesIO()
    wb.save(buf)
    return buf.getvalue()


# ─── Estado inicial ────────────────────────────────────────────────────────────

def _init_state() -> None:
    defaults = {
        "tabela_final":    None,
        "analytics_cache": None,
        "selected_col":    None,
        "excel_filters":   {c: [] for c in BUSINESS_COLS},
        "sci_df":    None, "sci_filename":  "", "sci_sheet":  None,
        "por_df":    None, "por_filename":  "", "por_sheet":  None,
        "arq3_df":   None, "arq3_filename": "", "arq3_sheet": None,
        "wizard_cfg": {},
    }
    for k, v in defaults.items():
        if k not in st.session_state:
            st.session_state[k] = v

_init_state()

# ─── Carga automática do banco ────────────────────────────────────────────────
if _ss("tabela_final") is None:
    try:
        with session_scope() as s:
            df_db = TabelaFinalRepository(s).load()
        if not df_db.empty:
            st.session_state["tabela_final"]    = df_db
            st.session_state["analytics_cache"] = precompute_all(df_db)
    except Exception as e:
        log.warning("Carga automatica: %s", e)

# ═══════════════════════════════════════════════════════════════════════════════
#  SIDEBAR
# ═══════════════════════════════════════════════════════════════════════════════
with st.sidebar:
    st.markdown(
        '<div style="padding:18px 14px 14px;border-bottom:1px solid #1E293B;'
        'margin-bottom:14px;">'
        '<span style="display:block;font-size:16px;font-weight:700;'
        'color:#F1F5F9;letter-spacing:-.02em;line-height:1.2;">ITX Analisys</span>'
        '<span style="display:block;font-size:9px;font-weight:700;'
        'color:#334155;letter-spacing:.12em;text-transform:uppercase;'
        'margin-top:4px;">1.0</span>'
        '</div>',
        unsafe_allow_html=True,
    )

    _PAGES = ["Carregar Arquivos", "Gerar Tabela", "Tabela"]
    if "nav_page" not in st.session_state:
        st.session_state["nav_page"] = _PAGES[0]

    _cur = st.session_state["nav_page"]
    for _pg in _PAGES:
        if st.button(_pg, key=f"nav_{_pg}", use_container_width=True):
            st.session_state["nav_page"] = _pg
            st.rerun()

    _safe_cur = _cur.replace("'", "\\'")
    st.markdown(f"""
<script>
(function(){{
  const cur = '{_safe_cur}';
  const btns = window.parent.document.querySelectorAll(
    '[data-testid="stSidebarUserContent"] [data-testid="baseButton-secondary"] p'
  );
  btns.forEach(function(p){{
    const btn = p.closest('button');
    if (!btn) return;
    if (p.textContent.trim() === cur) {{
      btn.style.setProperty('color', '#F1F5F9', 'important');
      btn.style.setProperty('background', 'rgba(27,95,191,.18)', 'important');
      btn.style.setProperty('border-left', '2px solid #1B5FBF', 'important');
    }}
  }});
}})();
</script>""", unsafe_allow_html=True)

    page = _cur

    tf_sb = _ss("tabela_final")
    if tf_sb is not None:
        n_sb = len(tf_sb)
        st.markdown(
            f'<div style="margin-top:12px;padding:10px 12px;background:#060D1A;'
            f'border-left:2px solid #1B5FBF;">'
            f'<span style="display:block;font-size:9px;font-weight:700;'
            f'letter-spacing:.1em;text-transform:uppercase;color:#334155;'
            f'margin-bottom:3px;">Quantidade de Rotas</span>'
            f'<span style="display:block;font-size:22px;font-weight:700;'
            f'color:#F1F5F9;line-height:1.1;">{n_sb:,}</span>'
            f'</div>',
            unsafe_allow_html=True,
        )

    st.markdown("<hr>", unsafe_allow_html=True)
    st.markdown(
        f'<span style="font-size:10px;color:#334155;">{Path(_DB).name}</span>',
        unsafe_allow_html=True,
    )
    if st.button("Reiniciar sessao", key="btn_reset"):
        for k in list(st.session_state.keys()):
            del st.session_state[k]
        st.rerun()

# ─── Roteamento ───────────────────────────────────────────────────────────────
p = page


# ═══════════════════════════════════════════════════════════════════════════════
#  PAGINA 1 — CARREGAR ARQUIVOS
# ═══════════════════════════════════════════════════════════════════════════════
if p == "Carregar Arquivos":
    _page_header("Carregar Arquivos", "E aqui onde voce importa seus arquivos")

    st.markdown(
        '<p style="font-size:12px;color:#6B7280;margin:0 0 18px;">'
        'Faca o upload dos arquivos citados abaixo.</p>',
        unsafe_allow_html=True,
    )

    def _handle_upload(uploaded_file, prefix: str, label: str):
        if uploaded_file is None:
            return
        ext = uploaded_file.name.split(".")[-1].lower()
        raw = uploaded_file.read()
        sheet = None
        if ext in ("xlsx", "xls"):
            sheets = list_sheets(_io.BytesIO(raw))
            if len(sheets) > 1:
                sheet = st.selectbox(f"Planilha — {label}", sheets, key=f"sheet_{prefix}")
            elif sheets:
                sheet = sheets[0]
        try:
            df = read_file(_io.BytesIO(raw), filename=uploaded_file.name, sheet=sheet)
            df, _ = apply_column_normalization(df)
            df = strip_whitespace(df)
            df = infer_and_coerce_types(df)
            st.session_state[f"{prefix}_df"]       = df
            st.session_state[f"{prefix}_filename"] = uploaded_file.name
            st.session_state[f"{prefix}_sheet"]    = sheet
        except Exception as e:
            _alert(f"Erro ao ler {uploaded_file.name}: {e}", "error")
            log.error("Upload %s: %s", prefix, e, exc_info=True)

    c1, c2, c3 = st.columns(3, gap="medium")
    for col, prefix, label in [
        (c1, "sci",  "Science"),
        (c2, "por",  "Portal de Cadastros"),
        (c3, "arq3", "Arquivo 3 — Referencia"),
    ]:
        with col:
            _label(label)
            f = st.file_uploader(
                label, type=["xlsx", "xls", "csv", "parquet"],
                key=f"up_{prefix}", label_visibility="collapsed",
            )
            _handle_upload(f, prefix, label)
            dfx = _ss(f"{prefix}_df")
            if dfx is not None:
                fn = _ss(f"{prefix}_filename") or "—"
                st.markdown(
                    f'<div style="display:flex;align-items:center;gap:8px;'
                    f'padding:7px 0;border-bottom:1px solid #E2E8F0;margin-top:6px;">'
                    f'<span style="width:6px;height:6px;border-radius:50%;'
                    f'background:#16A34A;flex-shrink:0;display:inline-block;"></span>'
                    f'<span style="font-size:12px;font-weight:600;color:#1C2536;">{fn}</span>'
                    f'<span style="font-size:11px;color:#9CA3AF;margin-left:4px;">'
                    f'{len(dfx):,} linhas · {len(dfx.columns)} cols</span>'
                    f'</div>',
                    unsafe_allow_html=True,
                )

    _divider()

    for prefix, label in [("sci","Science"), ("por","Portal"), ("arq3","Arquivo 3")]:
        dfx = _ss(f"{prefix}_df")
        if dfx is not None:
            with st.expander(f"Previa — {label}  ({len(dfx):,} linhas)"):
                st.dataframe(dfx.head(15), use_container_width=True, hide_index=True)

    if _ss("sci_df") is not None and _ss("por_df") is not None:
        _alert("Science e Portal carregados. "
               "Prossiga para <strong>Gerar Tabela</strong>.", "ok")


# ═══════════════════════════════════════════════════════════════════════════════
#  PAGINA 2 — GERAR TABELA
# ═══════════════════════════════════════════════════════════════════════════════
elif p == "Gerar Tabela":
    _page_header("Gerar Tabela", "Pipeline · Consolidacao · Deduplicacao · Persistencia")

    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")
    cfg     = _ss("wizard_cfg") or {}

    if sci_df is None or por_df is None:
        _alert("Carregue os arquivos Science e Portal na etapa anterior.", "warn")
        st.stop()

    c1, c2, c3 = st.columns(3)
    c1.metric("Science",   f"{len(sci_df):,}")
    c2.metric("Portal",    f"{len(por_df):,}")
    c3.metric("Arquivo 3", f"{len(arq3_df):,}" if arq3_df is not None else "—")

    _divider()

    col_run, col_clr = st.columns([5, 1])
    with col_clr:
        if st.button("Limpar", key="btn_clr"):
            for k in ("sci_df", "por_df", "arq3_df", "wizard_cfg"):
                st.session_state.pop(k, None)
            st.rerun()
    with col_run:
        run = st.button("Gerar Tabela", type="primary",
                        key="btn_merge", use_container_width=True)

    if run:
        with st.spinner("Executando pipeline de consolidacao..."):
            try:
                merged_df, report = build_merged_df(
                    science_df=sci_df.copy(),
                    portal_df=por_df.copy(),
                    ref_df=arq3_df.copy() if arq3_df is not None else None,
                    uf_map={}, join_keys_sci=[], join_keys_por=[], config=cfg,
                )
                biz_cols  = [c for c in BUSINESS_COLS if c in merged_df.columns]
                result_df = merged_df[biz_cols].copy()

                with session_scope() as s:
                    repo = TabelaFinalRepository(s)
                    inserted, skipped = repo.upsert(result_df)
                    final_df = repo.load()

                st.session_state["tabela_final"]    = final_df
                st.session_state["analytics_cache"] = precompute_all(final_df)
                st.session_state["selected_col"]    = None
                st.session_state["excel_filters"]   = {c: [] for c in BUSINESS_COLS}

                if inserted > 0 and skipped == 0:
                    _alert(f"<strong>{inserted:,} rotas novas adicionadas.</strong> "
                           f"Total: {len(final_df):,} rotas.", "ok")
                elif inserted > 0:
                    _alert(f"<strong>{inserted:,} rotas novas adicionadas.</strong> "
                           f"{skipped:,} ja existiam e foram ignoradas. "
                           f"Total: {len(final_df):,} rotas.", "ok")
                else:
                    _alert(f"Nenhuma rota nova. {skipped:,} ja existiam. "
                           f"Total: {len(final_df):,} rotas.", "info")

                log.info("Pipeline OK: inserted=%d skipped=%d total=%d",
                         inserted, skipped, len(final_df))

            except Exception as e:
                _alert(f"Erro no pipeline: {e}", "error")
                log.error("Pipeline: %s", e, exc_info=True)
                import traceback
                st.code(traceback.format_exc())
                st.stop()

    tf = _ss("tabela_final")
    if tf is not None and not tf.empty:
        _divider()
        _alert(f"Tabela disponivel — <strong>{len(tf):,} rotas</strong>. "
               f"Acesse <strong>Tabela</strong> na sidebar.", "info")


# ═══════════════════════════════════════════════════════════════════════════════
#  PAGINA 3 — TABELA
# ═══════════════════════════════════════════════════════════════════════════════
elif p == "Tabela":
    _page_header("Tabela", "Visualizacao · Dashboard · Exportacao")

    tf = _ss("tabela_final")
    if tf is None or tf.empty:
        _alert("Nenhuma rota disponivel. Gere a Tabela primeiro.", "warn")
        st.stop()

    cache   = _ss("analytics_cache") or {}
    sel_col = _ss("selected_col")
    ts_str  = datetime.now().strftime("%Y-%m-%d_%H%M")

    _section_title("Dashboard por coluna")
    col_btns = st.columns(len(BUSINESS_COLS))
    for i, col in enumerate(BUSINESS_COLS):
        with col_btns[i]:
            if st.button(
                col,
                key=f"col_btn_{col}",
                use_container_width=True,
                type="primary" if sel_col == col else "secondary",
            ):
                st.session_state["selected_col"] = col
                st.rerun()

    _divider()

    # Filtros Streamlit — controlam tabela E exports
    tf_filtered = _render_excel_filters(tf)

    _divider()

    render_interactive_table(
        tf_filtered,
        selected_col=sel_col,
        height=480,
        component_key="main_table",
    )

    if sel_col:
        _divider()
        render_column_dashboard(tf_filtered, sel_col, cache=None)

    _divider()
    st.markdown(
        '<span style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        'text-transform:uppercase;color:#6B7280;display:block;'
        'padding:10px 0 8px;">Exportar</span>',
        unsafe_allow_html=True,
    )

    df_exp = tf_filtered
    ex1, ex2, ex3, ex4 = st.columns(4, gap="small")

    with ex1:
        try:
            csv_bytes = df_exp.to_csv(index=False).encode("utf-8-sig")
            st.download_button(
                "Exportar CSV",
                data=csv_bytes,
                file_name=f"tabela_{ts_str}.csv",
                mime="text/csv",
                key="btn_csv",
                use_container_width=True,
            )
        except Exception as e:
            _alert(f"CSV: {e}", "error")

    with ex2:
        _xls_key = "export_xlsx_data"
        if _xls_key not in st.session_state or st.session_state[_xls_key] is None:
            if st.button("Gerar Excel", key="btn_gen_xlsx", use_container_width=True):
                with st.spinner("Gerando Excel..."):
                    try:
                        st.session_state[_xls_key] = _make_excel_professional(df_exp)
                    except Exception as e:
                        _alert(f"Excel: {e}", "error")
                        log.error("Excel: %s", e, exc_info=True)
                st.rerun()
        else:
            st.download_button(
                "Baixar Excel",
                data=st.session_state[_xls_key],
                file_name=f"tabela_{ts_str}.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                key="btn_xlsx",
                use_container_width=True,
            )
            if st.button("Regerar", key="btn_regen_xlsx", use_container_width=True):
                st.session_state[_xls_key] = None
                st.rerun()

    with ex3:
        _pptx_key = "export_pptx_geral_data"
        if _pptx_key not in st.session_state or st.session_state[_pptx_key] is None:
            if st.button("Gerar PPTX Geral", key="btn_gen_pptx_g", use_container_width=True):
                with st.spinner("Gerando PPTX..."):
                    try:
                        st.session_state[_pptx_key] = build_general_pptx(df_exp)
                    except Exception as e:
                        _alert(f"PPTX geral: {e}", "error")
                        log.error("PPTX geral: %s", e, exc_info=True)
                st.rerun()
        else:
            st.download_button(
                "Baixar PPTX Geral",
                data=st.session_state[_pptx_key],
                file_name=f"dashboard_geral_{ts_str}.pptx",
                mime="application/vnd.openxmlformats-officedocument.presentationml.presentation",
                key="btn_pptx_geral",
                use_container_width=True,
            )
            if st.button("Regerar", key="btn_regen_pptx_g", use_container_width=True):
                st.session_state[_pptx_key] = None
                st.rerun()

    with ex4:
        if not sel_col:
            st.markdown(
                '<div style="font-size:11px;color:#9CA3AF;padding-top:10px;'
                'text-align:center;">Selecione uma coluna acima para exportar o dashboard.</div>',
                unsafe_allow_html=True,
            )
        else:
            _pcol_key = f"export_pptx_col_{sel_col}_data"
            if _pcol_key not in st.session_state or st.session_state[_pcol_key] is None:
                if st.button(f"Gerar PPTX {sel_col}", key="btn_gen_pptx_c", use_container_width=True):
                    with st.spinner(f"Gerando PPTX {sel_col}..."):
                        try:
                            st.session_state[_pcol_key] = build_column_pptx(df_exp, sel_col)
                        except Exception as e:
                            _alert(f"PPTX coluna: {e}", "error")
                            log.error("PPTX col: %s", e, exc_info=True)
                    st.rerun()
            else:
                st.download_button(
                    f"Baixar PPTX {sel_col}",
                    data=st.session_state[_pcol_key],
                    file_name=f"dashboard_{sel_col}_{ts_str}.pptx",
                    mime="application/vnd.openxmlformats-officedocument.presentationml.presentation",
                    key="btn_pptx_col",
                    use_container_width=True,
                )
                if st.button("Regerar", key="btn_regen_pptx_c", use_container_width=True):
                    st.session_state[_pcol_key] = None
                    st.rerun()

    st.markdown("<div style='height:36px;'></div>", unsafe_allow_html=True)


# ─── Rodape ───────────────────────────────────────────────────────────────────
st.markdown(
    '<div style="border-top:1px solid #E2E8F0;margin-top:40px;padding-top:12px;'
    'text-align:center;font-size:10px;color:#9CA3AF;letter-spacing:.04em;">'
    'ITX Analisys · 1.0</div>',
    unsafe_allow_html=True,
)
