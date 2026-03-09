"""
app.py — VIVOHUB · Data Merger v2
Fluxo principal:
  1. Ao abrir: carrega Tabela Final do banco automaticamente
  2. Carregar Arquivos → gerar hash → deduplicar → salvar
  3. Gerar Tabela Final → exibir grade interativa
  4. Clique no cabeçalho → dashboard instantâneo
  5. Exportar PPTX (coluna ou geral)

Nenhuma tela técnica é exibida na interface.
"""
from __future__ import annotations

import io
import os
from datetime import datetime
from pathlib import Path

import pandas as pd
import streamlit as st

# ─── Configuração da página ──────────────────────────────────────────────────
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

# ─── Banco ───────────────────────────────────────────────────────────────────
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

# ─── CSS global ──────────────────────────────────────────────────────────────
st.markdown("""
<style>
@import url('https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap');

/* MODO CLARO FORÇADO */
html, body, .stApp,
[data-testid="stAppViewContainer"],
[data-testid="stAppViewBlockContainer"],
[data-testid="stMain"], section[data-testid="stMain"],
.main, .block-container, .main .block-container {
    background-color: #F8F9FA !important;
    color: #111827 !important;
}
.main .block-container {
    padding-top: 0 !important;
    padding-bottom: 2rem !important;
    max-width: 100% !important;
}

/* TIPOGRAFIA */
html, body, *, *::before, *::after {
    font-family: 'IBM Plex Sans', 'Segoe UI', Arial, sans-serif !important;
}
[data-testid="stMarkdownContainer"] *,
[data-testid="stVerticalBlock"] p,
[data-testid="stVerticalBlock"] span,
.element-container p, .element-container span {
    color: #111827 !important;
}

/* SIDEBAR */
[data-testid="stSidebar"],
[data-testid="stSidebar"] > div,
[data-testid="stSidebar"] section,
[data-testid="stSidebarUserContent"] { background-color: #0F172A !important; }
[data-testid="stSidebar"] * { color: #CBD5E1 !important; background-color: transparent !important; }
[data-testid="stSidebar"] hr { border-color: #1E293B !important; margin: 8px 0 !important; }
[data-testid="stSidebar"] [data-testid="stRadio"] label {
    font-size: 12px !important; font-weight: 500 !important;
    color: #94A3B8 !important; padding: 4px 6px !important; display: block !important;
}
[data-testid="stSidebar"] [data-testid="stRadio"] label:hover { color: #F1F5F9 !important; }
/* Botão reiniciar — discreto */
[data-testid="stSidebar"] button {
    background: transparent !important;
    border: 1px solid #1E293B !important;
    color: #475569 !important;
    font-size: 10px !important; font-weight: 500 !important;
    letter-spacing: .05em !important;
    width: 100% !important; padding: 4px 8px !important; margin-top: 4px !important;
    text-transform: none !important;
}
[data-testid="stSidebar"] button:hover { border-color: #334155 !important; color: #94A3B8 !important; }

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
    letter-spacing: .09em !important; text-transform: uppercase !important;
    color: #6B7280 !important;
}
[data-testid="stMetricValue"] > div { font-size: 20px !important; font-weight: 700 !important; color: #111827 !important; }
[data-testid="stMetricDelta"]        { font-size: 11px !important; color: #6B7280 !important; }

/* BOTÕES */
button[kind="primary"],
[data-testid="baseButton-primary"] {
    background-color: #1B5FBF !important; border: none !important;
    border-radius: 0 !important; color: #FFFFFF !important;
    font-size: 11px !important; font-weight: 700 !important;
    letter-spacing: .06em !important; text-transform: uppercase !important;
}
button[kind="primary"]:hover,
[data-testid="baseButton-primary"]:hover { background-color: #1549A0 !important; }
button[kind="secondary"],
[data-testid="baseButton-secondary"] {
    border-radius: 0 !important; background: #FFFFFF !important;
    border: 1px solid #D1D5DB !important; color: #374151 !important;
    font-size: 11px !important; font-weight: 600 !important;
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
input, [data-baseweb="input"] input, [data-baseweb="select"] > div {
    background-color: #FFFFFF !important; color: #111827 !important;
    border-color: #D1D5DB !important; border-radius: 0 !important; font-size: 12px !important;
}
[data-testid="stTextInput"] label,
[data-testid="stSelectbox"] label,
[data-testid="stFileUploader"] label { color: #374151 !important; font-size: 11px !important; font-weight: 600 !important; }

/* EXPANDERS */
[data-testid="stExpander"] {
    background: #FFFFFF !important; border: 1px solid #E2E8F0 !important;
    border-radius: 0 !important; margin-bottom: 6px !important;
}
[data-testid="stExpander"] summary {
    background: #F8F9FA !important; color: #1C2536 !important;
    font-size: 12px !important; font-weight: 600 !important; padding: 9px 14px !important;
}
[data-testid="stExpander"] summary:hover { background: #EFF6FF !important; color: #1B5FBF !important; }
/* Corrige label "keyboard_double_arrow" / ícone literal */
[data-testid="stExpander"] summary svg { display: inline !important; }
[data-testid="stExpander"] summary span[data-testid] { display: none !important; }

/* ALERTAS */
[data-testid="stAlert"] { border-radius: 0 !important; border-left-width: 3px !important; font-size: 12px !important; }
[data-testid="stAlert"] p { color: inherit !important; }

/* FILE UPLOADER */
[data-testid="stFileUploader"] section {
    background: #FFFFFF !important; border: 1.5px dashed #D1D5DB !important; border-radius: 0 !important;
}
[data-testid="stFileUploader"] section:hover { border-color: #1B5FBF !important; background: #F0F7FF !important; }

/* DATAFRAMES */
[data-testid="stDataFrame"] { border: 1px solid #E2E8F0 !important; }

/* OCULTA chrome desnecessário */
#MainMenu, footer, header { visibility: hidden; }
[data-testid="stToolbar"]  { display: none !important; }

/* FOCO */
*:focus-visible { outline: 2px solid #1B5FBF !important; outline-offset: 2px !important; }
</style>
""", unsafe_allow_html=True)

# ─── Imports internos ────────────────────────────────────────────────────────
from app.core.merge import build_merged_df
from app.core.normalize import (apply_column_normalization,
                                 infer_and_coerce_types, strip_whitespace)
from app.core.hash_utils import BUSINESS_COLS
from app.core.analytics import precompute_all
from app.db.tf_repository import TabelaFinalRepository
from app.io.readers import list_sheets, read_file
from app.ui.interactive_table import render_interactive_table
from app.ui.dashboard import render_column_dashboard
from app.export.pptx_builder import build_column_pptx, build_general_pptx
from app.utils.ids import version_tag
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

# ─── Utilitários de UI ────────────────────────────────────────────────────────

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
        f'padding:10px 14px;margin:6px 0;font-size:12px;color:{fg};font-weight:500;">'
        f'{text}</div>',
        unsafe_allow_html=True,
    )

def _divider():
    st.markdown('<div style="height:1px;background:#E2E8F0;margin:14px 0;"></div>',
                unsafe_allow_html=True)

def _label(text: str):
    st.markdown(
        f'<div style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        f'text-transform:uppercase;color:#9CA3AF;margin-bottom:5px;">{text}</div>',
        unsafe_allow_html=True,
    )

def _page_header(title: str, subtitle: str = ""):
    st.markdown(
        f'<div style="background:#0F172A;border-bottom:2px solid #1B5FBF;'
        f'padding:14px 24px 12px;margin-bottom:18px;">'
        f'<div style="font-size:17px;font-weight:700;color:#F1F5F9;margin:0 0 2px;">{title}</div>'
        f'<div style="font-size:10px;font-weight:600;color:#475569;letter-spacing:.08em;'
        f'text-transform:uppercase;">{subtitle}</div>'
        f'</div>',
        unsafe_allow_html=True,
    )

# ─── Estado inicial ───────────────────────────────────────────────────────────

def _init_state() -> None:
    defaults = {
        "tabela_final":     None,   # DataFrame com 8 colunas de negócio
        "analytics_cache":  None,   # dict col → {kpis, freq}
        "selected_col":     None,   # coluna selecionada para dashboard
        "sci_df": None, "sci_filename": "", "sci_hash": "", "sci_sheet": None,
        "por_df": None, "por_filename": "", "por_hash": "", "por_sheet": None,
        "arq3_df": None, "arq3_filename": "", "arq3_hash": "", "arq3_sheet": None,
        "wizard_cfg": {},
        "upload_msg": None,  # (text, kind)
    }
    for k, v in defaults.items():
        if k not in st.session_state:
            st.session_state[k] = v

_init_state()

# ─── Carga automática ao iniciar ─────────────────────────────────────────────
if _ss("tabela_final") is None:
    try:
        with session_scope() as s:
            df_db = TabelaFinalRepository(s).load()
        if not df_db.empty:
            st.session_state["tabela_final"]    = df_db
            st.session_state["analytics_cache"] = precompute_all(df_db)
    except Exception as e:
        log.warning("Carga automática falhou: %s", e)

# ─── SIDEBAR ─────────────────────────────────────────────────────────────────
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
        "2. Gerar Tabela Final",
        "3. Tabela Final",
    ], key="nav", label_visibility="collapsed")

    st.markdown("<hr>", unsafe_allow_html=True)

    # Status compacto
    def _dot(label: str, key: str):
        ok  = st.session_state.get(key) is not None
        dot = "#22C55E" if ok else "#1E293B"
        fg  = "#CBD5E1" if ok else "#475569"
        st.markdown(
            f'<div style="display:flex;align-items:center;gap:8px;padding:3px 0;">'
            f'<span style="width:6px;height:6px;border-radius:50%;background:{dot};'
            f'flex-shrink:0;display:inline-block;"></span>'
            f'<span style="font-size:11px;font-weight:500;color:{fg};">{label}</span>'
            f'</div>',
            unsafe_allow_html=True,
        )

    _dot("Science",   "sci_df")
    _dot("Portal",    "por_df")
    _dot("Arquivo 3", "arq3_df")
    _dot("Resultado", "tabela_final")

    tf = _ss("tabela_final")
    if tf is not None:
        n = len(tf)
        st.markdown(
            f'<div style="margin-top:10px;padding:9px 12px;background:#060D1A;'
            f'border-left:2px solid #1B5FBF;">'
            f'<div style="font-size:9px;font-weight:700;letter-spacing:.1em;'
            f'text-transform:uppercase;color:#334155;margin-bottom:2px;">Rotas</div>'
            f'<div style="font-size:20px;font-weight:700;color:#F1F5F9;line-height:1.1;">'
            f'{n:,}</div></div>',
            unsafe_allow_html=True,
        )

    st.markdown("<hr>", unsafe_allow_html=True)
    st.markdown(
        f'<div style="font-size:10px;color:#334155;margin-bottom:6px;">{Path(_DB).name}</div>',
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
    _page_header("Carregar Arquivos", "Importação — Science · Portal · Arquivo 3")

    st.markdown(
        '<p style="font-size:12px;color:#6B7280;margin:0 0 16px;">'
        'Faça o upload dos arquivos. O processamento é 100% local.</p>',
        unsafe_allow_html=True,
    )

    def _handle_upload(uploaded_file, prefix: str, label: str):
        if uploaded_file is None:
            return
        ext = uploaded_file.name.split(".")[-1].lower()
        raw = uploaded_file.read()
        sheet = None
        if ext in ("xlsx", "xls"):
            import io as _io
            sheets = list_sheets(_io.BytesIO(raw))
            if len(sheets) > 1:
                sheet = st.selectbox(f"Planilha — {label}", sheets,
                                     key=f"sheet_{prefix}")
            elif sheets:
                sheet = sheets[0]
        try:
            import io as _io
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
        (c3, "arq3", "Arquivo 3 — Referência"),
    ]:
        with col:
            _label(label)
            f = st.file_uploader(label, type=["xlsx","xls","csv","parquet"],
                                 key=f"up_{prefix}", label_visibility="collapsed")
            _handle_upload(f, prefix, label)
            df = _ss(f"{prefix}_df")
            if df is not None:
                st.markdown(
                    f'<div style="display:flex;align-items:center;gap:8px;padding:7px 0;'
                    f'border-bottom:1px solid #E2E8F0;">'
                    f'<span style="width:6px;height:6px;border-radius:50%;background:#16A34A;'
                    f'flex-shrink:0;display:inline-block;"></span>'
                    f'<span style="font-size:12px;font-weight:600;color:#111827;">'
                    f'{_ss(f"{prefix}_filename") or "—"}</span>'
                    f'<span style="font-size:11px;color:#9CA3AF;">'
                    f'{len(df):,} linhas · {len(df.columns)} colunas</span>'
                    f'</div>',
                    unsafe_allow_html=True,
                )

    _divider()

    # Previews colapsados
    for prefix, label in [("sci","Science"), ("por","Portal de Cadastros"), ("arq3","Arquivo 3")]:
        df = _ss(f"{prefix}_df")
        if df is not None:
            with st.expander(f"{label} — {len(df):,} linhas"):
                st.dataframe(df.head(15), use_container_width=True, hide_index=True)

    sci_ok  = _ss("sci_df")  is not None
    por_ok  = _ss("por_df")  is not None
    if sci_ok and por_ok:
        _alert("Science e Portal carregados. Prossiga para <strong>Gerar Tabela Final</strong>.", "ok")

# ══════════════════════════════════════════════════════════════════════════════
# PÁGINA 2 — GERAR TABELA FINAL
# ══════════════════════════════════════════════════════════════════════════════
elif p == "Gerar Tabela Final":
    _page_header("Gerar Tabela Final", "Pipeline · Consolidação · Deduplicação · Persistência")

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
            for k in ("sci_df","por_df","arq3_df","wizard_cfg"):
                st.session_state.pop(k, None)
            st.rerun()
    with col_run:
        run = st.button("Gerar Tabela Final", type="primary",
                        key="btn_merge", use_container_width=True)

    if run:
        with st.spinner("Executando pipeline..."):
            try:
                merged_df, report = build_merged_df(
                    science_df=sci_df.copy(),
                    portal_df=por_df.copy(),
                    ref_df=arq3_df.copy() if arq3_df is not None else None,
                    uf_map={}, join_keys_sci=[], join_keys_por=[], config=cfg,
                )

                # Mantém apenas colunas de negócio
                biz_cols = [c for c in BUSINESS_COLS if c in merged_df.columns]
                result_df = merged_df[biz_cols].copy()

                # Deduplicação e persistência
                with session_scope() as s:
                    repo = TabelaFinalRepository(s)
                    inserted, skipped = repo.upsert(result_df)
                    final_df = repo.load()

                st.session_state["tabela_final"]    = final_df
                st.session_state["analytics_cache"] = precompute_all(final_df)
                st.session_state["selected_col"]    = None

                # Mensagem clara para o usuário
                if inserted > 0 and skipped == 0:
                    _alert(f"<strong>{inserted:,} rotas novas adicionadas.</strong> "
                           f"Total no banco: {len(final_df):,} rotas.", "ok")
                elif inserted > 0:
                    _alert(f"<strong>{inserted:,} rotas novas adicionadas.</strong> "
                           f"{skipped:,} já existiam e foram ignoradas. "
                           f"Total no banco: {len(final_df):,} rotas.", "ok")
                else:
                    _alert(f"Nenhuma rota nova. {skipped:,} rotas já existiam no banco. "
                           f"Total: {len(final_df):,} rotas.", "info")

                log.info("Pipeline concluído: inserted=%d skipped=%d total=%d",
                         inserted, skipped, len(final_df))

            except Exception as e:
                _alert(f"Erro no pipeline: {e}", "error")
                log.error("Pipeline error: %s", e, exc_info=True)
                import traceback
                st.code(traceback.format_exc())
                st.stop()

    # Se já existe tabela, mostra resumo
    tf = _ss("tabela_final")
    if tf is not None and not tf.empty:
        _divider()
        _alert(f"Tabela Final disponível — <strong>{len(tf):,} rotas</strong>. "
               f"Acesse <strong>Tabela Final</strong> para visualizar.", "info")

# ══════════════════════════════════════════════════════════════════════════════
# PÁGINA 3 — TABELA FINAL
# ══════════════════════════════════════════════════════════════════════════════
elif p == "Tabela Final":
    _page_header("Tabela Final", "Visualização · Dashboard por coluna · Exportação")

    tf = _ss("tabela_final")

    if tf is None or tf.empty:
        _alert("Nenhuma rota disponível. Gere a Tabela Final primeiro.", "warn")
        st.stop()

    cache    = _ss("analytics_cache") or {}
    sel_col  = _ss("selected_col")

    # ── Barra de exportação (sempre visível) ──────────────────────────────
    _label("Exportar")
    ts_str = datetime.now().strftime("%Y-%m-%d_%H%M")
    eb1, eb2, eb3, eb4 = st.columns(4, gap="small")

    with eb1:
        csv_bytes = tf.to_csv(index=False).encode("utf-8-sig")
        st.download_button(
            "Exportar CSV",
            data=csv_bytes,
            file_name=f"tabela_final_{ts_str}.csv",
            mime="text/csv",
            key="btn_csv",
            use_container_width=True,
        )

    with eb2:
        try:
            import io as _io
            xlsx_buf = _io.BytesIO()
            with pd.ExcelWriter(xlsx_buf, engine="openpyxl") as writer:
                tf.to_excel(writer, index=False, sheet_name="Tabela Final")
            st.download_button(
                "Exportar Excel",
                data=xlsx_buf.getvalue(),
                file_name=f"tabela_final_{ts_str}.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                key="btn_xlsx",
                use_container_width=True,
            )
        except Exception as e:
            _alert(f"Erro ao gerar Excel: {e}", "error")

    with eb3:
        try:
            pptx_gen = build_general_pptx(tf)
            st.download_button(
                "Dashboard Geral (PPTX)",
                data=pptx_gen,
                file_name=f"dashboard_geral_{ts_str}.pptx",
                mime="application/vnd.openxmlformats-officedocument.presentationml.presentation",
                key="btn_pptx_geral",
                use_container_width=True,
            )
        except Exception as e:
            _alert(f"Erro ao gerar PPTX geral: {e}", "error")

    with eb4:
        if sel_col:
            try:
                pptx_col = build_column_pptx(tf, sel_col)
                st.download_button(
                    f"Dashboard {sel_col} (PPTX)",
                    data=pptx_col,
                    file_name=f"dashboard_{sel_col}_{ts_str}.pptx",
                    mime="application/vnd.openxmlformats-officedocument.presentationml.presentation",
                    key="btn_pptx_col",
                    use_container_width=True,
                )
            except Exception as e:
                _alert(f"Erro ao gerar PPTX da coluna: {e}", "error")
        else:
            st.markdown(
                '<div style="font-size:11px;color:#9CA3AF;padding-top:28px;text-align:center;">'
                'Selecione uma coluna para exportar o dashboard específico.</div>',
                unsafe_allow_html=True,
            )

    _divider()

    # ── Instrução de uso ───────────────────────────────────────────────────
    st.markdown(
        '<div style="background:#0F172A;padding:10px 16px;margin-bottom:12px;">'
        '<span style="font-size:12px;color:#94A3B8;">Selecione uma coluna para ver o dashboard instantâneo:</span>'
        '</div>',
        unsafe_allow_html=True,
    )

    # ── Seletor de coluna (via botões inline) ──────────────────────────────
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
    render_interactive_table(tf, selected_col=sel_col, height=420, component_key="main_table")

    # ── Dashboard instantâneo ──────────────────────────────────────────────
    if sel_col:
        _divider()
        render_column_dashboard(tf, sel_col, cache=cache)

# ─── Rodapé ───────────────────────────────────────────────────────────────────
st.markdown(
    '<div style="border-top:1px solid #E2E8F0;margin-top:40px;padding-top:10px;'
    'text-align:center;font-size:10px;color:#9CA3AF;letter-spacing:.05em;">'
    'VIVOHUB · Data Merger v2 · Processamento 100% local</div>',
    unsafe_allow_html=True,
)

