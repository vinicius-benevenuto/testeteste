"""
ui/mapping_wizard.py
Assistente de Mapeamento passo a passo com Streamlit.
Permite configurar regras para cada coluna de saída.
"""
from __future__ import annotations
import json
from typing import Dict, List, Optional, Tuple

import pandas as pd
import streamlit as st

from app.core.map_rules import (
    DEFAULT_MAPPING, OUTPUT_COLUMNS, suggest_mappings, validate_mapping,
)


_RULE_TYPES = {
    "coalesce":  "COALESCE (primeiro não vazio)",
    "concat":    "Concatenar colunas",
    "literal":   "Valor fixo / constante",
    "reference": "Tabela de referência (upload)",
}


def _source_editor(prefix: str, sources: List[Dict],
                    science_cols: List[str], portal_cols: List[str]) -> List[Dict]:
    """Editor de lista de fontes (table + col) para regras coalesce/concat."""
    updated: List[Dict] = []
    for i, src in enumerate(sources):
        c1, c2, c3 = st.columns([2, 3, 1])
        with c1:
            table = st.selectbox(
                f"Tabela #{i+1}", ["science", "portal"],
                index=0 if src.get("table") == "science" else 1,
                key=f"{prefix}_tbl_{i}",
            )
        with c2:
            opts = science_cols if table == "science" else portal_cols
            col_val = src.get("col", "")
            idx = opts.index(col_val) if col_val in opts else 0
            col = st.selectbox(
                f"Coluna #{i+1}", opts if opts else ["(nenhuma)"],
                index=idx if opts else 0,
                key=f"{prefix}_col_{i}",
            )
        with c3:
            st.write("")
            st.write("")
            if st.button("✕", key=f"{prefix}_del_{i}", help="Remover"):
                continue  # pula esta entrada (remove)
        updated.append({"table": table, "col": col})
    return updated


def run_mapping_wizard(
    science_cols: List[str],
    portal_cols: List[str],
    current_mapping: Optional[Dict] = None,
) -> Tuple[Dict, Dict, List[str], str, int, bool]:
    """
    Renderiza o assistente de mapeamento.
    Retorna:
        (mapping, literal_defaults, join_keys, join_type, fuzzy_threshold, use_fuzzy)
    """
    st.subheader("🗺️ Assistente de Mapeamento de Colunas")
    st.markdown("""
    Configure como cada coluna de saída será preenchida a partir das tabelas de entrada.
    As sugestões automáticas foram geradas com base nos nomes das colunas detectadas.
    """)

    # Auto-sugestão
    auto_mapping = suggest_mappings(science_cols, portal_cols)
    if current_mapping:
        base_mapping = current_mapping
    else:
        base_mapping = auto_mapping

    mapping: Dict = {}
    literal_defaults: Dict = {}

    st.markdown("---")
    st.markdown("### 1️⃣ Regras por coluna de saída")

    for out_col in OUTPUT_COLUMNS:
        with st.expander(f"**{out_col}**", expanded=(out_col in ("Tipo de Rota", "Central", "OPERADORA"))):
            rule = base_mapping.get(out_col, {"type": "literal", "value": ""})
            rtype = st.selectbox(
                "Tipo de regra",
                list(_RULE_TYPES.keys()),
                format_func=lambda k: _RULE_TYPES[k],
                index=list(_RULE_TYPES.keys()).index(rule.get("type", "literal")),
                key=f"wiz_type_{out_col}",
            )

            if rtype == "literal":
                val = st.text_input(
                    "Valor fixo",
                    value=str(rule.get("value", "")),
                    key=f"wiz_val_{out_col}",
                    help="Deixe vazio para campo em branco.",
                )
                mapping[out_col] = {"type": "literal", "value": val}
                literal_defaults[out_col] = val

            elif rtype in ("coalesce", "concat"):
                sources = list(rule.get("sources", [{"table": "science", "col": ""}]))
                st.caption("Fontes (ordem de precedência para COALESCE; ordem de concatenação para CONCAT):")
                sources = _source_editor(f"wiz_src_{out_col}", sources, science_cols, portal_cols)

                # Botão adicionar fonte
                if st.button(f"+ Adicionar fonte para {out_col}", key=f"wiz_add_{out_col}"):
                    sources.append({"table": "science", "col": science_cols[0] if science_cols else ""})

                extra: Dict = {}
                if rtype == "concat":
                    extra["separator"] = st.text_input(
                        "Separador", value=rule.get("separator", " | "),
                        key=f"wiz_sep_{out_col}",
                    )
                mapping[out_col] = {"type": rtype, "sources": sources, **extra}

            elif rtype == "reference":
                ref_file = st.file_uploader(
                    "Upload da tabela de referência (CSV/XLSX)",
                    type=["csv", "xlsx"],
                    key=f"wiz_ref_{out_col}",
                )
                join_col = st.text_input(
                    "Coluna-chave na tabela de referência",
                    key=f"wiz_ref_key_{out_col}",
                    help="Ex.: CNL ou Central",
                )
                value_col = st.text_input(
                    f"Coluna com o valor de {out_col}",
                    key=f"wiz_ref_val_{out_col}",
                )
                if ref_file:
                    # Salva referência em session_state para uso posterior
                    st.session_state[f"ref_df_{out_col}"] = ref_file.getvalue()
                    st.success(f"Tabela de referência carregada para {out_col}")
                mapping[out_col] = {
                    "type": "reference",
                    "ref_col": join_col,
                    "value_col": value_col,
                    "target": out_col,
                }

    # Validação
    warnings = validate_mapping(mapping, science_cols, portal_cols)
    if warnings:
        for w in warnings:
            st.warning(w)

    # ── Chaves de junção ──────────────────────────────────────────────────
    st.markdown("---")
    st.markdown("### 2️⃣ Chaves de junção entre Science e Portal")
    st.markdown("""
    Selecione as colunas de saída que serão usadas para cruzar as duas tabelas.
    Recomendado: **Central** e/ou **Tipo de Rota**.
    """)

    key_options = [c for c in OUTPUT_COLUMNS if mapping.get(c, {}).get("type") in ("coalesce", "concat")]
    join_keys = st.multiselect(
        "Colunas-chave de junção",
        options=OUTPUT_COLUMNS,
        default=[k for k in ["Central", "Tipo de Rota"] if k in OUTPUT_COLUMNS],
        key="wiz_join_keys",
        help="As mesmas colunas mapeadas acima serão usadas como chaves de junção.",
    )

    join_type = st.selectbox(
        "Tipo de junção",
        ["outer", "left", "right", "inner"],
        format_func=lambda v: {
            "outer": "OUTER — preserva todas as linhas (recomendado)",
            "left": "LEFT — preserva todas do Science",
            "right": "RIGHT — preserva todas do Portal",
            "inner": "INNER — apenas linhas com match em ambos",
        }.get(v, v),
        key="wiz_join_type",
        help="OUTER é o padrão para não perder linhas.",
    )

    # ── Fuzzy matching ─────────────────────────────────────────────────────
    st.markdown("---")
    st.markdown("### 3️⃣ Fuzzy matching (opcional)")
    use_fuzzy = st.checkbox(
        "Ativar correspondência aproximada (fuzzy)",
        value=False,
        key="wiz_fuzzy",
        help="Útil quando os nomes de Central têm pequenas variações. Requer thefuzz.",
    )
    fuzzy_threshold = 90
    if use_fuzzy:
        fuzzy_threshold = st.slider(
            "Limiar de confiança (%)",
            min_value=50, max_value=100, value=90, step=5,
            key="wiz_fuzzy_thresh",
            help="Valor abaixo desse % é descartado como não-match.",
        )
        st.info("ℹ️ Instale `pip install thefuzz python-levenshtein` para usar esta opção.")

    return mapping, literal_defaults, join_keys, join_type, fuzzy_threshold, use_fuzzy
