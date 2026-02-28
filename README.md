"""
app.py — Data Merger v2
Execute: streamlit run app.py
"""
from __future__ import annotations
import os
from pathlib import Path

import streamlit as st

st.set_page_config(
    page_title="Data Merger v2",
    page_icon="🔗",
    layout="wide",
    initial_sidebar_state="expanded",
)

# Carrega .env se existir
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass

# Inicializa banco + seeds automáticos na primeira execução
from app.db.session import init_db

_DB = os.environ.get("DATABASE_PATH", str(Path("data") / "app.db"))
init_db(_DB)

# Carrega seeds automaticamente se tabelas estiverem vazias
from app.db.session import session_scope
from app.db.repository import Repository

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
        print(f"[AVISO] Erro ao carregar seeds: {e}")

_auto_load_seeds()

# CSS
st.markdown("""
<style>
*:focus-visible { outline: 3px solid #4f8ef7 !important; outline-offset: 2px; }
[data-testid="stSidebar"] { background-color: #0f1117; }
[data-testid="stSidebar"] * { color: #e8e8e8 !important; }
.main-header {
    background: linear-gradient(90deg,#3a0ca3,#7209b7);
    padding:1rem 1.5rem; border-radius:8px; margin-bottom:1rem;
}
[data-testid="baseButton-primary"] { background-color:#7209b7!important; }
.skip-nav { position:absolute; top:-40px; left:0; background:#000;
            color:#fff; padding:8px; z-index:9999; border-radius:4px; }
.skip-nav:focus { top:0; }
</style>
""", unsafe_allow_html=True)

st.markdown('<a href="#main" class="skip-nav">Ir para conteúdo</a>',
            unsafe_allow_html=True)

from app.ui.pages import (
    _init_state, render_upload_page, render_mapping_page,
    render_merge_page, render_seeds_page, render_history_page,
    render_validation_page, render_logs_page,
)

_init_state()

# ── Sidebar ────────────────────────────────────────────────────────────────
with st.sidebar:
    st.markdown("""
    <div style="text-align:center;padding:1rem 0">
        <h2 style="margin:0">🔗 Data Merger v2</h2>
        <small>Science + Portal + Arq3</small>
    </div>
    """, unsafe_allow_html=True)
    st.markdown("---")

    page = st.radio("Etapa:", [
        "1. 📂 Carregar Arquivos",
        "2. 🗺️ Mapeamento",
        "3. ⚙️ Gerar Tabela Final",
        "4. ✅ Validação",
        "5. 🌱 Seeds / Referências",
        "6. 🕐 Histórico",
        "7. 📋 Logs",
    ], key="nav", label_visibility="collapsed")

    st.markdown("---")
    def _chk(label, key):
        ok = st.session_state.get(key) is not None
        st.markdown(f"{'✅' if ok else '⏳'} {label}")
    _chk("Science",    "sci_df")
    _chk("Portal",     "por_df")
    _chk("Arquivo 3",  "arq3_df")
    _chk("Mapeamento", "wizard_cfg")
    _chk("Resultado",  "merged_df")

    if st.session_state.get("merged_df") is not None:
        n = len(st.session_state["merged_df"])
        st.caption(f"**{n:,}** linhas no resultado")

    st.markdown("---")
    st.caption(f"🗄️ `{Path(_DB).name}`")
    if st.button("🔄 Reiniciar sessão", key="btn_reset"):
        for k in list(st.session_state.keys()):
            del st.session_state[k]
        st.rerun()

# ── Conteúdo ───────────────────────────────────────────────────────────────
st.markdown('<div id="main"></div>', unsafe_allow_html=True)
st.markdown("""
<div class="main-header">
    <h1 style="margin:0;color:white">🔗 Data Merger v2</h1>
    <p style="margin:0;opacity:.85">Science · Portal de Cadastros · Arquivo 3 de Referência</p>
</div>
""", unsafe_allow_html=True)

p = page.split(". ", 1)[-1]
if "Carregar"    in p: render_upload_page()
elif "Mapeamento" in p: render_mapping_page()
elif "Gerar"      in p: render_merge_page()
elif "Validação"  in p: render_validation_page()
elif "Seeds"      in p: render_seeds_page()
elif "Histórico"  in p: render_history_page()
elif "Logs"       in p: render_logs_page()

st.markdown("---")
st.markdown("<div style='text-align:center;color:#666;font-size:.8rem'>"
            "Data Merger v2 · 100% local · nenhum dado enviado externamente</div>",
            unsafe_allow_html=True)
