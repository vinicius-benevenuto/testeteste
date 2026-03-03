"""
ui/mapping_wizard.py
Assistente de Mapeamento: configura todas as colunas necessárias para o merge.
"""
from __future__ import annotations
from typing import Dict, List

import streamlit as st


def run_mapping_wizard(
    science_cols: List[str],
    portal_cols: List[str],
    arq3_cols: List[str],
) -> Dict:
    st.subheader("🗺️ Assistente de Mapeamento")
    st.markdown("Configure como cada coluna de saída será preenchida. "
                "As sugestões foram geradas automaticamente.")

    cfg: Dict = {}
    sci  = science_cols or []
    por  = portal_cols  or []
    arq3 = arq3_cols    or []

    def _sel(label, options, default, key, help=""):
        opts = list(options)
        idx  = opts.index(default) if default in opts else 0
        return st.selectbox(label, opts, index=idx, key=key, help=help)

    def _guess(cols, *candidates):
        up = {c.strip().upper(): c for c in cols}
        for cand in candidates:
            hit = up.get(cand.strip().upper())
            if hit:
                return hit
        return cols[0] if cols else ""

    NONE = "(nenhuma)"

    # ── 1. Coluna CN/DDD no Science ────────────────────────────────────
    with st.expander("1️⃣ Colunas para derivar UF", expanded=True):
        st.caption(
            "O app deriva UF em cascata: **Arquivo 3** → CN/DDD direto → CNL/PPI. "
            "Configure as colunas abaixo."
        )
        st.markdown("**A — Arquivo 3** (fonte principal — já funciona via Central)")
        st.info("✅ UF e CLUSTER são lidos automaticamente do Arquivo 3 pela coluna Central. "
                "Configure as colunas do Arquivo 3 na seção 8️⃣ abaixo.")

        st.markdown("**B — CN/DDD direto no Science** (fallback)")
        cfg["cn_sci_col"] = _sel(
            "Coluna CN/DDD no Science",
            [NONE] + sci,
            _guess(sci, "Área Ponta B", "Area Ponta B", "CN", "DDD", "AREA_PONTA_B"),
            "wiz_cn_sci",
            "Coluna com DDD numérico (ex: 43, 11). Mapeia direto para UF.",
        )

        st.markdown("**C — CNL/PPI** (fallback)")
        cfg["cnl_sci_col"] = _sel(
            "Coluna CNL no Science", [NONE] + sci,
            _guess(sci, "CNL", "CNL_PPI", "Num SSI"),
            "wiz_cnl_sci",
        )
        cfg["cnl_por_col"] = _sel(
            "Coluna CNL no Portal", [NONE] + por,
            _guess(por, "CNL_PPI", "PPI", "CNL"),
            "wiz_cnl_por",
        )

    # ── 2. Tipo de Rota ────────────────────────────────────────────────
    with st.expander("2️⃣ Tipo de Rota", expanded=True):
        cfg["tipo_rota_portal_col"] = _sel(
            "Tipo de Rota — Portal (prioritário)", [NONE] + por,
            _guess(por, "TIPO_ROTA", "TIPO"),
            "wiz_tr_por",
        )
        cfg["tipo_rota_sci_col"] = _sel(
            "Tipo de Rota — Science (fallback)", [NONE] + sci,
            _guess(sci, "Sinalização da Rota", "Tipo da Rota", "Tipo"),
            "wiz_tr_sci",
        )

    # ── 3. Central ─────────────────────────────────────────────────────
    with st.expander("3️⃣ Central", expanded=True):
        cfg["central_portal_col"] = _sel(
            "Central — Portal (prioritário)", [NONE] + por,
            _guess(por, "CENTRAL"),
            "wiz_ce_por",
        )
        cfg["central_sci_col"] = _sel(
            "Central — Science (fallback)", [NONE] + sci,
            _guess(sci, "Central Origem", "Central Interna"),
            "wiz_ce_sci",
        )

    # ── 4. Rótulos de Linha ────────────────────────────────────────────
    with st.expander("4️⃣ Rótulos de Linha (Portal)", expanded=False):
        cfg["label_e_col"] = _sel(
            "LABEL_E (entrada)", [NONE] + por,
            _guess(por, "LABEL_E"),
            "wiz_le",
        )
        cfg["label_s_col"] = _sel(
            "LABEL_S (saída)", [NONE] + por,
            _guess(por, "LABEL_S"),
            "wiz_ls",
        )
        cfg["concat_labels"] = st.checkbox(
            "Concatenar LABEL_E | LABEL_S",
            value=True, key="wiz_concat_labels",
        )
        cfg["label_sep"] = st.text_input(
            "Separador", value=" | ", key="wiz_label_sep",
        ) if cfg["concat_labels"] else " | "

    # ── 5. OPERADORA ──────────────────────────────────────────────────
    with st.expander("5️⃣ OPERADORA — fallback Science", expanded=False):
        cfg["operadora_sci_col"] = _sel(
            "Operadora Science", [NONE] + sci,
            _guess(sci, "Operadora Origem", "OP Origem"),
            "wiz_op_sci",
        )

    # ── 6. Denominação ────────────────────────────────────────────────
    with st.expander("6️⃣ Denominação — fallback Science", expanded=False):
        cfg["denominacao_sci_col"] = _sel(
            "Denominação Science", [NONE] + sci,
            _guess(sci, "Descrição", "Descricao"),
            "wiz_dn_sci",
        )

    # ── 7. Chaves de junção ───────────────────────────────────────────
    st.markdown("---")
    st.markdown("### 7️⃣ Chaves de Junção Science ↔ Portal")
    st.caption("Colunas que identificam a mesma rota nas duas tabelas. "
               "O app normaliza para UPPER antes de comparar.")

    st.info(
        "ℹ️ **Science = VIVO-SMP | Portal = VIVO-STFC.** "
        "Por padrão as bases são **concatenadas** — cada linha mantém sua identidade. "
        "Configure chaves abaixo **somente** se quiser unir rotas que existam nas duas bases."
    )
    join_sci = st.multiselect(
        "Colunas do Science para junção (deixe vazio para concatenar):",
        options=sci,
        default=[],
        key="wiz_join_sci",
    )
    join_por = st.multiselect(
        "Colunas correspondentes no Portal (mesma ordem):",
        options=por,
        default=[],
        key="wiz_join_por",
    )

    if len(join_sci) != len(join_por):
        st.warning("⚠️ Selecione o mesmo número de colunas em Science e Portal.")

    cfg["join_keys_sci"] = join_sci
    cfg["join_keys_por"] = join_por

    join_type = st.radio(
        "Tipo de junção:",
        options=["outer", "inner", "left", "right"],
        format_func=lambda v: {
            "outer": "OUTER — mantém todas as linhas (recomendado)",
            "left":  "LEFT — mantém todas do Science",
            "right": "RIGHT — mantém todas do Portal",
            "inner": "INNER — apenas matches",
        }.get(v, v),
        key="wiz_join_type",
    )
    cfg["join_type"] = join_type

    # ── 8. Arquivo 3 — colunas explícitas ─────────────────────────────
    if arq3:
        st.markdown("---")
        st.markdown("### 8️⃣ Arquivo 3 — Mapeamento de Colunas")
        st.caption(
            "⚠️ **Configure aqui as colunas do Arquivo 3.** "
            "O CLUSTER e a UF só serão preenchidos corretamente se estas colunas estiverem certas."
        )
        with st.expander("🗂️ Colunas do Arquivo 3", expanded=True):
            col1, col2 = st.columns(2)
            with col1:
                cfg["arq3_central_col"] = _sel(
                    "Central",
                    [NONE] + arq3,
                    _guess(arq3, "Central", "CENTRAL", "Central Origem"),
                    "wiz_arq3_central",
                    "Coluna que contém o nome da Central no Arquivo 3",
                )
                cfg["arq3_uf_col"] = _sel(
                    "UF",
                    [NONE] + arq3,
                    _guess(arq3, "UF", "uf", "Estado", "ESTADO"),
                    "wiz_arq3_uf",
                    "Coluna que contém a sigla do estado (ex: PR, SP)",
                )
                cfg["arq3_cluster_col"] = _sel(
                    "CLUSTER ⭐",
                    [NONE] + arq3,
                    _guess(arq3, "CLUSTER", "Cluster", "cluster",
                           "AGRUPAMENTO", "Agrupamento", "CLUSTER_NOME"),
                    "wiz_arq3_cluster",
                    "Coluna que contém o identificador de Cluster — obrigatória para preencher CLUSTER",
                )
            with col2:
                cfg["arq3_tipo_rota_col"] = _sel(
                    "Tipo de Rota",
                    [NONE] + arq3,
                    _guess(arq3, "Tipo de Rota", "TIPO DE ROTA", "TIPO_ROTA"),
                    "wiz_arq3_tr",
                )
                cfg["arq3_rotulos_col"] = _sel(
                    "Rótulos de Linha",
                    [NONE] + arq3,
                    _guess(arq3, "Rótulos de Linha", "ROTULOS DE LINHA",
                           "ROTULOS_DE_LINHA", "LABEL_E", "Rótulo"),
                    "wiz_arq3_rotulos",
                )
                cfg["arq3_operadora_col"] = _sel(
                    "OPERADORA",
                    [NONE] + arq3,
                    _guess(arq3, "OPERADORA", "Operadora"),
                    "wiz_arq3_op",
                )
                cfg["arq3_denominacao_col"] = _sel(
                    "Denominação",
                    [NONE] + arq3,
                    _guess(arq3, "Denominação", "DENOMINAÇÃO", "Denominacao"),
                    "wiz_arq3_den",
                )

        # Mostra colunas do Arquivo 3 para referência
        with st.expander("📋 Todas as colunas do Arquivo 3 (para consulta)", expanded=False):
            for i, c in enumerate(arq3):
                st.text(f"{i+1:3d}. {c}")

    return cfg
