"""
export/pptx_builder.py
======================
Geração de dashboards em PPTX usando python-pptx nativo (sem dependências externas).
Dois modos:
  - build_column_pptx(df, col_name)  → bytes (4 slides)
  - build_general_pptx(df)           → bytes (9+ slides)
"""
from __future__ import annotations

import io
from datetime import datetime
from typing import Optional

import pandas as pd
from pptx import Presentation
from pptx.chart.data import ChartData
from pptx.dml.color import RGBColor
from pptx.enum.chart import XL_CHART_TYPE
from pptx.enum.text import PP_ALIGN
from pptx.util import Inches, Pt, Emu

# ── Paleta corporativa ─────────────────────────────────────────────────────
_DARK   = RGBColor(0x0F, 0x17, 0x2A)
_ACCENT = RGBColor(0x1B, 0x5F, 0xBF)
_WHITE  = RGBColor(0xFF, 0xFF, 0xFF)
_LIGHT  = RGBColor(0xF8, 0xF9, 0xFA)
_MUTED  = RGBColor(0x6B, 0x72, 0x80)
_OK     = RGBColor(0x16, 0xA3, 0x4A)

_SLIDE_W = Inches(13.33)
_SLIDE_H = Inches(7.5)

_PALETTE = [
    RGBColor(0x1B, 0x5F, 0xBF),
    RGBColor(0x16, 0xA3, 0x4A),
    RGBColor(0xD9, 0x77, 0x06),
    RGBColor(0xDC, 0x26, 0x26),
    RGBColor(0x6D, 0x28, 0xD9),
    RGBColor(0x0E, 0x7A, 0x8B),
]

_TOP_N = 15  # máximo de categorias antes de agrupar em "Outros"


# ── Análise de frequência ──────────────────────────────────────────────────

def _freq(df: pd.DataFrame, col: str, top_n: int = _TOP_N) -> pd.DataFrame:
    """Frequência da coluna, agrupando cauda em 'Outros'."""
    s = df[col].fillna("").astype(str).str.strip()
    s = s.replace("", "(Sem valor)")
    vc = s.value_counts()
    total = len(s)
    if len(vc) > top_n:
        top   = vc.iloc[:top_n]
        other = vc.iloc[top_n:].sum()
        top   = pd.concat([top, pd.Series({"Outros": other})])
        vc    = top
    freq_df = pd.DataFrame({
        "Categoria": vc.index,
        "Quantidade": vc.values,
        "Percentual": (vc.values / total * 100).round(1),
    })
    return freq_df


def _kpis(df: pd.DataFrame, col: str) -> dict:
    s = df[col].fillna("").astype(str).str.strip()
    total  = len(s)
    nulls  = (s == "").sum()
    unique = s.replace("", pd.NA).dropna().nunique()
    vc     = s.replace("", pd.NA).dropna().value_counts()
    pct_top1 = round(vc.iloc[0] / total * 100, 1) if len(vc) else 0
    pct_top3 = round(vc.iloc[:3].sum() / total * 100, 1) if len(vc) >= 3 else round(vc.sum() / total * 100, 1) if len(vc) else 0
    return {
        "n":        total,
        "pct_null": round(nulls / total * 100, 1) if total else 0,
        "unique":   unique,
        "pct_top1": pct_top1,
        "pct_top3": pct_top3,
    }


# ── Helpers de slide ──────────────────────────────────────────────────────

def _new_prs() -> Presentation:
    prs = Presentation()
    prs.slide_width  = _SLIDE_W
    prs.slide_height = _SLIDE_H
    return prs


def _blank_slide(prs: Presentation):
    layout = prs.slide_layouts[6]  # blank
    return prs.slides.add_slide(layout)


def _add_rect(slide, x, y, w, h, fill_color, line_color=None):
    shape = slide.shapes.add_shape(1, x, y, w, h)  # MSO_SHAPE_TYPE.RECTANGLE=1
    shape.fill.solid()
    shape.fill.fore_color.rgb = fill_color
    if line_color:
        shape.line.color.rgb = line_color
    else:
        shape.line.fill.background()
    return shape


def _add_text(slide, text: str, x, y, w, h,
              font_size: int = 14, bold: bool = False,
              color: RGBColor = None, align=PP_ALIGN.LEFT,
              wrap: bool = True):
    txBox = slide.shapes.add_textbox(x, y, w, h)
    tf = txBox.text_frame
    tf.word_wrap = wrap
    p = tf.paragraphs[0]
    p.alignment = align
    run = p.add_run()
    run.text = text
    run.font.size  = Pt(font_size)
    run.font.bold  = bold
    run.font.color.rgb = color or _DARK
    return txBox


def _slide_header(slide, title: str, subtitle: str = ""):
    """Faixa escura no topo com título."""
    _add_rect(slide, Inches(0), Inches(0), _SLIDE_W, Inches(1.1), _DARK)
    _add_text(slide, title,
              Inches(0.4), Inches(0.1), Inches(10), Inches(0.6),
              font_size=22, bold=True, color=_WHITE, align=PP_ALIGN.LEFT)
    if subtitle:
        _add_text(slide, subtitle,
                  Inches(0.4), Inches(0.65), Inches(10), Inches(0.35),
                  font_size=10, color=RGBColor(0x47, 0x55, 0x69), align=PP_ALIGN.LEFT)
    # Linha de acento
    _add_rect(slide, Inches(0), Inches(1.08), _SLIDE_W, Emu(36000), _ACCENT)


def _slide_footer(slide, left_text: str = ""):
    y = _SLIDE_H - Inches(0.3)
    _add_rect(slide, Inches(0), y, _SLIDE_W, Inches(0.3), _DARK)
    ts = datetime.now().strftime("%d/%m/%Y %H:%M")
    _add_text(slide, f"VIVOHUB · Data Merger v2 · {ts}",
              Inches(0.3), y, Inches(9), Inches(0.28),
              font_size=8, color=RGBColor(0x47, 0x55, 0x69))
    if left_text:
        _add_text(slide, left_text,
                  Inches(10), y, Inches(3), Inches(0.28),
                  font_size=8, color=RGBColor(0x47, 0x55, 0x69), align=PP_ALIGN.RIGHT)


def _add_kpi_boxes(slide, kpis: dict, y_start=Inches(1.25)):
    """Linha de 5 KPI boxes."""
    labels = ["Registros", "% Nulos", "Únicos", "% Top 1", "% Top 3"]
    values = [
        f"{kpis['n']:,}",
        f"{kpis['pct_null']:.1f}%",
        f"{kpis['unique']:,}",
        f"{kpis['pct_top1']:.1f}%",
        f"{kpis['pct_top3']:.1f}%",
    ]
    box_w = Inches(2.4)
    gap   = Inches(0.15)
    x0    = Inches(0.4)
    for i, (lbl, val) in enumerate(zip(labels, values)):
        x = x0 + i * (box_w + gap)
        _add_rect(slide, x, y_start, box_w, Inches(1.0), _LIGHT,
                  line_color=RGBColor(0xE2, 0xE8, 0xF0))
        _add_rect(slide, x, y_start, Emu(36000), Inches(1.0), _ACCENT)
        _add_text(slide, lbl,
                  x + Inches(0.08), y_start + Inches(0.06), box_w, Inches(0.3),
                  font_size=8, color=_MUTED)
        _add_text(slide, val,
                  x + Inches(0.08), y_start + Inches(0.3), box_w, Inches(0.55),
                  font_size=20, bold=True, color=_DARK)


def _add_bar_chart(slide, freq_df: pd.DataFrame, x, y, w, h, title: str = ""):
    """Gráfico de barras horizontal (categorias ordenadas por frequência)."""
    cd = ChartData()
    cd.categories = freq_df["Categoria"].tolist()
    cd.add_series("Quantidade", freq_df["Quantidade"].tolist())
    chart = slide.shapes.add_chart(
        XL_CHART_TYPE.BAR_CLUSTERED, x, y, w, h, cd
    ).chart
    chart.has_title = bool(title)
    if title:
        chart.chart_title.text_frame.text = title
        chart.chart_title.text_frame.paragraphs[0].runs[0].font.size = Pt(11)
    chart.has_legend = False
    # Estilo da série
    series = chart.series[0]
    series.format.fill.solid()
    series.format.fill.fore_color.rgb = _ACCENT
    # Eixos
    chart.value_axis.has_major_gridlines = True
    chart.category_axis.tick_labels.font.size = Pt(9)
    chart.value_axis.tick_labels.font.size = Pt(9)
    return chart


def _add_pie_chart(slide, freq_df: pd.DataFrame, x, y, w, h, title: str = ""):
    """Gráfico de pizza/doughnut."""
    cd = ChartData()
    cd.categories = freq_df["Categoria"].tolist()
    cd.add_series("Participação", freq_df["Quantidade"].tolist())
    chart = slide.shapes.add_chart(
        XL_CHART_TYPE.DOUGHNUT, x, y, w, h, cd
    ).chart
    chart.has_title = bool(title)
    if title:
        chart.chart_title.text_frame.text = title
        chart.chart_title.text_frame.paragraphs[0].runs[0].font.size = Pt(11)
    chart.has_legend = True
    chart.legend.font.size = Pt(8)
    # Cores das fatias
    for i, point in enumerate(chart.series[0].points):
        point.format.fill.solid()
        point.format.fill.fore_color.rgb = _PALETTE[i % len(_PALETTE)]
    return chart


def _add_freq_table(slide, freq_df: pd.DataFrame, x, y, w, h):
    """Tabela de frequências: Categoria | Quantidade | %."""
    rows_count = min(len(freq_df) + 1, 20)  # cabeçalho + dados
    tbl = slide.shapes.add_table(rows_count, 3, x, y, w, h).table
    # Larguras
    tbl.columns[0].width = int(w * 0.55)
    tbl.columns[1].width = int(w * 0.22)
    tbl.columns[2].width = int(w * 0.23)

    def _set_cell(row_idx, col_idx, text, bold=False, bg=None, fg=_DARK):
        cell = tbl.cell(row_idx, col_idx)
        cell.text = text
        tf = cell.text_frame
        tf.paragraphs[0].runs[0].font.size = Pt(9)
        tf.paragraphs[0].runs[0].font.bold = bold
        tf.paragraphs[0].runs[0].font.color.rgb = fg
        if bg:
            cell.fill.solid()
            cell.fill.fore_color.rgb = bg

    # Cabeçalho
    for ci, (hdr, fg) in enumerate(zip(["Categoria", "Qtd", "%"], [_WHITE, _WHITE, _WHITE])):
        _set_cell(0, ci, hdr, bold=True, bg=_DARK, fg=_WHITE)

    # Dados
    for ri, (_, row) in enumerate(freq_df.head(rows_count - 1).iterrows(), start=1):
        bg = _LIGHT if ri % 2 == 0 else None
        _set_cell(ri, 0, str(row["Categoria"]), bg=bg)
        _set_cell(ri, 1, f"{int(row['Quantidade']):,}", bg=bg)
        _set_cell(ri, 2, f"{row['Percentual']:.1f}%", bg=bg)


# ── API pública ────────────────────────────────────────────────────────────

def build_column_pptx(df: pd.DataFrame, col_name: str) -> bytes:
    """
    Gera PPTX de dashboard para uma coluna (4 slides).
    Slide 1: Capa + KPIs
    Slide 2: Gráfico principal (barras ou pizza)
    Slide 3: Tabela de frequências
    Slide 4: Contextualização (análise textual)
    """
    prs  = _new_prs()
    freq = _freq(df, col_name)
    kpi  = _kpis(df, col_name)
    ts   = datetime.now().strftime("%d/%m/%Y %H:%M")
    use_pie = kpi["unique"] <= 6

    # ── Slide 1: Capa + KPIs ──────────────────────────────────────────────
    sl = _blank_slide(prs)
    _slide_header(sl, f"Dashboard — {col_name}", f"Gerado em {ts} · {kpi['n']:,} registros")
    _add_kpi_boxes(sl, kpi, y_start=Inches(1.4))
    _slide_footer(sl, col_name)

    # Descrição analítica
    analysis = _col_analysis_text(df, col_name, kpi, freq)
    _add_text(sl, analysis,
              Inches(0.4), Inches(2.6), Inches(12.5), Inches(4.3),
              font_size=11, color=_DARK)

    # ── Slide 2: Gráfico principal ────────────────────────────────────────
    sl2 = _blank_slide(prs)
    _slide_header(sl2, f"{col_name} — Distribuição",
                  "Frequência por categoria")
    _slide_footer(sl2, col_name)

    if use_pie:
        _add_pie_chart(sl2, freq,
                       Inches(2.5), Inches(1.3), Inches(8.0), Inches(5.8),
                       title=f"Distribuição — {col_name}")
    else:
        _add_bar_chart(sl2, freq,
                       Inches(0.4), Inches(1.3), Inches(12.4), Inches(5.8),
                       title=f"Top {min(len(freq), _TOP_N)} — {col_name}")

    # ── Slide 3: Tabela de frequências ────────────────────────────────────
    sl3 = _blank_slide(prs)
    _slide_header(sl3, f"{col_name} — Tabela de Frequências", "Categoria | Quantidade | %")
    _slide_footer(sl3, col_name)
    _add_freq_table(sl3, freq,
                    Inches(0.5), Inches(1.3), Inches(12.3), Inches(5.8))

    # ── Slide 4: Segundo gráfico / complementar ───────────────────────────
    sl4 = _blank_slide(prs)
    _slide_header(sl4, f"{col_name} — Concentração Top 3 vs. Resto",
                  "Análise de concentração / dominância")
    _slide_footer(sl4, col_name)

    # Agrupa em Top 3 vs Outros
    top3 = freq.head(3).copy()
    rest_qty = freq.iloc[3:]["Quantidade"].sum() if len(freq) > 3 else 0
    if rest_qty > 0:
        other_row = pd.DataFrame([{"Categoria": "Demais", "Quantidade": rest_qty,
                                    "Percentual": round(rest_qty / kpi["n"] * 100, 1)}])
        top3 = pd.concat([top3, other_row], ignore_index=True)

    if len(top3) >= 2:
        _add_pie_chart(sl4, top3,
                       Inches(2.5), Inches(1.3), Inches(8.0), Inches(5.8),
                       title=f"Concentração Top 3 — {col_name}")

    buf = io.BytesIO()
    prs.save(buf)
    return buf.getvalue()


def build_general_pptx(df: pd.DataFrame) -> bytes:
    """
    Gera PPTX de dashboard geral (9+ slides):
    Capa, Resumo, UF, OPERADORA, Tipo de Rota, Central, CLUSTER,
    Rótulos de Linha, Denominação, Conclusões.
    """
    prs = _new_prs()
    ts  = datetime.now().strftime("%d/%m/%Y %H:%M")
    total = len(df)

    # ── Capa ──────────────────────────────────────────────────────────────
    sl = _blank_slide(prs)
    _add_rect(sl, Inches(0), Inches(0), _SLIDE_W, _SLIDE_H, _DARK)
    _add_rect(sl, Inches(0), Inches(3.5), _SLIDE_W, Emu(54000), _ACCENT)
    _add_text(sl, "VIVOHUB",
              Inches(1), Inches(1.2), Inches(11), Inches(0.8),
              font_size=14, bold=False, color=RGBColor(0x47, 0x55, 0x69), align=PP_ALIGN.CENTER)
    _add_text(sl, "Dashboard Geral de Rotas",
              Inches(1), Inches(1.9), Inches(11), Inches(1.0),
              font_size=36, bold=True, color=_WHITE, align=PP_ALIGN.CENTER)
    _add_text(sl, f"Data Merger v2  ·  {ts}  ·  {total:,} rotas",
              Inches(1), Inches(2.9), Inches(11), Inches(0.5),
              font_size=13, color=RGBColor(0x94, 0xA3, 0xB8), align=PP_ALIGN.CENTER)

    # ── Resumo geral ──────────────────────────────────────────────────────
    sl = _blank_slide(prs)
    _slide_header(sl, "Resumo Executivo", f"Visão geral — {total:,} rotas · {ts}")
    _slide_footer(sl)

    # KPIs globais
    non_empty = lambda c: (df[c].fillna("").astype(str).str.strip() != "").sum()
    pct_fill  = lambda c: round(non_empty(c) / total * 100, 1) if total else 0
    from app.core.hash_utils import BUSINESS_COLS

    kpi_items = [
        ("Total de Rotas",  f"{total:,}",      None),
        ("Colunas",         "8",               None),
        ("UFs",             str(df["UF"].nunique()),       None),
        ("Operadoras",      str(df["OPERADORA"].nunique()), None),
        ("Centrais",        str(df["Central"].nunique()),   None),
        ("% Preenchimento", f"{sum(pct_fill(c) for c in BUSINESS_COLS)/8:.1f}%", None),
    ]
    bw = Inches(2.0); gap = Inches(0.12); x0 = Inches(0.3)
    for i, (lbl, val, _) in enumerate(kpi_items):
        x = x0 + i * (bw + gap)
        _add_rect(sl, x, Inches(1.4), bw, Inches(0.9), _LIGHT,
                  line_color=RGBColor(0xE2, 0xE8, 0xF0))
        _add_rect(sl, x, Inches(1.4), Emu(32000), Inches(0.9), _ACCENT)
        _add_text(sl, lbl, x+Inches(0.06), Inches(1.45), bw, Inches(0.28),
                  font_size=8, color=_MUTED)
        _add_text(sl, val, x+Inches(0.06), Inches(1.68), bw, Inches(0.5),
                  font_size=18, bold=True, color=_DARK)

    # Tabela de preenchimento por coluna
    _add_text(sl, "Preenchimento por coluna",
              Inches(0.4), Inches(2.5), Inches(12), Inches(0.4),
              font_size=11, bold=True, color=_DARK)
    fill_data = pd.DataFrame([
        {"Coluna": c, "Preenchidas": non_empty(c), "% Fill": f"{pct_fill(c):.1f}%"}
        for c in BUSINESS_COLS
    ])
    _add_freq_table(sl, fill_data.rename(columns={"Coluna":"Categoria","Preenchidas":"Quantidade","% Fill":"Percentual"}),
                    Inches(0.4), Inches(3.0), Inches(12.2), Inches(4.2))

    # ── Slides por coluna ─────────────────────────────────────────────────
    col_order = ["UF", "OPERADORA", "Tipo de Rota", "Central", "CLUSTER",
                 "Rótulos de Linha", "REDE", "Denominação"]

    for col in col_order:
        if col not in df.columns:
            continue
        freq = _freq(df, col)
        kpi  = _kpis(df, col)
        use_pie = kpi["unique"] <= 6

        sl = _blank_slide(prs)
        _slide_header(sl, f"Análise — {col}",
                      f"{kpi['unique']} valores únicos · {kpi['pct_null']:.1f}% sem valor")
        _slide_footer(sl, col)

        # Mini KPIs no topo
        _add_kpi_boxes(sl, kpi, y_start=Inches(1.25))

        # Gráfico
        if use_pie:
            _add_pie_chart(sl, freq,
                           Inches(0.4), Inches(2.4), Inches(6.0), Inches(4.8),
                           title="Distribuição")
        else:
            _add_bar_chart(sl, freq.head(12),
                           Inches(0.4), Inches(2.4), Inches(7.5), Inches(4.8),
                           title=f"Top {min(12, len(freq))}")

        # Tabela ao lado
        _add_freq_table(sl, freq.head(14),
                        Inches(8.0) if not use_pie else Inches(6.6),
                        Inches(2.4), Inches(5.1), Inches(4.8))

    # ── Slide de conclusões ───────────────────────────────────────────────
    sl = _blank_slide(prs)
    _slide_header(sl, "Conclusões e Próximos Passos", "Baseado nos dados processados")
    _slide_footer(sl)

    uf_top = df["UF"].value_counts().index[0] if total else "—"
    op_top = df["OPERADORA"].value_counts().index[0] if total else "—"
    lines = [
        f"• Total de rotas consolidadas: {total:,}",
        f"• UF com maior concentração: {uf_top}",
        f"• Operadora dominante: {op_top}",
        "",
        "Próximos Passos:",
        "• Validar rotas com UF em branco",
        "• Conferir rotas com CLUSTER vazio",
        "• Revisar Denominações duplicadas",
        "• Exportar para validação com equipe de campo",
    ]
    _add_text(sl, "\n".join(lines),
              Inches(0.6), Inches(1.4), Inches(12.0), Inches(5.5),
              font_size=14, color=_DARK)

    buf = io.BytesIO()
    prs.save(buf)
    return buf.getvalue()


# ── Análise textual por coluna ────────────────────────────────────────────

def _col_analysis_text(df: pd.DataFrame, col: str, kpi: dict,
                        freq: pd.DataFrame) -> str:
    """Texto analítico específico por coluna (para slide de capa)."""
    top1 = freq.iloc[0]["Categoria"] if len(freq) else "—"
    top1_pct = freq.iloc[0]["Percentual"] if len(freq) else 0

    lines = [f"Coluna: {col}",
             f"Total de registros: {kpi['n']:,}  |  Valores únicos: {kpi['unique']:,}  "
             f"|  Sem valor: {kpi['pct_null']:.1f}%",
             "",
             f"Categoria mais frequente: {top1} ({top1_pct:.1f}%)",
             f"Top 3 concentram: {kpi['pct_top3']:.1f}% dos registros",
             ""]

    col_up = col.upper().replace(" ", "_")

    if col_up == "REDE":
        lines += ["Análise REDE: Verifica distribuição entre VIVO-SMP (dados) e VIVO-STFC (voz)."]
    elif col_up == "UF":
        lines += ["Análise UF: Ranking por estado e participação regional no total de rotas."]
    elif col_up == "CLUSTER":
        lines += ["Análise CLUSTER: Frequência por grupo de roteamento. Verificar % vazio."]
    elif col_up == "TIPO_DE_ROTA":
        lines += ["Análise Tipo de Rota: Mix entre INTERNA e ITX-SIP_AS."]
    elif col_up == "CENTRAL":
        lines += ["Análise Central: Top 10 centrais com maior concentração de rotas. Curva de Pareto."]
    elif col_up in ("RÓTULOS_DE_LINHA", "ROTULOS_DE_LINHA"):
        lines += ["Análise Rótulos de Linha: Rótulos únicos. Verificar duplicidades."]
    elif col_up == "OPERADORA":
        lines += ["Análise OPERADORA: Participação por operadora. Cross com UF e Tipo de Rota."]
    elif col_up in ("DENOMINAÇÃO", "DENOMINACAO"):
        lines += ["Análise Denominação: Padrões e termos recorrentes. Detectar duplicidades."]

    return "\n".join(lines)
