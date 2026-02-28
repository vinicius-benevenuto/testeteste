"""core — lógica de negócio."""
from app.core.map_rules import OUTPUT_COLUMNS, derive_rede, build_ref_index, lookup_ref, build_output_row
from app.core.merge import build_merged_df, count_join_matches
from app.core.normalize import apply_column_normalization, strip_whitespace
from app.core.validate import generate_report, quality_summary
__all__ = ["OUTPUT_COLUMNS","derive_rede","build_ref_index","lookup_ref","build_output_row",
           "build_merged_df","count_join_matches","apply_column_normalization",
           "strip_whitespace","generate_report","quality_summary"]
