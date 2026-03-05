"""
app.py — VIVOHUB · Data Merger v2
Execute: streamlit run app.py
"""
from __future__ import annotations
import os
from pathlib import Path

import streamlit as st

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

from app.db.session import init_db, session_scope
from app.db.repository import Repository

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
    except Exception as e:
        print(f"[AVISO] Seeds: {e}")


_auto_load_seeds()

# ─────────────────────────────────────────────────────────────────────────────
# CSS GLOBAL
# Força modo claro independente das preferências do sistema.
# Textos em fundos escuros (sidebar, header) sempre com cor explícita.
# ─────────────────────────────────────────────────────────────────────────────
st.markdown("""
<style>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap');

/* MODO CLARO FORÇADO */
html, body,
.stApp,
[data-testid="stAppViewContainer"],
[data-testid="stAppViewBlockContainer"],
[data-testid="stMain"],
section[data-testid="stMain"],
.main,
.block-container,
.main .block-container {
    background-color: #F8F9FA !important;
    color: #111827 !important;
}
.main .block-container {
    padding-top: 0 !important;
    padding-bottom: 2.5rem !important;
    max-width: 100% !important;
}

/* TIPOGRAFIA */
html, body, *, *::before, *::after {
    font-family: 'IBM Plex Sans', 'Segoe UI', 'Helvetica Neue', Arial, sans-serif !important;
    box-sizing: border-box;
}
[data-testid="stMarkdownContainer"] *,
[data-testid="stVerticalBlock"] p,
[data-testid="stVerticalBlock"] span,
[data-testid="stVerticalBlock"] li,
.element-container p,
.element-container span {
    color: #111827 !important;
}

/* SIDEBAR */
[data-testid="stSidebar"],
[data-testid="stSidebar"] > div,
[data-testid="stSidebar"] section,
[data-testid="stSidebarUserContent"] {
    background-color: #0F172A !important;
}
[data-testid="stSidebar"] * {
    color: #CBD5E1 !important;
    background-color: transparent !important;
}
[data-testid="stSidebar"] hr { border-color: #1E293B !important; margin: 8px 0 !important; }
[data-testid="stSidebar"] [data-testid="stRadio"] label {
    font-size: 12px !important; font-weight: 500 !important;
    color: #94A3B8 !important; letter-spacing: .01em !important;
    padding: 4px 6px !important; display: block !important;
}
[data-testid="stSidebar"] [data-testid="stRadio"] label:hover { color: #F1F5F9 !important; }
[data-testid="stSidebar"] button {
    background: transparent !important; border: 1px solid #1E293B !important;
    color: #64748B !important; font-size: 10px !important; font-weight: 600 !important;
    letter-spacing: .07em !important; text-transform: uppercase !important;
    width: 100% !important; padding: 5px 8px !important; margin-top: 4px !important;
}
[data-testid="stSidebar"] button:hover { border-color: #1B5FBF !important; color: #93C5FD !important; }

/* MÉTRICAS */
[data-testid="metric-container"] {
    background: #FFFFFF !important; border: 1px solid #E2E8F0 !important;
    border-left: 3px solid #1B5FBF !important; padding: 10px 14px !important;
}
[data-testid="metric-container"] * { background: transparent !important; }
[data-testid="stMetricLabel"] > div,
[data-testid="stMetricLabel"] label,
[data-testid="metric-container"] label {
    font-size: 9px !important; font-weight: 700 !important;
    letter-spacing: .09em !important; text-transform: uppercase !important; color: #6B7280 !important;
}
[data-testid="stMetricValue"] > div { font-size: 22px !important; font-weight: 700 !important; color: #111827 !important; }
[data-testid="stMetricDelta"]       { font-size: 11px !important; color: #6B7280 !important; }

/* BOTÕES */
button[kind="primary"],
[data-testid="baseButton-primary"] {
    background-color: #1B5FBF !important; border: none !important; border-radius: 0 !important;
    color: #FFFFFF !important; font-size: 11px !important; font-weight: 700 !important;
    letter-spacing: .06em !important; text-transform: uppercase !important;
}
button[kind="primary"]:hover,
[data-testid="baseButton-primary"]:hover { background-color: #1549A0 !important; }

button[kind="secondary"],
[data-testid="baseButton-secondary"] {
    border-radius: 0 !important; background: #FFFFFF !important;
    border: 1px solid #D1D5DB !important; color: #374151 !important;
    font-size: 11px !important; font-weight: 600 !important; letter-spacing: .04em !important;
}
button[kind="secondary"]:hover,
[data-testid="baseButton-secondary"]:hover { border-color: #1B5FBF !important; color: #1B5FBF !important; }

[data-testid="stDownloadButton"] button {
    border-radius: 0 !important; background: #FFFFFF !important;
    border: 1px solid #D1D5DB !important; color: #374151 !important;
    font-size: 10px !important; font-weight: 700 !important;
    letter-spacing: .06em !important; text-transform: uppercase !important;
}
[data-testid="stDownloadButton"] button:hover { border-color: #1B5FBF !important; color: #1B5FBF !important; }

/* INPUTS */
input, textarea,
[data-baseweb="input"] input,
[data-baseweb="textarea"] textarea {
    background-color: #FFFFFF !important; color: #111827 !important;
    border-color: #D1D5DB !important; border-radius: 0 !important; font-size: 12px !important;
}
[data-baseweb="select"] > div {
    background-color: #FFFFFF !important; color: #111827 !important;
    border-color: #D1D5DB !important; border-radius: 0 !important; font-size: 12px !important;
}
[data-testid="stTextInput"] label,
[data-testid="stSelectbox"] label,
[data-testid="stMultiSelect"] label,
[data-testid="stTextArea"] label { color: #374151 !important; font-size: 11px !important; font-weight: 600 !important; }
[data-baseweb="tag"]             { background: #EFF6FF !important; border-radius: 0 !important; }
[data-baseweb="tag"] span        { color: #1B5FBF !important; }

/* EXPANDERS */
[data-testid="stExpander"] {
    background: #FFFFFF !important; border: 1px solid #E2E8F0 !important;
    border-radius: 0 !important; margin-bottom: 6px !important;
}
[data-testid="stExpander"] summary {
    background: #F8F9FA !important; color: #1C2536 !important;
    font-size: 12px !important; font-weight: 600 !important;
    letter-spacing: .02em !important; padding: 9px 14px !important;
}
[data-testid="stExpander"] summary:hover { background: #EFF6FF !important; color: #1B5FBF !important; }
[data-testid="stExpanderDetails"]        { background: #FFFFFF !important; }
[data-testid="stExpanderDetails"] *      { color: #111827 !important; }

/* ALERTAS */
[data-testid="stAlert"] {
    border-radius: 0 !important; border-left-width: 3px !important;
    font-size: 12px !important; font-weight: 500 !important;
}
[data-testid="stAlert"] p { color: inherit !important; }

/* FILE UPLOADER */
[data-testid="stFileUploader"] section {
    background: #FFFFFF !important; border: 1.5px dashed #D1D5DB !important; border-radius: 0 !important;
}
[data-testid="stFileUploader"] section:hover { border-color: #1B5FBF !important; background: #F0F7FF !important; }
[data-testid="stFileUploader"] * { color: #374151 !important; }

/* DATAFRAMES */
[data-testid="stDataFrame"] { border: 1px solid #E2E8F0 !important; }

/* MISC */
#MainMenu, footer, header  { visibility: hidden; }
[data-testid="stToolbar"]  { display: none !important; }
*:focus-visible            { outline: 2px solid #1B5FBF !important; outline-offset: 2px !important; }
</style>
""", unsafe_allow_html=True)

# ─────────────────────────────────────────────────────────────────────────────
from app.ui.pages import (
    _init_state, render_upload_page, render_mapping_page,
    render_merge_page, render_seeds_page, render_history_page,
    render_validation_page, render_logs_page, render_diagnostico_page,
)

_init_state()

# ── SIDEBAR ───────────────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown(
        '<div style="padding:18px 12px 14px;border-bottom:1px solid #1E293B;margin-bottom:12px;">'
        '<div style="font-size:16px;font-weight:700;color:#F1F5F9;letter-spacing:-.02em;">VIVOHUB</div>'
        '<div style="font-size:9px;font-weight:700;color:#334155;letter-spacing:.12em;'
        'text-transform:uppercase;margin-top:3px;">Data Merger v2</div>'
        '</div>',
        unsafe_allow_html=True,
    )

    page = st.radio("", [
        "1. Carregar Arquivos",
        "2. Mapeamento",
        "3. Gerar Tabela Final",
        "4. Validação",
        "5. Seeds / Referências",
        "6. Histórico",
        "7. Logs",
        "8. Diagnóstico",
    ], key="nav", label_visibility="collapsed")

    st.markdown("<hr>", unsafe_allow_html=True)

    def _status(label: str, key: str) -> None:
        ok  = st.session_state.get(key) is not None
        dot = "#22C55E" if ok else "#1E293B"
        fg  = "#E2E8F0" if ok else "#475569"
        st.markdown(
            f'<div style="display:flex;align-items:center;gap:9px;padding:3px 0;">'
            f'<span style="width:6px;height:6px;border-radius:50%;background:{dot};'
            f'flex-shrink:0;display:inline-block;"></span>'
            f'<span style="font-size:11px;font-weight:500;color:{fg};">{label}</span>'
            f'</div>',
            unsafe_allow_html=True,
        )

    _status("Science",    "sci_df")
    _status("Portal",     "por_df")
    _status("Arquivo 3",  "arq3_df")
    _status("Mapeamento", "wizard_cfg")
    _status("Resultado",  "merged_df")

    merged = st.session_state.get("merged_df")
    if merged is not None:
        n = len(merged)
        st.markdown(
            f'<div style="margin-top:10px;padding:10px 12px;background:#060D1A;'
            f'border-left:2px solid #1B5FBF;">'
            f'<div style="font-size:9px;font-weight:700;letter-spacing:.1em;'
            f'text-transform:uppercase;color:#334155;margin-bottom:3px;">Resultado</div>'
            f'<div style="font-size:20px;font-weight:700;color:#F1F5F9;line-height:1.1;">'
            f'{n:,}<span style="font-size:11px;font-weight:400;color:#475569;"> linhas</span>'
            f'</div></div>',
            unsafe_allow_html=True,
        )

    st.markdown("<hr>", unsafe_allow_html=True)
    st.markdown(
        f'<div style="font-size:10px;color:#334155;margin-bottom:8px;">{Path(_DB).name}</div>',
        unsafe_allow_html=True,
    )
    if st.button("Reiniciar sessão", key="btn_reset"):
        for k in list(st.session_state.keys()):
            del st.session_state[k]
        st.rerun()

# ── HEADER DE PÁGINA ──────────────────────────────────────────────────────────
p = page.split(". ", 1)[-1]
_PAGE_META = {
    "Carregar Arquivos":   ("Carregar Arquivos",       "Importação — Science · Portal · Arquivo 3"),
    "Mapeamento":          ("Mapeamento de Colunas",   "Configuração das chaves e campos de junção"),
    "Gerar Tabela Final":  ("Gerar Tabela Final",      "Execução do pipeline · Consolidação de rotas"),
    "Validação":           ("Relatório de Qualidade",  "Validação e integridade dos dados processados"),
    "Seeds / Referências": ("Seeds e Referências",     "Tabelas de referência CN→UF e CNL"),
    "Histórico":           ("Histórico de Versões",    "Versões salvas do resultado consolidado"),
    "Logs":                ("Logs e Auditoria",        "Registro de operações e eventos do sistema"),
    "Diagnóstico":         ("Diagnóstico",             "Análise de correspondência entre fontes"),
}
title, subtitle = _PAGE_META.get(p, (p, ""))

# Inline styles garantem renderização correta independente do tema
st.markdown(
    f'<div style="background:#0F172A;border-bottom:2px solid #1B5FBF;'
    f'padding:16px 24px 13px;margin-bottom:20px;">'
    f'<div style="font-size:17px;font-weight:700;color:#F1F5F9;letter-spacing:-.01em;'
    f'margin:0 0 3px;">{title}</div>'
    f'<div style="font-size:10px;font-weight:600;color:#475569;letter-spacing:.09em;'
    f'text-transform:uppercase;">{subtitle}</div>'
    f'</div>',
    unsafe_allow_html=True,
)

# ── ROTEAMENTO ────────────────────────────────────────────────────────────────
if   "Carregar"    in p: render_upload_page()
elif "Mapeamento"  in p: render_mapping_page()
elif "Gerar"       in p: render_merge_page()
elif "Validação"   in p: render_validation_page()
elif "Seeds"       in p: render_seeds_page()
elif "Histórico"   in p: render_history_page()
elif "Logs"        in p: render_logs_page()
elif "Diagnóstico" in p: render_diagnostico_page()

# ── RODAPÉ ────────────────────────────────────────────────────────────────────
st.markdown(
    '<div style="border-top:1px solid #E2E8F0;margin-top:40px;padding-top:12px;'
    'text-align:center;font-size:10px;color:#9CA3AF;letter-spacing:.05em;">'
    'VIVOHUB · Data Merger v2 · Processamento 100% local · Nenhum dado transmitido externamente'
    '</div>',
    unsafe_allow_html=True,
)