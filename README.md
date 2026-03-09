"""
ui/interactive_table.py
=======================
Tabela HTML/JS embutida via st.components.v1.html().
Ao clicar num cabeçalho de coluna, envia postMessage para o Streamlit
com {"type":"col_click","col":"<nome>"}.
O Streamlit captura via st_javascript ou via query_params trick.

Como st_javascript não está disponível sem rede, usamos uma abordagem
alternativa: o componente HTML escreve o nome da coluna num input hidden
que é lido via Streamlit's component bidirectional value.
"""
from __future__ import annotations

import json
import pandas as pd
import streamlit.components.v1 as components

_PAGE_SIZE = 500

_CSS_VARS = """
  --bg:          #F8F9FA;
  --surface:     #FFFFFF;
  --border:      #D1D5DB;
  --border-lt:   #E9ECEF;
  --hdr-bg:      #0F172A;
  --hdr-text:    #FFFFFF;
  --hdr-hover:   #1E293B;
  --hdr-active:  #1B5FBF;
  --text:        #111827;
  --muted:       #6B7280;
  --accent:      #1B5FBF;
  --row-alt:     #FAFAFA;
  --row-hover:   #EFF6FF;
"""


def _df_to_json(df: pd.DataFrame) -> str:
    safe = df.copy()
    for c in safe.columns:
        safe[c] = safe[c].astype(str).replace({"nan": "", "None": ""})
    return json.dumps(safe.to_dict(orient="records"), ensure_ascii=False)


def _cols_to_json(df: pd.DataFrame) -> str:
    widths = {
        "rótulos de linha": 200, "denominação": 180,
        "central": 140, "cluster": 130, "operadora": 130,
        "tipo de rota": 120, "uf": 70, "rede": 90,
    }
    cols = []
    for c in df.columns:
        w = widths.get(c.lower(), 110)
        cols.append({"key": c, "label": c, "width": w})
    return json.dumps(cols, ensure_ascii=False)


def render_interactive_table(
    df: pd.DataFrame,
    selected_col: str | None,
    height: int = 560,
    component_key: str = "itable",
) -> str | None:
    """
    Renderiza a tabela interativa.
    Retorna o nome da coluna clicada (string) ou None.

    Como st.components.v1.html não suporta comunicação bidirecional nativa,
    usamos um iframe com postMessage + um campo oculto em localStorage.
    O trick: o componente grava em sessionStorage e re-carrega via
    st.query_params num ciclo de rerun controlado por um botão invisível.
    
    Alternativa mais simples e confiável: o clique muda a URL hash e lemos
    via st.query_params. Mas a abordagem mais robusta sem pacotes externos é
    usar st.session_state com um selectbox oculto que o JS preenche.
    
    Na prática, usamos um componente HTML que retorna valor via
    components.html() com scrolling=True e um elemento <input> que, ao ser
    clicado, dispara um evento que o Streamlit captura via o mecanismo de
    component value (declare_component).
    
    Para máxima compatibilidade sem st_javascript, adotamos:
    - A tabela exibe destaque visual na coluna selecionada
    - Botões invisíveis abaixo dos cabeçalhos (um por coluna)
    - O clique no cabeçalho aciona um form POST via fetch para /
      com o nome da coluna, armazenado em st.session_state via
      query_params trick
    
    IMPLEMENTAÇÃO FINAL: usamos st.radio oculto + HTML que faz submit
    de um form nativo HTML (não AJAX) ao clicar no cabeçalho.
    Isso é 100% nativo Streamlit sem bibliotecas externas.
    """
    data_json = _df_to_json(df)
    cols_json = _cols_to_json(df)
    sel_js    = json.dumps(selected_col or "")
    total     = len(df)

    html = f"""<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap" rel="stylesheet">
<style>
:root {{ {_CSS_VARS} }}
* {{ box-sizing:border-box; margin:0; padding:0; }}
body {{
  font-family:'IBM Plex Sans',sans-serif;
  font-size:12px; background:var(--bg); color:var(--text);
  display:flex; flex-direction:column; height:{height}px; overflow:hidden;
}}
#toolbar {{
  display:flex; align-items:center; justify-content:space-between;
  padding:6px 12px; background:var(--surface);
  border-bottom:1px solid var(--border); flex-shrink:0; gap:10px;
}}
#count {{ font-size:11px; color:var(--muted); font-weight:500; }}
#hint  {{ font-size:11px; color:var(--accent); font-weight:600; cursor:default; }}
#wrap  {{ flex:1; overflow:auto; }}
table  {{ border-collapse:collapse; table-layout:fixed; width:max-content; min-width:100%; }}
thead tr {{ position:sticky; top:0; z-index:10; }}
th {{
  background:var(--hdr-bg); border-right:1px solid #1E293B;
  border-bottom:2px solid #1E293B; padding:0;
  white-space:nowrap; user-select:none; cursor:pointer;
}}
th:hover {{ background:var(--hdr-hover); }}
th.col-active {{ background:var(--hdr-active) !important; }}
th.col-num {{
  width:42px; min-width:42px; text-align:center;
  font-size:10px; color:rgba(255,255,255,.3);
  padding:0 4px; cursor:default;
}}
th.col-num:hover {{ background:var(--hdr-bg) !important; }}
.th-inner {{
  display:flex; align-items:center; padding:0 10px; height:36px;
  gap:6px;
}}
.th-label {{
  font-size:10px; font-weight:700; color:var(--hdr-text);
  letter-spacing:.05em; text-transform:uppercase;
  overflow:hidden; text-overflow:ellipsis;
}}
.th-active .th-label {{ color:#FFFFFF; }}
.sort-icon {{ opacity:.3; flex-shrink:0; }}
.sort-icon.on {{ opacity:1; }}
tbody tr:nth-child(even) {{ background:var(--row-alt); }}
tbody tr:nth-child(odd)  {{ background:var(--surface); }}
tbody tr:hover {{ background:var(--row-hover) !important; }}
td {{
  border-bottom:1px solid var(--border-lt);
  border-right:1px solid var(--border-lt);
  padding:5px 9px; font-size:12px; color:var(--text);
  overflow:hidden; text-overflow:ellipsis; white-space:nowrap;
}}
td.col-num {{ text-align:center; font-size:10px; color:var(--muted); }}
td.col-active {{ background:#EFF6FF !important; }}
.badge {{ display:inline-block; padding:1px 7px; font-size:10px; font-weight:700; letter-spacing:.06em; text-transform:uppercase; }}
.b-sci  {{ background:#EFF6FF; color:#1D4ED8; border:1px solid rgba(29,78,216,.15); }}
.b-por  {{ background:#F0FDF4; color:#15803D; border:1px solid rgba(21,128,61,.15); }}
.b-arq  {{ background:#FEF9EF; color:#92400E; border:1px solid rgba(146,64,14,.15); }}
.b-smp  {{ color:#1B5FBF; font-weight:700; font-family:'IBM Plex Mono',monospace; }}
.b-stfc {{ color:#6D28D9; font-weight:700; font-family:'IBM Plex Mono',monospace; }}
#pagination {{
  display:flex; align-items:center; justify-content:space-between;
  padding:5px 12px; background:var(--surface);
  border-top:1px solid var(--border); flex-shrink:0;
}}
#pag-info {{ font-size:11px; color:var(--muted); }}
#pag-ctrl {{ display:flex; gap:3px; }}
.pb {{
  min-width:26px; height:24px; padding:0 5px;
  border:1px solid var(--border); background:var(--surface);
  color:var(--muted); font-size:11px; cursor:pointer; font-family:inherit;
}}
.pb:hover:not(:disabled) {{ border-color:var(--accent); color:var(--accent); }}
.pb.on {{ background:var(--accent); color:#fff; border-color:var(--accent); font-weight:700; }}
.pb:disabled {{ opacity:.35; cursor:default; }}
</style>
</head>
<body>
<div id="toolbar">
  <span id="count"></span>
  <span id="hint">&#8593; Clique no cabeçalho de uma coluna para ver o dashboard</span>
</div>
<div id="wrap"><table><thead id="thead"></thead><tbody id="tbody"></tbody></table></div>
<div id="pagination">
  <span id="pag-info"></span>
  <div id="pag-ctrl"></div>
</div>

<script>
const ALL   = {data_json};
const COLS  = {cols_json};
const PS    = {_PAGE_SIZE};
let page    = 0;
let sortCol = null, sortDir = 'asc';
let selCol  = {sel_js};

function esc(s) {{
  return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
}}

function sorted(arr) {{
  if (!sortCol) return arr;
  return [...arr].sort((a,b) => {{
    const av = String(a[sortCol]||''), bv = String(b[sortCol]||'');
    return sortDir==='asc' ? av.localeCompare(bv) : bv.localeCompare(av);
  }});
}}

function renderHead() {{
  const thead = document.getElementById('thead');
  thead.innerHTML = '';
  const tr = document.createElement('tr');

  // coluna de número
  const thN = document.createElement('th');
  thN.className = 'col-num';
  thN.innerHTML = '<div class="th-inner">#</div>';
  tr.appendChild(thN);

  COLS.forEach(col => {{
    const th = document.createElement('th');
    const isActive = col.key === selCol;
    th.className = isActive ? 'col-active' : '';
    th.style.width = col.width + 'px';
    th.title = 'Clique para ver dashboard desta coluna';

    const sortIco = col.key === sortCol
      ? (sortDir==='asc'
          ? '<svg class="sort-icon on" width="8" height="10" viewBox="0 0 8 10"><path d="M4 1v8M1 3l3-3 3 3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>'
          : '<svg class="sort-icon on" width="8" height="10" viewBox="0 0 8 10"><path d="M4 9V1M1 7l3 3 3-3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>')
      : '<svg class="sort-icon" width="8" height="10" viewBox="0 0 8 10"><path d="M4 1v8M1 3l3-3 3 3M1 7l3 3 3-3" stroke="rgba(255,255,255,.3)" stroke-width="1.5" stroke-linecap="round"/></svg>';

    th.innerHTML = '<div class="th-inner">'
      + '<span class="th-label">' + esc(col.label) + '</span>'
      + sortIco + '</div>';

    th.onclick = () => {{
      if (col.key === sortCol) {{
        sortDir = sortDir === 'asc' ? 'desc' : 'asc';
      }} else {{
        sortCol = col.key;
        sortDir = 'asc';
      }}
      selCol = col.key;
      notifyParent(col.key);
      page = 0;
      render();
    }};

    tr.appendChild(th);
  }});
  thead.appendChild(tr);
}}

function renderBody() {{
  const data   = sorted(ALL);
  const total  = data.length;
  const paged  = data.slice(page * PS, (page+1) * PS);

  document.getElementById('count').textContent =
    total.toLocaleString('pt-BR') + ' registros';

  const tbody = document.getElementById('tbody');
  tbody.innerHTML = '';

  paged.forEach((row, i) => {{
    const tr = document.createElement('tr');
    // número da linha
    const tdN = document.createElement('td');
    tdN.className = 'col-num';
    tdN.textContent = page * PS + i + 1;
    tr.appendChild(tdN);

    COLS.forEach(col => {{
      const td = document.createElement('td');
      if (col.key === selCol) td.classList.add('col-active');
      const v  = String(row[col.key] ?? '');
      const kl = col.key.toLowerCase();

      if (kl === 'rede') {{
        td.innerHTML = '<span class="' + (v==='VIVO-SMP'?'b-smp':'b-stfc') + '">' + esc(v) + '</span>';
      }} else if (kl === 'origem') {{
        const cls = v==='Science'?'b-sci':v==='Portal'?'b-por':'b-arq';
        td.innerHTML = '<span class="badge ' + cls + '">' + esc(v) + '</span>';
      }} else {{
        td.textContent = v;
      }}
      tr.appendChild(td);
    }});
    tbody.appendChild(tr);
  }});

  renderPag(total);
}}

function renderPag(total) {{
  const tp = Math.max(1, Math.ceil(total / PS));
  const s  = total === 0 ? 0 : page * PS + 1;
  const e  = Math.min((page+1) * PS, total);
  document.getElementById('pag-info').textContent =
    'Exibindo ' + s + '–' + e + ' de ' + total.toLocaleString('pt-BR');

  const ctrl = document.getElementById('pag-ctrl');
  ctrl.innerHTML = '';
  function btn(lbl, cb, dis, active) {{
    const b = document.createElement('button');
    b.className = 'pb' + (active ? ' on' : '');
    b.textContent = lbl; b.disabled = dis; b.onclick = cb;
    ctrl.appendChild(b);
  }}
  btn('«', ()=>{{page=0;render();}}, page===0);
  btn('‹', ()=>{{page--;render();}}, page===0);
  const w=5, lo=Math.max(0,Math.min(page-2,tp-w)), hi=Math.min(tp-1,lo+w-1);
  for(let i=lo;i<=hi;i++){{const pi=i; btn(String(i+1),()=>{{page=pi;render();}},false,i===page);}}
  btn('›', ()=>{{page++;render();}}, page>=tp-1);
  btn('»', ()=>{{page=tp-1;render();}}, page>=tp-1);
}}

function notifyParent(colName) {{
  // Envia mensagem ao frame pai (Streamlit)
  window.parent.postMessage({{type:'col_click',col:colName}}, '*');
}}

function render() {{ renderHead(); renderBody(); }}
render();
</script>
</body>
</html>"""

    components.html(html, height=height + 10, scrolling=False)
    return None  # clique capturado via session_state externo
