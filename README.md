"""
ui/interactive_table.py  v3
===========================
Tabela HTML/JS com filtros inline estilo Excel:
  • Cada cabeçalho tem ícone ▼ de filtro
  • Clique no ▼ abre dropdown com checkboxes dos valores únicos
  • Filtragem 100% no lado do cliente (JavaScript) — sem roundtrip ao Streamlit
  • Clique no nome da coluna ainda seleciona para dashboard
  • Indicador visual (▼ azul) quando filtro ativo
"""
from __future__ import annotations

import json
import base64
import pandas as pd

_PAGE_SIZE = 500

_CSS_VARS = """
  --bg:        #F8F9FA;
  --surface:   #FFFFFF;
  --border:    #D1D5DB;
  --border-lt: #E9ECEF;
  --hdr-bg:    #0F172A;
  --hdr-text:  #FFFFFF;
  --hdr-hover: #1E293B;
  --hdr-sel:   #1B5FBF;
  --text:      #111827;
  --muted:     #6B7280;
  --accent:    #1B5FBF;
  --row-alt:   #FAFAFA;
  --row-hover: #EFF6FF;
"""


def _df_to_json(df: pd.DataFrame) -> str:
    safe = df.copy()
    for c in safe.columns:
        safe[c] = safe[c].astype(str).replace({"nan": "", "None": ""})
    return json.dumps(safe.to_dict(orient="records"), ensure_ascii=False)


def _cols_to_json(df: pd.DataFrame) -> str:
    widths = {
        "rótulos de linha": 200, "denominação": 180,
        "central": 145, "cluster": 120, "operadora": 130,
        "tipo de rota": 125, "uf": 70, "rede": 100,
    }
    return json.dumps(
        [{"key": c, "label": c,
          "width": widths.get(c.lower(), 115)}
         for c in df.columns],
        ensure_ascii=False,
    )


def render_interactive_table(
    df: pd.DataFrame,
    selected_col: str | None,
    height: int = 560,
    component_key: str = "itable",
) -> None:
    data_json = _df_to_json(df)
    cols_json = _cols_to_json(df)
    sel_js    = json.dumps(selected_col or "")

    html = f"""<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap" rel="stylesheet">
<style>
:root {{{_CSS_VARS}}}
*{{box-sizing:border-box;margin:0;padding:0;}}
body{{
  font-family:'IBM Plex Sans',sans-serif;
  font-size:12px;background:var(--bg);color:var(--text);
  display:flex;flex-direction:column;height:{height}px;overflow:hidden;
}}

/* ── TOOLBAR ──────────────────────────────────────────────────────── */
#toolbar{{
  display:flex;align-items:center;justify-content:space-between;
  padding:6px 12px;background:var(--surface);
  border-bottom:1px solid var(--border);flex-shrink:0;gap:10px;
}}
#count{{font-size:11px;color:var(--muted);font-weight:500;}}
#hint {{font-size:11px;color:var(--accent);font-weight:500;}}

/* ── WRAPPER ──────────────────────────────────────────────────────── */
#wrap{{flex:1;overflow:auto;position:relative;}}

/* ── TABELA ───────────────────────────────────────────────────────── */
table{{border-collapse:collapse;table-layout:fixed;width:max-content;min-width:100%;}}
thead tr{{position:sticky;top:0;z-index:20;}}

th{{
  background:var(--hdr-bg);
  border-right:1px solid #1E293B;
  border-bottom:2px solid #1E293B;
  white-space:nowrap;user-select:none;padding:0;
}}
th.col-num{{
  width:42px;min-width:42px;
  font-size:10px;color:rgba(255,255,255,.25);
  padding:0 4px;text-align:center;cursor:default;
}}
th.col-sel{{background:var(--hdr-sel)!important;}}

/* ── CABEÇALHO INTERNO ────────────────────────────────────────────── */
.th-wrap{{
  display:flex;align-items:stretch;height:36px;
}}
/* Área clicável para selecionar coluna (dashboard) */
.th-name{{
  flex:1;display:flex;align-items:center;
  padding:0 8px;gap:5px;cursor:pointer;min-width:0;
  border:none;background:transparent;
}}
.th-name:hover{{background:var(--hdr-hover);}}
.th-label{{
  font-size:10px;font-weight:700;color:#FFFFFF;
  letter-spacing:.05em;text-transform:uppercase;
  overflow:hidden;text-overflow:ellipsis;white-space:nowrap;
}}
.sort-ico{{opacity:.25;flex-shrink:0;}}
.sort-ico.on{{opacity:1;}}

/* ── BOTÃO FILTRO (▼) ─────────────────────────────────────────────── */
.th-filter{{
  display:flex;align-items:center;justify-content:center;
  width:24px;min-width:24px;cursor:pointer;flex-shrink:0;
  border-left:1px solid rgba(255,255,255,.07);
  border:none;background:transparent;
  font-size:9px;color:rgba(255,255,255,.35);
  transition:color .15s,background .15s;
}}
.th-filter:hover{{background:rgba(255,255,255,.1);color:#fff;}}
.th-filter.active{{color:#60A5FA;background:rgba(27,95,191,.25);}}

/* ── DROPDOWN DE FILTRO ───────────────────────────────────────────── */
.fdd{{
  position:fixed;z-index:9999;
  background:#FFFFFF;border:1px solid #D1D5DB;
  box-shadow:0 4px 20px rgba(0,0,0,.14);
  border-radius:4px;min-width:200px;max-width:280px;
  display:none;overflow:hidden;
}}
.fdd.open{{display:block;}}
.fdd-head{{
  display:flex;align-items:center;justify-content:space-between;
  padding:8px 10px;border-bottom:1px solid #E9ECEF;
  background:#F8F9FA;
}}
.fdd-title{{font-size:11px;font-weight:700;color:#1C2536;text-transform:uppercase;letter-spacing:.05em;}}
.fdd-actions{{display:flex;gap:6px;}}
.fdd-btn{{
  font-size:10px;color:#1B5FBF;cursor:pointer;
  background:none;border:none;font-family:inherit;font-weight:600;padding:0;
}}
.fdd-btn:hover{{text-decoration:underline;}}
.fdd-search{{
  padding:6px 10px;border-bottom:1px solid #E9ECEF;
}}
.fdd-search input{{
  width:100%;padding:4px 8px;font-size:11px;
  border:1px solid #D1D5DB;border-radius:3px;
  font-family:inherit;outline:none;color:#1C2536;
}}
.fdd-search input:focus{{border-color:#1B5FBF;}}
.fdd-list{{max-height:200px;overflow-y:auto;padding:4px 0;}}
.fdd-item{{
  display:flex;align-items:center;gap:8px;
  padding:5px 10px;cursor:pointer;font-size:12px;color:#374151;
}}
.fdd-item:hover{{background:#EFF6FF;}}
.fdd-item input[type=checkbox]{{
  width:13px;height:13px;accent-color:#1B5FBF;cursor:pointer;flex-shrink:0;
}}
.fdd-item label{{cursor:pointer;flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
.fdd-footer{{
  padding:7px 10px;border-top:1px solid #E9ECEF;
  display:flex;justify-content:flex-end;
}}
.fdd-apply{{
  font-size:11px;font-weight:700;
  background:#1B5FBF;color:#fff;
  border:none;border-radius:3px;
  padding:4px 14px;cursor:pointer;font-family:inherit;
}}
.fdd-apply:hover{{background:#1549A0;}}

/* ── BADGE FILTRO ATIVO ───────────────────────────────────────────── */
#active-filters{{
  display:flex;align-items:center;gap:6px;flex-wrap:wrap;
  padding:4px 12px;background:var(--surface);
  border-bottom:1px solid var(--border);flex-shrink:0;
  min-height:0;
}}
#active-filters:empty{{display:none;}}
.af-chip{{
  display:inline-flex;align-items:center;gap:4px;
  background:#EFF6FF;border:1px solid #BFDBFE;
  border-radius:2px;padding:2px 8px;font-size:10px;
  color:#1B5FBF;font-weight:600;
}}
.af-x{{cursor:pointer;font-size:12px;line-height:1;color:#1B5FBF;margin-left:2px;}}
.af-x:hover{{color:#DC2626;}}

/* ── CÉLULAS ──────────────────────────────────────────────────────── */
tbody tr:nth-child(even){{background:var(--row-alt);}}
tbody tr:nth-child(odd) {{background:var(--surface);}}
tbody tr:hover{{background:var(--row-hover)!important;}}
td{{
  border-bottom:1px solid var(--border-lt);
  border-right:1px solid var(--border-lt);
  padding:5px 9px;font-size:12px;color:var(--text);
  overflow:hidden;text-overflow:ellipsis;white-space:nowrap;
}}
td.col-num{{text-align:center;font-size:10px;color:var(--muted);}}
td.col-sel{{background:#EFF6FF!important;}}
.b-smp {{color:#1B5FBF;font-weight:700;font-family:'IBM Plex Mono',monospace;}}
.b-stfc{{color:#6D28D9;font-weight:700;font-family:'IBM Plex Mono',monospace;}}

/* ── PAGINAÇÃO ────────────────────────────────────────────────────── */
#pagination{{
  display:flex;align-items:center;justify-content:space-between;
  padding:5px 12px;background:var(--surface);
  border-top:1px solid var(--border);flex-shrink:0;
}}
#pag-info{{font-size:11px;color:var(--muted);}}
#pag-ctrl{{display:flex;gap:3px;}}
.pb{{
  min-width:26px;height:24px;padding:0 5px;
  border:1px solid var(--border);background:var(--surface);
  color:var(--muted);font-size:11px;cursor:pointer;font-family:inherit;
}}
.pb:hover:not(:disabled){{border-color:var(--accent);color:var(--accent);}}
.pb.on{{background:var(--accent);color:#fff;border-color:var(--accent);font-weight:700;}}
.pb:disabled{{opacity:.35;cursor:default;}}
</style>
</head>
<body>

<div id="toolbar">
  <span id="count"></span>
  <span id="hint">Clique no nome da coluna para ver o dashboard. Use o botao de filtro para filtrar valores.</span>
</div>
<div id="active-filters"></div>
<div id="wrap">
  <table><thead id="thead"></thead><tbody id="tbody"></tbody></table>
</div>
<div id="pagination">
  <span id="pag-info"></span>
  <div id="pag-ctrl"></div>
</div>

<!-- Dropdown flutuante (único, reutilizado por todas as colunas) -->
<div class="fdd" id="fdd">
  <div class="fdd-head">
    <span class="fdd-title" id="fdd-col-name"></span>
    <div class="fdd-actions">
      <button class="fdd-btn" id="fdd-selall">Marcar todos</button>
      <button class="fdd-btn" id="fdd-clear" >Limpar</button>
    </div>
  </div>
  <div class="fdd-search"><input type="text" id="fdd-search" placeholder="Pesquisar..."></div>
  <div class="fdd-list" id="fdd-list"></div>
  <div class="fdd-footer"><button class="fdd-apply" id="fdd-apply">Aplicar</button></div>
</div>

<script>
/* ── Dados ──────────────────────────────────────────────────────────── */
const ALL_ROWS = {data_json};
const COLS     = {cols_json};
const PS       = {_PAGE_SIZE};

let selCol   = {sel_js};   // coluna selecionada para dashboard
let sortCol  = null;
let sortDir  = 'asc';
let page     = 0;

// Filtros ativos: {{ colKey: Set<string> }}
const activeFilters = {{}};

// Todos os valores únicos por coluna (para o dropdown)
const uniqueVals = {{}};
COLS.forEach(c => {{
  const vals = [...new Set(ALL_ROWS.map(r => String(r[c.key] ?? '')))].sort();
  uniqueVals[c.key] = vals;
}});

/* ── Estado do dropdown ─────────────────────────────────────────────── */
let fddCol  = null;   // coluna cujo dropdown está aberto
let fddSel  = new Set();  // seleção temporária no dropdown

/* ── Filtro + ordenação ─────────────────────────────────────────────── */
function getFiltered() {{
  let rows = ALL_ROWS;
  for (const [col, vals] of Object.entries(activeFilters)) {{
    if (vals.size === 0) continue;
    rows = rows.filter(r => vals.has(String(r[col] ?? '')));
  }}
  if (sortCol) {{
    rows = [...rows].sort((a,b) => {{
      const av = String(a[sortCol]||''), bv = String(b[sortCol]||'');
      return sortDir==='asc' ? av.localeCompare(bv,'pt') : bv.localeCompare(av,'pt');
    }});
  }}
  return rows;
}}

/* ── Render head ────────────────────────────────────────────────────── */
function renderHead() {{
  const thead = document.getElementById('thead');
  thead.innerHTML = '';
  const tr = document.createElement('tr');

  // Coluna #
  const thN = document.createElement('th');
  thN.className = 'col-num';
  thN.innerHTML = '<div style="padding:0 4px;height:36px;display:flex;align-items:center;justify-content:center;color:rgba(255,255,255,.25);font-size:10px;">#</div>';
  tr.appendChild(thN);

  COLS.forEach(col => {{
    const th = document.createElement('th');
    th.style.width = col.width + 'px';
    if (col.key === selCol) th.className = 'col-sel';

    const hasFilter = activeFilters[col.key] && activeFilters[col.key].size > 0;
    const sortIco   = col.key === sortCol
      ? (sortDir==='asc'
          ? '<svg class="sort-ico on" width="8" height="10" viewBox="0 0 8 10"><path d="M4 1v8M1 3l3-3 3 3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>'
          : '<svg class="sort-ico on" width="8" height="10" viewBox="0 0 8 10"><path d="M4 9V1M1 7l3 3 3-3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>')
      : '<svg class="sort-ico" width="8" height="10" viewBox="0 0 8 10"><path d="M4 1v8M1 3l3-3 3 3M1 7l3 3 3-3" stroke="rgba(255,255,255,.3)" stroke-width="1.5" stroke-linecap="round"/></svg>';

    th.innerHTML =
      '<div class="th-wrap">' +
        '<div class="th-name" data-col="' + esc(col.key) + '">' +
          '<span class="th-label">' + esc(col.label) + '</span>' +
          sortIco +
        '</div>' +
        '<div class="th-filter ' + (hasFilter ? 'active' : '') + '" ' +
             'data-col="' + esc(col.key) + '" title="Filtrar ' + esc(col.label) + '">' +
          (hasFilter ? 'F' : 'f') +
        '</div>' +
      '</div>';

    // Clique no nome → seleciona coluna para dashboard + ordena
    th.querySelector('.th-name').addEventListener('click', e => {{
      e.stopPropagation();
      if (col.key === sortCol) {{
        sortDir = sortDir === 'asc' ? 'desc' : 'asc';
      }} else {{
        sortCol = col.key; sortDir = 'asc';
      }}
      selCol = col.key;
      notifyParent(col.key);
      page = 0;
      render();
    }});

    // Clique no ▼ → abre dropdown de filtro
    th.querySelector('.th-filter').addEventListener('click', e => {{
      e.stopPropagation();
      openFilter(col.key, e.currentTarget);
    }});

    tr.appendChild(th);
  }});
  thead.appendChild(tr);
}}

/* ── Render body ────────────────────────────────────────────────────── */
function renderBody() {{
  const data  = getFiltered();
  const total = data.length;
  const paged = data.slice(page * PS, (page+1) * PS);

  document.getElementById('count').textContent =
    total.toLocaleString('pt-BR') + ' registros' +
    (total < ALL_ROWS.length ? ' (filtrado de ' + ALL_ROWS.length.toLocaleString('pt-BR') + ')' : '');

  const tbody = document.getElementById('tbody');
  tbody.innerHTML = '';

  paged.forEach((row, i) => {{
    const tr = document.createElement('tr');
    const tdN = document.createElement('td');
    tdN.className = 'col-num';
    tdN.textContent = page * PS + i + 1;
    tr.appendChild(tdN);

    COLS.forEach(col => {{
      const td  = document.createElement('td');
      if (col.key === selCol) td.classList.add('col-sel');
      const v   = String(row[col.key] ?? '');
      const kl  = col.key.toLowerCase();
      if (kl === 'rede') {{
        td.innerHTML = '<span class="' + (v==='VIVO-SMP'?'b-smp':'b-stfc') + '">' + esc(v) + '</span>';
      }} else {{
        td.textContent = v;
      }}
      tr.appendChild(td);
    }});
    tbody.appendChild(tr);
  }});
  renderPag(total);
}}

/* ── Chips de filtros ativos ────────────────────────────────────────── */
function renderChips() {{
  const bar = document.getElementById('active-filters');
  bar.innerHTML = '';
  for (const [col, vals] of Object.entries(activeFilters)) {{
    if (!vals || vals.size === 0) continue;
    const colLabel = COLS.find(c=>c.key===col)?.label || col;
    const chip = document.createElement('span');
    chip.className = 'af-chip';
    const vlist = [...vals].slice(0,2).join(', ') + (vals.size > 2 ? ' +' + (vals.size-2) : '');
    chip.innerHTML = esc(colLabel) + ': ' + esc(vlist)
      + '<span class="af-x" data-col="' + esc(col) + '">[x]</span>';
    chip.querySelector('.af-x').addEventListener('click', e => {{
      const c = e.target.dataset.col;
      delete activeFilters[c];
      page = 0; render();
    }});
    bar.appendChild(chip);
  }}
}}

/* ── Paginação ──────────────────────────────────────────────────────── */
function renderPag(total) {{
  const tp = Math.max(1, Math.ceil(total / PS));
  const s  = total === 0 ? 0 : page * PS + 1;
  const e  = Math.min((page+1) * PS, total);
  document.getElementById('pag-info').textContent =
    'Exibindo ' + s + '–' + e + ' de ' + total.toLocaleString('pt-BR');
  const ctrl = document.getElementById('pag-ctrl');
  ctrl.innerHTML = '';
  const btn = (lbl, cb, dis, on) => {{
    const b = document.createElement('button');
    b.className = 'pb' + (on ? ' on' : '');
    b.textContent = lbl; b.disabled = dis; b.onclick = cb;
    ctrl.appendChild(b);
  }};
  btn('|<', ()=>{{page=0;render();}}, page===0);
  btn('<',  ()=>{{page--;render();}}, page===0);
  const w=5, lo=Math.max(0,Math.min(page-2,tp-w)), hi=Math.min(tp-1,lo+w-1);
  for(let i=lo;i<=hi;i++){{const pi=i;btn(String(i+1),()=>{{page=pi;render();}},false,i===page);}}
  btn('>',  ()=>{{page++;render();}}, page>=tp-1);
  btn('>|', ()=>{{page=tp-1;render();}}, page>=tp-1);
}}

/* ── Dropdown de filtro ─────────────────────────────────────────────── */
function openFilter(colKey, anchor) {{
  const fdd = document.getElementById('fdd');

  // Toggle: fecha se já está aberta para esta coluna
  if (fddCol === colKey && fdd.classList.contains('open')) {{
    closeFilter(); return;
  }}

  fddCol = colKey;
  const colLabel = COLS.find(c=>c.key===colKey)?.label || colKey;
  document.getElementById('fdd-col-name').textContent = colLabel;
  document.getElementById('fdd-search').value = '';

  // Seleção temporária: cópia do filtro ativo
  fddSel = activeFilters[colKey]
    ? new Set(activeFilters[colKey])
    : new Set(uniqueVals[colKey]);  // se sem filtro, todos marcados

  buildList('');
  fdd.classList.add('open');
  positionDropdown(anchor);
}}

function positionDropdown(anchor) {{
  const fdd  = document.getElementById('fdd');
  const rect = anchor.getBoundingClientRect();
  fdd.style.top  = (rect.bottom + 2) + 'px';
  fdd.style.left = Math.min(rect.left, window.innerWidth - 290) + 'px';
}}

function buildList(search) {{
  const list  = document.getElementById('fdd-list');
  list.innerHTML = '';
  const vals  = uniqueVals[fddCol] || [];
  const lower = search.toLowerCase();
  const shown = search ? vals.filter(v => v.toLowerCase().includes(lower)) : vals;

  shown.forEach(v => {{
    const label = v === '' ? '(Sem valor)' : v;
    const item  = document.createElement('div');
    item.className = 'fdd-item';
    const chk = document.createElement('input');
    chk.type = 'checkbox';
    chk.checked = fddSel.has(v);
    chk.addEventListener('change', () => {{
      if (chk.checked) fddSel.add(v);
      else             fddSel.delete(v);
    }});
    const lbl = document.createElement('label');
    lbl.textContent = label;
    lbl.addEventListener('click', () => {{
      chk.checked = !chk.checked;
      if (chk.checked) fddSel.add(v);
      else             fddSel.delete(v);
    }});
    item.appendChild(chk);
    item.appendChild(lbl);
    list.appendChild(item);
  }});
}}

function closeFilter() {{
  const fdd = document.getElementById('fdd');
  fdd.classList.remove('open');
  fddCol = null;
}}

function applyFilter() {{
  if (!fddCol) return;
  const allVals = new Set(uniqueVals[fddCol] || []);
  // Se todas as opções estão marcadas, remove o filtro (= sem restrição)
  const allSelected = [...allVals].every(v => fddSel.has(v));
  if (allSelected || fddSel.size === 0) {{
    delete activeFilters[fddCol];
  }} else {{
    activeFilters[fddCol] = new Set(fddSel);
  }}
  closeFilter();
  page = 0;
  render();
}}

// Eventos do dropdown
document.getElementById('fdd-apply').addEventListener('click', applyFilter);
document.getElementById('fdd-selall').addEventListener('click', () => {{
  fddSel = new Set(uniqueVals[fddCol] || []);
  buildList(document.getElementById('fdd-search').value);
}});
document.getElementById('fdd-clear').addEventListener('click', () => {{
  fddSel = new Set();
  buildList(document.getElementById('fdd-search').value);
}});
document.getElementById('fdd-search').addEventListener('input', e => {{
  buildList(e.target.value);
}});

// Fecha dropdown ao clicar fora
document.addEventListener('click', e => {{
  const fdd = document.getElementById('fdd');
  if (!fdd.contains(e.target) && !e.target.closest('.th-filter')) {{
    if (fdd.classList.contains('open')) closeFilter();
  }}
}});

/* ── postMessage para Streamlit ─────────────────────────────────────── */
function notifyParent(colName) {{
  window.parent.postMessage({{type:'col_click',col:colName}}, '*');
}}

/* ── Utilitário ─────────────────────────────────────────────────────── */
function esc(s) {{
  return String(s)
    .replace(/&/g,'&amp;').replace(/</g,'&lt;')
    .replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}}

/* ── Render principal ───────────────────────────────────────────────── */
function render() {{
  renderHead();
  renderBody();
  renderChips();
}}

render();
</script>
</body>
</html>"""

    # Encoda o HTML em base64 e usa um <iframe> nativo via st.markdown.
    # Isso evita o placeholder "Drag and drop file here" que o
    # st.components.v1.html() mostra enquanto o componente ainda está carregando.
    import base64
    html_b64 = base64.b64encode(html.encode("utf-8")).decode("utf-8")
    st.markdown(
        f'<iframe src="data:text/html;base64,{html_b64}" '
        f'width="100%" height="{height + 10}px" '
        f'style="border:none;display:block;overflow:hidden;" '
        f'scrolling="no" frameborder="0"></iframe>',
        unsafe_allow_html=True,
    )
