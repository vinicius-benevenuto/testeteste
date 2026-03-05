"""
ui/pages.py — Páginas do VIVOHUB. Visual corporativo, sem emojis.
"""
from __future__ import annotations
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

# ── Tokens de design ──────────────────────────────────────────────────────────
_BG      = "#F8F9FA"
_SURFACE = "#FFFFFF"
_BORDER  = "#E2E8F0"
_TEXT    = "#111827"
_MUTED   = "#6B7280"
_FAINT   = "#9CA3AF"
_HEADING = "#0F172A"
_ACCENT  = "#1B5FBF"
_ACC_LT  = "#EFF6FF"
_OK      = "#16A34A";  _OK_BG  = "#F0FDF4"
_WARN    = "#D97706";  _WN_BG  = "#FFFBEB"
_ERR     = "#DC2626";  _ER_BG  = "#FEF2F2"


# ── Primitivos de UI ──────────────────────────────────────────────────────────

def _divider() -> None:
    st.markdown(
        f'<div style="height:1px;background:{_BORDER};margin:16px 0;"></div>',
        unsafe_allow_html=True,
    )


def _section(title: str, tag: str = "") -> None:
    tag_html = (
        f'<span style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        f'text-transform:uppercase;color:{_FAINT};margin-right:10px;">{tag}</span>'
    ) if tag else ""
    st.markdown(
        f'<div style="margin:22px 0 8px;">{tag_html}'
        f'<span style="font-size:13px;font-weight:700;color:{_HEADING};">{title}</span>'
        f'<div style="height:1px;background:{_BORDER};margin-top:7px;"></div></div>',
        unsafe_allow_html=True,
    )


def _label(text: str) -> None:
    st.markdown(
        f'<div style="font-size:9px;font-weight:700;letter-spacing:.1em;'
        f'text-transform:uppercase;color:{_FAINT};margin-bottom:5px;">{text}</div>',
        unsafe_allow_html=True,
    )


def _alert(text: str, kind: str = "info") -> None:
    bg, border, fg = {
        "info":  (_ACC_LT, _ACCENT, "#1E3A5F"),
        "ok":    (_OK_BG,  _OK,     "#14532D"),
        "warn":  (_WN_BG,  _WARN,   "#78350F"),
        "error": (_ER_BG,  _ERR,    "#7F1D1D"),
    }.get(kind, (_ACC_LT, _ACCENT, "#1E3A5F"))
    st.markdown(
        f'<div style="background:{bg};border-left:3px solid {border};'
        f'padding:10px 14px;margin:6px 0;font-size:12px;color:{fg};font-weight:500;">'
        f'{text}</div>',
        unsafe_allow_html=True,
    )


def _kv_bar(items: list) -> None:
    """Barra de estatísticas estilo key–value."""
    cells = "".join(
        f'<div style="padding:0 24px 0 0;">'
        f'<div style="font-size:9px;font-weight:700;letter-spacing:.09em;'
        f'text-transform:uppercase;color:{_FAINT};margin-bottom:3px;">{k}</div>'
        f'<div style="font-size:18px;font-weight:700;color:{c or _TEXT};">{v}</div>'
        f'</div>'
        for k, v, c in items
    )
    st.markdown(
        f'<div style="display:flex;flex-wrap:wrap;padding:14px 16px;'
        f'background:{_SURFACE};border:1px solid {_BORDER};margin:8px 0;">{cells}</div>',
        unsafe_allow_html=True,
    )


def _file_pill(filename: str, rows: int, cols: int) -> None:
    st.markdown(
        f'<div style="display:flex;align-items:center;gap:10px;padding:8px 0;'
        f'border-bottom:1px solid {_BORDER};">'
        f'<span style="width:6px;height:6px;border-radius:50%;background:{_OK};'
        f'flex-shrink:0;display:inline-block;"></span>'
        f'<span style="font-size:12px;font-weight:600;color:{_TEXT};">{filename}</span>'
        f'<span style="font-size:11px;color:{_FAINT};">{rows:,} linhas · {cols} colunas</span>'
        f'</div>',
        unsafe_allow_html=True,
    )


# ── Estado da sessão ──────────────────────────────────────────────────────────

def _ss(k, default=None):
    return st.session_state.get(k, default)


def _init_state() -> None:
    defaults = {
        "sci_df": None,  "sci_filename": "",  "sci_import_id": None,
        "sci_col_map": {}, "sci_hash": "",    "sci_sheet": None,
        "por_df": None,  "por_filename": "",  "por_import_id": None,
        "por_col_map": {}, "por_hash": "",    "por_sheet": None,
        "arq3_df": None, "arq3_filename": "", "arq3_import_id": None,
        "arq3_col_map": {}, "arq3_hash": "",  "arq3_sheet": None,
        "merged_df": None, "merge_report": None, "version_id": None,
        "wizard_cfg": {}, "uf_map": {}, "seeds_loaded": False,
    }
    for k, v in defaults.items():
        if k not in st.session_state:
            st.session_state[k] = v


# ── Upload helper ─────────────────────────────────────────────────────────────

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
        st.session_state[f"{prefix}_df"]       = df
        st.session_state[f"{prefix}_filename"] = uploaded_file.name
        st.session_state[f"{prefix}_raw"]      = raw
        st.session_state[f"{prefix}_col_map"]  = col_map
        st.session_state[f"{prefix}_hash"]     = file_hash(raw)
        st.session_state[f"{prefix}_sheet"]    = sheet
        _alert(f"{uploaded_file.name} — {len(df):,} linhas · {len(df.columns)} colunas", "ok")
    except Exception as e:
        _alert(f"Erro ao ler {uploaded_file.name}: {e}", "error")
        log.error("Upload error: %s", e, exc_info=True)


# ── Página 1: Carregar Arquivos ───────────────────────────────────────────────

def render_upload_page() -> None:
    st.markdown(
        f'<p style="font-size:12px;color:{_MUTED};margin:0 0 18px;">'
        'Faça o upload dos três arquivos de origem. '
        'Todo o processamento ocorre localmente — nenhum dado é transmitido externamente.'
        '</p>',
        unsafe_allow_html=True,
    )

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
            if f:
                _handle_upload(f, prefix, label)
            df = _ss(f"{prefix}_df")
            if df is not None:
                _file_pill(_ss(f"{prefix}_filename") or "—", len(df), len(df.columns))

    _divider()

    sci_ok  = _ss("sci_df")  is not None
    por_ok  = _ss("por_df")  is not None
    arq3_ok = _ss("arq3_df") is not None
    if sci_ok and por_ok:
        suffix = " Arquivo 3 carregado." if arq3_ok else " Arquivo 3 não carregado (opcional)."
        _alert("Science e Portal carregados." + suffix, "ok")

    # Seeds
    with session_scope() as s:
        repo  = Repository(s)
        cnl_n = repo.cnl_count()
        uf_n  = repo.cn_to_uf_count()
    _kv_bar([("Registros CNL", f"{cnl_n:,}", None),
             ("Mapeamentos CN→UF", str(uf_n),  None)])
    if cnl_n == 0:
        _alert("Tabela CNL vazia. Acesse <strong>Seeds / Referências</strong> para carregar.", "warn")

    # Previews
    for prefix, label in [("sci","Science"), ("por","Portal de Cadastros"), ("arq3","Arquivo 3")]:
        df = _ss(f"{prefix}_df")
        if df is not None:
            with st.expander(f"{label} — {len(df):,} linhas · {len(df.columns)} colunas"):
                st.dataframe(df.head(20), use_container_width=True, hide_index=True)
                st.markdown(
                    f'<div style="font-size:11px;color:{_FAINT};margin-top:4px;">'
                    f'Colunas: {", ".join(df.columns.tolist())}</div>',
                    unsafe_allow_html=True,
                )


# ── Página 2: Mapeamento ──────────────────────────────────────────────────────

def render_mapping_page() -> None:
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")
    if sci_df is None or por_df is None:
        _alert("Carregue os arquivos Science e Portal antes de configurar o mapeamento.", "warn")
        return

    cfg = run_mapping_wizard(
        science_cols=list(sci_df.columns),
        portal_cols=list(por_df.columns),
        arq3_cols=list(arq3_df.columns) if arq3_df is not None else [],
    )

    join_sci = cfg.get("join_keys_sci", [])
    join_por = cfg.get("join_keys_por", [])
    if join_sci and join_por and len(join_sci) == len(join_por):
        _divider()
        _section("Preview de correspondências por chave de junção")
        for ks, kp in zip(join_sci, join_por):
            if ks in sci_df.columns and kp in por_df.columns:
                stats = count_join_matches(sci_df, por_df, ks, kp)
                st.markdown(
                    f'<div style="font-size:12px;font-weight:600;color:{_HEADING};margin:10px 0 6px;">'
                    f'<code>{ks}</code> (Science) &nbsp;↔&nbsp; <code>{kp}</code> (Portal)</div>',
                    unsafe_allow_html=True,
                )
                mc1, mc2, mc3 = st.columns(3)
                mc1.metric("Correspondências", stats.get("matches", 0))
                mc2.metric("Somente Science",  stats.get("sci_only", 0))
                mc3.metric("Somente Portal",   stats.get("por_only", 0))

    _divider()
    if st.button("Salvar Configuração", type="primary", key="btn_save_cfg"):
        st.session_state["wizard_cfg"] = cfg
        _alert("Configuração salva. Prossiga para <strong>Gerar Tabela Final</strong>.", "ok")


# ── Página 3: Gerar Tabela Final ──────────────────────────────────────────────

def render_merge_page() -> None:
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")
    cfg     = _ss("wizard_cfg") or {}

    if sci_df is None or por_df is None:
        _alert("Carregue os arquivos antes de executar o pipeline.", "warn")
        return

    r1, r2, r3 = st.columns(3)
    r1.metric("Linhas Science", f"{len(sci_df):,}")
    r2.metric("Linhas Portal",  f"{len(por_df):,}")
    r3.metric("Arquivo 3",      f"{len(arq3_df):,}" if arq3_df is not None else "—")

    _divider()

    if not cfg:
        _alert("Configuração não salva. Acesse <strong>Mapeamento</strong> e clique em Salvar.", "warn")
    else:
        id_rota = cfg.get("id_rota_sci_col", "(não definida)")
        _alert(
            f"Configuração ativa &nbsp;·&nbsp; "
            f"Central Science: <strong>{cfg.get('central_sci_col','auto')}</strong> &nbsp;·&nbsp; "
            f"ID Rota: <strong>{id_rota}</strong> &nbsp;·&nbsp; "
            f"Central Portal: <strong>{cfg.get('central_portal_col','CENTRAL')}</strong>",
            "info",
        )
        if id_rota in ("(não definida)", "(nenhuma)", ""):
            _alert("ID Rota não configurada — Etapa 3 do Science será ignorada.", "warn")

    col_run, col_clr = st.columns([5, 1])
    with col_clr:
        if st.button("Limpar cache", key="btn_clear_cache"):
            for k in ("merged_df", "merge_report", "wizard_cfg"):
                st.session_state.pop(k, None)
            st.rerun()
    with col_run:
        run = st.button("Executar Pipeline", type="primary",
                        key="btn_merge", use_container_width=True)

    if run:
        with st.spinner("Executando pipeline..."):
            try:
                merged, report = build_merged_df(
                    science_df=sci_df.copy(), portal_df=por_df.copy(),
                    ref_df=arq3_df.copy() if arq3_df is not None else None,
                    uf_map={}, join_keys_sci=[], join_keys_por=[], config=cfg,
                )
                st.session_state["merged_df"]    = merged
                st.session_state["merge_report"] = report
                _alert(f"{len(merged):,} linhas geradas com sucesso.", "ok")
            except Exception as e:
                _alert(f"Erro na execução do pipeline: {e}", "error")
                log.error("Pipeline error: %s", e, exc_info=True)
                import traceback; st.code(traceback.format_exc())
                return

    merged_df = _ss("merged_df")
    if merged_df is None:
        with session_scope() as s:
            merged_df = Repository(s).load_merged_df()
        if merged_df is not None and not merged_df.empty:
            _alert("Exibindo última versão salva no banco de dados.", "info")
            st.session_state["merged_df"] = merged_df
    if merged_df is None or merged_df.empty:
        return

    report = _ss("merge_report") or {}

    # ── Relatório ─────────────────────────────────────────────────────────
    _divider()
    with st.expander("Relatório do Pipeline", expanded=True):
        _label("Science — Etapas 1 a 4")
        s1, s2, s3, s4, s5 = st.columns(5)
        s1.metric("Total inicial",    report.get("total_inicial_science", 0))
        s2.metric("Filtro Central",   report.get("total_apos_filtro_central", 0),
                  delta=report.get("total_apos_filtro_central",0)-report.get("total_inicial_science",0))
        s3.metric("Filtro Data",      report.get("total_apos_filtro_data", 0),
                  delta=report.get("total_apos_filtro_data",0)-report.get("total_apos_filtro_central",0))
        s4.metric("Valid. Rotas",     report.get("total_apos_validacao_rotas", 0),
                  delta=report.get("total_apos_validacao_rotas",0)-report.get("total_apos_filtro_data",0))
        s5.metric("Enriq. Arq3",      report.get("total_enriquecido_arq3", 0))

        st.markdown("<div style='margin-top:10px;'></div>", unsafe_allow_html=True)
        _label("Portal — Etapas 5 a 9")
        p1, p2, p3 = st.columns(3)
        p1.metric("Total processado",   report.get("total_portal_processado", 0))
        p2.metric("Encontrado no Arq3", report.get("total_encontrado_arq3_portal", 0))
        p3.metric("Novas rotas",        report.get("total_novas_rotas_criadas", 0))

        st.markdown("<div style='margin-top:10px;'></div>", unsafe_allow_html=True)
        _label("Arq3 + Resultado Final")
        g1, g2, g3, g4, g5, g6 = st.columns(6)
        g1.metric("Arq3 sem match",    report.get("total_arq3_sem_match", 0))
        g2.metric("Total linhas",      report.get("total_resultado_final", len(merged_df)))
        g3.metric("UF em branco",      report.get("uf_missing", 0),         delta_color="inverse")
        g4.metric("CLUSTER em branco", report.get("cluster_missing", 0),    delta_color="inverse")
        g5.metric("Inconsistências",   report.get("total_inconsistencias",0),delta_color="inverse")
        g6.metric("Redução Science",   f"{report.get('reducao_science_pct',0)}%")

        st.markdown("<div style='margin-top:10px;'></div>", unsafe_allow_html=True)
        _label("Validações Automáticas — R1 · R2 · R3")
        q1, q2, q3, q4 = st.columns(4)
        q1.metric("Correções REDE (R1)",  report.get("correcoes_rede", 0),
                  delta_color="inverse" if report.get("correcoes_rede",0) > 0 else "off",
                  help="REDE corrigida conforme origem (imutável)")
        q2.metric("Tipo de Rota (R2)",    report.get("correcoes_tipo_rota", 0),
                  help="Tipo de Rota preenchido com ITX-SIP_AS")
        q3.metric("UF via CNL_PPI (R3)",  report.get("uf_preenchido_r3", 0),
                  help="UF derivada com sucesso via CNL_PPI")
        q4.metric("UF não derivada (R3)", report.get("uf_invalido_r3", 0),
                  delta_color="inverse" if report.get("uf_invalido_r3",0) > 0 else "off",
                  help="CNL_PPI inválido ou CN não encontrado")

        if report.get("uf_divergente_r3", 0) > 0:
            _alert(
                f"<strong>{report['uf_divergente_r3']} registro(s)</strong> com UF divergente "
                f"entre Arq3 e CNL_PPI — CNL_PPI aplicado como fonte primária.", "warn",
            )

        qlogs = report.get("_quality_logs", [])
        if qlogs:
            with st.expander(f"Log de Correções Automáticas — {len(qlogs)} entradas"):
                df_ql = pd.DataFrame(qlogs)
                cols_show = [c for c in ["regra","rotulo","antes","depois","mensagem"]
                             if c in df_ql.columns]
                st.dataframe(df_ql[cols_show], use_container_width=True, hide_index=True)

        incons = report.get("_inconsistencias", [])
        if incons:
            st.markdown(
                f'<div style="font-size:12px;font-weight:700;color:{_HEADING};margin:14px 0 6px;">'
                f'Inconsistências — {len(incons)} ocorrências</div>',
                unsafe_allow_html=True,
            )
            df_i = pd.DataFrame(incons)
            for etapa in df_i["etapa"].unique():
                g = df_i[df_i["etapa"] == etapa]
                with st.expander(f"{etapa} — {len(g)} ocorrência(s)"):
                    st.dataframe(g[["chave","motivo"]], use_container_width=True, hide_index=True)

    # ── Grade ─────────────────────────────────────────────────────────────
    _divider()
    display_cols = [c for c in merged_df.columns if not c.startswith("_")]
    filtered_df  = show_grid(merged_df[display_cols], key="merge_grid",
                             title="Resultado Final", height=620)

    # ── Exportação ────────────────────────────────────────────────────────
    _divider()
    _section("Exportar")
    ex1, ex2, ex3, ex4 = st.columns(4)
    with ex1:
        if st.button("Salvar no banco de dados", key="btn_save", use_container_width=True):
            _save_to_db(merged_df)
    with ex2:
        st.download_button("Exportar CSV", to_csv_bytes(filtered_df),
                           f"resultado_{version_tag()}.csv", "text/csv",
                           key="btn_csv", use_container_width=True)
    with ex3:
        try:
            st.download_button("Exportar XLSX", to_xlsx_bytes(filtered_df),
                               f"resultado_{version_tag()}.xlsx",
                               "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                               key="btn_xlsx", use_container_width=True)
        except Exception as e:
            _alert(f"Erro ao gerar XLSX: {e}", "error")
    with ex4:
        clean = {k: v for k, v in report.items() if not k.startswith("_")}
        st.download_button("Relatório JSON", to_mapping_json(clean),
                           f"report_{version_tag()}.json", "application/json",
                           key="btn_json", use_container_width=True)
    if report.get("_inconsistencias"):
        st.download_button("Exportar Inconsistências CSV",
                           to_csv_bytes(pd.DataFrame(report["_inconsistencias"])),
                           f"inconsistencias_{version_tag()}.csv", "text/csv",
                           key="btn_incons")


def _save_to_db(merged_df: pd.DataFrame) -> None:
    cfg = _ss("wizard_cfg") or {}
    try:
        with session_scope() as s:
            repo = Repository(s)
            sci_id = _ss("sci_import_id")
            if not sci_id and _ss("sci_df") is not None:
                sci_id = repo.save_import("SCIENCE", _ss("sci_filename") or "science.xlsx",
                    _ss("sci_sheet"), _ss("sci_hash") or "",
                    st.session_state["sci_df"], _ss("sci_col_map") or {})
                st.session_state["sci_import_id"] = sci_id
            por_id = _ss("por_import_id")
            if not por_id and _ss("por_df") is not None:
                por_id = repo.save_import("PORTAL", _ss("por_filename") or "portal.xlsx",
                    _ss("por_sheet"), _ss("por_hash") or "",
                    st.session_state["por_df"], _ss("por_col_map") or {})
                st.session_state["por_import_id"] = por_id
            arq3_id = _ss("arq3_import_id")
            if not arq3_id and _ss("arq3_df") is not None:
                arq3_id = repo.save_import("ARQ3", _ss("arq3_filename") or "arquivo3.xlsx",
                    _ss("arq3_sheet"), _ss("arq3_hash") or "",
                    st.session_state["arq3_df"], _ss("arq3_col_map") or {})
                repo.save_arq3(arq3_id, st.session_state["arq3_df"], _ss("arq3_col_map") or {})
                st.session_state["arq3_import_id"] = arq3_id
            vid = repo.save_merge_version(
                version_tag(), sci_id, por_id, arq3_id,
                mapping=cfg, join_keys=cfg.get("join_keys_sci",[]),
                join_type=cfg.get("join_type","outer"), fuzzy_threshold=90,
                merged_df=merged_df,
                rows_sci=len(st.session_state.get("sci_df") or []),
                rows_por=len(st.session_state.get("por_df") or []),
            )
            st.session_state["version_id"] = vid
            repo.add_log("INFO", f"Merge salvo version_id={vid}")
        _alert(f"Resultado salvo. ID da versão: <code>{vid[:8]}...</code>", "ok")
    except Exception as e:
        _alert(f"Erro ao salvar: {e}", "error")
        log.error("DB save: %s", e, exc_info=True)


# ── Página 4: Seeds ───────────────────────────────────────────────────────────

def render_seeds_page() -> None:
    with session_scope() as s:
        repo  = Repository(s)
        cnl_n = repo.cnl_count()
        uf_n  = repo.cn_to_uf_count()
    _kv_bar([("Registros CNL", f"{cnl_n:,}", None), ("Mapeamentos CN→UF", str(uf_n), None)])

    import os
    from pathlib import Path
    seed_dir = Path("seeds")

    _section("Seeds automáticos")
    c1, c2 = st.columns(2, gap="medium")
    with c1:
        _label("seeds/cnl.sql")
        cnl_sql = seed_dir / "cnl.sql"
        if cnl_sql.exists():
            if st.button("Carregar CNL SQL", key="btn_cnl_seed", use_container_width=True):
                with session_scope() as s:
                    n = Repository(s).load_cnl_seeds(str(cnl_sql))
                _alert(f"{n} statements CNL executados.", "ok")
        else:
            _alert("seeds/cnl.sql não encontrado.", "warn")
    with c2:
        _label("seeds/cn_to_uf.csv")
        uf_csv = seed_dir / "cn_to_uf.csv"
        if uf_csv.exists():
            if st.button("Carregar CN → UF", key="btn_uf_seed", use_container_width=True):
                with session_scope() as s:
                    n = Repository(s).load_cn_to_uf_csv(str(uf_csv))
                _alert(f"{n} mapeamentos carregados.", "ok")
        else:
            _alert("seeds/cn_to_uf.csv não encontrado.", "warn")

    _divider()
    _section("Upload manual")
    uf_file = st.file_uploader("cn_to_uf.csv personalizado", type=["csv"], key="up_uf_csv")
    if uf_file:
        import io, tempfile
        tmp = tempfile.NamedTemporaryFile(suffix=".csv", delete=False)
        tmp.write(uf_file.read()); tmp.flush()
        try:
            with session_scope() as s:
                n = Repository(s).load_cn_to_uf_csv(tmp.name)
            _alert(f"{n} mapeamentos carregados.", "ok")
        finally:
            os.unlink(tmp.name)

    _divider()
    _section("Testar lookup de UF")
    test_cnl = st.text_input("COD_CNL:", key="test_cnl", placeholder="Ex.: 11ABCD")
    if test_cnl:
        with session_scope() as s:
            uf = Repository(s).get_uf_for_cnl(test_cnl)
        if uf:
            _alert(f"CNL <code>{test_cnl}</code> → UF: <strong>{uf}</strong>", "ok")
        else:
            _alert(f"CNL <code>{test_cnl}</code> não encontrado.", "warn")


# ── Página 5: Histórico ───────────────────────────────────────────────────────

def render_history_page() -> None:
    with session_scope() as s:
        repo     = Repository(s)
        versions = repo.list_versions()
        imports  = repo.list_imports()
    if not versions:
        _alert("Nenhum resultado salvo ainda.", "info"); return

    st.dataframe(pd.DataFrame(versions), use_container_width=True, hide_index=True)
    _divider()
    sel = st.selectbox("Versão:", [v["id"] for v in versions],
                        format_func=lambda x: next(
                            (f"{v['tag']} — {v['rows_merged']:,} linhas"
                             for v in versions if v["id"] == x), x),
                        key="hist_sel")
    if st.button("Carregar versão selecionada", key="btn_hist_load", type="primary"):
        with session_scope() as s2:
            loaded = Repository(s2).load_merged_df(sel)
        st.session_state["merged_df"] = loaded
        _alert(f"{len(loaded):,} linhas carregadas.", "ok")
        show_grid(loaded[[c for c in loaded.columns if not c.startswith("_")]],
                  key="hist_grid", title="Versão carregada")
    if imports:
        _divider()
        with st.expander(f"Importações registradas — {len(imports)}"):
            st.dataframe(pd.DataFrame(imports), use_container_width=True, hide_index=True)


# ── Página 6: Validação ───────────────────────────────────────────────────────

def render_validation_page() -> None:
    merged = _ss("merged_df")
    if merged is None:
        with session_scope() as s:
            merged = Repository(s).load_merged_df()
    if merged is None or merged.empty:
        _alert("Nenhum resultado gerado ainda.", "info"); return

    q = quality_summary(merged)
    c1, c2, c3, c4 = st.columns(4)
    c1.metric("Total linhas",      q.get("total", 0))
    c2.metric("UF em branco",      q.get("UF_missing", 0))
    c3.metric("CLUSTER em branco", q.get("CLUSTER_missing", 0))
    c4.metric("Sem match Arq3",    q.get("arq3_no_match", 0))

    _divider()
    uf_pending = merged[merged["UF"].eq("") | merged["UF"].isna()]
    if not uf_pending.empty:
        with st.expander(f"Linhas sem UF — {len(uf_pending)} registros"):
            st.dataframe(
                uf_pending[[c for c in uf_pending.columns if not c.startswith("_")]].head(50),
                use_container_width=True, hide_index=True,
            )
    _divider()
    for df, name in [(_ss("sci_df"),"Science"),(_ss("por_df"),"Portal"),(merged,"Resultado")]:
        if df is not None:
            with st.expander(f"Relatório de colunas — {name}"):
                st.dataframe(pd.DataFrame(generate_report(df, name)["colunas"]),
                             use_container_width=True, hide_index=True)


# ── Página 7: Logs ────────────────────────────────────────────────────────────

def render_logs_page() -> None:
    with session_scope() as s:
        logs = Repository(s).get_logs(200)
    if not logs:
        _alert("Nenhum log registrado.", "info"); return

    df_l = pd.DataFrame(logs)[["timestamp","level","message"]]
    lvl_filter = st.multiselect("Nível:", ["INFO","WARNING","ERROR","DEBUG"],
                                 default=["INFO","WARNING","ERROR"], key="log_lvl")
    st.dataframe(df_l[df_l["level"].isin(lvl_filter)], use_container_width=True, hide_index=True)
    _divider()
    st.download_button("Exportar log", logs_to_text(logs),
                       f"logs_{version_tag()}.txt", key="btn_log_dl")


# ── Página 8: Diagnóstico ─────────────────────────────────────────────────────

def render_diagnostico_page() -> None:
    arq3_df = _ss("arq3_df")
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    cfg     = _ss("wizard_cfg") or {}

    from app.core.pipeline import (
        _find_col, _extract_arq3_cols, _build_arq3_rotulo_index,
        _build_arq3_central_index, CENTRAL_PORTAL_MAP, CN_TO_UF,
    )
    if arq3_df is None:
        _alert("Carregue o Arquivo 3 para executar o diagnóstico.", "warn"); return

    # ── 1. Colunas do Arquivo 3 ────────────────────────────────────────────
    _section("Colunas detectadas no Arquivo 3", "Etapa 1")
    arq3_cols    = _extract_arq3_cols(arq3_df, cfg)
    rot_col      = arq3_cols.get("rotulos", "")
    rotulo_index = _build_arq3_rotulo_index(arq3_df, rot_col)

    d1, d2, d3 = st.columns(3)
    d1.metric("Rótulos únicos",      len(rotulo_index))
    d2.metric("Linhas no Arquivo 3", len(arq3_df))
    d3.metric("Colunas detectadas",  sum(1 for v in arq3_cols.values() if v))

    field_map = {
        "REDE": arq3_cols.get("rede",""), "Central": arq3_cols.get("central",""),
        "UF": arq3_cols.get("uf",""), "CLUSTER": arq3_cols.get("cluster",""),
        "Tipo de Rota": arq3_cols.get("tipo",""), "Rótulos de Linha": arq3_cols.get("rotulos",""),
        "OPERADORA": arq3_cols.get("operadora",""), "Denominação": arq3_cols.get("den",""),
    }
    st.markdown(
        f'<div style="background:{_SURFACE};border:1px solid {_BORDER};margin:8px 0 12px;">',
        unsafe_allow_html=True,
    )
    for field, col in field_map.items():
        if col and col in arq3_df.columns:
            samples = (arq3_df[col].dropna().astype(str).str.strip()
                      .pipe(lambda s: s[s.str.len() > 0]).unique()[:4])
            st.markdown(
                f'<div style="display:flex;align-items:center;gap:12px;padding:7px 14px;'
                f'border-bottom:1px solid {_BORDER};font-size:12px;">'
                f'<span style="width:6px;height:6px;border-radius:50%;background:{_OK};'
                f'flex-shrink:0;display:inline-block;"></span>'
                f'<span style="min-width:130px;font-weight:600;color:{_TEXT};">{field}</span>'
                f'<code style="font-size:11px;background:#F3F4F6;color:{_ACCENT};'
                f'padding:1px 7px;">{col}</code>'
                f'<span style="color:{_FAINT};font-size:11px;">'
                f'{" &nbsp;·&nbsp; ".join(samples)}</span></div>',
                unsafe_allow_html=True,
            )
        else:
            st.markdown(
                f'<div style="display:flex;align-items:center;gap:12px;padding:7px 14px;'
                f'border-bottom:1px solid {_BORDER};font-size:12px;">'
                f'<span style="width:6px;height:6px;border-radius:50%;background:{_BORDER};'
                f'flex-shrink:0;display:inline-block;"></span>'
                f'<span style="min-width:130px;font-weight:600;color:{_FAINT};">{field}</span>'
                f'<span style="color:{_ERR};font-size:11px;">Não detectada — configure em Mapeamento</span>'
                f'</div>',
                unsafe_allow_html=True,
            )
    st.markdown("</div>", unsafe_allow_html=True)

    with st.expander(f"Todas as colunas — {len(arq3_df.columns)} colunas"):
        for i, c in enumerate(arq3_df.columns):
            sample = arq3_df[c].dropna().iloc[0] if not arq3_df[c].dropna().empty else ""
            st.text(f"{i+1:3d}. {c!r:40s}  ex: {str(sample)[:60]}")

    # ── 2. Diagnóstico Science ─────────────────────────────────────────────
    if sci_df is not None:
        _divider()
        _section("Diagnóstico Science — Etapa 3 (amostra 20 linhas)", "Etapa 2")
        col_cen    = _find_col(list(sci_df.columns), cfg.get("central_sci_col",""),
                               "Central","CENTRAL","Central Interna","Central Origem")
        col_id_rot = _find_col(list(sci_df.columns), cfg.get("id_rota_sci_col",""),
                               "ID Rota","ID_ROTA","IDROTA","Id Rota")
        _alert(
            f"Coluna Central: <strong>{col_cen or 'não encontrada'}</strong> &nbsp;·&nbsp; "
            f"Coluna ID Rota: <strong>{col_id_rot or 'não encontrada'}</strong>", "info",
        )
        matches, no_matches = [], []
        for _, row in sci_df.head(20).iterrows():
            d = {c: str(row.get(c,"")).strip() for c in sci_df.columns}
            cv = d.get(col_cen,"").upper()    if col_cen    else ""
            rv = d.get(col_id_rot,"").upper() if col_id_rot else ""
            ch = (cv[:7]+"_"+rv) if cv and rv else ""
            fd = rotulo_index.get(ch)
            if fd:
                matches.append({"Central":cv,"ID Rota":rv,"Chave":ch,
                                 "CLUSTER":fd.get(arq3_cols.get("cluster",""),""),
                                 "UF":fd.get(arq3_cols.get("uf",""),"")})
            else:
                no_matches.append({"Central":cv,"ID Rota":rv,"Chave tentada":ch or "(sem chave)"})

        mc1, mc2 = st.columns(2)
        mc1.metric("Com correspondência", len(matches))
        mc2.metric("Sem correspondência", len(no_matches))
        if matches:
            with st.expander(f"Com correspondência — {len(matches)}"):
                st.dataframe(pd.DataFrame(matches), use_container_width=True, hide_index=True)
        if no_matches:
            with st.expander(f"Sem correspondência — {len(no_matches)}"):
                st.dataframe(pd.DataFrame(no_matches), use_container_width=True, hide_index=True)
                if rot_col:
                    st.markdown(
                        f'<div style="font-size:11px;font-weight:600;color:{_HEADING};'
                        f'margin:8px 0 4px;">Exemplos de rótulos no Arquivo 3:</div>',
                        unsafe_allow_html=True,
                    )
                    st.write(sorted(list(rotulo_index.keys()))[:20])

    # ── 3. Diagnóstico Portal ──────────────────────────────────────────────
    if por_df is not None:
        _divider()
        _section("Diagnóstico Portal — Etapas 5 a 8 (amostra 20 linhas)", "Etapa 3")
        col_cen_p = _find_col(list(por_df.columns), cfg.get("central_portal_col",""), "CENTRAL")
        col_le    = _find_col(list(por_df.columns), cfg.get("label_e_col",""),         "LABEL_E")
        col_ls    = _find_col(list(por_df.columns), cfg.get("label_s_col",""),         "LABEL_S")
        _alert(
            f"CENTRAL: <strong>{col_cen_p or 'não encontrada'}</strong> &nbsp;·&nbsp; "
            f"LABEL_E: <strong>{col_le or 'não encontrada'}</strong> &nbsp;·&nbsp; "
            f"LABEL_S: <strong>{col_ls or 'não encontrada'}</strong>", "info",
        )
        centrais_sem_mapa = [
            c for c in (por_df[col_cen_p].dropna().unique() if col_cen_p else [])
            if str(c).strip().upper() not in set(CENTRAL_PORTAL_MAP.keys())
        ]
        if centrais_sem_mapa:
            _alert(f"{len(centrais_sem_mapa)} valores de CENTRAL sem mapeamento: "
                   f"{centrais_sem_mapa[:5]}", "warn")

        matches_p, no_matches_p = [], []
        for _, row in por_df.head(20).iterrows():
            d       = {c: str(row.get(c,"")).strip() for c in por_df.columns}
            cr      = d.get(col_cen_p,"").upper() if col_cen_p else ""
            cc      = CENTRAL_PORTAL_MAP.get(cr, cr)
            le      = d.get(col_le,"") if col_le else ""
            ls      = d.get(col_ls,"") if col_ls else ""
            label   = le if len(le)>2 else ls if len(ls)>2 else le
            rotulo  = (cc[:7]+"_"+label[:6]).upper() if cc and label else ""
            fd      = rotulo_index.get(rotulo)
            matches_p.append({"CENTRAL":cr,"Conv.":cc,"LABEL":label,"Rótulo":rotulo,
                              "CLUSTER":fd.get(arq3_cols.get("cluster",""),"") if fd else "NOVA ROTA",
                              "UF":fd.get(arq3_cols.get("uf",""),"") if fd else "?"})
            if not fd:
                no_matches_p.append({"CENTRAL":cr,"Rótulo tentado":rotulo})

        pc1, pc2 = st.columns(2)
        pc1.metric("Encontrado no Arquivo 3", len(matches_p) - len(no_matches_p))
        pc2.metric("Nova rota",               len(no_matches_p))
        with st.expander("Amostra completa — 20 linhas"):
            st.dataframe(pd.DataFrame(matches_p), use_container_width=True, hide_index=True)

    # ── 4. Mapa CENTRAL ────────────────────────────────────────────────────
    _divider()
    _section("Mapa de Conversão CENTRAL — Portal", "Etapa 4")
    st.dataframe(
        pd.DataFrame([{"CENTRAL (Portal)":k,"CENTRAL (Convertida)":v}
                      for k,v in CENTRAL_PORTAL_MAP.items()]),
        use_container_width=True, hide_index=True,
    )

    # ── 5. CN → UF ─────────────────────────────────────────────────────────
    _divider()
    with st.expander(f"Tabela CN → UF — {len(CN_TO_UF)} DDDs"):
        st.dataframe(
            pd.DataFrame([{"CN":k,"UF":v}
                          for k,v in sorted(CN_TO_UF.items(), key=lambda x: int(x[0]))]),
            use_container_width=True, hide_index=True,
        )