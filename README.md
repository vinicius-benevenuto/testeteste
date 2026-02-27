"""
app.ui — interface Streamlit.
Expõe as funções de renderização de cada página.
"""
from app.ui.pages import (
    render_history_page,
    render_logs_page,
    render_mapping_page,
    render_merge_page,
    render_upload_page,
    render_validation_page,
)

__all__ = [
    "render_upload_page",
    "render_mapping_page",
    "render_merge_page",
    "render_validation_page",
    "render_history_page",
    "render_logs_page",
]
