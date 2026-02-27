"""
app.py — Ponto de entrada principal do Data Merger
Execute com: streamlit run app.py
"""
from __future__ import annotations

import os
from pathlib import Path

import streamlit as st

# ── Configuração da página ─────────────────────────────────────────────────
st.set_page_config(
    page_title="Data Merger — Science + Portal",
    page_icon="🔗",
    layout="wide",
    initial_sidebar_state="expanded",
    menu_items={
        "About": "Data Merger v1.0 — Combina tabelas Science e Portal de Cadastros.",
    },
)

# ── Carrega .env se disponível ─────────────────────────────────────────────
try:
    from dotenv import load_dotenv  # type: ignore
    load_dotenv()
except ImportError:
    pass

# ── Inicializa banco ────────────────────────────────────────────────────────
from app.db.session import init_db

_db_path = os.environ.get("DATABASE_PATH", str(Path("data") / "app.db"))
init_db(_db_path)

# ── Importa páginas ─────────────────────────────────────────────────────────
from app.ui.pages import (
    _init_state,
    render_history_page,
    render_logs_page,
    render_mapping_page,
    render_merge_page,
    render_upload_page,
    render_validation_page,
)

# ── CSS customizado ─────────────────────────────────────────────────────────
st.markdown("""
<style>
/* ---- Acessibilidade: foco visível ---- */
*:focus-visible {
    outline: 3px solid #4f8ef7 !important;
    outline-offset: 2px;
}
/* ---- Sidebar ---- */
[data-testid="stSidebar"] {
    background-color: #1a1a2e;
}
[data-testid="stSidebar"] * {
    color: #e0e0e0 !important;
}
/* ---- Header strip ---- */
.main-header {
    background: linear-gradient(90deg, #2d6a4f 0%, #1b4332 100%);
    padding: 1rem 1.5rem;
    border-radius: 8px;
    margin-bottom: 1rem;
    color: white;
}
/* ---- Botão primário ---- */
[data-testid="baseButton-primary"] {
    background-color: #2d6a4f !important;
    border: none;
    font-weight: 600;
}
/* ---- Skip-nav para acessibilidade ---- */
.skip-nav {
    position: absolute;
    top: -40px;
    left: 0;
    background: #000;
    color: #fff;
    padding: 8px;
    z-index: 9999;
    border-radius: 4px;
}
.skip-nav:focus { top: 0; }
/* ---- Métricas ---- */
[data-testid="stMetric"] {
    background: #f0f4ff;
    border-radius: 8px;
    padding: 0.5rem 1rem;
}
</style>
""", unsafe_allow_html=True)

# Skip navigation (acessibilidade)
st.markdown('<a href="#main-content" class="skip-nav">Ir para conteúdo principal</a>',
            unsafe_allow_html=True)

# ── Estado global ────────────────────────────────────────────────────────────
_init_state()

# ── Sidebar / Navegação ──────────────────────────────────────────────────────
with st.sidebar:
    st.markdown("""
    <div style="text-align:center; padding: 1rem 0;">
        <h2 style="margin:0">🔗 Data Merger</h2>
        <small>Science + Portal de Cadastros</small>
    </div>
    """, unsafe_allow_html=True)

    st.markdown("---")
    st.markdown("### 📋 Navegação")

    page = st.radio(
        "Selecione uma etapa:",
        options=[
            "1. 📂 Carregar Arquivos",
            "2. 🗺️ Mapeamento",
            "3. ⚙️ Gerar Tabela Final",
            "4. ✅ Validação",
            "5. 🕐 Histórico",
            "6. 📋 Logs",
        ],
        key="nav_page",
        label_visibility="collapsed",
    )

    st.markdown("---")
    st.markdown("### ℹ️ Status")

    sci_ok = st.session_state.get("sci_df") is not None
    por_ok = st.session_state.get("por_df") is not None
    map_ok = st.session_state.get("current_mapping") is not None
    mer_ok = st.session_state.get("merged_df") is not None

    def _status(label: str, ok: bool) -> None:
        icon = "✅" if ok else "⏳"
        st.markdown(f"{icon} {label}")

    _status("Science carregado", sci_ok)
    _status("Portal carregado", por_ok)
    _status("Mapeamento configurado", map_ok)
    _status("Merge gerado", mer_ok)

    if mer_ok:
        n = len(st.session_state["merged_df"])
        st.caption(f"**{n:,}** linhas no resultado")

    st.markdown("---")
    db_path_display = Path(_db_path).resolve()
    st.caption(f"🗄️ Banco: `{db_path_display}`")

    # Botão reset
    if st.button("🔄 Reiniciar Sessão", key="btn_reset"):
        for k in list(st.session_state.keys()):
            del st.session_state[k]
        st.rerun()


# ── Conteúdo principal ────────────────────────────────────────────────────────
st.markdown('<div id="main-content"></div>', unsafe_allow_html=True)

# Header
st.markdown("""
<div class="main-header" role="banner">
    <h1 style="margin:0; color:white">🔗 Data Merger — Science + Portal de Cadastros</h1>
    <p style="margin:0; opacity:0.85">Combine, filtre e persista dados de telecomunicações com rastreabilidade completa.</p>
</div>
""", unsafe_allow_html=True)

# Roteamento
page_key = page.split(". ", 1)[-1].strip()

if "Carregar" in page_key:
    render_upload_page()
elif "Mapeamento" in page_key:
    render_mapping_page()
elif "Gerar" in page_key:
    render_merge_page()
elif "Validação" in page_key:
    render_validation_page()
elif "Histórico" in page_key:
    render_history_page()
elif "Logs" in page_key:
    render_logs_page()

# Footer
st.markdown("---")
st.markdown(
    "<div style='text-align:center; color:#888; font-size:0.8rem'>"
    "Data Merger v1.0 · Dados processados 100% localmente · "
    "Nenhuma informação enviada para servidores externos."
    "</div>",
    unsafe_allow_html=True,
)

