"""
export/pptx_builder.py
======================
Geração de dashboards em PPTX com python-pptx nativo.
  build_column_pptx(df, col_name) → bytes  (4 slides)
  build_general_pptx(df)          → bytes  (9+ slides)

CORREÇÕES v3:
  • _safe_qty/_safe_pct → nunca lançam "Unknown format code 'd' for str"
  • _kpis() → retorna float/int Python nativos, nunca numpy
  • _add_freq_table() → Percentual aceita str ou float
  • Verificações de len() antes de .index[0]
  • Todas colunas tratadas com guard "if col in df.columns"
"""
from __future__ import annotations

import io
from datetime import datetime

import pandas as pd
from pptx import Presentation
from pptx.chart.data import ChartData
from pptx.dml.color import RGBColor
from pptx.enum.chart import XL_CHART_TYPE
from pptx.enum.text import PP_ALIGN
from pptx.util import Inches, Pt, Emu

# ── Paleta ────────────────────────────────────────────────────────────────────
_DARK   = RGBColor(0x0F, 0x17, 0x2A)
_ACCENT = RGBColor(0x1B, 0x5F, 0xBF)
_WHITE  = RGBColor(0xFF, 0xFF, 0xFF)
_LIGHT  = RGBColor(0xF8, 0xF9, 0xFA)
_MUTED  = RGBColor(0x6B, 0x72, 0x80)
_SLATE  = RGBColor(0x47, 0x55, 0x69)

_SLIDE_W = Inches(13.33)
_SLIDE_H = Inches(7.5)
_TOP_N   = 15

_PALETTE = [
    RGBColor(0x1B, 0x5F, 0xBF), RGBColor(0x16, 0xA3, 0x4A),
    RGBColor(0xD9, 0x77, 0x06), RGBColor(0xDC, 0x26, 0x26),
    RGBColor(0x6D, 0x28, 0xD9), RGBColor(0x0E, 0x7A, 0x8B),
    RGBColor(0xDB, 0x27, 0x77), RGBColor(0x05, 0x96, 0x68),
]


# ── Formatação segura ─────────────────────────────────────────────────────────

def _safe_str(v) -> str:
    if v is None:
        return ""
    s = str(v)
    return "" if s in ("nan", "None", "NaN") else s


def _safe_qty(v) -> str:
    """Qualquer valor → '1.234' sem erro."""
    try:
        return f"{int(float(_safe_str(v) or '0')):,}"
    except Exception:
        return _safe_str(v)


def _safe_pct(v) -> str:
    """Qualquer valor → '12.3%' sem erro. Strings já formatadas são passadas."""
    try:
        s = _safe_str(v).strip()
        if s.endswith("%"):
            return s           # já formatado — não reformata
        return f"{float(s):.1f}%"
    except Exception:
        return _safe_str(v)


# ── Análise ───────────────────────────────────────────────────────────────────

def _freq(df: pd.DataFrame, col: str, top_n: int = _TOP_N) -> pd.DataFrame:
    if col not in df.columns:
        return pd.DataFrame(columns=["Categoria", "Quantidade", "Percentual"])
    s     = df[col].fillna("").astype(str).str.strip().replace("", "(Sem valor)")
    vc    = s.value_counts()
    total = max(int(len(s)), 1)
    if len(vc) > top_n:
        top   = vc.iloc[:top_n]
        other = int(vc.iloc[top_n:].sum())
        vc    = pd.concat([top, pd.Series({"Outros": other})])
    return pd.DataFrame({
        "Categoria":  [_safe_str(c) for c in vc.index],
        "Quantidade": [int(x) for x in vc.values],          # int Python nativo
        "Percentual": [round(float(x) / total * 100, 1) for x in vc.values],  # float nativo
    })


def _kpis(df: pd.DataFrame, col: str) -> dict:
    if col not in df.columns:
        return {"n": 0, "pct_null": 0.0, "unique": 0, "pct_top1": 0.0, "pct_top3": 0.0}
    s     = df[col].fillna("").astype(str).str.strip()
    total = max(int(len(s)), 1)
    nulls = int((s == "").sum())
    real  = s[s != ""]
    vc    = real.value_counts()
    return {
        "n":        total,
        "pct_null": round(nulls / total * 100, 1),
        "unique":   int(real.nunique()),
        "pct_top1": round(float(vc.iloc[0]) / total * 100, 1) if len(vc) >= 1 else 0.0,
        "pct_top3": round(float(vc.iloc[:3].sum()) / total * 100, 1) if len(vc) >= 1 else 0.0,
    }


# ── Primitivos de slide ───────────────────────────────────────────────────────

def _new_prs() -> Presentation:
    prs = Presentation()
    prs.slide_width  = _SLIDE_W
    prs.slide_height = _SLIDE_H
    return prs


def _blank_slide(prs: Presentation):
    return prs.slides.add_slide(prs.slide_layouts[6])


def _add_rect(slide, x, y, w, h, fill: RGBColor, line: RGBColor = None):
    sh = slide.shapes.add_shape(1, x, y, w, h)
    sh.fill.solid()
    sh.fill.fore_color.rgb = fill
    if line:
        sh.line.color.rgb = line
    else:
        sh.line.fill.background()
    return sh


def _add_text(slide, text: str, x, y, w, h, *,
              font_size: int = 12, bold: bool = False,
              color: RGBColor = None, align=PP_ALIGN.LEFT, wrap: bool = True):
    txb = slide.shapes.add_textbox(x, y, w, h)
    tf  = txb.text_frame
    tf.word_wrap = wrap
    p   = tf.paragraphs[0]
    p.alignment = align
    run = p.add_run()
    run.text           = _safe_str(text)
    run.font.size      = Pt(font_size)
    run.font.bold      = bold
    run.font.color.rgb = color or _DARK
    return txb


def _slide_header(slide, title: str, subtitle: str = ""):
    _add_rect(slide, Inches(0), Inches(0), _SLIDE_W, Inches(1.1), _DARK)
    _add_text(slide, title,
              Inches(0.4), Inches(0.08), Inches(12), Inches(0.65),
              font_size=22, bold=True, color=_WHITE)
    if subtitle:
        _add_text(slide, subtitle,
                  Inches(0.4), Inches(0.68), Inches(12), Inches(0.34),
                  font_size=10, color=_SLATE)
    _add_rect(slide, Inches(0), Inches(1.08), _SLIDE_W, Emu(36000), _ACCENT)


def _slide_footer(slide, right_text: str = ""):
    y = _SLIDE_H - Inches(0.3)
    _add_rect(slide, Inches(0), y, _SLIDE_W, Inches(0.3), _DARK)
    ts = datetime.now().strftime("%d/%m/%Y %H:%M")
    _add_text(slide, f"VIVOHUB · Data Merger v2 · {ts}",
              Inches(0.3), y, Inches(9), Inches(0.28), font_size=8, color=_SLATE)
    if right_text:
        _add_text(slide, right_text,
                  Inches(10), y, Inches(3), Inches(0.28),
                  font_size=8, color=_SLATE, align=PP_ALIGN.RIGHT)


def _add_kpi_boxes(slide, kpi: dict, y_start=Inches(1.25)):
    labels = ["Registros",        "% Nulos",              "Únicos",
              "% Top 1",          "% Top 3"]
    values = [f"{kpi['n']:,}",    f"{kpi['pct_null']:.1f}%", f"{kpi['unique']:,}",
              f"{kpi['pct_top1']:.1f}%", f"{kpi['pct_top3']:.1f}%"]
    bw, gap, x0 = Inches(2.4), Inches(0.14), Inches(0.4)
    for i, (lbl, val) in enumerate(zip(labels, values)):
        x = x0 + i * (bw + gap)
        _add_rect(slide, x, y_start, bw, Inches(1.0), _LIGHT,
                  line=RGBColor(0xE2, 0xE8, 0xF0))
        _add_rect(slide, x, y_start, Emu(34000), Inches(1.0), _ACCENT)
        _add_text(slide, lbl, x + Inches(0.08), y_start + Inches(0.05),
                  bw, Inches(0.28), font_size=8, color=_MUTED)
        _add_text(slide, val, x + Inches(0.08), y_start + Inches(0.28),
                  bw, Inches(0.6), font_size=20, bold=True, color=_DARK)


def _add_bar_chart(slide, freq_df: pd.DataFrame, x, y, w, h, title: str = ""):
    if freq_df is None or freq_df.empty:
        return None
    cd = ChartData()
    cd.categories = [_safe_str(c) for c in freq_df["Categoria"]]
    cd.add_series("Quantidade", [int(v) for v in freq_df["Quantidade"]])
    chart = slide.shapes.add_chart(XL_CHART_TYPE.BAR_CLUSTERED, x, y, w, h, cd).chart
    chart.has_title  = bool(title)
    chart.has_legend = False
    if title and chart.has_title:
        chart.chart_title.text_frame.text = title
        chart.chart_title.text_frame.paragraphs[0].runs[0].font.size = Pt(11)
    s = chart.series[0]
    s.format.fill.solid()
    s.format.fill.fore_color.rgb = _ACCENT
    chart.value_axis.has_major_gridlines = True
    chart.category_axis.tick_labels.font.size = Pt(8)
    chart.value_axis.tick_labels.font.size    = Pt(8)
    return chart


def _add_pie_chart(slide, freq_df: pd.DataFrame, x, y, w, h, title: str = ""):
    if freq_df is None or len(freq_df) < 2:
        return None
    cd = ChartData()
    cd.categories = [_safe_str(c) for c in freq_df["Categoria"]]
    cd.add_series("Participação", [int(v) for v in freq_df["Quantidade"]])
    chart = slide.shapes.add_chart(XL_CHART_TYPE.DOUGHNUT, x, y, w, h, cd).chart
    chart.has_title  = bool(title)
    chart.has_legend = True
    if title and chart.has_title:
        chart.chart_title.text_frame.text = title
        chart.chart_title.text_frame.paragraphs[0].runs[0].font.size = Pt(11)
    chart.legend.font.size = Pt(8)
    for i, pt in enumerate(chart.series[0].points):
        pt.format.fill.solid()
        pt.format.fill.fore_color.rgb = _PALETTE[i % len(_PALETTE)]
    return chart


def _add_freq_table(slide, freq_df: pd.DataFrame, x, y, w, h):
    """Tabela de frequências tolerante a qualquer tipo em Quantidade/Percentual."""
    if freq_df is None or freq_df.empty:
        return
    rows_n = min(len(freq_df) + 1, 20)
    tbl    = slide.shapes.add_table(rows_n, 3, x, y, w, h).table
    tbl.columns[0].width = int(w * 0.55)
    tbl.columns[1].width = int(w * 0.22)
    tbl.columns[2].width = int(w * 0.23)

    def _c(r, c, txt, bold=False, bg: RGBColor = None, fg: RGBColor = _DARK):
        cell = tbl.cell(r, c)
        cell.text = _safe_str(txt)
        p = cell.text_frame.paragraphs[0]
        if p.runs:
            run = p.runs[0]
            run.font.size      = Pt(9)
            run.font.bold      = bold
            run.font.color.rgb = fg
        if bg:
            cell.fill.solid()
            cell.fill.fore_color.rgb = bg

    for ci, hdr in enumerate(["Categoria", "Qtd", "%"]):
        _c(0, ci, hdr, bold=True, bg=_DARK, fg=_WHITE)

    for ri, (_, row) in enumerate(freq_df.head(rows_n - 1).iterrows(), start=1):
        bg = _LIGHT if ri % 2 == 0 else None
        _c(ri, 0, _safe_str(row.get("Categoria", "")), bg=bg)
        _c(ri, 1, _safe_qty(row.get("Quantidade", "")), bg=bg)
        _c(ri, 2, _safe_pct(row.get("Percentual", "")), bg=bg)


# ── Análise textual ───────────────────────────────────────────────────────────

def _col_analysis_text(df: pd.DataFrame, col: str, kpi: dict,
                        freq: pd.DataFrame) -> str:
    top1     = _safe_str(freq.iloc[0]["Categoria"])  if len(freq) >= 1 else "—"
    top1_pct = freq.iloc[0]["Percentual"]             if len(freq) >= 1 else 0.0
    lines = [
        f"Coluna: {col}",
        f"Total: {kpi['n']:,}  |  Únicos: {kpi['unique']:,}  |  Sem valor: {kpi['pct_null']:.1f}%",
        "",
        f"Categoria dominante: {top1}  ({_safe_pct(top1_pct)})",
        f"Top 3 acumulam: {kpi['pct_top3']:.1f}% dos registros",
        "",
    ]
    col_up = col.upper().replace(" ", "_")
    hints = {
        "REDE":             "Distribuição entre VIVO-SMP (dados) e VIVO-STFC (voz).",
        "UF":               "Ranking por estado — participação regional no total de rotas.",
        "CLUSTER":          "Frequência por grupo de roteamento. Verificar % vazio.",
        "TIPO_DE_ROTA":     "Mix entre INTERNA e ITX-SIP_AS.",
        "CENTRAL":          "Top 10 centrais com maior concentração de rotas — Pareto.",
        "ROTULOS_DE_LINHA": "Rótulos únicos. Verificar possíveis duplicidades.",
        "OPERADORA":        "Participação por operadora. Cross com UF e Tipo de Rota.",
        "DENOMINACAO":      "Padrões e termos recorrentes. Identificar duplicidades.",
    }
    for key, msg in hints.items():
        if key in col_up or col_up in key:
            lines.append(msg)
            break
    return "\n".join(lines)


# ── API pública ───────────────────────────────────────────────────────────────

def build_column_pptx(df: pd.DataFrame, col_name: str) -> bytes:
    """4 slides: KPIs | Gráfico | Tabela | Concentração Top3."""
    prs     = _new_prs()
    freq    = _freq(df, col_name)
    kpi     = _kpis(df, col_name)
    ts      = datetime.now().strftime("%d/%m/%Y %H:%M")
    use_pie = kpi["unique"] <= 6

    # Slide 1 — KPIs + análise
    sl = _blank_slide(prs)
    _slide_header(sl, f"Dashboard — {col_name}",
                  f"Gerado em {ts}  ·  {kpi['n']:,} registros")
    _add_kpi_boxes(sl, kpi, y_start=Inches(1.35))
    _slide_footer(sl, col_name)
    _add_text(sl, _col_analysis_text(df, col_name, kpi, freq),
              Inches(0.4), Inches(2.55), Inches(12.5), Inches(4.3),
              font_size=11, color=_DARK)

    # Slide 2 — Gráfico principal
    sl2 = _blank_slide(prs)
    _slide_header(sl2, f"{col_name} — Distribuição", "Frequência por categoria")
    _slide_footer(sl2, col_name)
    if use_pie and len(freq) >= 2:
        _add_pie_chart(sl2, freq,
                       Inches(2.5), Inches(1.25), Inches(8.0), Inches(5.9),
                       title=f"Distribuição — {col_name}")
    else:
        _add_bar_chart(sl2, freq,
                       Inches(0.4), Inches(1.25), Inches(12.4), Inches(5.9),
                       title=f"Top {min(len(freq), _TOP_N)} — {col_name}")

    # Slide 3 — Tabela
    sl3 = _blank_slide(prs)
    _slide_header(sl3, f"{col_name} — Tabela de Frequências",
                  "Categoria  |  Quantidade  |  %")
    _slide_footer(sl3, col_name)
    _add_freq_table(sl3, freq,
                    Inches(0.5), Inches(1.25), Inches(12.3), Inches(5.9))

    # Slide 4 — Concentração
    sl4 = _blank_slide(prs)
    _slide_header(sl4, f"{col_name} — Concentração Top 3 vs. Demais",
                  "Análise de dominância")
    _slide_footer(sl4, col_name)
    top3 = freq.head(3).copy()
    rest = int(freq.iloc[3:]["Quantidade"].sum()) if len(freq) > 3 else 0
    if rest > 0:
        top3 = pd.concat([top3, pd.DataFrame([{
            "Categoria": "Demais",
            "Quantidade": rest,
            "Percentual": round(rest / kpi["n"] * 100, 1),
        }])], ignore_index=True)
    if len(top3) >= 2:
        _add_pie_chart(sl4, top3,
                       Inches(2.5), Inches(1.25), Inches(8.0), Inches(5.9),
                       title=f"Concentração — {col_name}")

    buf = io.BytesIO()
    prs.save(buf)
    return buf.getvalue()


def build_general_pptx(df: pd.DataFrame) -> bytes:
    """9+ slides: Capa | Resumo | por coluna | Conclusões."""
    prs   = _new_prs()
    ts    = datetime.now().strftime("%d/%m/%Y %H:%M")
    total = max(int(len(df)), 1)

    # Slide 0 — Capa
    sl = _blank_slide(prs)
    _add_rect(sl, Inches(0), Inches(0), _SLIDE_W, _SLIDE_H, _DARK)
    _add_rect(sl, Inches(0), Inches(3.45), _SLIDE_W, Emu(54000), _ACCENT)
    _add_text(sl, "VIVOHUB",
              Inches(1), Inches(1.2), Inches(11), Inches(0.7),
              font_size=14, color=_SLATE, align=PP_ALIGN.CENTER)
    _add_text(sl, "Dashboard Geral de Rotas",
              Inches(1), Inches(1.85), Inches(11), Inches(1.1),
              font_size=36, bold=True, color=_WHITE, align=PP_ALIGN.CENTER)
    _add_text(sl, f"Data Merger v2  ·  {ts}  ·  {total:,} rotas",
              Inches(1), Inches(2.9), Inches(11), Inches(0.45),
              font_size=13, color=RGBColor(0x94, 0xA3, 0xB8), align=PP_ALIGN.CENTER)

    # Slide 1 — Resumo executivo
    sl = _blank_slide(prs)
    _slide_header(sl, "Resumo Executivo",
                  f"{total:,} rotas consolidadas  ·  {ts}")
    _slide_footer(sl)

    from app.core.hash_utils import BUSINESS_COLS
    non_empty = lambda c: int((df[c].fillna("").astype(str).str.strip() != "").sum()) \
                          if c in df.columns else 0
    pct_fill  = lambda c: round(non_empty(c) / total * 100, 1)

    cols_present = [c for c in BUSINESS_COLS if c in df.columns]
    avg_fill = round(sum(pct_fill(c) for c in cols_present) / max(len(cols_present), 1), 1)

    kpi_cards = [
        ("Total de Rotas",   f"{total:,}"),
        ("UFs distintas",    str(df["UF"].nunique())        if "UF"        in df.columns else "—"),
        ("Operadoras",       str(df["OPERADORA"].nunique()) if "OPERADORA" in df.columns else "—"),
        ("Centrais",         str(df["Central"].nunique())   if "Central"   in df.columns else "—"),
        ("Colunas",          "8"),
        ("% Preenchimento",  f"{avg_fill:.1f}%"),
    ]
    bw, gap, x0 = Inches(2.0), Inches(0.12), Inches(0.3)
    for i, (lbl, val) in enumerate(kpi_cards):
        x = x0 + i * (bw + gap)
        _add_rect(sl, x, Inches(1.38), bw, Inches(0.88), _LIGHT,
                  line=RGBColor(0xE2, 0xE8, 0xF0))
        _add_rect(sl, x, Inches(1.38), Emu(30000), Inches(0.88), _ACCENT)
        _add_text(sl, lbl, x + Inches(0.06), Inches(1.43), bw, Inches(0.28),
                  font_size=8, color=_MUTED)
        _add_text(sl, val, x + Inches(0.06), Inches(1.65), bw, Inches(0.5),
                  font_size=18, bold=True, color=_DARK)

    _add_text(sl, "Preenchimento por coluna",
              Inches(0.4), Inches(2.44), Inches(12), Inches(0.38),
              font_size=11, bold=True, color=_DARK)

    # IMPORTANTE: Percentual como float (não str) para _safe_pct funcionar
    fill_data = pd.DataFrame([
        {"Categoria":  c,
         "Quantidade": non_empty(c),      # int
         "Percentual": pct_fill(c)}       # float — _safe_pct converte corretamente
        for c in BUSINESS_COLS if c in df.columns
    ])
    _add_freq_table(sl, fill_data,
                    Inches(0.4), Inches(2.9), Inches(12.2), Inches(4.2))

    # Slides por coluna
    col_order = ["UF", "OPERADORA", "Tipo de Rota", "Central",
                 "CLUSTER", "Rótulos de Linha", "REDE", "Denominação"]
    for col in col_order:
        if col not in df.columns:
            continue
        freq    = _freq(df, col)
        kpi     = _kpis(df, col)
        use_pie = kpi["unique"] <= 6

        sl = _blank_slide(prs)
        _slide_header(sl, f"Análise — {col}",
                      f"{kpi['unique']} valores únicos  ·  {kpi['pct_null']:.1f}% sem valor")
        _slide_footer(sl, col)
        _add_kpi_boxes(sl, kpi, y_start=Inches(1.2))

        if use_pie and len(freq) >= 2:
            _add_pie_chart(sl, freq,
                           Inches(0.4), Inches(2.35), Inches(6.0), Inches(4.8),
                           title="Distribuição")
            tbl_x, tbl_w = Inches(6.65), Inches(6.5)
        else:
            _add_bar_chart(sl, freq.head(12),
                           Inches(0.4), Inches(2.35), Inches(7.5), Inches(4.8),
                           title=f"Top {min(12, len(freq))}")
            tbl_x, tbl_w = Inches(8.1), Inches(5.0)

        _add_freq_table(sl, freq.head(14), tbl_x, Inches(2.35), tbl_w, Inches(4.8))

    # Slide final — Conclusões
    sl = _blank_slide(prs)
    _slide_header(sl, "Conclusões e Próximos Passos",
                  "Baseado nos dados processados")
    _slide_footer(sl)

    uf_top = _safe_str(df["UF"].value_counts().index[0]) \
             if "UF" in df.columns and len(df) > 0 else "—"
    op_top = _safe_str(df["OPERADORA"].value_counts().index[0]) \
             if "OPERADORA" in df.columns and len(df) > 0 else "—"

    _add_text(sl, "\n".join([
        f"• Total de rotas consolidadas: {total:,}",
        f"• UF com maior concentração:   {uf_top}",
        f"• Operadora dominante:          {op_top}",
        "",
        "Próximos Passos:",
        "• Validar rotas com UF em branco",
        "• Conferir rotas com CLUSTER vazio",
        "• Revisar Denominações duplicadas",
        "• Exportar para validação com equipe de campo",
    ]), Inches(0.6), Inches(1.35), Inches(12.0), Inches(5.6),
        font_size=13, color=_DARK)

    buf = io.BytesIO()
    prs.save(buf)
    return buf.getvalue()
