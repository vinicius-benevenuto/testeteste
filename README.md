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

    if st.button("🚀 Gerar Tabela Combinada", type="primary", key="btn_merge"):
        with st.spinner("Processando..."):
            try:
                # Carrega uf_map do banco
                with session_scope() as s:
                    repo = Repository(s)
                    all_cnls = (
                        list(sci_df.get(cfg.get("cnl_sci_col","CNL"), pd.Series()).dropna())
                        + list(por_df.get(cfg.get("cnl_por_col","CNL_PPI"), pd.Series()).dropna())
                    )
                    uf_map = repo.resolve_ufs_batch([str(c) for c in all_cnls])
                st.session_state["uf_map"] = uf_map

                merged, report = build_merged_df(
                    science_df=sci_df.copy(),
                    portal_df=por_df.copy(),
                    ref_df=arq3_df.copy() if arq3_df is not None else None,
                    uf_map=uf_map,
                    join_keys_sci=cfg.get("join_keys_sci", []),
                    join_keys_por=cfg.get("join_keys_por", []),
                    join_type=cfg.get("join_type", "outer"),
                    config=cfg,
                )
                st.session_state["merged_df"]    = merged
                st.session_state["merge_report"] = report
                st.success(f"✅ {len(merged):,} linhas geradas.")
            except Exception as e:
                st.error(f"❌ Erro: {e}")
                log.error("Merge error: %s", e, exc_info=True)
                return

    merged_df = _ss("merged_df")
    if merged_df is None:
        # Tenta carregar a última versão persistida
        with session_scope() as s:
            repo = Repository(s)
            merged_df = repo.load_merged_df()
        if merged_df is not None and not merged_df.empty:
            st.info("ℹ️ Exibindo última versão salva no banco.")
            st.session_state["merged_df"] = merged_df

    if merged_df is None or merged_df.empty:
        return

    # Qualidade
    report = _ss("merge_report") or {}
    with st.expander("📋 Relatório do Merge"):
        q = quality_summary(merged_df)
        rc1, rc2, rc3, rc4 = st.columns(4)
        rc1.metric("Total linhas",   q.get("total", 0))
        rc2.metric("UF em branco",   q.get("UF_missing", 0))
        rc3.metric("Cluster em branco", q.get("CLUSTER_missing", 0))
        rc4.metric("Sem match Arq3", q.get("arq3_no_match", 0))

        sb = q.get("source_breakdown", {})
        if sb:
            st.markdown("**Origem das linhas:**")
            for tag, cnt in sb.items():
                st.write(f"  - `{tag}`: {cnt:,}")

    # Grid com floating filters
    display_cols = [c for c in merged_df.columns if not c.startswith("_")]
    filtered_df = show_grid(
        merged_df[display_cols],
        key="merge_grid",
        title="📊 Tabela Final Combinada",
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
        st.download_button("⬇️ Mapeamento JSON",
                           to_mapping_json({"config": cfg}),
                           f"mapping_{version_tag()}.json",
                           "application/json", key="btn_map_json")


def _save_to_db(merged_df: pd.DataFrame) -> None:
    try:
        with session_scope() as s:
            repo = Repository(s)
            cfg  = _ss("wizard_cfg") or {}

            # Science
            sci_id = _ss("sci_import_id")
            if not sci_id and _ss("sci_df") is not None:
                sci_id = repo.save_import(
                    "SCIENCE", _ss("sci_filename") or "science.xlsx",
                    _ss("sci_sheet"), _ss("sci_hash") or "",
                    st.session_state["sci_df"], _ss("sci_col_map") or {})
                st.session_state["sci_import_id"] = sci_id

            # Portal
            por_id = _ss("por_import_id")
            if not por_id and _ss("por_df") is not None:
                por_id = repo.save_import(
                    "PORTAL", _ss("por_filename") or "portal.xlsx",
                    _ss("por_sheet"), _ss("por_hash") or "",
                    st.session_state["por_df"], _ss("por_col_map") or {})
                st.session_state["por_import_id"] = por_id

            # Arquivo 3
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
    st.header("🔬 Diagnóstico de Mapeamento")
    st.markdown("""
    Esta página mostra **por que** colunas ficam vazias — exibe os valores reais
    que o app usa como chave para buscar no Arquivo 3.
    """)

    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")
    cfg     = _ss("wizard_cfg") or {}

    if sci_df is None and por_df is None:
        st.warning("⚠️ Carregue os arquivos primeiro.")
        return

    # ── 1. Mostra colunas encontradas ────────────────────────────────────
    st.markdown("### 1️⃣ Colunas configuradas no Mapeamento")
    campos = {
        "Central (Portal)":        cfg.get("central_portal_col", "CENTRAL"),
        "Central (Science)":       cfg.get("central_sci_col", "Central Origem"),
        "Tipo de Rota (Portal)":   cfg.get("tipo_rota_portal_col", "TIPO_ROTA"),
        "Tipo de Rota (Science)":  cfg.get("tipo_rota_sci_col", "Sinalização da Rota"),
        "CNL (Science)":           cfg.get("cnl_sci_col", "CNL"),
        "CNL (Portal)":            cfg.get("cnl_por_col", "CNL_PPI"),
        "LABEL_E (Portal)":        cfg.get("label_e_col", "LABEL_E"),
        "LABEL_S (Portal)":        cfg.get("label_s_col", "LABEL_S"),
        "OPERADORA (Science)":     cfg.get("operadora_sci_col", "Operadora Origem"),
    }
    rows = []
    for campo, col_name in campos.items():
        fonte = "Science" if "Science" in campo else "Portal"
        df_check = sci_df if fonte == "Science" else por_df
        if df_check is None:
            rows.append({"Campo": campo, "Coluna configurada": col_name,
                         "Existe no arquivo?": "—", "Exemplo de valor": "—"})
            continue
        existe = col_name in df_check.columns if col_name != "(nenhuma)" else False
        exemplo = ""
        if existe:
            serie = df_check[col_name].replace("", pd.NA).dropna()
            exemplo = str(serie.iloc[0]) if not serie.empty else "(vazio)"
        rows.append({
            "Campo":              campo,
            "Coluna configurada": col_name,
            "Existe no arquivo?": "✅ Sim" if existe else "❌ NÃO ENCONTRADA",
            "Exemplo de valor":   exemplo,
        })
    st.dataframe(pd.DataFrame(rows), width="stretch", hide_index=True)

    # ── 2. Amostra das chaves de busca no Arquivo 3 ──────────────────────
    st.markdown("### 2️⃣ Como o app monta a chave de busca no Arquivo 3")
    st.markdown("""
    O app busca cada rota no Arquivo 3 usando **duas tentativas em ordem**:
    1. `Central (normalizado UPPER)` + `Tipo de Rota (normalizado UPPER)`
    2. `Central (normalizado UPPER)` + `Rótulos de Linha (do Portal)`

    Se nenhuma bater → CLUSTER, UF (via Arq3), Denominação ficam vazios.
    """)

    if sci_df is not None or por_df is not None:
        ce_por = cfg.get("central_portal_col","CENTRAL")
        ce_sci = cfg.get("central_sci_col","Central Origem")
        tr_por = cfg.get("tipo_rota_portal_col","TIPO_ROTA")
        tr_sci = cfg.get("tipo_rota_sci_col","Sinalização da Rota")

        sample_rows = []
        df_src = por_df if por_df is not None else sci_df
        n_show = min(10, len(df_src))
        for i in range(n_show):
            por_r = df_src.iloc[i].to_dict() if por_df is not None else {}
            sci_r = sci_df.iloc[min(i, len(sci_df)-1)].to_dict() if sci_df is not None else {}
            from app.core.map_rules import coalesce
            central = coalesce(
                str(por_r.get(ce_por,"") or ""),
                str(sci_r.get(ce_sci,"") or "")
            ).strip().upper()
            tipo = coalesce(
                str(por_r.get(tr_por,"") or ""),
                str(sci_r.get(tr_sci,"") or "")
            ).strip().upper()
            sample_rows.append({
                "Central (chave)": central or "(vazio!)",
                "Tipo de Rota (chave)": tipo or "(vazio!)",
                "Chave final": f"{central}|{tipo}" if central and tipo else "⚠️ CHAVE INCOMPLETA",
            })
        st.dataframe(pd.DataFrame(sample_rows), width="stretch", hide_index=True)

        vazios_central = sum(1 for r in sample_rows if "(vazio!)" in r["Central (chave)"])
        if vazios_central > 0:
            st.error(f"⚠️ {vazios_central}/{n_show} linhas com Central vazia — o lookup no Arquivo 3 vai falhar para estas!")

    # ── 3. Amostra das chaves disponíveis no Arquivo 3 ───────────────────
    if arq3_df is not None:
        st.markdown("### 3️⃣ Chaves disponíveis no Arquivo 3 (primeiras 20 linhas)")
        from app.core.map_rules import build_ref_index
        idx = build_ref_index(arq3_df)
        st.caption(f"Total de chaves indexadas: **{len(idx)}**")

        from app.utils.encoding import normalize_column_label
        # Mostra colunas Central e Tipo de Rota do Arquivo 3
        cols_arq3 = list(arq3_df.columns)
        central_col = next((c for c in cols_arq3 if "central" in c.lower()), None)
        tipo_col    = next((c for c in cols_arq3 if "tipo" in c.lower() and "rota" in c.lower()), None)
        rotulo_col  = next((c for c in cols_arq3 if "r" in c.lower() and "tulo" in c.lower()), None)

        show_cols = [c for c in [central_col, tipo_col, rotulo_col,
                                  "CLUSTER","UF","OPERADORA","Denominação","Denominacao"]
                     if c and c in arq3_df.columns]
        if show_cols:
            st.dataframe(arq3_df[show_cols].head(20), width="stretch", hide_index=True)
        else:
            st.info("Colunas de referência não identificadas automaticamente.")
            st.dataframe(arq3_df.head(10), width="stretch", hide_index=True)

        st.markdown("**Exemplo de chaves indexadas (primeiras 10):**")
        for k in list(idx.keys())[:10]:
            st.code(k)
    else:
        st.warning("⚠️ Arquivo 3 não carregado — sem referência para CLUSTER, UF, Denominação e Rótulos padronizados.")

# ── Página 7: Logs ─────────────────────────────────────────────────────────

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
