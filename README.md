"""
ui/mapping_wizard.py
Assistente de Mapeamento: permite ao usuário confirmar/editar:
  - qual coluna contém CNL em cada tabela
  - chaves de junção
  - colunas de Tipo de Rota, Central, OPERADORA, Rótulos
  - opção concat LABEL_E|LABEL_S
"""
from __future__ import annotations
from typing import Dict, List, Tuple

import streamlit as st


def run_mapping_wizard(
    science_cols: List[str],
    portal_cols: List[str],
    arq3_cols: List[str],
) -> Dict:
    """
    Renderiza o wizard e retorna o dicionário de config com todas as escolhas.
    """
    st.subheader("🗺️ Assistente de Mapeamento")
    st.markdown("""
    Configure como cada coluna de saída será preenchida.
    As sugestões abaixo foram geradas automaticamente.
    """)

    cfg: Dict = {}
    sci = science_cols or []
    por = portal_cols  or []

    def _sel(label, options, default, key, help=""):
        opts = list(options)
        idx = opts.index(default) if default in opts else 0
        return st.selectbox(label, opts, index=idx, key=key, help=help)

    def _guess(cols, *candidates):
        """Encontra a primeira coluna que bate com qualquer candidato."""
        cols_up = {c.strip().upper(): c for c in cols}
        for cand in candidates:
            if cand.upper() in cols_up:
                return cols_up[cand.upper()]
        return cols[0] if cols else ""

    # ── 1. Colunas CNL ─────────────────────────────────────────────────
    with st.expander("1️⃣ Coluna de CNL (para derivar UF)", expanded=True):
        st.caption("O app usa CNL → CN → UF via tabelas de referência.")
        cfg["cnl_sci_col"] = _sel(
            "Coluna CNL no Science", ["(nenhuma)"] + sci,
            _guess(sci, "CNL", "CNL_PPI", "Num SSI"),
            "wiz_cnl_sci",
            "Coluna que contém o código CNL na planilha Science",
        )
        cfg["cnl_por_col"] = _sel(
            "Coluna CNL no Portal", ["(nenhuma)"] + por,
            _guess(por, "CNL_PPI", "PPI", "CNL"),
            "wiz_cnl_por",
            "Coluna que contém o código CNL/PPI no Portal",
        )

    # ── 2. Tipo de Rota ────────────────────────────────────────────────
    with st.expander("2️⃣ Tipo de Rota", expanded=True):
        cfg["tipo_rota_portal_col"] = _sel(
            "Tipo de Rota — Portal (prioritário)", ["(nenhuma)"] + por,
            _guess(por, "TIPO_ROTA", "TIPO"),
            "wiz_tr_por",
        )
        cfg["tipo_rota_sci_col"] = _sel(
            "Tipo de Rota — Science (fallback)", ["(nenhuma)"] + sci,
            _guess(sci, "Sinalização da Rota", "Tipo da Rota", "Tipo"),
            "wiz_tr_sci",
        )

    # ── 3. Central ─────────────────────────────────────────────────────
    with st.expander("3️⃣ Central", expanded=True):
        cfg["central_portal_col"] = _sel(
            "Central — Portal (prioritário)", ["(nenhuma)"] + por,
            _guess(por, "CENTRAL"),
            "wiz_ce_por",
        )
        cfg["central_sci_col"] = _sel(
            "Central — Science (fallback)", ["(nenhuma)"] + sci,
            _guess(sci, "Central Origem", "Central Interna"),
            "wiz_ce_sci",
        )

    # ── 4. Rótulos de Linha ────────────────────────────────────────────
    with st.expander("4️⃣ Rótulos de Linha (Portal)", expanded=True):
        cfg["label_e_col"] = _sel(
            "LABEL_E (entrada)", ["(nenhuma)"] + por,
            _guess(por, "LABEL_E"),
            "wiz_le",
        )
        cfg["label_s_col"] = _sel(
            "LABEL_S (saída)", ["(nenhuma)"] + por,
            _guess(por, "LABEL_S"),
            "wiz_ls",
        )
        cfg["concat_labels"] = st.checkbox(
            "Concatenar LABEL_E | LABEL_S quando ambos existirem",
            value=True, key="wiz_concat_labels",
        )
        cfg["label_sep"] = st.text_input(
            "Separador", value=" | ", key="wiz_label_sep",
        ) if cfg["concat_labels"] else " | "

    # ── 5. OPERADORA ──────────────────────────────────────────────────
    with st.expander("5️⃣ OPERADORA — coluna Science (fallback)", expanded=False):
        cfg["operadora_sci_col"] = _sel(
            "Operadora Science", ["(nenhuma)"] + sci,
            _guess(sci, "Operadora Origem", "OP Origem"),
            "wiz_op_sci",
            "Arquivo 3 → Portal.EMPRESA → esta coluna (precedência fixa)",
        )

    # ── 6. Denominação ────────────────────────────────────────────────
    with st.expander("6️⃣ Denominação — fallback Science", expanded=False):
        cfg["denominacao_sci_col"] = _sel(
            "Denominação Science", ["(nenhuma)"] + sci,
            _guess(sci, "Descrição", "Descricao"),
            "wiz_dn_sci",
        )

    # ── 7. Chaves de junção ───────────────────────────────────────────
    st.markdown("---")
    st.markdown("### 7️⃣ Chaves de Junção Science ↔ Portal")
    st.markdown("""
    Selecione quais colunas identificam a mesma rota nas duas tabelas.
    O app normaliza para UPPER antes de comparar.
    """)

    join_sci = st.multiselect(
        "Colunas do Science para junção:",
        options=sci,
        default=[c for c in [
            _guess(sci, "Central Interna", "Central Origem"),
            _guess(sci, "Tipo da Rota", "Sinalização da Rota"),
        ] if c],
        key="wiz_join_sci",
    )
    join_por = st.multiselect(
        "Colunas correspondentes no Portal (mesma ordem):",
        options=por,
        default=[c for c in [
            _guess(por, "CENTRAL"),
            _guess(por, "TIPO_ROTA"),
        ] if c],
        key="wiz_join_por",
    )

    if len(join_sci) != len(join_por):
        st.error("⚠️ O número de colunas de junção deve ser igual nos dois lados.")

    cfg["join_keys_sci"] = join_sci
    cfg["join_keys_por"] = join_por

    join_type = st.selectbox(
        "Tipo de junção",
        ["outer", "left", "right", "inner"],
        format_func=lambda v: {
            "outer": "OUTER — preserva todas as linhas (recomendado)",
            "left":  "LEFT — preserva todas do Science",
            "right": "RIGHT — preserva todas do Portal",
            "inner": "INNER — apenas matches",
        }.get(v, v),
        key="wiz_join_type",
    )
    cfg["join_type"] = join_type

    # ── 8. Arquivo 3 — chave de match ────────────────────────────────
    if arq3_cols:
        st.markdown("---")
        st.markdown("### 8️⃣ Arquivo 3 — Chave de Match")
        st.caption(
            "O app usa [Central + Tipo de Rota] e [Central + Rótulos de Linha] "
            "para encontrar linhas correspondentes no Arquivo 3. "
            "Isso é automático — não é necessário configurar."
        )

    return cfg
