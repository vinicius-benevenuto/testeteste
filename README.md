"""
ui/pages.py
P�ginas principais do app Streamlit.
Cada função render_* desenha uma seção/aba completa.
"""
from __future__ import annotations
import json
from typing import Optional

import pandas as pd
import streamlit as st

from app.core.map_rules import OUTPUT_COLUMNS
from app.core.merge import build_merged_df, count_join_matches
from app.core.normalize import apply_column_normalization, infer_and_coerce_types, strip_whitespace
from app.core.validate import check_duplicates, generate_report, validate_output
from app.db.repository import Repository
from app.db.session import session_scope
from app.io.readers import list_sheets, read_file
from app.io.writers import logs_to_text, to_csv_bytes, to_mapping_json, to_xlsx_bytes
from app.ui.grid import show_grid
from app.ui.mapping_wizard import run_mapping_wizard
from app.utils.ids import file_hash, version_tag
from app.utils.logging_utils import get_logger

log = get_logger(__name__)

# ── Helpers de estado ─────────────────────────────────────────────────────

def _ss(key: str, default=None):
    return st.session_state.get(key, default)


def _init_state() -> None:
    defaults = {
        "sci_df": None, "sci_filename": "", "sci_import_id": None,
        "por_df": None, "por_filename": "", "por_import_id": None,
        "merged_df": None, "merge_report": None, "version_id": None,
        "current_mapping": None, "literal_defaults": {},
        "join_keys": [], "join_type": "outer",
        "fuzzy_threshold": 90, "use_fuzzy": False,
    }
    for k, v in defaults.items():
        if k not in st.session_state:
            st.session_state[k] = v


# ── Renderizadores ────────────────────────────────────────────────────────

def render_upload_page() -> None:
    """Passo 1: Upload de arquivos + preview."""
    st.header("📂 1. Carregar Arquivos")
    st.markdown("""
    Faça upload dos dois arquivos de planilha. Todos os dados são processados
    **localmente** — nenhuma informação é enviada para servidores externos.
    """)

    col1, col2 = st.columns(2)

    # Science
    with col1:
        st.subheader("Tabela Science")
        sci_file = st.file_uploader(
            "Selecione o arquivo Science",
            type=["xlsx", "xls", "csv", "parquet"],
            key="uploader_science",
            help="Suportado: .xlsx, .xls, .csv, .parquet",
        )
        if sci_file:
            _handle_upload(sci_file, "science")

    # Portal
    with col2:
        st.subheader("Tabela Portal de Cadastros")
        por_file = st.file_uploader(
            "Selecione o arquivo Portal",
            type=["xlsx", "xls", "csv", "parquet"],
            key="uploader_portal",
            help="Suportado: .xlsx, .xls, .csv, .parquet",
        )
        if por_file:
            _handle_upload(por_file, "portal")

    # Previews
    sci_df = _ss("sci_df")
    por_df = _ss("por_df")

    if sci_df is not None:
        with st.expander(f"👁️ Preview Science ({len(sci_df):,} linhas)", expanded=True):
            st.dataframe(sci_df.head(50), use_container_width=True, hide_index=True)
            st.caption(f"Colunas ({len(sci_df.columns)}): {', '.join(sci_df.columns.tolist())}")

    if por_df is not None:
        with st.expander(f"👁️ Preview Portal ({len(por_df):,} linhas)", expanded=True):
            st.dataframe(por_df.head(50), use_container_width=True, hide_index=True)
            st.caption(f"Colunas ({len(por_df.columns)}): {', '.join(por_df.columns.tolist())}")

    if sci_df is not None and por_df is not None:
        st.success("✅ Ambos os arquivos carregados! Prossiga para o Mapeamento.")


def _handle_upload(uploaded_file, source: str) -> None:
    """Processa o arquivo enviado e armazena no session_state."""
    ext = uploaded_file.name.split(".")[-1].lower()
    raw_bytes = uploaded_file.read()
    prefix = "sci" if source == "science" else "por"

    # Seleção de planilha para XLSX/XLS
    sheet: Optional[str] = None
    if ext in ("xlsx", "xls"):
        import io
        sheets = list_sheets(io.BytesIO(raw_bytes))
        if len(sheets) > 1:
            sheet = st.selectbox(
                f"Planilha ({source})",
                sheets,
                key=f"sheet_{source}",
            )
        elif sheets:
            sheet = sheets[0]

    try:
        import io
        df = read_file(io.BytesIO(raw_bytes), filename=uploaded_file.name, sheet=sheet)
        df, col_map = apply_column_normalization(df)
        df = strip_whitespace(df)
        df = infer_and_coerce_types(df)

        st.session_state[f"{prefix}_df"]       = df
        st.session_state[f"{prefix}_filename"] = uploaded_file.name
        st.session_state[f"{prefix}_raw"]      = raw_bytes
        st.session_state[f"{prefix}_col_map"]  = col_map
        st.session_state[f"{prefix}_hash"]     = file_hash(raw_bytes)
        st.session_state[f"{prefix}_sheet"]    = sheet

        st.success(f"✅ {uploaded_file.name}: {len(df):,} linhas, {len(df.columns)} colunas carregadas.")
    except Exception as e:
        st.error(f"❌ Erro ao ler {uploaded_file.name}: {e}")
        log.error("Erro upload: %s", e, exc_info=True)


def render_mapping_page() -> None:
    """Passo 2: Assistente de Mapeamento."""
    sci_df = _ss("sci_df")
    por_df = _ss("por_df")

    if sci_df is None or por_df is None:
        st.warning("⚠️ Carregue os dois arquivos primeiro (Passo 1).")
        return

    mapping, literal_defaults, join_keys, join_type, fuzzy_threshold, use_fuzzy = (
        run_mapping_wizard(
            science_cols=list(sci_df.columns),
            portal_cols=list(por_df.columns),
            current_mapping=_ss("current_mapping"),
        )
    )

    if join_keys:
        st.markdown("---")
        st.markdown("### 📊 Pré-visualização das chaves de junção")
        for key in join_keys:
            rule = mapping.get(key, {})
            sources = rule.get("sources", [])
            sci_src = next((s["col"] for s in sources if s["table"] == "science"), None)
            por_src = next((s["col"] for s in sources if s["table"] == "portal"), None)
            if sci_src and por_src and sci_src in sci_df.columns and por_src in por_df.columns:
                stats = count_join_matches(sci_df, por_df, sci_src, por_src)
                st.markdown(f"**{key}** (`{sci_src}` ↔ `{por_src}`):")
                c1, c2, c3 = st.columns(3)
                c1.metric("Matches", stats["matches"])
                c2.metric("Só Science", stats["sci_only"])
                c3.metric("Só Portal", stats["por_only"])

    st.markdown("---")
    if st.button("💾 Salvar Mapeamento na Sessão", type="primary", key="btn_save_mapping"):
        st.session_state["current_mapping"]  = mapping
        st.session_state["literal_defaults"] = literal_defaults
        st.session_state["join_keys"]        = join_keys
        st.session_state["join_type"]        = join_type
        st.session_state["fuzzy_threshold"]  = fuzzy_threshold
        st.session_state["use_fuzzy"]        = use_fuzzy
        st.success("✅ Mapeamento salvo! Prossiga para Gerar Tabela Final.")


def render_merge_page() -> None:
    """Passo 3: Gerar tabela final + salvar no SQLite."""
    sci_df = _ss("sci_df")
    por_df = _ss("por_df")
    mapping = _ss("current_mapping")

    if sci_df is None or por_df is None:
        st.warning("⚠️ Carregue os dois arquivos primeiro.")
        return
    if not mapping:
        st.warning("⚠️ Configure o mapeamento primeiro (Passo 2).")
        return

    st.header("⚙️ 3. Gerar Tabela Final")

    col1, col2 = st.columns(2)
    with col1:
        st.info(f"Science: **{len(sci_df):,}** linhas")
    with col2:
        st.info(f"Portal: **{len(por_df):,}** linhas")

    if st.button("🚀 Gerar Tabela Combinada", type="primary", key="btn_merge"):
        with st.spinner("Processando merge..."):
            try:
                merged_df, report = build_merged_df(
                    science_df=sci_df.copy(),
                    portal_df=por_df.copy(),
                    mapping=mapping,
                    join_keys=_ss("join_keys") or [],
                    join_type=_ss("join_type") or "outer",
                    fuzzy=_ss("use_fuzzy") or False,
                    fuzzy_threshold=_ss("fuzzy_threshold") or 90,
                    literal_defaults=_ss("literal_defaults") or {},
                )
                st.session_state["merged_df"]   = merged_df
                st.session_state["merge_report"] = report
                st.success(f"✅ Merge concluído: **{len(merged_df):,}** linhas geradas.")
            except Exception as e:
                st.error(f"❌ Erro no merge: {e}")
                log.error("Erro merge: %s", e, exc_info=True)
                return

    merged_df = _ss("merged_df")
    if merged_df is None:
        return

    # Validações
    warnings = validate_output(merged_df)
    for w in warnings:
        st.warning(w)

    # Report
    report = _ss("merge_report") or {}
    with st.expander("📋 Relatório do Merge"):
        c1, c2, c3 = st.columns(3)
        c1.metric("Linhas Science", report.get("rows_science", 0))
        c2.metric("Linhas Portal", report.get("rows_portal", 0))
        c3.metric("Linhas Resultado", report.get("rows_merged", 0))
        null_counts = report.get("null_counts", {})
        if null_counts:
            st.markdown("**Valores vazios por coluna:**")
            nc_df = pd.DataFrame(
                [{"Coluna": k, "Vazios": v,
                  "% Vazio": f"{v/max(report.get('rows_merged',1),1)*100:.1f}%"}
                 for k, v in null_counts.items()]
            )
            st.dataframe(nc_df, use_container_width=True, hide_index=True)

    # Grid filtrado
    filtered_df = show_grid(merged_df, key_prefix="merge_grid",
                            title="📊 Tabela Final Combinada")

    # Ações de persistência e exportação
    st.markdown("---")
    st.markdown("### 💾 Salvar e Exportar")
    action_cols = st.columns(4)

    with action_cols[0]:
        if st.button("🗄️ Salvar no SQLite", key="btn_save_db"):
            _save_to_db(merged_df)

    with action_cols[1]:
        csv_data = to_csv_bytes(filtered_df)
        st.download_button(
            "⬇️ Exportar CSV",
            data=csv_data,
            file_name=f"resultado_{version_tag()}.csv",
            mime="text/csv",
            key="btn_export_csv",
        )

    with action_cols[2]:
        try:
            xlsx_data = to_xlsx_bytes(filtered_df)
            st.download_button(
                "⬇️ Exportar XLSX",
                data=xlsx_data,
                file_name=f"resultado_{version_tag()}.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                key="btn_export_xlsx",
            )
        except Exception as e:
            st.error(f"XLSX indisponível: {e}")

    with action_cols[3]:
        mapping_json = to_mapping_json({
            "mapping": mapping,
            "join_keys": _ss("join_keys"),
            "join_type": _ss("join_type"),
        })
        st.download_button(
            "⬇️ Baixar Mapeamento JSON",
            data=mapping_json,
            file_name=f"mapping_{version_tag()}.json",
            mime="application/json",
            key="btn_export_mapping",
        )


def _save_to_db(merged_df: pd.DataFrame) -> None:
    """Salva importações + merge no SQLite."""
    try:
        with session_scope() as session:
            repo = Repository(session)
            sci_import_id = _ss("sci_import_id")
            por_import_id = _ss("por_import_id")

            # Salva Science se ainda não foi salvo
            if not sci_import_id and _ss("sci_df") is not None:
                sci_import_id = repo.save_import(
                    source="science",
                    filename=_ss("sci_filename") or "science.xlsx",
                    sheet=_ss("sci_sheet"),
                    file_hash=_ss("sci_hash") or "",
                    df=st.session_state["sci_df"],
                    column_map=_ss("sci_col_map") or {},
                )
                st.session_state["sci_import_id"] = sci_import_id

            # Salva Portal se ainda não foi salvo
            if not por_import_id and _ss("por_df") is not None:
                por_import_id = repo.save_import(
                    source="portal",
                    filename=_ss("por_filename") or "portal.xlsx",
                    sheet=_ss("por_sheet"),
                    file_hash=_ss("por_hash") or "",
                    df=st.session_state["por_df"],
                    column_map=_ss("por_col_map") or {},
                )
                st.session_state["por_import_id"] = por_import_id

            # Salva versão do merge
            version_id = repo.save_merge_version(
                tag=version_tag(),
                science_import_id=sci_import_id,
                portal_import_id=por_import_id,
                mapping=_ss("current_mapping") or {},
                join_keys=_ss("join_keys") or [],
                join_type=_ss("join_type") or "outer",
                fuzzy_threshold=_ss("fuzzy_threshold") or 90,
                merged_df=merged_df,
                rows_science=len(st.session_state.get("sci_df", [])) if st.session_state.get("sci_df") is not None else 0,
                rows_portal=len(st.session_state.get("por_df", [])) if st.session_state.get("por_df") is not None else 0,
            )
            st.session_state["version_id"] = version_id
            repo.add_log("INFO", f"Merge salvo: version_id={version_id}")

        st.success(f"✅ Dados salvos no SQLite! version_id: `{version_id[:8]}...`")
        log.info("DB save OK: version_id=%s", version_id)
    except Exception as e:
        st.error(f"❌ Erro ao salvar no banco: {e}")
        log.error("Erro DB save: %s", e, exc_info=True)


def render_history_page() -> None:
    """Passo 4: Histórico de versões salvas no banco."""
    st.header("🕐 Histórico de Versões")

    try:
        with session_scope() as session:
            repo = Repository(session)
            versions = repo.list_versions()
            imports  = repo.list_imports()

        if not versions:
            st.info("Nenhum merge salvo ainda.")
        else:
            vers_df = pd.DataFrame(versions)
            st.subheader("Versões de Merge")
            st.dataframe(vers_df, use_container_width=True, hide_index=True)

            sel_id = st.selectbox(
                "Carregar versão para visualizar:",
                [v["id"] for v in versions],
                format_func=lambda x: next(
                    (f"{v['tag']} — {v['rows_merged']} linhas ({x[:8]}...)"
                     for v in versions if v["id"] == x), x),
                key="hist_version_sel",
            )
            if st.button("📂 Carregar Versão", key="btn_load_version"):
                with session_scope() as s:
                    r = Repository(s)
                    loaded = r.load_merged_df(sel_id)
                st.session_state["merged_df"] = loaded
                st.success(f"✅ {len(loaded):,} linhas carregadas da versão `{sel_id[:8]}...`")
                st.dataframe(loaded.head(50), use_container_width=True, hide_index=True)

        if imports:
            with st.expander("📁 Importações Salvas"):
                imp_df = pd.DataFrame(imports)
                st.dataframe(imp_df, use_container_width=True, hide_index=True)

    except Exception as e:
        st.error(f"Erro ao acessar histórico: {e}")


def render_logs_page() -> None:
    """Exibe logs do banco e permite download."""
    st.header("📋 Logs e Auditoria")
    try:
        with session_scope() as session:
            repo = Repository(session)
            logs = repo.get_logs(limit=200)

        if not logs:
            st.info("Nenhum log registrado ainda.")
            return

        log_df = pd.DataFrame(logs)[["timestamp", "level", "message"]]
        level_filter = st.multiselect(
            "Filtrar por nível:", ["INFO", "WARNING", "ERROR", "DEBUG"],
            default=["INFO", "WARNING", "ERROR"], key="log_level_filter",
        )
        filtered = log_df[log_df["level"].isin(level_filter)]
        st.dataframe(filtered, use_container_width=True, hide_index=True)

        log_bytes = logs_to_text(logs)
        st.download_button(
            "⬇️ Baixar Log Completo",
            data=log_bytes,
            file_name=f"logs_{version_tag()}.txt",
            mime="text/plain",
            key="btn_download_logs",
        )
    except Exception as e:
        st.error(f"Erro ao carregar logs: {e}")


def render_validation_page() -> None:
    """Relatório de qualidade dos dados carregados."""
    st.header("✅ Relatório de Qualidade")
    sci_df = _ss("sci_df")
    por_df = _ss("por_df")
    merged_df = _ss("merged_df")

    for df, name in [(sci_df, "Science"), (por_df, "Portal"), (merged_df, "Resultado")]:
        if df is not None:
            with st.expander(f"📊 {name} ({len(df):,} linhas)", expanded=(name == "Resultado")):
                report = generate_report(df, name)
                rep_df = pd.DataFrame(report["colunas"])
                st.dataframe(rep_df, use_container_width=True, hide_index=True)

                if merged_df is not None and name == "Resultado":
                    join_keys = _ss("join_keys") or []
                    if join_keys:
                        dup_report = check_duplicates(df, join_keys)
                        c1, c2 = st.columns(2)
                        c1.metric("Duplicatas (pelas chaves)", dup_report["duplicatas"])
                        c2.metric("Únicos", dup_report["unicos"])
