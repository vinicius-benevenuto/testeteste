"""
ui/pages.py — Páginas principais do app Streamlit.
"""
from __future__ import annotations
import json
from typing import Optional

import pandas as pd
import streamlit as st

from app.core.merge import build_merged_df, count_join_matches
from app.core.normalize import (apply_column_normalization,
                                 infer_and_coerce_types, strip_whitespace)
from app.core.validate import generate_report, quality_summary
from app.db.repository import Repository
from app.db.session import session_scope
from app.io.readers import list_sheets, read_file
from app.io.writers import logs_to_text, to_csv_bytes, to_mapping_json, to_xlsx_bytes
from app.ui.grid import show_grid
from app.ui.mapping_wizard import run_mapping_wizard
from app.utils.ids import file_hash, version_tag
from app.utils.logging_utils import get_logger

log = get_logger(__name__)


# ── helpers de estado ──────────────────────────────────────────────────────

def _ss(k, default=None):
    return st.session_state.get(k, default)


def _init_state() -> None:
    defaults = {
        "sci_df": None, "sci_filename": "", "sci_import_id": None,
        "sci_col_map": {}, "sci_hash": "", "sci_sheet": None,
        "por_df": None, "por_filename": "", "por_import_id": None,
        "por_col_map": {}, "por_hash": "", "por_sheet": None,
        "arq3_df": None, "arq3_filename": "", "arq3_import_id": None,
        "arq3_col_map": {}, "arq3_hash": "", "arq3_sheet": None,
        "merged_df": None, "merge_report": None, "version_id": None,
        "wizard_cfg": {}, "uf_map": {},
        "seeds_loaded": False,
    }
    for k, v in defaults.items():
        if k not in st.session_state:
            st.session_state[k] = v


# ── Upload helper ──────────────────────────────────────────────────────────

def _handle_upload(uploaded_file, prefix: str, source_label: str) -> None:
    ext = uploaded_file.name.split(".")[-1].lower()
    raw = uploaded_file.read()
    import io
    sheet: Optional[str] = None
    if ext in ("xlsx", "xls"):
        sheets = list_sheets(io.BytesIO(raw))
        if len(sheets) > 1:
            sheet = st.selectbox(f"Planilha — {source_label}",
                                  sheets, key=f"sheet_{prefix}")
        elif sheets:
            sheet = sheets[0]
    try:
        df = read_file(io.BytesIO(raw), filename=uploaded_file.name, sheet=sheet)
        df, col_map = apply_column_normalization(df)
        df = strip_whitespace(df)
        df = infer_and_coerce_types(df)
        st.session_state[f"{prefix}_df"]        = df
        st.session_state[f"{prefix}_filename"]  = uploaded_file.name
        st.session_state[f"{prefix}_raw"]       = raw
        st.session_state[f"{prefix}_col_map"]   = col_map
        st.session_state[f"{prefix}_hash"]      = file_hash(raw)
        st.session_state[f"{prefix}_sheet"]     = sheet
        st.success(f"✅ {uploaded_file.name}: {len(df):,} linhas · {len(df.columns)} colunas")
    except Exception as e:
        st.error(f"❌ Erro ao ler {uploaded_file.name}: {e}")
        log.error("Upload error: %s", e, exc_info=True)


# ── Página 1: Upload ────────────────────────────────────────────────────────

def render_upload_page() -> None:
    st.header("📂 1. Carregar Arquivos")
    st.markdown("""
    Faça upload dos três arquivos. Dados processados **100% localmente**.
    """)

    c1, c2, c3 = st.columns(3)

    with c1:
        st.subheader("Planilha Science")
        f = st.file_uploader("Science (xlsx/csv/parquet)",
                             type=["xlsx","xls","csv","parquet"],
                             key="up_sci")
        if f: _handle_upload(f, "sci", "Science")

    with c2:
        st.subheader("Portal de Cadastros")
        f = st.file_uploader("Portal (xlsx/csv/parquet)",
                             type=["xlsx","xls","csv","parquet"],
                             key="up_por")
        if f: _handle_upload(f, "por", "Portal")

    with c3:
        st.subheader("Arquivo 3 (Referência)")
        f = st.file_uploader("Arquivo 3 (xlsx/csv/parquet)",
                             type=["xlsx","xls","csv","parquet"],
                             key="up_arq3")
        if f: _handle_upload(f, "arq3", "Arquivo 3")

    # Previews
    for prefix, label in [("sci","Science"), ("por","Portal"), ("arq3","Arquivo 3")]:
        df = _ss(f"{prefix}_df")
        if df is not None:
            with st.expander(f"👁️ Preview {label} ({len(df):,} linhas)", expanded=False):
                st.dataframe(df.head(20), width="stretch", hide_index=True)
                st.caption(f"Colunas: {', '.join(df.columns.tolist())}")

    sci_ok  = _ss("sci_df")  is not None
    por_ok  = _ss("por_df")  is not None
    arq3_ok = _ss("arq3_df") is not None

    if sci_ok and por_ok:
        msg = "✅ Science e Portal carregados."
        msg += " ✅ Arquivo 3 carregado." if arq3_ok else " ℹ️ Arquivo 3 não carregado (opcional)."
        st.success(msg)

    # Seeds status
    with session_scope() as s:
        repo = Repository(s)
        cnl_n  = repo.cnl_count()
        uf_n   = repo.cn_to_uf_count()
    st.info(f"🗄️ Seeds: {cnl_n:,} registros CNL · {uf_n} mapeamentos CN→UF")
    if cnl_n == 0:
        st.warning("⚠️ Tabela CNL vazia. Execute: **Ferramentas → Carregar Seeds** para habilitar derivação de UF.")


# ── Página 2: Mapeamento ───────────────────────────────────────────────────

def render_mapping_page() -> None:
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")

    if sci_df is None or por_df is None:
        st.warning("⚠️ Carregue Science e Portal primeiro (Passo 1).")
        return

    cfg = run_mapping_wizard(
        science_cols=list(sci_df.columns),
        portal_cols=list(por_df.columns),
        arq3_cols=list(arq3_df.columns) if arq3_df is not None else [],
    )

    # Preview de matches
    join_sci = cfg.get("join_keys_sci", [])
    join_por = cfg.get("join_keys_por", [])
    if join_sci and join_por and len(join_sci) == len(join_por):
        st.markdown("---")
        st.markdown("### 📊 Preview de matches por chave de junção")
        for ks, kp in zip(join_sci, join_por):
            if ks in sci_df.columns and kp in por_df.columns:
                stats = count_join_matches(sci_df, por_df, ks, kp)
                st.markdown(f"**`{ks}`** (Science) ↔ **`{kp}`** (Portal):")
                mc1, mc2, mc3 = st.columns(3)
                mc1.metric("Matches", stats.get("matches", 0))
                mc2.metric("Só Science", stats.get("sci_only", 0))
                mc3.metric("Só Portal", stats.get("por_only", 0))

    st.markdown("---")
    if st.button("💾 Salvar Configuração", type="primary", key="btn_save_cfg"):
        st.session_state["wizard_cfg"] = cfg
        st.success("✅ Configuração salva! Prossiga para Gerar Tabela Final.")


# ── Página 3: Merge ────────────────────────────────────────────────────────

def render_merge_page() -> None:
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")
    cfg     = _ss("wizard_cfg") or {}

    if sci_df is None or por_df is None:
        st.warning("⚠️ Carregue os arquivos primeiro (Passo 1).")
        return

    st.header("⚙️ 3. Gerar Tabela Final")

    c1, c2, c3 = st.columns(3)
    c1.metric("Linhas Science", f"{len(sci_df):,}")
    c2.metric("Linhas Portal",  f"{len(por_df):,}")
    c3.metric("Arquivo 3",      f"{len(arq3_df):,}" if arq3_df is not None else "—")

    cfg_saved = _ss("wizard_cfg") or {}
    if not cfg_saved:
        st.warning("⚠️ Configuração não salva. Vá para **2. Mapeamento** e clique **Salvar**.")
    else:
        id_rota = cfg_saved.get("id_rota_sci_col", "(não definida)")
        st.info(
            f"📋 Config ativa — "
            f"Central Science: **{cfg_saved.get('central_sci_col','auto')}** | "
            f"ID Rota: **{id_rota}** | "
            f"Central Portal: **{cfg_saved.get('central_portal_col','CENTRAL')}** | "
            f"CLUSTER Arq3: **{cfg_saved.get('arq3_cluster_col','auto')}**"
        )
        if id_rota in ("(não definida)", "(nenhuma)", ""):
            st.warning("⚠️ **ID Rota não configurada** — Etapa 3 do Science será ignorada. "
                       "Configure em **2. Mapeamento → 3b.**")

    col_r1, col_r2 = st.columns([3, 1])
    with col_r2:
        if st.button("🗑️ Limpar cache", key="btn_clear_cache"):
            st.session_state.pop("merged_df", None)
            st.session_state.pop("merge_report", None)
            st.session_state.pop("wizard_cfg", None)
            st.rerun()

    if st.button("🚀 Executar Pipeline Completo", type="primary", key="btn_merge"):
        with st.spinner("Executando pipeline (9 etapas)..."):
            try:
                merged, report = build_merged_df(
                    science_df=sci_df.copy(),
                    portal_df=por_df.copy(),
                    ref_df=arq3_df.copy() if arq3_df is not None else None,
                    uf_map={},
                    join_keys_sci=[],
                    join_keys_por=[],
                    config=cfg,
                )
                st.session_state["merged_df"]    = merged
                st.session_state["merge_report"] = report
                st.success(f"✅ {len(merged):,} linhas geradas.")
            except Exception as e:
                st.error(f"❌ Erro: {e}")
                log.error("Pipeline error: %s", e, exc_info=True)
                import traceback
                st.code(traceback.format_exc())
                return

    merged_df = _ss("merged_df")
    if merged_df is None:
        with session_scope() as s:
            repo = Repository(s)
            merged_df = repo.load_merged_df()
        if merged_df is not None and not merged_df.empty:
            st.info("ℹ️ Exibindo última versão salva no banco.")
            st.session_state["merged_df"] = merged_df

    if merged_df is None or merged_df.empty:
        return

    report = _ss("merge_report") or {}

    # ── Relatório do pipeline ──────────────────────────────────────────
    with st.expander("📋 Relatório Completo do Pipeline", expanded=True):
        st.markdown("#### Science — Etapas 1 a 4")
        s1, s2, s3, s4, s5 = st.columns(5)
        s1.metric("Total inicial",         report.get("total_inicial_science", 0))
        s2.metric("Após filtro Central",   report.get("total_apos_filtro_central", 0),
                  delta=report.get("total_apos_filtro_central", 0)
                       - report.get("total_inicial_science", 0))
        s3.metric("Após filtro data",      report.get("total_apos_filtro_data", 0),
                  delta=report.get("total_apos_filtro_data", 0)
                       - report.get("total_apos_filtro_central", 0))
        s4.metric("Após validação rotas",  report.get("total_apos_validacao_rotas", 0),
                  delta=report.get("total_apos_validacao_rotas", 0)
                       - report.get("total_apos_filtro_data", 0))
        s5.metric("Enriquecido Arq3",      report.get("total_enriquecido_arq3", 0))

        st.markdown("#### Portal — Etapas 5 a 9")
        p1, p2, p3 = st.columns(3)
        p1.metric("Total processado",     report.get("total_portal_processado", 0))
        p2.metric("Encontrado no Arq3",   report.get("total_encontrado_arq3_portal", 0))
        p3.metric("Novas rotas criadas",  report.get("total_novas_rotas_criadas", 0),
                  delta=report.get("total_novas_rotas_criadas", 0),
                  delta_color="inverse" if report.get("total_novas_rotas_criadas", 0) > 0
                               else "off")

        st.markdown("#### Resultado Final")
        g1, g2, g3, g4, g5 = st.columns(5)
        g1.metric("Total linhas",           report.get("total_resultado_final",
                                                       len(merged_df)))
        g2.metric("UF em branco",           report.get("uf_missing", 0),
                  delta_color="inverse")
        g3.metric("CLUSTER em branco",      report.get("cluster_missing", 0),
                  delta_color="inverse")
        g4.metric("Inconsistências",        report.get("total_inconsistencias", 0),
                  delta_color="inverse")
        g5.metric("Redução Science",        f"{report.get('reducao_science_pct', 0)}%",
                  delta_color="inverse")

        # Inconsistências detalhadas
        all_incons = report.get("_inconsistencias", [])
        if all_incons:
            st.markdown(f"#### ⚠️ Inconsistências Detalhadas ({len(all_incons)})")
            df_incons = pd.DataFrame(all_incons)
            # Agrupa por etapa
            for etapa in df_incons["etapa"].unique():
                grupo = df_incons[df_incons["etapa"] == etapa]
                with st.expander(f"Etapa {etapa} — {len(grupo)} ocorrências"):
                    st.dataframe(grupo[["chave", "motivo"]],
                                 use_container_width=True, hide_index=True)

    # Grid
    display_cols = [c for c in merged_df.columns if not c.startswith("_")]
    filtered_df = show_grid(
        merged_df[display_cols],
        key="merge_grid",
        title="📊 Resultado Final",
        height=540,
    )

    # Exportações
    st.markdown("---")
    st.markdown("### 💾 Salvar e Exportar")
    ex1, ex2, ex3, ex4 = st.columns(4)

    with ex1:
        if st.button("🗄️ Salvar no SQLite", key="btn_save"):
            _save_to_db(merged_df)

    with ex2:
        st.download_button("⬇️ CSV (filtrado)",
                           to_csv_bytes(filtered_df),
                           f"resultado_{version_tag()}.csv",
                           "text/csv", key="btn_csv")
    with ex3:
        try:
            st.download_button("⬇️ XLSX (filtrado)",
                               to_xlsx_bytes(filtered_df),
                               f"resultado_{version_tag()}.xlsx",
                               "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                               key="btn_xlsx")
        except Exception as e:
            st.error(f"XLSX: {e}")

    with ex4:
        st.download_button("⬇️ Relatório JSON",
                           to_mapping_json({k: v for k, v in report.items()
                                            if k != "_inconsistencias"}),
                           f"relatorio_{version_tag()}.json",
                           "application/json", key="btn_rep_json")

    # Exportar inconsistências
    incons_list = report.get("_inconsistencias", [])
    if incons_list:
        st.download_button(
            "⬇️ Inconsistências CSV",
            to_csv_bytes(pd.DataFrame(incons_list)),
            f"inconsistencias_{version_tag()}.csv",
            "text/csv",
            key="btn_incons_csv",
        )


def _save_to_db(merged_df: pd.DataFrame) -> None:
    try:
        with session_scope() as s:
            repo = Repository(s)
            cfg  = _ss("wizard_cfg") or {}

            sci_id = _ss("sci_import_id")
            if not sci_id and _ss("sci_df") is not None:
                sci_id = repo.save_import(
                    "SCIENCE", _ss("sci_filename") or "science.xlsx",
                    _ss("sci_sheet"), _ss("sci_hash") or "",
                    st.session_state["sci_df"], _ss("sci_col_map") or {})
                st.session_state["sci_import_id"] = sci_id

            por_id = _ss("por_import_id")
            if not por_id and _ss("por_df") is not None:
                por_id = repo.save_import(
                    "PORTAL", _ss("por_filename") or "portal.xlsx",
                    _ss("por_sheet"), _ss("por_hash") or "",
                    st.session_state["por_df"], _ss("por_col_map") or {})
                st.session_state["por_import_id"] = por_id

            arq3_id = _ss("arq3_import_id")
            if not arq3_id and _ss("arq3_df") is not None:
                arq3_id = repo.save_import(
                    "ARQ3", _ss("arq3_filename") or "arquivo3.xlsx",
                    _ss("arq3_sheet"), _ss("arq3_hash") or "",
                    st.session_state["arq3_df"], _ss("arq3_col_map") or {})
                repo.save_arq3(arq3_id, st.session_state["arq3_df"],
                               _ss("arq3_col_map") or {})
                st.session_state["arq3_import_id"] = arq3_id

            vid = repo.save_merge_version(
                version_tag(), sci_id, por_id, arq3_id,
                mapping=cfg, join_keys=cfg.get("join_keys_sci", []),
                join_type=cfg.get("join_type", "outer"),
                fuzzy_threshold=90,
                merged_df=merged_df,
                rows_sci=len(st.session_state.get("sci_df") or []),
                rows_por=len(st.session_state.get("por_df") or []),
            )
            st.session_state["version_id"] = vid
            repo.add_log("INFO", f"Merge salvo version_id={vid}")

        st.success(f"✅ Salvo! version_id: `{vid[:8]}...`")
    except Exception as e:
        st.error(f"❌ Erro ao salvar: {e}")
        log.error("DB save: %s", e, exc_info=True)


# ── Página 4: Seeds ─────────────────────────────────────────────────────────

def render_seeds_page() -> None:
    st.header("🌱 Ferramentas — Seeds e Referências")

    with session_scope() as s:
        repo = Repository(s)
        cnl_n = repo.cnl_count()
        uf_n  = repo.cn_to_uf_count()

    st.info(f"Estado atual: **{cnl_n:,}** registros CNL · **{uf_n}** mapeamentos CN→UF")

    import os
    from pathlib import Path

    st.markdown("### 📋 Carregar Seeds automáticos")
    seed_dir = Path("seeds")
    cnl_sql  = seed_dir / "cnl.sql"
    uf_csv   = seed_dir / "cn_to_uf.csv"

    c1, c2 = st.columns(2)
    with c1:
        if cnl_sql.exists():
            if st.button("▶️ Carregar seeds/cnl.sql", key="btn_cnl_seed"):
                with session_scope() as s:
                    repo = Repository(s)
                    n = repo.load_cnl_seeds(str(cnl_sql))
                st.success(f"✅ {n} statements CNL executados.")
        else:
            st.warning("seeds/cnl.sql não encontrado.")

    with c2:
        if uf_csv.exists():
            if st.button("▶️ Carregar seeds/cn_to_uf.csv", key="btn_uf_seed"):
                with session_scope() as s:
                    repo = Repository(s)
                    n = repo.load_cn_to_uf_csv(str(uf_csv))
                st.success(f"✅ {n} mapeamentos CN→UF carregados.")
        else:
            st.warning("seeds/cn_to_uf.csv não encontrado.")

    st.markdown("### 📤 Upload manual de seeds")
    uf_file = st.file_uploader("Upload cn_to_uf.csv personalizado",
                                type=["csv"], key="up_uf_csv")
    if uf_file:
        import io, tempfile
        tmp = tempfile.NamedTemporaryFile(suffix=".csv", delete=False)
        tmp.write(uf_file.read())
        tmp.flush()
        try:
            with session_scope() as s:
                repo = Repository(s)
                n = repo.load_cn_to_uf_csv(tmp.name)
            st.success(f"✅ {n} mapeamentos carregados.")
        finally:
            os.unlink(tmp.name)

    st.markdown("### 🔍 Testar lookup de UF")
    test_cnl = st.text_input("Digite um COD_CNL para testar:", key="test_cnl")
    if test_cnl:
        with session_scope() as s:
            repo = Repository(s)
            uf = repo.get_uf_for_cnl(test_cnl)
        if uf:
            st.success(f"CNL `{test_cnl}` → UF: **{uf}**")
        else:
            st.warning(f"CNL `{test_cnl}` não encontrado nas tabelas de referência.")


# ── Página 5: Histórico ─────────────────────────────────────────────────────

def render_history_page() -> None:
    st.header("🕐 Histórico de Versões")
    with session_scope() as s:
        repo = Repository(s)
        versions = repo.list_versions()
        imports  = repo.list_imports()

    if not versions:
        st.info("Nenhum merge salvo ainda.")
    else:
        df_v = pd.DataFrame(versions)
        st.dataframe(df_v, width="stretch", hide_index=True)

        sel = st.selectbox("Carregar versão:",
                            [v["id"] for v in versions],
                            format_func=lambda x: next(
                                (f"{v['tag']} — {v['rows_merged']} linhas"
                                 for v in versions if v["id"] == x), x),
                            key="hist_sel")
        if st.button("📂 Carregar", key="btn_hist_load"):
            with session_scope() as s2:
                repo2 = Repository(s2)
                loaded = repo2.load_merged_df(sel)
            st.session_state["merged_df"] = loaded
            st.success(f"✅ {len(loaded):,} linhas carregadas.")
            show_grid(loaded[[c for c in loaded.columns if not c.startswith("_")]],
                      key="hist_grid", title="Versão carregada")

    if imports:
        with st.expander("📁 Importações"):
            st.dataframe(pd.DataFrame(imports), width="stretch", hide_index=True)


# ── Página 6: Validação ────────────────────────────────────────────────────

def render_validation_page() -> None:
    st.header("✅ Relatório de Qualidade")
    merged = _ss("merged_df")
    if merged is None:
        with session_scope() as s:
            repo = Repository(s)
            merged = repo.load_merged_df()
    if merged is None or merged.empty:
        st.info("Nenhum resultado gerado ainda.")
        return

    q = quality_summary(merged)
    cols = st.columns(4)
    cols[0].metric("Total linhas",   q.get("total", 0))
    cols[1].metric("UF em branco",   q.get("UF_missing", 0))
    cols[2].metric("Cluster em branco", q.get("CLUSTER_missing", 0))
    cols[3].metric("Sem match Arq3", q.get("arq3_no_match", 0))

    # Amostras de pendentes
    uf_pending = merged[merged["UF"].eq("") | merged["UF"].isna()]
    if not uf_pending.empty:
        with st.expander(f"⚠️ Linhas sem UF ({len(uf_pending)})"):
            display = [c for c in uf_pending.columns if not c.startswith("_")]
            st.dataframe(uf_pending[display].head(50),
                         width="stretch", hide_index=True)

    for df, name in [(_ss("sci_df"), "Science"), (_ss("por_df"), "Portal"),
                     (merged, "Resultado")]:
        if df is not None:
            with st.expander(f"📊 {name}"):
                rep = generate_report(df, name)
                st.dataframe(pd.DataFrame(rep["colunas"]),
                             width="stretch", hide_index=True)



# ── Página de Diagnóstico ────────────────────────────────────────────────

def render_diagnostico_page() -> None:
    st.header("🔬 Diagnóstico")

    arq3_df = _ss("arq3_df")
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    cfg     = _ss("wizard_cfg") or {}

    from app.core.pipeline import (
        _find_col, _extract_arq3_cols, _build_arq3_rotulo_index,
        _build_arq3_central_index, CENTRAL_PORTAL_MAP, CN_TO_UF,
    )

    if arq3_df is None:
        st.warning("Carregue o Arquivo 3 para ver o diagnóstico.")
        return

    # ── 1. Colunas detectadas no Arquivo 3 ──────────────────────────────
    st.subheader("1️⃣ Colunas detectadas no Arquivo 3")
    arq3_cols = _extract_arq3_cols(arq3_df, cfg)
    rot_col  = arq3_cols.get("rotulos", "")
    rotulo_index = _build_arq3_rotulo_index(arq3_df, rot_col)

    c1, c2, c3 = st.columns(3)
    c1.metric("Rótulos únicos",    len(rotulo_index))
    c2.metric("Linhas no Arq3",    len(arq3_df))
    c3.metric("Colunas detectadas",sum(1 for v in arq3_cols.values() if v))

    cols = list(arq3_df.columns)
    NONE = "(nenhuma)"

    field_map = {
        "REDE":             arq3_cols.get("rede", ""),
        "Central":          arq3_cols.get("central", ""),
        "UF":               arq3_cols.get("uf", ""),
        "CLUSTER ⭐":       arq3_cols.get("cluster", ""),
        "Tipo de Rota":     arq3_cols.get("tipo", ""),
        "Rótulos de Linha": arq3_cols.get("rotulos", ""),
        "OPERADORA":        arq3_cols.get("operadora", ""),
        "Denominação":      arq3_cols.get("den", ""),
    }

    for field, col in field_map.items():
        if col and col in arq3_df.columns:
            samples = (arq3_df[col].dropna().astype(str).str.strip()
                      .pipe(lambda s: s[s.str.len() > 0]).unique()[:5])
            st.markdown(f"✅ **{field}** → `{col}` — ex: *{' | '.join(samples)}*")
        else:
            st.error(f"❌ **{field}** → não detectada — configure em **2. Mapeamento → 8️⃣**")

    if not arq3_cols.get("cluster"):
        st.error("🚨 CLUSTER não detectado. Configure em **2. Mapeamento → 8️⃣**")

    with st.expander("📋 Todas as colunas do Arquivo 3"):
        for i, c in enumerate(cols):
            sample = arq3_df[c].dropna().iloc[0] if not arq3_df[c].dropna().empty else ""
            st.text(f"{i+1:3d}. {c!r:40s} ex: {str(sample)[:60]}")

    # ── 2. Diagnóstico Science (etapa 3 — amostra 20) ────────────────────
    if sci_df is not None:
        st.markdown("---")
        st.subheader("2️⃣ Diagnóstico Science — Etapa 3 (amostra 20 linhas)")

        col_cen    = _find_col(list(sci_df.columns),
                               cfg.get("central_sci_col", ""),
                               "Central", "CENTRAL",
                               "Central Interna", "Central Origem")
        col_id_rot = _find_col(list(sci_df.columns),
                               cfg.get("id_rota_sci_col", ""),
                               "ID Rota", "ID_ROTA", "IDROTA", "Id Rota")

        st.info(f"Coluna Central: **{col_cen or '❌ não encontrada'}** | "
                f"Coluna ID Rota: **{col_id_rot or '❌ não encontrada'}**")

        sample_sci = sci_df.head(20)
        matches, no_matches = [], []

        for _, row in sample_sci.iterrows():
            d = {c: str(row.get(c, "")).strip() for c in sci_df.columns}
            central_val = d.get(col_cen, "").upper()  if col_cen    else ""
            id_rota_val = d.get(col_id_rot, "").upper() if col_id_rot else ""
            chave = (central_val[:7] + "_" + id_rota_val) if central_val and id_rota_val else ""
            found = rotulo_index.get(chave)
            if found:
                matches.append({"Central": central_val, "ID Rota": id_rota_val,
                                 "Chave": chave,
                                 "CLUSTER": found.get(arq3_cols.get("cluster",""), ""),
                                 "UF": found.get(arq3_cols.get("uf",""), "")})
            else:
                no_matches.append({"Central": central_val, "ID Rota": id_rota_val,
                                   "Chave tentada": chave or "(sem chave)"})

        mc1, mc2 = st.columns(2)
        mc1.metric("✅ Com match", len(matches))
        mc2.metric("❌ Sem match", len(no_matches))

        if matches:
            with st.expander(f"✅ Matches ({len(matches)})"):
                st.dataframe(pd.DataFrame(matches), use_container_width=True, hide_index=True)
        if no_matches:
            with st.expander(f"❌ Sem match ({len(no_matches)})"):
                st.dataframe(pd.DataFrame(no_matches), use_container_width=True, hide_index=True)
                if rot_col:
                    sample_rotulos = sorted(list(rotulo_index.keys()))[:20]
                    st.markdown("**Exemplos de rótulos no Arq3:**")
                    st.write(sample_rotulos)

    # ── 3. Diagnóstico Portal (etapas 5-8 — amostra 20) ──────────────────
    if por_df is not None:
        st.markdown("---")
        st.subheader("3️⃣ Diagnóstico Portal — Etapas 5-8 (amostra 20 linhas)")

        col_cen_p = _find_col(list(por_df.columns),
                              cfg.get("central_portal_col", ""), "CENTRAL")
        col_le    = _find_col(list(por_df.columns),
                              cfg.get("label_e_col", ""), "LABEL_E")
        col_ls    = _find_col(list(por_df.columns),
                              cfg.get("label_s_col", ""), "LABEL_S")

        st.info(f"CENTRAL: **{col_cen_p or '❌'}** | LABEL_E: **{col_le or '❌'}** | "
                f"LABEL_S: **{col_ls or '❌'}**")

        centrais_portal = por_df[col_cen_p].dropna().unique() if col_cen_p else []
        centrais_mapa = set(CENTRAL_PORTAL_MAP.keys())
        centrais_sem_mapa = [c for c in centrais_portal
                             if str(c).strip().upper() not in centrais_mapa]
        if centrais_sem_mapa:
            st.warning(f"⚠️ {len(centrais_sem_mapa)} valores de CENTRAL sem mapeamento: "
                       f"{centrais_sem_mapa[:5]}")

        sample_por = por_df.head(20)
        matches_p, no_matches_p = [], []

        for _, row in sample_por.iterrows():
            d = {c: str(row.get(c, "")).strip() for c in por_df.columns}
            cen_raw  = d.get(col_cen_p, "").upper() if col_cen_p else ""
            cen_conv = CENTRAL_PORTAL_MAP.get(cen_raw, cen_raw)
            le_val   = d.get(col_le, "") if col_le else ""
            ls_val   = d.get(col_ls, "") if col_ls else ""
            label    = le_val if len(le_val) > 2 else ls_val if len(ls_val) > 2 else le_val
            rotulo   = (cen_conv[:7] + "_" + label[:6]).upper() if cen_conv and label else ""
            found    = rotulo_index.get(rotulo)
            if found:
                matches_p.append({"CENTRAL": cen_raw, "→": cen_conv,
                                  "LABEL": label, "Rótulo": rotulo,
                                  "CLUSTER": found.get(arq3_cols.get("cluster",""), ""),
                                  "UF": found.get(arq3_cols.get("uf",""), "")})
            else:
                matches_p.append({"CENTRAL": cen_raw, "→": cen_conv,
                                  "LABEL": label, "Rótulo": rotulo,
                                  "CLUSTER": "⚠️ NOVA ROTA", "UF": "?"})
                no_matches_p.append({"CENTRAL": cen_raw, "Rótulo tentado": rotulo})

        pc1, pc2 = st.columns(2)
        pc1.metric("✅ No Arq3", len(matches_p) - len(no_matches_p))
        pc2.metric("⚠️ Nova rota", len(no_matches_p))

        with st.expander("Ver todas (amostra 20)"):
            st.dataframe(pd.DataFrame(matches_p), use_container_width=True, hide_index=True)

    # ── 4. Mapa CENTRAL Portal ────────────────────────────────────────────
    st.markdown("---")
    st.subheader("4️⃣ Mapa de Conversão CENTRAL Portal")
    df_mapa = pd.DataFrame(
        [{"CENTRAL (Portal)": k, "CENTRAL (Convertida)": v}
         for k, v in CENTRAL_PORTAL_MAP.items()]
    )
    st.dataframe(df_mapa, use_container_width=True, hide_index=True)

    # ── 5. CN → UF ───────────────────────────────────────────────────────
    with st.expander("5️⃣ Tabela CN → UF (67 DDDs)"):
        df_cn = pd.DataFrame(
            [{"CN": k, "UF": v} for k, v in sorted(CN_TO_UF.items(), key=lambda x: int(x[0]))]
        )
        st.dataframe(df_cn, use_container_width=True, hide_index=True)

def render_logs_page() -> None:
    st.header("📋 Logs e Auditoria")
    with session_scope() as s:
        repo = Repository(s)
        logs = repo.get_logs(200)
    if not logs:
        st.info("Nenhum log ainda.")
        return
    df_l = pd.DataFrame(logs)[["timestamp", "level", "message"]]
    lvl_filter = st.multiselect("Nível:", ["INFO","WARNING","ERROR","DEBUG"],
                                 default=["INFO","WARNING","ERROR"],
                                 key="log_lvl")
    st.dataframe(df_l[df_l["level"].isin(lvl_filter)],
                 width="stretch", hide_index=True)
    st.download_button("⬇️ Baixar log", logs_to_text(logs),
                       f"logs_{version_tag()}.txt", key="btn_log_dl")
