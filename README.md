"""
app.py — VIVOHUB · Data Merger v2
Fluxo: Carregar Arquivos → Gerar Tabela Final → Tabela Final → Dashboard → Exportar

CORREÇÕES v4:
  • CSS: botão colapso sidebar restaurado (all:revert + exclusões precisas)
  • CSS: títulos/headers com color explícito + !important resistente a override
  • CSS: sem seletor "*" global que esmague inline-styles em divs escuras
  • Excel: _make_excel_professional() com openpyxl puro, sem format-code errors
  • PPTX: redireciona para pptx_builder v3 (sem bugs de format code)
  • Exportações CSV/Excel/PPTX sempre no final da página (abaixo da tabela e dashboard)
"""
from __future__ import annotations

import io as _stdlib_io
import os
from datetime import datetime
from pathlib import Path

import pandas as pd
import streamlit as st

# ─── Configuração ─────────────────────────────────────────────────────────────
st.set_page_config(
    page_title="VIVOHUB — Data Merger",
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

def _auto_load_seeds() -> None:
    try:
        with session_scope() as s:
            repo = Repository(s)
            if repo.cnl_count() == 0:
                p = Path("seeds") / "cnl.sql"
                if p.exists() and p.stat().st_size > 0:
                    repo.load_cnl_seeds(str(p))
            if repo.cn_to_uf_count() == 0:
                p = Path("seeds") / "cn_to_uf.csv"
                if p.exists() and p.stat().st_size > 0:
                    repo.load_cn_to_uf_csv(str(p))
    except Exception:
        pass

_auto_load_seeds()

# ═══════════════════════════════════════════════════════════════════════════════
# CSS GLOBAL
# Estratégia:
#  1) Modo claro forçado nos containers genéricos do Streamlit
#  2) Sidebar escura apenas nos containers de CONTEÚDO — não no wrapper externo
#     (o wrapper externo contém o botão de colapso nativo, que precisa sobreviver)
#  3) Botão de colapso restaurado com all:revert para cancelar qualquer herança
#  4) Títulos page-header: color:#F1F5F9 em inline style, reforçado por CSS
#  5) H1-H4 explícitos com color escuro (áreas claras) — sem wildcard "*"
# ═══════════════════════════════════════════════════════════════════════════════
st.markdown("""
<style>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap');

/* ── MODO CLARO — containers principais ──────────────────────────────────── */
html, body { background: #F8F9FA !important; color: #111827 !important; }
.stApp,
[data-testid="stAppViewContainer"],
[data-testid="stMain"],
section[data-testid="stMain"],
.main, .block-container {
    background-color: #F8F9FA !important;
    color: #111827 !important;
}
.main .block-container {
    padding-top:    0     !important;
    padding-bottom: 2rem  !important;
    max-width:      100%  !important;
}

/* ── TIPOGRAFIA — seletores específicos, sem "*" ─────────────────────────── */
html, body,
.stApp *, .block-container * {
    font-family: 'IBM Plex Sans', 'Segoe UI', Arial, sans-serif !important;
}
h1, h2, h3, h4 { color: #111827 !important; }
p, li, span:not([style]) { color: #111827; }

/* ── SIDEBAR — apenas o conteúdo de usuário, não o wrapper inteiro ────────── */
[data-testid="stSidebarUserContent"],
[data-testid="stSidebarUserContent"] > * {
    background-color: #0F172A !important;
}
/* Fundo escuro do painel visível da sidebar */
[data-testid="stSidebar"] > div:first-child {
    background-color: #0F172A !important;
}
/* Texto claro dentro do conteúdo da sidebar */
[data-testid="stSidebarUserContent"] p,
[data-testid="stSidebarUserContent"] span,
[data-testid="stSidebarUserContent"] label,
[data-testid="stSidebarUserContent"] div:not([style]) {
    color: #CBD5E1 !important;
}
[data-testid="stSidebar"] hr { border-color: #1E293B !important; }
[data-testid="stSidebar"] [data-testid="stRadio"] label {
    font-size: 12px !important; font-weight: 500 !important;
    color: #94A3B8 !important; padding: 5px 8px !important;
    display: block !important;
}
[data-testid="stSidebar"] [data-testid="stRadio"] label:hover {
    color: #F1F5F9 !important;
}

/* Botão Reiniciar — discreto */
[data-testid="stSidebarUserContent"] button {
    background:    transparent !important;
    border:        1px solid #1E293B !important;
    color:         #64748B !important;
    font-size:     10px !important;
    font-weight:   500 !important;
    width:         100% !important;
    padding:       5px 8px !important;
    margin-top:    4px !important;
    border-radius: 4px !important;
    text-transform: none !important;
}
[data-testid="stSidebarUserContent"] button:hover {
    border-color: #334155 !important;
    color:        #94A3B8 !important;
}

/* ── BOTÃO DE COLAPSO DA SIDEBAR ─────────────────────────────────────────── */
/* Cancela TODA herança de cor/background da sidebar escura */
[data-testid="stSidebarCollapseButton"],
[data-testid="stSidebarCollapseButton"] *,
[data-testid="collapsedControl"],
[data-testid="collapsedControl"] * {
    all: revert !important;
    font-family: 'IBM Plex Sans', sans-serif !important;
}
/* Garante visibilidade */
[data-testid="stSidebarCollapseButton"],
[data-testid="collapsedControl"] {
    display:    flex !important;
    visibility: visible !important;
    opacity:    1 !important;
}
[data-testid="stSidebarCollapseButton"] button,
[data-testid="collapsedControl"] button {
    display:    flex !important;
    visibility: visible !important;
    cursor:     pointer !important;
}
[data-testid="stSidebarCollapseButton"] svg,
[data-testid="collapsedControl"] svg {
    display:    block !important;
    visibility: visible !important;
}

/* ── HEADERS / TÍTULOS DE PÁGINA ─────────────────────────────────────────── */
/* O _page_header usa inline styles — mas forçamos via atributo data para segurança */
.page-header-title {
    color: #F1F5F9 !important;
    font-size: 18px !important;
    font-weight: 700 !important;
}
/* Garante que markdown dentro de divs escuras não sobreponha branco → escuro */
[data-testid="stMarkdownContainer"] div[style*="background:#0F172A"] *,
[data-testid="stMarkdownContainer"] div[style*="background: #0F172A"] * {
    color: #F1F5F9 !important;
}

/* ── EXPANDERS ───────────────────────────────────────────────────────────── */
[data-testid="stExpander"] {
    background:    #FFFFFF !important;
    border:        1px solid #E2E8F0 !important;
    border-radius: 4px !important;
    margin-bottom: 6px !important;
}
[data-testid="stExpander"] summary {
    background:    #F8F9FA !important;
    color:         #1C2536 !important;
    font-size:     12px !important;
    font-weight:   600 !important;
    padding:       9px 14px !important;
    border-radius: 4px !important;
}
[data-testid="stExpander"] summary:hover {
    background: #EFF6FF !important;
    color:      #1B5FBF !important;
}
/* Remove ícone literal "keyboard_double_arrow" / "expand_more" */
[data-testid="stExpander"] summary [data-testid="stExpanderToggleIcon"] { display: none !important; }
[data-testid="stExpanderDetails"] {
    background: #FFFFFF !important;
    padding:    12px !important;
}

/* ── MÉTRICAS ────────────────────────────────────────────────────────────── */
[data-testid="metric-container"] {
    background:   #FFFFFF !important;
    border:       1px solid #E2E8F0 !important;
    border-left:  3px solid #1B5FBF !important;
    padding:      10px 14px !important;
    border-radius: 0 !important;
}
[data-testid="metric-container"] * { background: transparent !important; }
[data-testid="stMetricLabel"] > div,
[data-testid="stMetricLabel"] label {
    font-size:       9px !important;
    font-weight:     700 !important;
    letter-spacing:  .09em !important;
    text-transform:  uppercase !important;
    color:           #6B7280 !important;
}
[data-testid="stMetricValue"] > div {
    font-size:   20px !important;
    font-weight: 700 !important;
    color:       #111827 !important;
}

/* ── BOTÕES PRIMÁRIOS ────────────────────────────────────────────────────── */
button[kind="primary"],
[data-testid="baseButton-primary"] {
    background-color: #1B5FBF !important;
    border:           none !important;
    border-radius:    3px !important;
    color:            #FFFFFF !important;
    font-size:        11px !important;
    font-weight:      700 !important;
    letter-spacing:   .05em !important;
    text-transform:   uppercase !important;
    padding:          6px 14px !important;
}
button[kind="primary"]:hover,
[data-testid="baseButton-primary"]:hover { background-color: #1549A0 !important; }

/* ── BOTÕES SECUNDÁRIOS ──────────────────────────────────────────────────── */
button[kind="secondary"],
[data-testid="baseButton-secondary"] {
    border-radius: 3px !important;
    background:    #FFFFFF !important;
    border:        1px solid #D1D5DB !important;
    color:         #374151 !important;
    font-size:     11px !important;
    font-weight:   500 !important;
    padding:       6px 14px !important;
}
button[kind="secondary"]:hover,
[data-testid="baseButton-secondary"]:hover {
    border-color: #1B5FBF !important;
    color:        #1B5FBF !important;
    background:   #EFF6FF !important;
}

/* ── DOWNLOAD BUTTONS ────────────────────────────────────────────────────── */
[data-testid="stDownloadButton"] button {
    border-radius: 3px !important;
    background:    #FFFFFF !important;
    border:        1px solid #D1D5DB !important;
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

/* ── INPUTS ──────────────────────────────────────────────────────────────── */
input, [data-baseweb="input"] input {
    background-color: #FFFFFF !important;
    color:            #111827 !important;
    border-color:     #D1D5DB !important;
    border-radius:    3px !important;
    font-size:        12px !important;
}
[data-testid="stTextInput"] label,
[data-testid="stSelectbox"] label,
[data-testid="stFileUploader"] label {
    color:       #374151 !important;
    font-size:   11px !important;
    font-weight: 600 !important;
}

/* ── ALERTAS ─────────────────────────────────────────────────────────────── */
[data-testid="stAlert"] {
    border-radius:    3px !important;
    border-left-width: 3px !important;
    font-size:        12px !important;
    font-weight:      500 !important;
}
[data-testid="stAlert"] p { color: inherit !important; }

/* ── FILE UPLOADER ───────────────────────────────────────────────────────── */
[data-testid="stFileUploader"] section {
    background:    #FFFFFF !important;
    border:        1.5px dashed #D1D5DB !important;
    border-radius: 4px !important;
}
[data-testid="stFileUploader"] section:hover {
    border-color: #1B5FBF !important;
    background:   #F0F7FF !important;
}

/* ── DATAFRAME ───────────────────────────────────────────────────────────── */
[data-testid="stDataFrame"] {
    border:        1px solid #E2E8F0 !important;
    border-radius: 4px !important;
}

/* ── HIDE CHROME DESNECESSÁRIO ───────────────────────────────────────────── */
#MainMenu, footer { visibility: hidden; }
[data-testid="stToolbar"] { display: none !important; }
/* Mantém header para botão de colapso funcionar — só oculta branding */
header { background: transparent !important; }
header a[href], header img,
header [class*="statusWidget"],
header [class*="reportStatus"] { display: none !important; }

/* ── BARRA DE EXPORT ─────────────────────────────────────────────────────── */
.export-section {
    background:   #F1F5F9;
    border-top:   2px solid #E2E8F0;
    padding:      14px 0 8px;
    margin-top:   20px;
}

/* ── FOCO ────────────────────────────────────────────────────────────────── */
*:focus-visible { outline: 2px solid #1B5FBF !important; outline-offset: 2px !important; }
</style>
""", unsafe_allow_html=True)

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
    styles = {
        "info":  ("#EFF6FF", "#1B5FBF", "#1E3A5F"),
        "ok":    ("#F0FDF4", "#16A34A", "#14532D"),
        "warn":  ("#FFFBEB", "#D97706", "#78350F"),
        "error": ("#FEF2F2", "#DC2626", "#7F1D1D"),
    }
    bg, bd, fg = styles.get(kind, styles["info"])
    st.markdown(
        f'<div style="background:{bg};border-left:3px solid {bd};'
        f'padding:10px 14px;margin:6px 0;font-size:12px;'
        f'color:{fg} !important;font-weight:500;border-radius:0 3px 3px 0;">'
        f'{text}</div>',
        unsafe_allow_html=True,
    )


def _divider():
    st.markdown('<div style="height:1px;background:#E2E8F0;margin:16px 0;"></div>',
                unsafe_allow_html=True)


def _label(text: str):
    st.markdown(
        f'<div style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        f'text-transform:uppercase;color:#9CA3AF !important;margin:12px 0 6px;">'
        f'{text}</div>',
        unsafe_allow_html=True,
    )


def _page_header(title: str, subtitle: str = ""):
    """Header de página — fundo escuro, título sempre branco."""
    st.markdown(
        f'<div style="background:#0F172A;border-bottom:2px solid #1B5FBF;'
        f'padding:16px 24px 13px;margin-bottom:20px;">'
        # Forçamos cor com atributo style diretamente no span de texto
        f'<span style="display:block;font-size:18px;font-weight:700;'
        f'color:#F1F5F9 !important;line-height:1.3;margin:0 0 3px;">{title}</span>'
        f'<span style="display:block;font-size:10px;font-weight:600;'
        f'color:#64748B !important;letter-spacing:.08em;text-transform:uppercase;">'
        f'{subtitle}</span>'
        f'</div>',
        unsafe_allow_html=True,
    )


def _section_title(text: str):
    """Título de seção secundário — fundo escuro, texto branco."""
    st.markdown(
        f'<div style="background:#0F172A;border-left:4px solid #1B5FBF;'
        f'padding:10px 16px;margin:14px 0 10px;">'
        f'<span style="display:block;font-size:13px;font-weight:700;'
        f'color:#F1F5F9 !important;">{text}</span>'
        f'</div>',
        unsafe_allow_html=True,
    )


# ─── Excel profissional ───────────────────────────────────────────────────────

def _make_excel_professional(df: pd.DataFrame) -> bytes:
    """
    XLSX estilo consultoria Big Four.
    • Linha 1: Título centralizado com fundo azul suave
    • Linha 2: Cabeçalhos — fundo #0F172A, texto branco, negrito
    • Dados:   zebra (branco / cinza claro), fonte Calibri 11
    • Bordas consistentes em todos os dados
    • Cabeçalho congelado em A3
    • Auto-fit de colunas (10–55 chars)
    • Auto-filtro no cabeçalho
    • Formatação condicional: células vazias → amarelo suave
    • Orientação landscape, área de impressão definida
    """
    from openpyxl import Workbook
    from openpyxl.styles import (Alignment, Border, Font,
                                  PatternFill, Side)
    from openpyxl.utils import get_column_letter
    from openpyxl.formatting.rule import CellIsRule

    wb = Workbook()
    ws = wb.active
    ws.title = "Tabela Final"

    # Paleta
    FILL_HDR   = PatternFill("solid", fgColor="0F172A")
    FILL_TITLE = PatternFill("solid", fgColor="EFF6FF")
    FILL_EVEN  = PatternFill("solid", fgColor="F1F5F9")
    FILL_ODD   = PatternFill("solid", fgColor="FFFFFF")
    FILL_EMPTY = PatternFill("solid", fgColor="FFFBEB")

    FONT_TITLE = Font(name="Calibri", size=13, bold=True,  color="0F172A")
    FONT_HDR   = Font(name="Calibri", size=10, bold=True,  color="FFFFFF")
    FONT_CELL  = Font(name="Calibri", size=10, bold=False, color="111827")

    THIN   = Side(style="thin",   color="D1D5DB")
    MEDIUM = Side(style="medium", color="0F172A")
    ACCENT = Side(style="medium", color="1B5FBF")

    BORDER_HDR  = Border(top=ACCENT,  left=THIN, right=THIN,  bottom=ACCENT)
    BORDER_CELL = Border(top=THIN,    left=THIN, right=THIN,  bottom=THIN)
    BORDER_TITLE = Border(
        top=Side(style="medium", color="1B5FBF"),
        bottom=Side(style="medium", color="1B5FBF"),
        left=Side(style="medium", color="1B5FBF"),
        right=Side(style="medium", color="1B5FBF"),
    )

    n_cols   = len(df.columns)
    n_rows   = len(df)
    last_col = get_column_letter(n_cols)

    # Linha 1 — título
    ws.merge_cells(f"A1:{last_col}1")
    tc = ws["A1"]
    tc.value = (f"VIVOHUB — Tabela Final de Rotas  |  "
                f"{datetime.now().strftime('%d/%m/%Y %H:%M')}  |  {n_rows:,} rotas")
    tc.font      = FONT_TITLE
    tc.fill      = FILL_TITLE
    tc.border    = BORDER_TITLE
    tc.alignment = Alignment(horizontal="center", vertical="center")
    ws.row_dimensions[1].height = 28

    # Linha 2 — cabeçalhos
    for ci, col in enumerate(df.columns, start=1):
        cell = ws.cell(row=2, column=ci, value=col)
        cell.font      = FONT_HDR
        cell.fill      = FILL_HDR
        cell.border    = BORDER_HDR
        cell.alignment = Alignment(horizontal="center", vertical="center")
    ws.row_dimensions[2].height = 22

    # Linhas de dados (a partir da linha 3)
    for ri, (_, row) in enumerate(df.iterrows(), start=3):
        fill = FILL_EVEN if ri % 2 == 0 else FILL_ODD
        for ci, col in enumerate(df.columns, start=1):
            raw = row[col]
            # Converte NaN/None → string vazia para o Excel
            if raw is None:
                val = ""
            else:
                try:
                    val = "" if pd.isna(raw) else str(raw)
                except Exception:
                    val = str(raw)
            cell = ws.cell(row=ri, column=ci, value=val)
            cell.font      = FONT_CELL
            cell.fill      = fill
            cell.border    = BORDER_CELL
            cell.alignment = Alignment(horizontal="left", vertical="center")
        ws.row_dimensions[ri].height = 18

    # Congela cabeçalho
    ws.freeze_panes = "A3"

    # Auto-fit largura
    for ci, col in enumerate(df.columns, start=1):
        try:
            max_len = max(
                len(str(col)),
                df[col].fillna("").astype(str).str.len().max() or 0,
            )
        except Exception:
            max_len = len(str(col))
        ws.column_dimensions[get_column_letter(ci)].width = min(55, max(10, int(max_len * 1.12) + 2))

    # Auto-filtro
    ws.auto_filter.ref = f"A2:{last_col}2"

    # Formatação condicional: células vazias em amarelo
    data_range = f"A3:{last_col}{n_rows + 2}"
    ws.conditional_formatting.add(
        data_range,
        CellIsRule(operator="equal", formula=['""'], fill=FILL_EMPTY),
    )

    # Impressão landscape
    ws.print_area            = f"A1:{last_col}{n_rows + 2}"
    ws.page_setup.orientation = "landscape"
    ws.page_setup.fitToPage  = True
    ws.page_setup.fitToWidth = 1

    buf = _stdlib_io.BytesIO()
    wb.save(buf)
    return buf.getvalue()


# ─── Estado inicial ───────────────────────────────────────────────────────────

def _init_state() -> None:
    defaults = {
        "tabela_final":    None,
        "analytics_cache": None,
        "selected_col":    None,
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
        log.warning("Carga automática: %s", e)

# ─── SIDEBAR ──────────────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown(
        '<div style="padding:18px 12px 14px;border-bottom:1px solid #1E293B;margin-bottom:12px;">'
        '<span style="display:block;font-size:16px;font-weight:700;color:#F1F5F9 !important;'
        'letter-spacing:-.02em;">VIVOHUB</span>'
        '<span style="display:block;font-size:9px;font-weight:700;color:#334155 !important;'
        'letter-spacing:.12em;text-transform:uppercase;margin-top:3px;">Data Merger v2</span>'
        '</div>',
        unsafe_allow_html=True,
    )

    page = st.radio("", [
        "1. Carregar Arquivos",
        "2. Gerar Tabela Final",
        "3. Tabela Final",
    ], key="nav", label_visibility="collapsed")

    st.markdown("<hr>", unsafe_allow_html=True)

    def _dot(label: str, key: str):
        ok  = st.session_state.get(key) is not None
        dot = "#22C55E" if ok else "#1E293B"
        fg  = "#CBD5E1" if ok else "#475569"
        st.markdown(
            f'<div style="display:flex;align-items:center;gap:8px;padding:3px 0;">'
            f'<span style="width:6px;height:6px;border-radius:50%;'
            f'background:{dot};flex-shrink:0;display:inline-block;"></span>'
            f'<span style="font-size:11px;font-weight:500;color:{fg} !important;">'
            f'{label}</span></div>',
            unsafe_allow_html=True,
        )

    _dot("Science",   "sci_df")
    _dot("Portal",    "por_df")
    _dot("Arquivo 3", "arq3_df")
    _dot("Resultado", "tabela_final")

    tf_sb = _ss("tabela_final")
    if tf_sb is not None:
        n = len(tf_sb)
        st.markdown(
            f'<div style="margin-top:10px;padding:9px 12px;background:#060D1A;'
            f'border-left:2px solid #1B5FBF;">'
            f'<span style="display:block;font-size:9px;font-weight:700;'
            f'letter-spacing:.1em;text-transform:uppercase;color:#334155 !important;'
            f'margin-bottom:2px;">Rotas no banco</span>'
            f'<span style="display:block;font-size:20px;font-weight:700;'
            f'color:#F1F5F9 !important;line-height:1.1;">{n:,}</span>'
            f'</div>',
            unsafe_allow_html=True,
        )

    st.markdown("<hr>", unsafe_allow_html=True)
    st.markdown(
        f'<span style="font-size:10px;color:#334155 !important;">'
        f'{Path(_DB).name}</span>',
        unsafe_allow_html=True,
    )
    if st.button("Reiniciar sessão", key="btn_reset"):
        for k in list(st.session_state.keys()):
            del st.session_state[k]
        st.rerun()

# ─── Roteamento ───────────────────────────────────────────────────────────────
p = page.split(". ", 1)[-1]


# ══════════════════════════════════════════════════════════════════════════════
# PÁGINA 1 — CARREGAR ARQUIVOS
# ══════════════════════════════════════════════════════════════════════════════
if p == "Carregar Arquivos":
    _page_header("Carregar Arquivos",
                 "Importação — Science · Portal de Cadastros · Arquivo 3")

    st.markdown(
        '<p style="font-size:12px;color:#6B7280;margin:0 0 16px;">'
        'Faça o upload dos arquivos de origem. O processamento é 100% local.</p>',
        unsafe_allow_html=True,
    )

    def _handle_upload(uploaded_file, prefix: str, label: str):
        if uploaded_file is None:
            return
        ext = uploaded_file.name.split(".")[-1].lower()
        raw = uploaded_file.read()
        sheet = None
        if ext in ("xlsx", "xls"):
            import io as _io2
            sheets = list_sheets(_io2.BytesIO(raw))
            if len(sheets) > 1:
                sheet = st.selectbox(f"Planilha — {label}", sheets,
                                     key=f"sheet_{prefix}")
            elif sheets:
                sheet = sheets[0]
        try:
            import io as _io2
            df = read_file(_io2.BytesIO(raw), filename=uploaded_file.name, sheet=sheet)
            df, _ = apply_column_normalization(df)
            df    = strip_whitespace(df)
            df    = infer_and_coerce_types(df)
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
        (c3, "arq3", "Arquivo 3 — Referência"),
    ]:
        with col:
            _label(label)
            f = st.file_uploader(
                label, type=["xlsx", "xls", "csv", "parquet"],
                key=f"up_{prefix}", label_visibility="collapsed",
            )
            _handle_upload(f, prefix, label)
            df_up = _ss(f"{prefix}_df")
            if df_up is not None:
                fn = _ss(f"{prefix}_filename") or "—"
                st.markdown(
                    f'<div style="display:flex;align-items:center;gap:8px;'
                    f'padding:7px 0;border-bottom:1px solid #E2E8F0;margin-top:6px;">'
                    f'<span style="width:6px;height:6px;border-radius:50%;'
                    f'background:#16A34A;flex-shrink:0;display:inline-block;"></span>'
                    f'<span style="font-size:12px;font-weight:600;color:#111827 !important;">'
                    f'{fn}</span>'
                    f'<span style="font-size:11px;color:#9CA3AF !important;">'
                    f'{len(df_up):,} linhas · {len(df_up.columns)} cols</span>'
                    f'</div>',
                    unsafe_allow_html=True,
                )

    _divider()

    for prefix, label in [("sci", "Science"), ("por", "Portal"), ("arq3", "Arquivo 3")]:
        df_up = _ss(f"{prefix}_df")
        if df_up is not None:
            with st.expander(f"Prévia — {label}  ({len(df_up):,} linhas)"):
                st.dataframe(df_up.head(15), use_container_width=True, hide_index=True)

    if _ss("sci_df") is not None and _ss("por_df") is not None:
        _alert("Science e Portal carregados. "
               "Prossiga para <strong>Gerar Tabela Final</strong>.", "ok")


# ══════════════════════════════════════════════════════════════════════════════
# PÁGINA 2 — GERAR TABELA FINAL
# ══════════════════════════════════════════════════════════════════════════════
elif p == "Gerar Tabela Final":
    _page_header("Gerar Tabela Final",
                 "Pipeline · Consolidação · Deduplicação · Persistência")

    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")
    cfg     = _ss("wizard_cfg") or {}

    if sci_df is None or por_df is None:
        _alert("Carregue os arquivos Science e Portal na etapa anterior.", "warn")
        st.stop()

    c1, c2, c3 = st.columns(3)
    c1.metric("Linhas Science", f"{len(sci_df):,}")
    c2.metric("Linhas Portal",  f"{len(por_df):,}")
    c3.metric("Arquivo 3",      f"{len(arq3_df):,}" if arq3_df is not None else "—")

    _divider()

    col_run, col_clr = st.columns([5, 1])
    with col_clr:
        if st.button("Limpar", key="btn_clr"):
            for k in ("sci_df", "por_df", "arq3_df", "wizard_cfg"):
                st.session_state.pop(k, None)
            st.rerun()
    with col_run:
        run = st.button("Gerar Tabela Final", type="primary",
                        key="btn_merge", use_container_width=True)

    if run:
        with st.spinner("Executando pipeline de consolidação..."):
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

                if inserted > 0 and skipped == 0:
                    _alert(f"<strong>{inserted:,} rotas novas adicionadas.</strong> "
                           f"Total: {len(final_df):,} rotas.", "ok")
                elif inserted > 0:
                    _alert(f"<strong>{inserted:,} rotas novas adicionadas.</strong> "
                           f"{skipped:,} já existiam e foram ignoradas. "
                           f"Total: {len(final_df):,} rotas.", "ok")
                else:
                    _alert(f"Nenhuma rota nova. {skipped:,} já existiam. "
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
        _alert(f"Tabela Final disponível — <strong>{len(tf):,} rotas</strong>. "
               f"Acesse <strong>Tabela Final</strong> na sidebar.", "info")


# ══════════════════════════════════════════════════════════════════════════════
# PÁGINA 3 — TABELA FINAL + DASHBOARD + EXPORTAÇÃO
# ══════════════════════════════════════════════════════════════════════════════
elif p == "Tabela Final":
    _page_header("Tabela Final", "Visualização · Dashboard · Exportação")

    tf = _ss("tabela_final")
    if tf is None or tf.empty:
        _alert("Nenhuma rota disponível. Gere a Tabela Final primeiro.", "warn")
        st.stop()

    cache   = _ss("analytics_cache") or {}
    sel_col = _ss("selected_col")
    ts_str  = datetime.now().strftime("%Y-%m-%d_%H%M")

    # ── Seletor de coluna ──────────────────────────────────────────────────
    _section_title("Selecione uma coluna para ver o dashboard instantâneo")
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

    # ── Tabela interativa ──────────────────────────────────────────────────
    render_interactive_table(tf, selected_col=sel_col, height=430,
                             component_key="main_table")

    # ── Dashboard (abaixo da tabela) ───────────────────────────────────────
    if sel_col:
        _divider()
        render_column_dashboard(tf, sel_col, cache=cache)

    # ══════════════════════════════════════════════════════════════════════
    # EXPORTAÇÕES — SEMPRE NO FINAL, ABAIXO DE TUDO
    # ══════════════════════════════════════════════════════════════════════
    _divider()
    st.markdown(
        '<div style="background:#F1F5F9;border-top:2px solid #E2E8F0;'
        'padding:14px 0 6px;margin-top:4px;">'
        '<span style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        'text-transform:uppercase;color:#6B7280 !important;">Exportar</span>'
        '</div>',
        unsafe_allow_html=True,
    )

    ex1, ex2, ex3, ex4 = st.columns(4, gap="small")

    # CSV ──────────────────────────────────────────────────────────────────
    with ex1:
        try:
            csv_bytes = tf.to_csv(index=False).encode("utf-8-sig")
            st.download_button(
                "⬇  Exportar CSV",
                data=csv_bytes,
                file_name=f"tabela_final_{ts_str}.csv",
                mime="text/csv",
                key="btn_csv",
                use_container_width=True,
            )
        except Exception as e:
            _alert(f"CSV: {e}", "error")
            log.error("CSV export: %s", e)

    # Excel profissional ───────────────────────────────────────────────────
    with ex2:
        try:
            xlsx_bytes = _make_excel_professional(tf)
            st.download_button(
                "⬇  Exportar Excel",
                data=xlsx_bytes,
                file_name=f"tabela_final_{ts_str}.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                key="btn_xlsx",
                use_container_width=True,
            )
        except Exception as e:
            _alert(f"Excel: {e}", "error")
            log.error("Excel export: %s", e, exc_info=True)

    # PPTX Geral ───────────────────────────────────────────────────────────
    with ex3:
        try:
            pptx_gen = build_general_pptx(tf)
            st.download_button(
                "⬇  Dashboard Geral (PPTX)",
                data=pptx_gen,
                file_name=f"dashboard_geral_{ts_str}.pptx",
                mime="application/vnd.openxmlformats-officedocument.presentationml.presentation",
                key="btn_pptx_geral",
                use_container_width=True,
            )
        except Exception as e:
            _alert(f"PPTX geral: {e}", "error")
            log.error("PPTX geral: %s", e, exc_info=True)

    # PPTX Coluna (só ativo quando coluna selecionada) ─────────────────────
    with ex4:
        if sel_col:
            try:
                pptx_col = build_column_pptx(tf, sel_col)
                st.download_button(
                    f"⬇  Dashboard {sel_col} (PPTX)",
                    data=pptx_col,
                    file_name=f"dashboard_{sel_col}_{ts_str}.pptx",
                    mime="application/vnd.openxmlformats-officedocument.presentationml.presentation",
                    key="btn_pptx_col",
                    use_container_width=True,
                )
            except Exception as e:
                _alert(f"PPTX coluna: {e}", "error")
                log.error("PPTX col: %s", e, exc_info=True)
        else:
            st.markdown(
                '<div style="font-size:11px;color:#9CA3AF !important;'
                'padding-top:30px;text-align:center;line-height:1.4;">'
                'Selecione uma coluna para exportar o dashboard específico.</div>',
                unsafe_allow_html=True,
            )

    st.markdown("<div style='height:28px;'></div>", unsafe_allow_html=True)


# ─── Rodapé ───────────────────────────────────────────────────────────────────
st.markdown(
    '<div style="border-top:1px solid #E2E8F0;margin-top:32px;padding-top:10px;'
    'text-align:center;font-size:10px;color:#9CA3AF !important;letter-spacing:.05em;">'
    'VIVOHUB · Data Merger v2 · Processamento 100% local</div>',
    unsafe_allow_html=True,
)
