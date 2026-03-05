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
    page_icon="assets/favicon.ico" if Path("assets/favicon.ico").exists() else None,
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

def _auto_load_seeds():
    try:
        with session_scope() as s:
            repo = Repository(s)
            if repo.cnl_count() == 0:
                cnl_sql = Path("seeds") / "cnl.sql"
                if cnl_sql.exists() and cnl_sql.stat().st_size > 0:
                    repo.load_cnl_seeds(str(cnl_sql))
            if repo.cn_to_uf_count() == 0:
                uf_csv = Path("seeds") / "cn_to_uf.csv"
                if uf_csv.exists() and uf_csv.stat().st_size > 0:
                    repo.load_cn_to_uf_csv(str(uf_csv))
    except Exception as e:
        print(f"[AVISO] Seeds: {e}")

_auto_load_seeds()

# ── CSS corporativo global ────────────────────────────────────────────────────
st.markdown("""
<style>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&display=swap');

/* Reset e tipografia base */
html, body, [class*="css"] {
    font-family: 'IBM Plex Sans', 'Segoe UI', 'Helvetica Neue', Arial, sans-serif !important;
}

/* Sidebar */
[data-testid="stSidebar"] {
    background-color: #1C2536 !important;
    border-right: 1px solid #2D3748;
}
[data-testid="stSidebar"] * {
    color: #E2E8F0 !important;
}
[data-testid="stSidebar"] .stRadio label {
    font-size: 13px !important;
    font-weight: 500 !important;
    letter-spacing: 0.01em;
    padding: 4px 0;
}
[data-testid="stSidebar"] [data-testid="stMarkdownContainer"] p {
    font-size: 11px !important;
    color: #94A3B8 !important;
}
[data-testid="stSidebar"] hr {
    border-color: #2D3748 !important;
    margin: 10px 0 !important;
}
[data-testid="stSidebar"] .stButton button {
    background: transparent !important;
    border: 1px solid #374151 !important;
    color: #94A3B8 !important;
    font-size: 11px !important;
    font-weight: 500 !important;
    letter-spacing: 0.03em !important;
    text-transform: uppercase !important;
    width: 100% !important;
    margin-top: 4px !important;
}
[data-testid="stSidebar"] .stButton button:hover {
    border-color: #1B5FBF !important;
    color: #93C5FD !important;
}

/* Área principal */
.main .block-container {
    padding-top: 1rem !important;
    padding-bottom: 2rem !important;
    max-width: 100% !important;
}

/* Header corporativo */
.corp-header {
    background: #1C2536;
    padding: 14px 24px;
    margin-bottom: 20px;
    border-bottom: 3px solid #1B5FBF;
    display: flex;
    align-items: baseline;
    gap: 16px;
}
.corp-header-title {
    font-size: 18px;
    font-weight: 700;
    color: #FFFFFF;
    letter-spacing: -0.01em;
    margin: 0;
}
.corp-header-sub {
    font-size: 11px;
    font-weight: 500;
    color: #94A3B8;
    letter-spacing: 0.05em;
    text-transform: uppercase;
    margin: 0;
}

/* Títulos de seção */
h1, h2, h3 { letter-spacing: -0.01em !important; }
h1 { font-size: 20px !important; font-weight: 700 !important; color: #111827 !important; }
h2 { font-size: 16px !important; font-weight: 700 !important; color: #1C2536 !important; }
h3 { font-size: 14px !important; font-weight: 600 !important; color: #1C2536 !important; }

/* Métricas */
[data-testid="metric-container"] {
    background: #F8F9FA;
    border: 1px solid #E9ECEF;
    padding: 10px 14px !important;
    border-left: 3px solid #1B5FBF;
}
[data-testid="metric-container"] label {
    font-size: 10px !important;
    font-weight: 700 !important;
    letter-spacing: 0.07em !important;
    text-transform: uppercase !important;
    color: #6B7280 !important;
}
[data-testid="metric-container"] [data-testid="stMetricValue"] {
    font-size: 22px !important;
    font-weight: 700 !important;
    color: #111827 !important;
}

/* Botões primários */
[data-testid="baseButton-primary"] {
    background-color: #1B5FBF !important;
    border: none !important;
    border-radius: 0 !important;
    font-size: 12px !important;
    font-weight: 700 !important;
    letter-spacing: 0.05em !important;
    text-transform: uppercase !important;
}
[data-testid="baseButton-primary"]:hover {
    background-color: #1A54A8 !important;
}

/* Botões secundários */
[data-testid="baseButton-secondary"] {
    border-radius: 0 !important;
    font-size: 12px !important;
    font-weight: 600 !important;
    letter-spacing: 0.03em !important;
}

/* Download buttons */
[data-testid="stDownloadButton"] button {
    border-radius: 0 !important;
    font-size: 11px !important;
    font-weight: 600 !important;
    letter-spacing: 0.04em !important;
    text-transform: uppercase !important;
    background: #FFFFFF !important;
    border: 1px solid #D1D5DB !important;
    color: #374151 !important;
}
[data-testid="stDownloadButton"] button:hover {
    border-color: #1B5FBF !important;
    color: #1B5FBF !important;
}

/* Expanders */
[data-testid="stExpander"] {
    border: 1px solid #E9ECEF !important;
    border-radius: 0 !important;
    margin-bottom: 6px !important;
}
[data-testid="stExpander"] summary {
    font-size: 12px !important;
    font-weight: 600 !important;
    letter-spacing: 0.02em !important;
    color: #1C2536 !important;
    padding: 8px 12px !important;
    background: #F8F9FA !important;
}
[data-testid="stExpander"] summary:hover {
    background: #EFF6FF !important;
    color: #1B5FBF !important;
}

/* Info / Warning / Success */
[data-testid="stAlert"] {
    border-radius: 0 !important;
    border-left-width: 3px !important;
    font-size: 12px !important;
    font-weight: 500 !important;
}

/* Selectbox / text inputs */
[data-testid="stSelectbox"] div[data-baseweb="select"] > div,
[data-testid="stTextInput"] input,
[data-testid="stMultiSelect"] div[data-baseweb="select"] > div {
    border-radius: 0 !important;
    font-size: 12px !important;
    border-color: #D1D5DB !important;
}

/* Dataframes nativos */
[data-testid="stDataFrame"] iframe {
    border: 1px solid #E9ECEF !important;
}

/* Tabs */
[data-testid="stTabs"] [data-baseweb="tab"] {
    font-size: 12px !important;
    font-weight: 600 !important;
    letter-spacing: 0.04em !important;
    text-transform: uppercase !important;
    border-radius: 0 !important;
}
[data-testid="stTabs"] [aria-selected="true"] {
    border-bottom: 2px solid #1B5FBF !important;
    color: #1B5FBF !important;
}

/* File uploader */
[data-testid="stFileUploader"] {
    border: 1.5px dashed #D1D5DB !important;
    border-radius: 0 !important;
    background: #FAFAFA !important;
    padding: 12px !important;
}
[data-testid="stFileUploader"]:hover {
    border-color: #1B5FBF !important;
    background: #EFF6FF !important;
}

/* Ocultar elementos desnecessários */
#MainMenu { visibility: hidden; }
footer    { visibility: hidden; }
header    { visibility: hidden; }

/* Focus acessível */
*:focus-visible { outline: 2px solid #1B5FBF !important; outline-offset: 2px; }
</style>
""", unsafe_allow_html=True)

from app.ui.pages import (
    _init_state, render_upload_page, render_mapping_page,
    render_merge_page, render_seeds_page, render_history_page,
    render_validation_page, render_logs_page, render_diagnostico_page,
)

_init_state()

# ── Sidebar ───────────────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown("""
    <div style="padding:16px 8px 12px;border-bottom:1px solid #2D3748;margin-bottom:12px;">
        <div style="font-size:15px;font-weight:700;color:#FFFFFF;letter-spacing:-0.01em;">
            VIVOHUB
        </div>
        <div style="font-size:10px;font-weight:600;color:#64748B;letter-spacing:0.1em;
                    text-transform:uppercase;margin-top:3px;">
            Data Merger v2
        </div>
    </div>
    """, unsafe_allow_html=True)

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

    def _chk(label, key):
        ok = st.session_state.get(key) is not None
        color  = "#22C55E" if ok else "#374151"
        symbol = "+" if ok else "—"
        st.markdown(
            f'<div style="display:flex;align-items:center;gap:8px;padding:3px 0;">'
            f'<span style="color:{color};font-weight:700;font-size:12px;">{symbol}</span>'
            f'<span style="font-size:12px;color:{"#E2E8F0" if ok else "#64748B"};">{label}</span>'
            f'</div>',
            unsafe_allow_html=True,
        )

    _chk("Science carregado",  "sci_df")
    _chk("Portal carregado",   "por_df")
    _chk("Arquivo 3 carregado","arq3_df")
    _chk("Mapeamento salvo",   "wizard_cfg")
    _chk("Resultado gerado",   "merged_df")

    if st.session_state.get("merged_df") is not None:
        n = len(st.session_state["merged_df"])
        st.markdown(
            f'<div style="margin-top:10px;padding:8px 10px;background:#0F172A;'
            f'border-left:3px solid #1B5FBF;">'
            f'<span style="font-size:11px;color:#94A3B8;">Resultado atual</span><br>'
            f'<span style="font-size:16px;font-weight:700;color:#FFFFFF;">{n:,}</span>'
            f'<span style="font-size:11px;color:#64748B;"> linhas</span>'
            f'</div>',
            unsafe_allow_html=True,
        )

    st.markdown("<hr>", unsafe_allow_html=True)
    st.markdown(
        f'<div style="font-size:10px;color:#475569;margin-bottom:6px;">'
        f'Banco: {Path(_DB).name}</div>',
        unsafe_allow_html=True,
    )
    if st.button("Reiniciar sessão", key="btn_reset"):
        for k in list(st.session_state.keys()):
            del st.session_state[k]
        st.rerun()

# ── Header corporativo ────────────────────────────────────────────────────────
p = page.split(". ", 1)[-1]
page_titles = {
    "Carregar Arquivos":  ("Carregar Arquivos",      "Importação de dados — Science · Portal · Arquivo 3"),
    "Mapeamento":         ("Mapeamento de Colunas",  "Configuração das chaves e campos de junção"),
    "Gerar Tabela Final": ("Gerar Tabela Final",     "Execução do pipeline e consolidação de rotas"),
    "Validação":          ("Relatório de Qualidade", "Validação e integridade dos dados processados"),
    "Seeds / Referências":("Seeds e Referências",    "Tabelas de referência CN→UF e CNL"),
    "Histórico":          ("Histórico de Versões",   "Versões salvas do resultado consolidado"),
    "Logs":               ("Logs e Auditoria",       "Registro de operações e eventos do sistema"),
    "Diagnóstico":        ("Diagnóstico",            "Análise de correspondência entre fontes e Arquivo 3"),
}
title, subtitle = page_titles.get(p, (p, ""))
st.markdown(
    f'<div class="corp-header">'
    f'<h1 class="corp-header-title">{title}</h1>'
    f'<span class="corp-header-sub">{subtitle}</span>'
    f'</div>',
    unsafe_allow_html=True,
)

# ── Roteamento ────────────────────────────────────────────────────────────────
if   "Carregar"    in p: render_upload_page()
elif "Mapeamento"  in p: render_mapping_page()
elif "Gerar"       in p: render_merge_page()
elif "Validação"   in p: render_validation_page()
elif "Seeds"       in p: render_seeds_page()
elif "Histórico"   in p: render_history_page()
elif "Logs"        in p: render_logs_page()
elif "Diagnóstico" in p: render_diagnostico_page()

# ── Rodapé ────────────────────────────────────────────────────────────────────
st.markdown(
    '<div style="border-top:1px solid #E9ECEF;margin-top:32px;padding-top:10px;'
    'text-align:center;font-size:10px;color:#9CA3AF;letter-spacing:0.05em;">'
    'VIVOHUB · Data Merger v2 · Processamento 100% local · Nenhum dado transmitido externamente'
    '</div>',
    unsafe_allow_html=True,
)

