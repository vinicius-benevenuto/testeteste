"""
ui/pages.py — Páginas do VIVOHUB. Padrão visual corporativo.
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


# ── Componentes de UI corporativos ────────────────────────────────────────────

def _section(title: str, subtitle: str = "") -> None:
    """Cabeçalho de seção com linha divisora."""
    if subtitle:
        st.markdown(
            f'<div style="margin:20px 0 10px;">'
            f'<div style="font-size:11px;font-weight:700;letter-spacing:0.1em;'
            f'text-transform:uppercase;color:#6B7280;margin-bottom:2px;">{subtitle}</div>'
            f'<div style="font-size:15px;font-weight:700;color:#1C2536;">{title}</div>'
            f'<div style="height:2px;background:#E9ECEF;margin-top:8px;"></div>'
            f'</div>',
            unsafe_allow_html=True,
        )
    else:
        st.markdown(
            f'<div style="margin:20px 0 10px;">'
            f'<div style="font-size:14px;font-weight:700;color:#1C2536;">{title}</div>'
            f'<div style="height:1px;background:#E9ECEF;margin-top:6px;"></div>'
            f'</div>',
            unsafe_allow_html=True,
        )


def _badge(text: str, variant: str = "neutral") -> str:
    """Badge inline HTML."""
    colors = {
        "neutral": ("#F3F4F6", "#374151", "#D1D5DB"),
        "blue":    ("#EFF6FF", "#1D4ED8", "rgba(29,78,216,.15)"),
        "green":   ("#F0FDF4", "#15803D", "rgba(21,128,61,.15)"),
        "amber":   ("#FEF9EF", "#92400E", "rgba(146,64,14,.15)"),
        "red":     ("#FEF2F2", "#DC2626", "rgba(220,38,38,.15)"),
    }
    bg, fg, border = colors.get(variant, colors["neutral"])
    return (
        f'<span style="display:inline-block;padding:1px 7px;font-size:10px;font-weight:700;'
        f'letter-spacing:0.06em;text-transform:uppercase;background:{bg};color:{fg};'
        f'border:1px solid {border};">{text}</span>'
    )


def _info_block(text: str) -> None:
    st.markdown(
        f'<div style="padding:10px 14px;background:#EFF6FF;border-left:3px solid #1B5FBF;'
        f'font-size:12px;color:#1E3A5F;margin:8px 0;">{text}</div>',
        unsafe_allow_html=True,
    )


def _warn_block(text: str) -> None:
    st.markdown(
        f'<div style="padding:10px 14px;background:#FFFBEB;border-left:3px solid #D97706;'
        f'font-size:12px;color:#78350F;margin:8px 0;">{text}</div>',
        unsafe_allow_html=True,
    )


def _ok_block(text: str) -> None:
    st.markdown(
        f'<div style="padding:10px 14px;background:#F0FDF4;border-left:3px solid #16A34A;'
        f'font-size:12px;color:#14532D;margin:8px 0;">{text}</div>',
        unsafe_allow_html=True,
    )


def _err_block(text: str) -> None:
    st.markdown(
        f'<div style="padding:10px 14px;background:#FEF2F2;border-left:3px solid #DC2626;'
        f'font-size:12px;color:#7F1D1D;margin:8px 0;">{text}</div>',
        unsafe_allow_html=True,
    )


def _divider() -> None:
    st.markdown(
        '<div style="height:1px;background:#E9ECEF;margin:18px 0;"></div>',
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
        _ok_block(f"{uploaded_file.name} — {len(df):,} linhas · {len(df.columns)} colunas")
    except Exception as e:
        _err_block(f"Erro ao ler {uploaded_file.name}: {e}")
        log.error("Upload error: %s", e, exc_info=True)


# ── Página 1: Carregar Arquivos ───────────────────────────────────────────────

def render_upload_page() -> None:
    st.markdown(
        '<p style="font-size:12px;color:#6B7280;margin-bottom:16px;">'
        'Faça o upload dos três arquivos de origem. '
        'Todo o processamento ocorre localmente — nenhum dado é transmitido externamente.'
        '</p>',
        unsafe_allow_html=True,
    )

    c1, c2, c3 = st.columns(3)

    with c1:
        st.markdown(
            '<div style="font-size:11px;font-weight:700;letter-spacing:0.08em;'
            'text-transform:uppercase;color:#6B7280;margin-bottom:8px;">Science</div>',
            unsafe_allow_html=True,
        )
        f = st.file_uploader("Arquivo Science", type=["xlsx","xls","csv","parquet"],
                              key="up_sci", label_visibility="collapsed")
        if f: _handle_upload(f, "sci", "Science")
        df = _ss("sci_df")
        if df is not None:
            st.markdown(
                f'<div style="font-size:11px;color:#22C55E;font-weight:600;margin-top:4px;">'
                f'{_ss("sci_filename")} &nbsp;·&nbsp; {len(df):,} linhas</div>',
                unsafe_allow_html=True,
            )

    with c2:
        st.markdown(
            '<div style="font-size:11px;font-weight:700;letter-spacing:0.08em;'
            'text-transform:uppercase;color:#6B7280;margin-bottom:8px;">Portal de Cadastros</div>',
            unsafe_allow_html=True,
        )
        f = st.file_uploader("Arquivo Portal", type=["xlsx","xls","csv","parquet"],
                              key="up_por", label_visibility="collapsed")
        if f: _handle_upload(f, "por", "Portal")
        df = _ss("por_df")
        if df is not None:
            st.markdown(
                f'<div style="font-size:11px;color:#22C55E;font-weight:600;margin-top:4px;">'
                f'{_ss("por_filename")} &nbsp;·&nbsp; {len(df):,} linhas</div>',
                unsafe_allow_html=True,
            )

    with c3:
        st.markdown(
            '<div style="font-size:11px;font-weight:700;letter-spacing:0.08em;'
            'text-transform:uppercase;color:#6B7280;margin-bottom:8px;">Arquivo 3 — Referência</div>',
            unsafe_allow_html=True,
        )
        f = st.file_uploader("Arquivo 3", type=["xlsx","xls","csv","parquet"],
                              key="up_arq3", label_visibility="collapsed")
        if f: _handle_upload(f, "arq3", "Arquivo 3")
        df = _ss("arq3_df")
        if df is not None:
            st.markdown(
                f'<div style="font-size:11px;color:#22C55E;font-weight:600;margin-top:4px;">'
                f'{_ss("arq3_filename")} &nbsp;·&nbsp; {len(df):,} linhas</div>',
                unsafe_allow_html=True,
            )

    _divider()

    # Status geral
    sci_ok  = _ss("sci_df")  is not None
    por_ok  = _ss("por_df")  is not None
    arq3_ok = _ss("arq3_df") is not None

    if sci_ok and por_ok:
        msg = "Science e Portal carregados."
        msg += " Arquivo 3 carregado." if arq3_ok else " Arquivo 3 não carregado (opcional)."
        _ok_block(msg)

    # Previews
    for prefix, label in [("sci","Science"), ("por","Portal de Cadastros"), ("arq3","Arquivo 3")]:
        df = _ss(f"{prefix}_df")
        if df is not None:
            with st.expander(f"Preview — {label}  ({len(df):,} linhas · {len(df.columns)} colunas)"):
                st.dataframe(df.head(20), use_container_width=True, hide_index=True)
                st.markdown(
                    f'<div style="font-size:11px;color:#6B7280;margin-top:4px;">'
                    f'Colunas: {", ".join(df.columns.tolist())}</div>',
                    unsafe_allow_html=True,
                )

    _divider()

    # Seeds status
    with session_scope() as s:
        repo = Repository(s)
        cnl_n = repo.cnl_count()
        uf_n  = repo.cn_to_uf_count()

    st.markdown(
        f'<div style="display:flex;gap:24px;padding:10px 14px;background:#F8F9FA;'
        f'border:1px solid #E9ECEF;font-size:12px;">'
        f'<span><span style="color:#6B7280;font-weight:600;text-transform:uppercase;'
        f'font-size:10px;letter-spacing:.06em;">CNL</span>&nbsp;&nbsp;'
        f'<strong>{cnl_n:,}</strong> registros</span>'
        f'<span><span style="color:#6B7280;font-weight:600;text-transform:uppercase;'
        f'font-size:10px;letter-spacing:.06em;">CN → UF</span>&nbsp;&nbsp;'
        f'<strong>{uf_n}</strong> mapeamentos</span>'
        f'</div>',
        unsafe_allow_html=True,
    )
    if cnl_n == 0:
        _warn_block("Tabela CNL vazia. Acesse <strong>Seeds / Referências</strong> para carregar.")


# ── Página 2: Mapeamento ──────────────────────────────────────────────────────

def render_mapping_page() -> None:
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")

    if sci_df is None or por_df is None:
        _warn_block("Carregue os arquivos Science e Portal antes de configurar o mapeamento.")
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
        _divider()
        _section("Preview de correspondências por chave de junção")
        for ks, kp in zip(join_sci, join_por):
            if ks in sci_df.columns and kp in por_df.columns:
                stats = count_join_matches(sci_df, por_df, ks, kp)
                st.markdown(
                    f'<div style="font-size:12px;font-weight:600;color:#1C2536;margin:10px 0 6px;">'
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
        _ok_block("Configuração salva. Prossiga para Gerar Tabela Final.")


# ── Página 3: Gerar Tabela Final ──────────────────────────────────────────────

def render_merge_page() -> None:
    sci_df  = _ss("sci_df")
    por_df  = _ss("por_df")
    arq3_df = _ss("arq3_df")
    cfg     = _ss("wizard_cfg") or {}

    if sci_df is None or por_df is None:
        _warn_block("Carregue os arquivos antes de executar o pipeline.")
        return

    # Resumo das fontes
    r1, r2, r3 = st.columns(3)
    r1.metric("Linhas Science", f"{len(sci_df):,}")
    r2.metric("Linhas Portal",  f"{len(por_df):,}")
    r3.metric("Arquivo 3",      f"{len(arq3_df):,}" if arq3_df is not None else "—")

    _divider()

    # Configuração ativa
    cfg_saved = _ss("wizard_cfg") or {}
    if not cfg_saved:
        _warn_block("Configuração não salva. Acesse <strong>Mapeamento</strong> e clique em Salvar.")
    else:
        id_rota = cfg_saved.get("id_rota_sci_col", "(não definida)")
        _info_block(
            f"Configuração ativa &nbsp;·&nbsp; "
            f"Central Science: <strong>{cfg_saved.get('central_sci_col','auto')}</strong> &nbsp;·&nbsp; "
            f"ID Rota: <strong>{id_rota}</strong> &nbsp;·&nbsp; "
            f"Central Portal: <strong>{cfg_saved.get('central_portal_col','CENTRAL')}</strong>"
        )
        if id_rota in ("(não definida)", "(nenhuma)", ""):
            _warn_block("ID Rota não configurada — Etapa 3 do Science será ignorada. "
                        "Configure em <strong>Mapeamento</strong>.")

    # Ações
    col_btn, col_clear = st.columns([4, 1])
    with col_clear:
        if st.button("Limpar cache", key="btn_clear_cache"):
            st.session_state.pop("merged_df", None)
            st.session_state.pop("merge_report", None)
            st.session_state.pop("wizard_cfg", None)
            st.rerun()
    with col_btn:
        run = st.button("Executar Pipeline", type="primary", key="btn_merge",
                        use_container_width=True)

    if run:
        with st.spinner("Executando pipeline..."):
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
                _ok_block(f"{len(merged):,} linhas geradas com sucesso.")
            except Exception as e:
                _err_block(f"Erro na execução do pipeline: {e}")
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
            _info_block("Exibindo última versão salva no banco de dados.")
            st.session_state["merged_df"] = merged_df

    if merged_df is None or merged_df.empty:
        return

    report = _ss("merge_report") or {}

    # ── Relatório do pipeline ─────────────────────────────────────────────
    _divider()
    with st.expander("Relatório do Pipeline", expanded=True):

        st.markdown(
            '<div style="font-size:11px;font-weight:700;letter-spacing:0.08em;'
            'text-transform:uppercase;color:#6B7280;margin-bottom:8px;">Science — Etapas 1 a 4</div>',
            unsafe_allow_html=True,
        )
        s1, s2, s3, s4, s5 = st.columns(5)
        s1.metric("Total inicial",       report.get("total_inicial_science", 0))
        s2.metric("Filtro Central",      report.get("total_apos_filtro_central", 0),
                  delta=report.get("total_apos_filtro_central", 0)
                       - report.get("total_inicial_science", 0))
        s3.metric("Filtro Data",         report.get("total_apos_filtro_data", 0),
                  delta=report.get("total_apos_filtro_data", 0)
                       - report.get("total_apos_filtro_central", 0))
        s4.metric("Validação Rotas",     report.get("total_apos_validacao_rotas", 0),
                  delta=report.get("total_apos_validacao_rotas", 0)
                       - report.get("total_apos_filtro_data", 0))
        s5.metric("Enriquecido Arq3",    report.get("total_enriquecido_arq3", 0))

        st.markdown(
            '<div style="font-size:11px;font-weight:700;letter-spacing:0.08em;'
            'text-transform:uppercase;color:#6B7280;margin:16px 0 8px;">Portal — Etapas 5 a 9</div>',
            unsafe_allow_html=True,
        )
        p1, p2, p3 = st.columns(3)
        p1.metric("Total processado",    report.get("total_portal_processado", 0))
        p2.metric("Encontrado no Arq3",  report.get("total_encontrado_arq3_portal", 0))
        p3.metric("Novas rotas",         report.get("total_novas_rotas_criadas", 0),
                  delta=report.get("total_novas_rotas_criadas", 0),
                  delta_color="inverse" if report.get("total_novas_rotas_criadas", 0) > 0 else "off")

        st.markdown(
            '<div style="font-size:11px;font-weight:700;letter-spacing:0.08em;'
            'text-transform:uppercase;color:#6B7280;margin:16px 0 8px;">Arq3 + Resultado Final</div>',
            unsafe_allow_html=True,
        )
        g1, g2, g3, g4, g5, g6 = st.columns(6)
        g1.metric("Arq3 sem match",      report.get("total_arq3_sem_match", 0))
        g2.metric("Total linhas",        report.get("total_resultado_final", len(merged_df)))
        g3.metric("UF em branco",        report.get("uf_missing", 0),        delta_color="inverse")
        g4.metric("CLUSTER em branco",   report.get("cluster_missing", 0),   delta_color="inverse")
        g5.metric("Inconsistências",     report.get("total_inconsistencias", 0), delta_color="inverse")
        g6.metric("Redução Science",     f"{report.get('reducao_science_pct', 0)}%")

        st.markdown(
            '<div style="font-size:11px;font-weight:700;letter-spacing:0.08em;'
            'text-transform:uppercase;color:#6B7280;margin:16px 0 8px;">Validações Automáticas — R1 · R2 · R3</div>',
            unsafe_allow_html=True,
        )
        q1, q2, q3, q4 = st.columns(4)
        q1.metric("Correções REDE (R1)",   report.get("correcoes_rede", 0),
                  delta_color="inverse" if report.get("correcoes_rede", 0) > 0 else "off",
                  help="Linhas onde REDE foi corrigida conforme origem (imutável)")
        q2.metric("Tipo de Rota (R2)",     report.get("correcoes_tipo_rota", 0),
                  help="Linhas com Tipo de Rota preenchido com ITX-SIP_AS")
        q3.metric("UF via CNL_PPI (R3)",   report.get("uf_preenchido_r3", 0),
                  help="Registros Portal com UF derivada via CNL_PPI")
        q4.metric("UF não derivada (R3)",  report.get("uf_invalido_r3", 0),
                  delta_color="inverse" if report.get("uf_invalido_r3", 0) > 0 else "off",
                  help="CNL_PPI inválido ou CN não encontrado na tabela")

        if report.get("uf_divergente_r3", 0) > 0:
            _warn_block(
                f"<strong>{report['uf_divergente_r3']} registro(s)</strong> com UF divergente entre "
                f"Arq3 e CNL_PPI — o valor do CNL_PPI foi aplicado (fonte primária). "
                f"Veja detalhes nas inconsistências abaixo."
            )

        # Log de correções automáticas
        quality_logs = report.get("_quality_logs", [])
        if quality_logs:
            with st.expander(f"Log de Correções Automáticas — {len(quality_logs)} entradas"):
                df_ql = pd.DataFrame(quality_logs)
                cols_show = [c for c in ["regra","rotulo","antes","depois","mensagem"]
                             if c in df_ql.columns]
                st.dataframe(df_ql[cols_show], use_container_width=True, hide_index=True)

        # Inconsistências
        all_incons = report.get("_inconsistencias", [])
        if all_incons:
            st.markdown(
                f'<div style="font-size:13px;font-weight:700;color:#1C2536;margin:14px 0 6px;">'
                f'Inconsistências Detalhadas — {len(all_incons)} ocorrências</div>',
                unsafe_allow_html=True,
            )
            df_incons = pd.DataFrame(all_incons)
            for etapa in df_incons["etapa"].unique():
                grupo = df_incons[df_incons["etapa"] == etapa]
                with st.expander(f"{etapa} — {len(grupo)} ocorrência(s)"):
                    st.dataframe(grupo[["chave","motivo"]],
                                 use_container_width=True, hide_index=True)

    # ── Grade de resultados ───────────────────────────────────────────────
    _divider()
    display_cols = [c for c in merged_df.columns if not c.startswith("_")]
    filtered_df = show_grid(
        merged_df[display_cols],
        key="merge_grid",
        title="Resultado Final",
        height=600,
    )

    # ── Exportação ────────────────────────────────────────────────────────
    _divider()
    _section("Salvar e Exportar")

    ex1, ex2, ex3, ex4 = st.columns(4)
    with ex1:
        if st.button("Salvar no banco de dados", key="btn_save", use_container_width=True):
            _save_to_db(merged_df)
    with ex2:
        st.download_button("Exportar CSV",
                           to_csv_bytes(filtered_df),
                           f"resultado_{version_tag()}.csv",
                           "text/csv",
                           key="btn_csv",
                           use_container_width=True)
    with ex3:
        try:
            st.download_button("Exportar XLSX",
                               to_xlsx_bytes(filtered_df),
                               f"resultado_{version_tag()}.xlsx",
                               "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                               key="btn_xlsx",
                               use_container_width=True)
        except Exception as e:
            _err_block(f"Erro ao gerar XLSX: {e}")
    with ex4:
        st.download_button("Relatório JSON",
                           to_mapping_json({k: v for k, v in report.items()
                                            if not k.startswith("_")}),
                           f"report_{version_tag()}.json",
                           "application/json",
                           key="btn_json",
                           use_container_width=True)

    if report.get("_inconsistencias"):
        _divider()
        st.download_button("Exportar Inconsistências CSV",
                           to_csv_bytes(pd.DataFrame(report["_inconsistencias"])),
                           f"inconsistencias_{version_tag()}.csv",
                           "text/csv",
                           key="btn_incons")


def _save_to_db(merged_df: pd.DataFrame) -> None:
    cfg = _ss("wizard_cfg") or {}
    try:
        with session_scope() as s:
            repo = Repository(s)

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

        _ok_block(f"Resultado salvo com sucesso. ID da versão: <code>{vid[:8]}...</code>")
    except Exception as e:
        _err_block(f"Erro ao salvar: {e}")
        log.error("DB save: %s", e, exc_info=True)


# ── Página 4: Seeds ───────────────────────────────────────────────────────────

def render_seeds_page() -> None:
    with session_scope() as s:
        repo  = Repository(s)
        cnl_n = repo.cnl_count()
        uf_n  = repo.cn_to_uf_count()

    st.markdown(
        f'<div style="display:flex;gap:32px;padding:12px 16px;background:#F8F9FA;'
        f'border:1px solid #E9ECEF;font-size:12px;margin-bottom:16px;">'
        f'<span><span style="font-size:10px;font-weight:700;letter-spacing:.07em;'
        f'text-transform:uppercase;color:#6B7280;">CNL</span>&nbsp;&nbsp;'
        f'<strong>{cnl_n:,}</strong> registros</span>'
        f'<span><span style="font-size:10px;font-weight:700;letter-spacing:.07em;'
        f'text-transform:uppercase;color:#6B7280;">CN → UF</span>&nbsp;&nbsp;'
        f'<strong>{uf_n}</strong> mapeamentos</span>'
        f'</div>',
        unsafe_allow_html=True,
    )

    import os
    from pathlib import Path

    _section("Seeds automáticos")
    seed_dir = Path("seeds")
    c1, c2 = st.columns(2)

    with c1:
        st.markdown('<div style="font-size:11px;color:#6B7280;margin-bottom:6px;">seeds/cnl.sql</div>',
                    unsafe_allow_html=True)
        cnl_sql = seed_dir / "cnl.sql"
        if cnl_sql.exists():
            if st.button("Carregar CNL SQL", key="btn_cnl_seed", use_container_width=True):
                with session_scope() as s:
                    n = Repository(s).load_cnl_seeds(str(cnl_sql))
                _ok_block(f"{n} statements CNL executados.")
        else:
            _warn_block("seeds/cnl.sql não encontrado.")

    with c2:
        st.markdown('<div style="font-size:11px;color:#6B7280;margin-bottom:6px;">seeds/cn_to_uf.csv</div>',
                    unsafe_allow_html=True)
        uf_csv = seed_dir / "cn_to_uf.csv"
        if uf_csv.exists():
            if st.button("Carregar CN → UF", key="btn_uf_seed", use_container_width=True):
                with session_scope() as s:
                    n = Repository(s).load_cn_to_uf_csv(str(uf_csv))
                _ok_block(f"{n} mapeamentos CN→UF carregados.")
        else:
            _warn_block("seeds/cn_to_uf.csv não encontrado.")

    _divider()
    _section("Upload manual")
    uf_file = st.file_uploader("Arquivo cn_to_uf.csv personalizado",
                                type=["csv"], key="up_uf_csv")
    if uf_file:
        import io, tempfile
        tmp = tempfile.NamedTemporaryFile(suffix=".csv", delete=False)
        tmp.write(uf_file.read()); tmp.flush()
        try:
            with session_scope() as s:
                n = Repository(s).load_cn_to_uf_csv(tmp.name)
            _ok_block(f"{n} mapeamentos carregados.")
        finally:
            os.unlink(tmp.name)

    _divider()
    _section("Testar lookup de UF")
    test_cnl = st.text_input("COD_CNL para teste:", key="test_cnl",
                              placeholder="Ex.: 11ABCD")
    if test_cnl:
        with session_scope() as s:
            uf = Repository(s).get_uf_for_cnl(test_cnl)
        if uf:
            _ok_block(f"CNL <code>{test_cnl}</code> → UF: <strong>{uf}</strong>")
        else:
            _warn_block(f"CNL <code>{test_cnl}</code> não encontrado nas tabelas de referência.")


# ── Página 5: Histórico ───────────────────────────────────────────────────────

def render_history_page() -> None:
    with session_scope() as s:
        repo     = Repository(s)
        versions = repo.list_versions()
        imports  = repo.list_imports()

    if not versions:
        _info_block("Nenhum resultado salvo ainda.")
        return

    df_v = pd.DataFrame(versions)
    st.dataframe(df_v, use_container_width=True, hide_index=True)

    _divider()
    sel = st.selectbox(
        "Selecionar versão:",
        [v["id"] for v in versions],
        format_func=lambda x: next(
            (f"{v['tag']} — {v['rows_merged']:,} linhas" for v in versions if v["id"] == x), x),
        key="hist_sel",
    )
    if st.button("Carregar versão selecionada", key="btn_hist_load", type="primary"):
        with session_scope() as s2:
            loaded = Repository(s2).load_merged_df(sel)
        st.session_state["merged_df"] = loaded
        _ok_block(f"{len(loaded):,} linhas carregadas.")
        show_grid(
            loaded[[c for c in loaded.columns if not c.startswith("_")]],
            key="hist_grid",
            title="Versão carregada",
        )

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
        _info_block("Nenhum resultado gerado ainda.")
        return

    q = quality_summary(merged)
    c1, c2, c3, c4 = st.columns(4)
    c1.metric("Total linhas",       q.get("total", 0))
    c2.metric("UF em branco",       q.get("UF_missing", 0))
    c3.metric("CLUSTER em branco",  q.get("CLUSTER_missing", 0))
    c4.metric("Sem match Arq3",     q.get("arq3_no_match", 0))

    _divider()

    uf_pending = merged[merged["UF"].eq("") | merged["UF"].isna()]
    if not uf_pending.empty:
        with st.expander(f"Linhas sem UF — {len(uf_pending)} registros"):
            display = [c for c in uf_pending.columns if not c.startswith("_")]
            st.dataframe(uf_pending[display].head(50),
                         use_container_width=True, hide_index=True)

    _divider()
    for df, name in [(_ss("sci_df"), "Science"), (_ss("por_df"), "Portal"), (merged, "Resultado")]:
        if df is not None:
            with st.expander(f"Relatório de colunas — {name}"):
                rep = generate_report(df, name)
                st.dataframe(pd.DataFrame(rep["colunas"]),
                             use_container_width=True, hide_index=True)


# ── Página 7: Logs ────────────────────────────────────────────────────────────

def render_logs_page() -> None:
    with session_scope() as s:
        logs = Repository(s).get_logs(200)

    if not logs:
        _info_block("Nenhum log registrado.")
        return

    df_l = pd.DataFrame(logs)[["timestamp","level","message"]]

    # Filtro de nível
    lvl_filter = st.multiselect(
        "Nível:",
        ["INFO","WARNING","ERROR","DEBUG"],
        default=["INFO","WARNING","ERROR"],
        key="log_lvl",
    )

    df_filtered = df_l[df_l["level"].isin(lvl_filter)]
    st.dataframe(df_filtered, use_container_width=True, hide_index=True)

    _divider()
    st.download_button(
        "Exportar log",
        logs_to_text(logs),
        f"logs_{version_tag()}.txt",
        key="btn_log_dl",
    )


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
        _warn_block("Carregue o Arquivo 3 para executar o diagnóstico.")
        return

    # ── 1. Colunas do Arquivo 3 ───────────────────────────────────────────
    _section("Colunas detectadas no Arquivo 3", "Etapa 1")
    arq3_cols    = _extract_arq3_cols(arq3_df, cfg)
    rot_col      = arq3_cols.get("rotulos", "")
    rotulo_index = _build_arq3_rotulo_index(arq3_df, rot_col)

    d1, d2, d3 = st.columns(3)
    d1.metric("Rótulos únicos",     len(rotulo_index))
    d2.metric("Linhas no Arquivo 3",len(arq3_df))
    d3.metric("Colunas detectadas", sum(1 for v in arq3_cols.values() if v))

    field_map = {
        "REDE":             arq3_cols.get("rede", ""),
        "Central":          arq3_cols.get("central", ""),
        "UF":               arq3_cols.get("uf", ""),
        "CLUSTER":          arq3_cols.get("cluster", ""),
        "Tipo de Rota":     arq3_cols.get("tipo", ""),
        "Rótulos de Linha": arq3_cols.get("rotulos", ""),
        "OPERADORA":        arq3_cols.get("operadora", ""),
        "Denominação":      arq3_cols.get("den", ""),
    }

    for field, col in field_map.items():
        if col and col in arq3_df.columns:
            samples = (arq3_df[col].dropna().astype(str).str.strip()
                      .pipe(lambda s: s[s.str.len() > 0]).unique()[:5])
            st.markdown(
                f'<div style="display:flex;align-items:baseline;gap:10px;padding:4px 0;'
                f'border-bottom:1px solid #F3F4F6;font-size:12px;">'
                f'<span style="color:#16A34A;font-weight:700;font-size:11px;">+</span>'
                f'<span style="font-weight:600;color:#1C2536;min-width:140px;">{field}</span>'
                f'<code style="font-size:11px;background:#F3F4F6;padding:1px 6px;">{col}</code>'
                f'<span style="color:#6B7280;font-size:11px;">{" · ".join(samples)}</span>'
                f'</div>',
                unsafe_allow_html=True,
            )
        else:
            st.markdown(
                f'<div style="display:flex;align-items:baseline;gap:10px;padding:4px 0;'
                f'border-bottom:1px solid #F3F4F6;font-size:12px;">'
                f'<span style="color:#DC2626;font-weight:700;font-size:11px;">—</span>'
                f'<span style="font-weight:600;color:#9CA3AF;min-width:140px;">{field}</span>'
                f'<span style="color:#DC2626;font-size:11px;">Não detectada — configure em Mapeamento</span>'
                f'</div>',
                unsafe_allow_html=True,
            )

    with st.expander(f"Todas as colunas do Arquivo 3 — {len(arq3_df.columns)} colunas"):
        for i, c in enumerate(arq3_df.columns):
            sample = arq3_df[c].dropna().iloc[0] if not arq3_df[c].dropna().empty else ""
            st.text(f"{i+1:3d}. {c!r:40s}  ex: {str(sample)[:60]}")

    # ── 2. Diagnóstico Science ────────────────────────────────────────────
    if sci_df is not None:
        _divider()
        _section("Diagnóstico Science — Etapa 3 (amostra 20 linhas)", "Etapa 2")

        col_cen    = _find_col(list(sci_df.columns), cfg.get("central_sci_col",""),
                               "Central","CENTRAL","Central Interna","Central Origem")
        col_id_rot = _find_col(list(sci_df.columns), cfg.get("id_rota_sci_col",""),
                               "ID Rota","ID_ROTA","IDROTA","Id Rota")

        _info_block(
            f"Coluna Central: <strong>{col_cen or 'não encontrada'}</strong> &nbsp;·&nbsp; "
            f"Coluna ID Rota: <strong>{col_id_rot or 'não encontrada'}</strong>"
        )

        matches, no_matches = [], []
        for _, row in sci_df.head(20).iterrows():
            d = {c: str(row.get(c,"")).strip() for c in sci_df.columns}
            central_val = d.get(col_cen,"").upper()   if col_cen    else ""
            id_rota_val = d.get(col_id_rot,"").upper() if col_id_rot else ""
            chave = (central_val[:7]+"_"+id_rota_val) if central_val and id_rota_val else ""
            found = rotulo_index.get(chave)
            if found:
                matches.append({"Central": central_val, "ID Rota": id_rota_val, "Chave": chave,
                                 "CLUSTER": found.get(arq3_cols.get("cluster",""),""),
                                 "UF": found.get(arq3_cols.get("uf",""),"")})
            else:
                no_matches.append({"Central": central_val, "ID Rota": id_rota_val,
                                   "Chave tentada": chave or "(sem chave)"})

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
                    st.markdown('<div style="font-size:11px;font-weight:600;color:#1C2536;'
                                'margin:8px 0 4px;">Exemplos de rótulos no Arquivo 3:</div>',
                                unsafe_allow_html=True)
                    st.write(sorted(list(rotulo_index.keys()))[:20])

    # ── 3. Diagnóstico Portal ─────────────────────────────────────────────
    if por_df is not None:
        _divider()
        _section("Diagnóstico Portal — Etapas 5 a 8 (amostra 20 linhas)", "Etapa 3")

        col_cen_p = _find_col(list(por_df.columns), cfg.get("central_portal_col",""), "CENTRAL")
        col_le    = _find_col(list(por_df.columns), cfg.get("label_e_col",""), "LABEL_E")
        col_ls    = _find_col(list(por_df.columns), cfg.get("label_s_col",""), "LABEL_S")

        _info_block(
            f"CENTRAL: <strong>{col_cen_p or 'não encontrada'}</strong> &nbsp;·&nbsp; "
            f"LABEL_E: <strong>{col_le or 'não encontrada'}</strong> &nbsp;·&nbsp; "
            f"LABEL_S: <strong>{col_ls or 'não encontrada'}</strong>"
        )

        centrais_portal   = por_df[col_cen_p].dropna().unique() if col_cen_p else []
        centrais_sem_mapa = [c for c in centrais_portal
                             if str(c).strip().upper() not in set(CENTRAL_PORTAL_MAP.keys())]
        if centrais_sem_mapa:
            _warn_block(
                f"{len(centrais_sem_mapa)} valores de CENTRAL sem mapeamento: "
                f"{centrais_sem_mapa[:5]}"
            )

        matches_p, no_matches_p = [], []
        for _, row in por_df.head(20).iterrows():
            d        = {c: str(row.get(c,"")).strip() for c in por_df.columns}
            cen_raw  = d.get(col_cen_p,"").upper() if col_cen_p else ""
            cen_conv = CENTRAL_PORTAL_MAP.get(cen_raw, cen_raw)
            le_val   = d.get(col_le,"") if col_le else ""
            ls_val   = d.get(col_ls,"") if col_ls else ""
            label    = le_val if len(le_val) > 2 else ls_val if len(ls_val) > 2 else le_val
            rotulo   = (cen_conv[:7]+"_"+label[:6]).upper() if cen_conv and label else ""
            found    = rotulo_index.get(rotulo)
            row_data = {"CENTRAL": cen_raw, "Conv.": cen_conv, "LABEL": label, "Rótulo": rotulo,
                        "CLUSTER": found.get(arq3_cols.get("cluster",""),"") if found else "NOVA ROTA",
                        "UF": found.get(arq3_cols.get("uf",""),"") if found else "?"}
            matches_p.append(row_data)
            if not found:
                no_matches_p.append({"CENTRAL": cen_raw, "Rótulo tentado": rotulo})

        pc1, pc2 = st.columns(2)
        pc1.metric("Encontrado no Arquivo 3", len(matches_p) - len(no_matches_p))
        pc2.metric("Nova rota",               len(no_matches_p))

        with st.expander("Amostra completa — 20 linhas"):
            st.dataframe(pd.DataFrame(matches_p), use_container_width=True, hide_index=True)

    # ── 4. Mapa CENTRAL Portal ────────────────────────────────────────────
    _divider()
    _section("Mapa de Conversão CENTRAL — Portal", "Etapa 4")
    df_mapa = pd.DataFrame(
        [{"CENTRAL (Portal)": k, "CENTRAL (Convertida)": v}
         for k, v in CENTRAL_PORTAL_MAP.items()]
    )
    st.dataframe(df_mapa, use_container_width=True, hide_index=True)

    # ── 5. Tabela CN → UF ─────────────────────────────────────────────────
    _divider()
    with st.expander(f"Tabela CN → UF — {len(CN_TO_UF)} DDDs"):
        df_cn = pd.DataFrame(
            [{"CN": k, "UF": v} for k, v in sorted(CN_TO_UF.items(), key=lambda x: int(x[0]))]
        )
        st.dataframe(df_cn, use_container_width=True, hide_index=True)
