"""
app.core — lógica de negócio.
Expõe as funções e constantes principais de cada submódulo.
"""
from app.core.map_rules import (
    DEFAULT_MAPPING,
    OUTPUT_COLUMNS,
    apply_rule,
    suggest_mappings,
    validate_mapping,
)
from app.core.merge import build_merged_df, count_join_matches
from app.core.normalize import (
    apply_column_normalization,
    infer_and_coerce_types,
    strip_whitespace,
)
from app.core.validate import check_duplicates, generate_report, validate_output

__all__ = [
    # map_rules
    "OUTPUT_COLUMNS",
    "DEFAULT_MAPPING",
    "suggest_mappings",
    "apply_rule",
    "validate_mapping",
    # merge
    "build_merged_df",
    "count_join_matches",
    # normalize
    "apply_column_normalization",
    "strip_whitespace",
    "infer_and_coerce_types",
    # validate
    "generate_report",
    "check_duplicates",
    "validate_output",
]
