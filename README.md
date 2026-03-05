"""
ui/grid.py
==========
Grade corporativa com filtros por coluna, estilo Excel.
Renderizada via st.components.v1.html() — sem dependências externas de Grid.

Comportamento dos filtros:
  - Cada coluna possui um ícone discreto que abre um dropdown
  - O dropdown lista todos os valores únicos da coluna com checkboxes
  - "Selecionar tudo" e "Limpar" funcionam como no Excel
  - Filtros múltiplos são cumulativos (interseção)
  - A grade atualiza imediatamente ao confirmar

API pública:
  show_grid(df, key, title, height) -> pd.DataFrame
"""
from __future__ import annotations

import json
from typing import Optional
import pandas as pd
import streamlit as st
import streamlit.components.v1 as components

_CSS_VARS = """
  --bg:           #F8F9FA;
  --surface:      #FFFFFF;
  --border:       #D1D5DB;
  --border-light: #E9ECEF;
  --header-bg:    #1C2536;
  --header-text:  #FFFFFF;
  --text:         #111827;
  --text-muted:   #6B7280;
  --accent:       #1B5FBF;
  --accent-light: #EFF6FF;
  --accent-dark:  #1A54A8;
  --row-hover:    #F3F4F6;
  --row-alt:      #FAFAFA;
  --danger:       #DC2626;
"""


def _df_to_json(df: pd.DataFrame) -> str:
    safe = df.copy()
    for col in safe.columns:
        safe[col] = safe[col].astype(str).replace({"nan": "", "None": ""})
    return json.dumps(safe.to_dict(orient="records"), ensure_ascii=False)


def _cols_to_json(df: pd.DataFrame) -> str:
    cols = []
    for c in df.columns:
        cl = c.lower()
        if any(k in cl for k in ("rótulo", "rotulo", "denominação", "denominacao", "descrição")):
            w = 190
        elif any(k in cl for k in ("central", "operadora", "tipo", "cluster")):
            w = 130
        elif any(k in cl for k in ("uf", "rede")):
            w = 100
        elif any(k in cl for k in ("origem",)):
            w = 90
        else:
            w = 110
        cols.append({"key": c, "label": c, "width": w})
    return json.dumps(cols, ensure_ascii=False)


def _build_html(df: pd.DataFrame, height: int) -> str:
    data_json = _df_to_json(df)
    cols_json = _cols_to_json(df)

    return f"""<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap" rel="stylesheet">
<style>
  :root {{ {_CSS_VARS} }}
  * {{ box-sizing: border-box; margin: 0; padding: 0; }}
  body {{
    font-family: 'IBM Plex Sans', 'Segoe UI', 'Helvetica Neue', sans-serif;
    font-size: 12px;
    background: var(--bg);
    color: var(--text);
    overflow: hidden;
  }}
  #toolbar {{
    display: flex; align-items: center; justify-content: space-between;
    padding: 6px 10px; background: var(--surface);
    border-bottom: 1px solid var(--border); gap: 8px; flex-shrink: 0;
  }}
  #toolbar-left {{ display: flex; align-items: center; gap: 10px; }}
  #record-count {{ font-size: 11px; color: var(--text-muted); font-weight: 500; white-space: nowrap; }}
  #active-filters {{
    display: none; align-items: center; gap: 6px; padding: 3px 10px;
    background: var(--accent-light); border: 1px solid rgba(27,95,191,0.2);
    font-size: 11px; color: var(--accent); font-weight: 500;
  }}
  #active-filters.visible {{ display: flex; }}
  #clear-all-btn {{ font-weight: 700; cursor: pointer; margin-left: 4px; }}
  #clear-all-btn:hover {{ text-decoration: underline; }}
  #table-wrap {{ overflow: auto; flex: 1; }}
  table {{ border-collapse: collapse; table-layout: fixed; width: max-content; min-width: 100%; }}
  thead tr {{ position: sticky; top: 0; z-index: 10; }}
  th {{
    background: var(--header-bg); border-right: 1px solid #374151;
    border-bottom: 2px solid #374151; padding: 0; white-space: nowrap; user-select: none;
  }}
  th.col-num {{
    width: 44px; min-width: 44px; text-align: center; font-size: 10px;
    font-weight: 400; color: rgba(255,255,255,0.3); letter-spacing: 0.05em;
    padding: 0 4px; cursor: default;
  }}
  .th-inner {{ display: flex; align-items: stretch; height: 36px; }}
  .th-sort {{
    flex: 1; display: flex; align-items: center; padding: 0 8px; gap: 5px;
    cursor: pointer; overflow: hidden; min-width: 0;
  }}
  .th-sort:hover {{ background: rgba(255,255,255,0.06); }}
  .th-label {{
    font-size: 11px; font-weight: 700; color: var(--header-text);
    letter-spacing: 0.05em; text-transform: uppercase;
    overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
  }}
  .sort-icon {{ flex-shrink: 0; opacity: 0.3; }}
  .sort-icon.active {{ opacity: 1; }}
  .th-filter-btn {{
    width: 28px; flex-shrink: 0; display: flex; align-items: center;
    justify-content: center; cursor: pointer; border-left: 1px solid #374151;
  }}
  .th-filter-btn:hover {{ background: rgba(255,255,255,0.08); }}
  .th-filter-btn.active {{ background: var(--accent); }}
  .th-active-bar {{ height: 2px; background: var(--accent); display: none; }}
  th.filtered .th-active-bar {{ display: block; }}
  tbody tr:nth-child(even) {{ background: var(--row-alt); }}
  tbody tr:nth-child(odd)  {{ background: var(--surface); }}
  tbody tr:hover {{ background: var(--row-hover) !important; }}
  td {{
    border-bottom: 1px solid var(--border-light); border-right: 1px solid var(--border-light);
    padding: 5px 9px; font-size: 12px; color: var(--text);
    overflow: hidden; text-overflow: ellipsis; white-space: nowrap;
  }}
  td.col-num {{
    text-align: center; font-size: 10px; color: var(--text-muted);
    font-family: 'IBM Plex Mono', 'Consolas', monospace; padding: 5px 4px;
  }}
  .cell-mono {{ font-family: 'IBM Plex Mono', 'Consolas', monospace; font-size: 11px; }}
  .cell-smp  {{ color:#1B5FBF; font-weight:600; font-family:'IBM Plex Mono','Consolas',monospace; font-size:11px; }}
  .cell-stfc {{ color:#6D28D9; font-weight:600; font-family:'IBM Plex Mono','Consolas',monospace; font-size:11px; }}
  .badge {{
    display: inline-block; padding: 1px 7px; font-size: 10px; font-weight: 700;
    letter-spacing: 0.06em; text-transform: uppercase;
  }}
  .badge-science {{ background:#EFF6FF; color:#1D4ED8; border:1px solid rgba(29,78,216,.15); }}
  .badge-portal  {{ background:#F0FDF4; color:#15803D; border:1px solid rgba(21,128,61,.15); }}
  .badge-arq3    {{ background:#FEF9EF; color:#92400E; border:1px solid rgba(146,64,14,.15); }}
  td.empty-state {{ text-align:center; padding:28px 0; color:var(--text-muted); font-style:italic; border:none; }}
  #pagination {{
    display: flex; align-items: center; justify-content: space-between;
    padding: 6px 10px; background: var(--surface);
    border-top: 1px solid var(--border); flex-shrink: 0;
  }}
  #pag-info {{ font-size: 11px; color: var(--text-muted); }}
  #pag-controls {{ display: flex; gap: 3px; }}
  .pag-btn {{
    min-width: 26px; height: 24px; padding: 0 5px; border: 1px solid var(--border);
    background: var(--surface); color: var(--text-muted); font-size: 11px;
    cursor: pointer; font-family: inherit;
  }}
  .pag-btn:hover:not(:disabled) {{ border-color: var(--accent); color: var(--accent); }}
  .pag-btn.active {{ background: var(--accent); color: #fff; border-color: var(--accent); font-weight: 700; }}
  .pag-btn:disabled {{ opacity: 0.35; cursor: default; }}
  #filter-overlay {{ display:none; position:fixed; inset:0; z-index:999; }}
  #filter-dropdown {{
    display: none; position: fixed; z-index: 1000; width: 240px;
    background: var(--surface); border: 1px solid var(--border);
    box-shadow: 0 4px 18px rgba(0,0,0,0.14), 0 1px 4px rgba(0,0,0,0.08);
  }}
  #filter-dropdown.open, #filter-overlay.open {{ display: block; }}
  #drop-header {{ padding: 7px 10px 6px; background: #F3F4F6; border-bottom: 1px solid var(--border-light); }}
  #drop-title {{ font-size:10px; font-weight:700; letter-spacing:0.08em; text-transform:uppercase; color:var(--text-muted); margin-bottom:5px; }}
  #drop-search {{
    width: 100%; padding: 4px 7px; font-size: 12px; border: 1px solid var(--border);
    background: var(--surface); color: var(--text); outline: none; font-family: inherit;
  }}
  #drop-search:focus {{ border-color: var(--accent); }}
  #drop-select-all {{
    display: flex; align-items: center; gap: 8px; padding: 6px 10px; cursor: pointer;
    border-bottom: 1px solid var(--border-light); background: #FAFAFA;
    font-weight: 600; font-size: 12px;
  }}
  #drop-select-all:hover {{ background: var(--accent-light); }}
  #drop-list {{ max-height: 210px; overflow-y: auto; }}
  .drop-item {{
    display: flex; align-items: center; gap: 8px; padding: 5px 10px; cursor: pointer;
    border-bottom: 1px solid var(--border-light); font-size: 12px;
  }}
  .drop-item:hover {{ background: var(--accent-light); }}
  .drop-item:last-child {{ border-bottom: none; }}
  #drop-footer {{
    display: flex; align-items: center; justify-content: space-between;
    padding: 6px 10px; background: #F3F4F6; border-top: 1px solid var(--border);
  }}
  .drop-btn-clear {{
    padding: 3px 10px; font-size: 11px; font-weight: 600; border: 1px solid var(--border);
    background: var(--surface); color: var(--text-muted); cursor: pointer;
    font-family: inherit; letter-spacing: 0.03em; text-transform: uppercase;
  }}
  .drop-btn-clear:hover {{ color: var(--danger); border-color: var(--danger); }}
  .drop-btn-apply {{
    padding: 3px 14px; font-size: 11px; font-weight: 600; border: none;
    background: var(--accent); color: #fff; cursor: pointer;
    font-family: inherit; letter-spacing: 0.03em; text-transform: uppercase;
  }}
  .drop-btn-apply:hover {{ background: var(--accent-dark); }}
  .chk {{
    width:14px; height:14px; flex-shrink:0; border:1.5px solid var(--border);
    background:var(--surface); display:flex; align-items:center; justify-content:center;
    position:relative;
  }}
  .chk.on  {{ background:var(--accent); border-color:var(--accent); }}
  .chk.ind {{ border-color:var(--accent); }}
  .chk.on::after {{
    content:''; display:block; width:8px; height:5px;
    border-left:2px solid #fff; border-bottom:2px solid #fff;
    transform:rotate(-45deg) translateY(-1px);
  }}
  .chk.ind::after {{ content:''; display:block; width:8px; height:1.5px; background:var(--accent); }}
  #root {{ display:flex; flex-direction:column; height:{height}px; overflow:hidden; }}
</style>
</head>
<body>
<div id="root">
  <div id="toolbar">
    <div id="toolbar-left">
      <div id="record-count"></div>
      <div id="active-filters">
        <span id="active-filters-text"></span>
        <span id="clear-all-btn" onclick="clearAllFilters()">&#215; Limpar tudo</span>
      </div>
    </div>
  </div>
  <div id="table-wrap">
    <table id="main-table">
      <thead id="thead"></thead>
      <tbody id="tbody"></tbody>
    </table>
  </div>
  <div id="pagination">
    <div id="pag-info"></div>
    <div id="pag-controls"></div>
  </div>
</div>

<div id="filter-overlay" onclick="closeDropdown()"></div>
<div id="filter-dropdown">
  <div id="drop-header">
    <div id="drop-title">Filtrar</div>
    <input id="drop-search" type="text" placeholder="Pesquisar valor..." oninput="renderDropList()" autocomplete="off">
  </div>
  <div id="drop-select-all" onclick="toggleSelectAll()">
    <div id="chk-all" class="chk"></div>
    <span>Selecionar tudo</span>
  </div>
  <div id="drop-list"></div>
  <div id="drop-footer">
    <button class="drop-btn-clear" onclick="clearLocal()">Limpar</button>
    <button class="drop-btn-apply" onclick="applyFilter()">Aplicar</button>
  </div>
</div>

<script>
const ALL_DATA  = {data_json};
const COLUMNS   = {cols_json};
const PAGE_SIZE = 500;

let filters  = {{}};
let sortCol  = null;
let sortDir  = 'asc';
let page     = 0;
let openKey  = null;
let localSel = new Set();

function getFiltered() {{
  let rows = ALL_DATA;
  for (const [key, vals] of Object.entries(filters)) {{
    if (vals && vals.size > 0) rows = rows.filter(r => vals.has(String(r[key] ?? '')));
  }}
  if (sortCol) {{
    rows = [...rows].sort((a, b) => {{
      const av = String(a[sortCol] ?? ''), bv = String(b[sortCol] ?? '');
      return sortDir === 'asc' ? av.localeCompare(bv,'pt-BR') : bv.localeCompare(av,'pt-BR');
    }});
  }}
  return rows;
}}

function uniqueVals(key) {{
  return [...new Set(ALL_DATA.map(r => String(r[key] ?? '')))].sort((a,b)=>a.localeCompare(b,'pt-BR'));
}}

function renderHeader() {{
  const thead = document.getElementById('thead');
  thead.innerHTML = '';
  const tr = document.createElement('tr');
  const thNum = document.createElement('th');
  thNum.className = 'col-num'; thNum.textContent = '#';
  tr.appendChild(thNum);

  COLUMNS.forEach(col => {{
    const th = document.createElement('th');
    th.id = 'th-' + col.key;
    th.style.width = col.width + 'px'; th.style.minWidth = col.width + 'px';
    if (filters[col.key]) th.classList.add('filtered');

    const inner = document.createElement('div'); inner.className = 'th-inner';
    const sortZone = document.createElement('div'); sortZone.className = 'th-sort';
    sortZone.onclick = () => handleSort(col.key);
    const lbl = document.createElement('span'); lbl.className = 'th-label'; lbl.textContent = col.label;
    const ico = document.createElement('span');
    ico.className = 'sort-icon' + (sortCol === col.key ? ' active' : '');
    ico.innerHTML = sortCol === col.key
      ? (sortDir==='asc'
        ? '<svg width="9" height="11" viewBox="0 0 9 11" fill="none"><path d="M4.5 1v9M1.5 3L4.5 1 7.5 3" stroke="#fff" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M1.5 8L4.5 10 7.5 8" stroke="rgba(255,255,255,0.2)" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>'
        : '<svg width="9" height="11" viewBox="0 0 9 11" fill="none"><path d="M4.5 1v9M1.5 3L4.5 1 7.5 3" stroke="rgba(255,255,255,0.2)" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/><path d="M1.5 8L4.5 10 7.5 8" stroke="#fff" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>')
      : '<svg width="9" height="11" viewBox="0 0 9 11" fill="none"><path d="M4.5 1v9M1.5 3L4.5 1 7.5 3M1.5 8L4.5 10 7.5 8" stroke="rgba(255,255,255,0.25)" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>';
    sortZone.appendChild(lbl); sortZone.appendChild(ico); inner.appendChild(sortZone);

    const fb = document.createElement('div');
    fb.className = 'th-filter-btn' + (filters[col.key] ? ' active' : '');
    fb.title = 'Filtrar: ' + col.label;
    fb.innerHTML = '<svg width="11" height="11" viewBox="0 0 11 11" fill="none"><path d="M1 2h9M2.5 5.5h6M4 9h3" stroke="' + (filters[col.key] ? '#fff' : 'rgba(255,255,255,0.45)') + '" stroke-width="1.6" stroke-linecap="round"/></svg>';
    fb.onclick = (e) => {{ e.stopPropagation(); openDropdown(col.key, fb); }};
    inner.appendChild(fb);
    th.appendChild(inner);
    const bar = document.createElement('div'); bar.className = 'th-active-bar';
    th.appendChild(bar);
    tr.appendChild(th);
  }});
  thead.appendChild(tr);
}}

function esc(s) {{ return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }}

function renderBody() {{
  const filtered = getFiltered();
  const total = ALL_DATA.length, shown = filtered.length;
  const paged = filtered.slice(page * PAGE_SIZE, (page+1) * PAGE_SIZE);

  document.getElementById('record-count').textContent =
    shown === total ? total.toLocaleString('pt-BR') + ' registros'
                    : shown.toLocaleString('pt-BR') + ' de ' + total.toLocaleString('pt-BR') + ' registros';

  const ac = Object.keys(filters).length;
  const afEl = document.getElementById('active-filters');
  if (ac > 0) {{
    afEl.classList.add('visible');
    document.getElementById('active-filters-text').textContent =
      ac + ' filtro' + (ac>1?'s':'') + ' ativo' + (ac>1?'s':'');
  }} else {{ afEl.classList.remove('visible'); }}

  const tbody = document.getElementById('tbody'); tbody.innerHTML = '';
  if (paged.length === 0) {{
    const tr = document.createElement('tr');
    const td = document.createElement('td');
    td.colSpan = COLUMNS.length+1; td.className = 'empty-state';
    td.textContent = 'Nenhum registro corresponde aos filtros selecionados.';
    tr.appendChild(td); tbody.appendChild(tr);
  }} else {{
    paged.forEach((row, i) => {{
      const idx = page * PAGE_SIZE + i + 1;
      const tr = document.createElement('tr');
      const tdN = document.createElement('td'); tdN.className = 'col-num'; tdN.textContent = idx;
      tr.appendChild(tdN);
      COLUMNS.forEach(col => {{
        const td = document.createElement('td');
        const v = String(row[col.key] ?? ''); const kl = col.key.toLowerCase();
        if (kl === 'rede') {{
          td.innerHTML = '<span class="' + (v==='VIVO-SMP'?'cell-smp':'cell-stfc') + '">' + esc(v) + '</span>';
        }} else if (kl === 'origem') {{
          const cls = v==='Science'?'badge-science':v==='Portal'?'badge-portal':'badge-arq3';
          td.innerHTML = '<span class="badge ' + cls + '">' + esc(v) + '</span>';
        }} else if (kl.includes('rotul') || kl==='central') {{
          td.innerHTML = '<span class="cell-mono">' + esc(v) + '</span>';
        }} else {{ td.textContent = v; }}
        tr.appendChild(td);
      }});
      tbody.appendChild(tr);
    }});
  }}
  renderPagination(shown);
}}

function renderPagination(total) {{
  const tp = Math.max(1, Math.ceil(total/PAGE_SIZE));
  const s = total===0?0:page*PAGE_SIZE+1, e = Math.min((page+1)*PAGE_SIZE, total);
  document.getElementById('pag-info').textContent =
    'Exibindo ' + s + '–' + e + ' de ' + total.toLocaleString('pt-BR') + ' registros';
  const ctrl = document.getElementById('pag-controls'); ctrl.innerHTML = '';
  function btn(label, cb, dis, active) {{
    const b = document.createElement('button'); b.className='pag-btn'+(active?' active':'');
    b.textContent=label; b.disabled=dis; b.onclick=cb; ctrl.appendChild(b);
  }}
  btn('«',()=>{{page=0;render();}},page===0);
  btn('‹',()=>{{page--;render();}},page===0);
  const w=5; let lo=Math.max(0,page-Math.floor(w/2)); let hi=Math.min(tp-1,lo+w-1); lo=Math.max(0,hi-w+1);
  for(let i=lo;i<=hi;i++){{const pi=i;btn(String(i+1),()=>{{page=pi;render();}},false,i===page);}}
  btn('›',()=>{{page++;render();}},page>=tp-1);
  btn('»',()=>{{page=tp-1;render();}},page>=tp-1);
}}

function handleSort(key) {{
  if(sortCol===key) sortDir=sortDir==='asc'?'desc':'asc'; else{{sortCol=key;sortDir='asc';}}
  page=0; render();
}}

function openDropdown(key, anchorEl) {{
  if(openKey===key){{closeDropdown();return;}}
  openKey=key;
  const vals=uniqueVals(key);
  localSel=filters[key]?new Set(filters[key]):new Set(vals);
  document.getElementById('drop-title').textContent='Filtrar: '+key;
  document.getElementById('drop-search').value='';
  renderDropList();
  const rect=anchorEl.getBoundingClientRect();
  const drop=document.getElementById('filter-dropdown');
  drop.style.top=(rect.bottom+2)+'px';
  drop.style.left=Math.min(rect.left,window.innerWidth-250)+'px';
  drop.classList.add('open');
  document.getElementById('filter-overlay').classList.add('open');
  document.getElementById('drop-search').focus();
}}

function closeDropdown() {{
  openKey=null;
  document.getElementById('filter-dropdown').classList.remove('open');
  document.getElementById('filter-overlay').classList.remove('open');
}}

function renderDropList() {{
  if(!openKey) return;
  const search=document.getElementById('drop-search').value.toLowerCase();
  const all=uniqueVals(openKey);
  const shown=search?all.filter(v=>v.toLowerCase().includes(search)):all;
  const allSel=shown.every(v=>localSel.has(v)), anySel=shown.some(v=>localSel.has(v));
  const ca=document.getElementById('chk-all');
  ca.className='chk'+(allSel?' on':(anySel?' ind':''));
  const list=document.getElementById('drop-list'); list.innerHTML='';
  shown.forEach(v=>{{
    const item=document.createElement('div'); item.className='drop-item';
    item.onclick=()=>{{localSel.has(v)?localSel.delete(v):localSel.add(v);renderDropList();}};
    const chk=document.createElement('div'); chk.className='chk'+(localSel.has(v)?' on':'');
    const lbl=document.createElement('span'); lbl.textContent=v===''?'(vazio)':v;
    if(v==='') lbl.style.fontStyle='italic';
    item.appendChild(chk); item.appendChild(lbl); list.appendChild(item);
  }});
}}

function toggleSelectAll() {{
  const search=document.getElementById('drop-search').value.toLowerCase();
  const all=uniqueVals(openKey);
  const shown=search?all.filter(v=>v.toLowerCase().includes(search)):all;
  const allSel=shown.every(v=>localSel.has(v));
  shown.forEach(v=>allSel?localSel.delete(v):localSel.add(v));
  renderDropList();
}}

function clearLocal()  {{ localSel=new Set(); renderDropList(); }}
function clearAllFilters() {{ filters={{}}; page=0; render(); }}

function applyFilter() {{
  if(!openKey) return;
  const all=uniqueVals(openKey);
  if(localSel.size===0||all.every(v=>localSel.has(v))) delete filters[openKey];
  else filters[openKey]=new Set(localSel);
  page=0; closeDropdown(); render();
}}

function render() {{ renderHeader(); renderBody(); }}
render();
</script>
</body>
</html>"""


def show_grid(
    df: pd.DataFrame,
    key: str = "grid",
    title: str = "",
    height: int = 540,
) -> pd.DataFrame:
    """
    Exibe grade corporativa com filtros estilo Excel.
    Retorna o DataFrame original (filtragem ocorre no cliente).
    """
    if title:
        st.markdown(f"#### {title}")

    display_cols = [c for c in df.columns if not c.startswith("_")]
    df_disp = df[display_cols].copy()

    if df_disp.empty:
        st.info("Nenhum dado para exibir.")
        return df_disp

    html = _build_html(df_disp, height=height)
    components.html(html, height=height + 60, scrolling=False)
    return df_disp

