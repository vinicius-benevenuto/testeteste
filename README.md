"""
ui/interactive_table.py
=======================
Tabela com filtros inline estilo Excel.
Download do filtrado via query_params do Streamlit.
"""
from __future__ import annotations

import json
import io
from datetime import datetime
import pandas as pd
import streamlit as st
import streamlit.components.v1 as components
from openpyxl import Workbook
from openpyxl.styles import Alignment, Border, Font, PatternFill, Side
from openpyxl.utils import get_column_letter

_PAGE_SIZE = 500

_CSS_VARS = """
  --bg:        #F8F9FA;
  --surface:   #FFFFFF;
  --border:    #D1D5DB;
  --border-lt: #E9ECEF;
  --hdr-bg:    #0F172A;
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
        "rotulos de linha": 200, "denominacao": 180,
        "central": 145, "cluster": 120, "operadora": 130,
        "tipo de rota": 125, "uf": 70, "rede": 100,
    }
    return json.dumps(
        [{"key": c, "label": c,
          "width": widths.get(
              c.lower().replace("ó","o").replace("ã","a").replace("ç","c").replace("é","e"),
              115)}
         for c in df.columns],
        ensure_ascii=False,
    )


def _make_excel(df: pd.DataFrame) -> bytes:
    wb = Workbook()
    ws = wb.active
    ws.title = "Tabela Filtrada"
    F_HEADER  = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
    F_CELL    = Font(name="Calibri", size=10, color="111827")
    F_TITLE   = Font(name="Calibri", size=12, bold=True, color="0F172A")
    FILL_HEAD = PatternFill("solid", fgColor="0F172A")
    FILL_TTL  = PatternFill("solid", fgColor="EFF6FF")
    FILL_EVEN = PatternFill("solid", fgColor="F1F5F9")
    FILL_ODD  = PatternFill("solid", fgColor="FFFFFF")
    TH  = Side(style="thin",   color="D1D5DB")
    ACC = Side(style="medium", color="1B5FBF")
    B_HEAD = Border(top=ACC, bottom=ACC, left=TH, right=TH)
    B_CELL = Border(top=TH,  bottom=TH,  left=TH, right=TH)
    B_TTL  = Border(top=ACC, bottom=ACC, left=ACC, right=ACC)
    n_cols   = len(df.columns)
    last_col = get_column_letter(n_cols)
    ws.merge_cells(f"A1:{last_col}1")
    tc = ws["A1"]
    tc.value = (f"ITX Analisys  |  {datetime.now().strftime('%d/%m/%Y %H:%M')}"
                f"  |  {len(df):,} rotas")
    tc.font = F_TITLE; tc.fill = FILL_TTL; tc.border = B_TTL
    tc.alignment = Alignment(horizontal="center", vertical="center")
    ws.row_dimensions[1].height = 26
    for ci, col in enumerate(df.columns, 1):
        c = ws.cell(row=2, column=ci, value=col)
        c.font = F_HEADER; c.fill = FILL_HEAD; c.border = B_HEAD
        c.alignment = Alignment(horizontal="center", vertical="center")
    ws.row_dimensions[2].height = 22
    for ri, (_, row) in enumerate(df.iterrows(), 3):
        fill = FILL_EVEN if ri % 2 == 0 else FILL_ODD
        for ci, col in enumerate(df.columns, 1):
            raw = row[col]
            try:
                val = "" if (raw is None or (not isinstance(raw, str) and pd.isna(raw))) else str(raw)
            except Exception:
                val = "" if raw is None else str(raw)
            c = ws.cell(row=ri, column=ci, value=val)
            c.font = F_CELL; c.fill = fill; c.border = B_CELL
            c.alignment = Alignment(horizontal="left", vertical="center")
        ws.row_dimensions[ri].height = 18
    ws.freeze_panes = "A3"
    for ci, col in enumerate(df.columns, 1):
        try:
            ml = max(len(str(col)), int(df[col].fillna("").astype(str).str.len().max() or 0))
        except Exception:
            ml = len(str(col))
        ws.column_dimensions[get_column_letter(ci)].width = min(55, max(10, int(ml * 1.12) + 2))
    ws.auto_filter.ref = f"A2:{last_col}2"
    buf = io.BytesIO()
    wb.save(buf)
    return buf.getvalue()


def render_interactive_table(
    df: pd.DataFrame,
    selected_col: str | None,
    height: int = 560,
    component_key: str = "itable",
) -> None:

    # ── Verifica download pendente (gravado via query_params pelo JS) ──────
    _dl_key  = f"_dl_{component_key}"
    _dlc_key = f"_dlc_{component_key}"
    qp = st.query_params
    if _dl_key in qp and _dlc_key in qp:
        try:
            rows = json.loads(qp[_dl_key])
            cols = json.loads(qp[_dlc_key])
            df_dl = pd.DataFrame(rows, columns=cols)
            xlsx  = _make_excel(df_dl)
            ts    = datetime.now().strftime("%Y-%m-%d_%H%M")
            del qp[_dl_key]
            del qp[_dlc_key]
            st.session_state[f"{component_key}_xlsx"] = xlsx
            st.session_state[f"{component_key}_ts"]   = ts
            st.session_state[f"{component_key}_n"]    = len(df_dl)
        except Exception as e:
            st.error(f"Erro ao preparar Excel: {e}")

    # Mostra botao de download se Excel esta pronto
    _xls_state = st.session_state.get(f"{component_key}_xlsx")
    if _xls_state is not None:
        n   = st.session_state.get(f"{component_key}_n", "?")
        ts  = st.session_state.get(f"{component_key}_ts", "")
        c1, c2 = st.columns([3, 1])
        with c1:
            st.download_button(
                label=f"Baixar Excel filtrado  ({n:,} linhas)" if isinstance(n, int) else "Baixar Excel filtrado",
                data=_xls_state,
                file_name=f"tabela_filtrada_{ts}.xlsx",
                mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                key=f"{component_key}_dl_btn",
                type="primary",
                use_container_width=True,
            )
        with c2:
            if st.button("Fechar", key=f"{component_key}_dl_close", use_container_width=True):
                st.session_state[f"{component_key}_xlsx"] = None
                st.rerun()

    data_json = _df_to_json(df)
    cols_json = _cols_to_json(df)
    sel_js    = json.dumps(selected_col or "")
    dl_key_js = json.dumps(_dl_key)
    dlc_key_js= json.dumps(_dlc_key)

    html = f"""<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap" rel="stylesheet">
<style>
:root {{{_CSS_VARS}}}
*{{box-sizing:border-box;margin:0;padding:0;}}
body{{font-family:'IBM Plex Sans',sans-serif;font-size:12px;background:var(--bg);color:var(--text);display:flex;flex-direction:column;height:{height}px;overflow:hidden;}}
#toolbar{{display:flex;align-items:center;justify-content:space-between;padding:6px 12px;background:var(--surface);border-bottom:1px solid var(--border);flex-shrink:0;gap:10px;}}
#count{{font-size:11px;color:var(--muted);font-weight:500;}}
#hint{{font-size:11px;color:var(--accent);font-weight:500;}}
#wrap{{flex:1;overflow:auto;}}
table{{border-collapse:collapse;table-layout:fixed;width:max-content;min-width:100%;}}
thead tr{{position:sticky;top:0;z-index:20;}}
th{{background:var(--hdr-bg);border-right:1px solid #1E293B;border-bottom:2px solid #1E293B;white-space:nowrap;user-select:none;padding:0;}}
th.col-num{{width:42px;min-width:42px;font-size:10px;color:rgba(255,255,255,.25);padding:0 4px;text-align:center;cursor:default;}}
th.col-sel{{background:var(--hdr-sel)!important;}}
.th-wrap{{display:flex;align-items:stretch;height:36px;}}
.th-name{{flex:1;display:flex;align-items:center;padding:0 8px;gap:5px;cursor:pointer;min-width:0;border:none;background:transparent;}}
.th-name:hover{{background:var(--hdr-hover);}}
.th-label{{font-size:10px;font-weight:700;color:#FFF;letter-spacing:.05em;text-transform:uppercase;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
.sort-ico{{opacity:.25;flex-shrink:0;}}.sort-ico.on{{opacity:1;}}
.th-filter{{display:flex;align-items:center;justify-content:center;width:24px;min-width:24px;cursor:pointer;flex-shrink:0;border:none;background:transparent;font-size:9px;color:rgba(255,255,255,.35);transition:color .15s,background .15s;}}
.th-filter:hover{{background:rgba(255,255,255,.1);color:#fff;}}
.th-filter.active{{color:#60A5FA;background:rgba(27,95,191,.25);}}
.fdd{{position:fixed;z-index:9999;background:#FFF;border:1px solid #D1D5DB;box-shadow:0 4px 20px rgba(0,0,0,.14);border-radius:4px;min-width:200px;max-width:280px;display:none;overflow:hidden;}}
.fdd.open{{display:block;}}
.fdd-head{{display:flex;align-items:center;justify-content:space-between;padding:8px 10px;border-bottom:1px solid #E9ECEF;background:#F8F9FA;}}
.fdd-title{{font-size:11px;font-weight:700;color:#1C2536;text-transform:uppercase;letter-spacing:.05em;}}
.fdd-actions{{display:flex;gap:6px;}}
.fdd-btn{{font-size:10px;color:#1B5FBF;cursor:pointer;background:none;border:none;font-family:inherit;font-weight:600;padding:0;}}
.fdd-btn:hover{{text-decoration:underline;}}
.fdd-search{{padding:6px 10px;border-bottom:1px solid #E9ECEF;}}
.fdd-search input{{width:100%;padding:4px 8px;font-size:11px;border:1px solid #D1D5DB;border-radius:3px;font-family:inherit;outline:none;color:#1C2536;}}
.fdd-search input:focus{{border-color:#1B5FBF;}}
.fdd-list{{max-height:200px;overflow-y:auto;padding:4px 0;}}
.fdd-item{{display:flex;align-items:center;gap:8px;padding:5px 10px;cursor:pointer;font-size:12px;color:#374151;}}
.fdd-item:hover{{background:#EFF6FF;}}
.fdd-item input[type=checkbox]{{width:13px;height:13px;accent-color:#1B5FBF;cursor:pointer;flex-shrink:0;}}
.fdd-item label{{cursor:pointer;flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
.fdd-footer{{padding:7px 10px;border-top:1px solid #E9ECEF;display:flex;justify-content:flex-end;}}
.fdd-apply{{font-size:11px;font-weight:700;background:#1B5FBF;color:#fff;border:none;border-radius:3px;padding:4px 14px;cursor:pointer;font-family:inherit;}}
.fdd-apply:hover{{background:#1549A0;}}
#active-filters{{display:flex;align-items:center;gap:6px;flex-wrap:wrap;padding:4px 12px;background:var(--surface);border-bottom:1px solid var(--border);flex-shrink:0;}}
#active-filters:empty{{display:none;}}
.af-chip{{display:inline-flex;align-items:center;gap:4px;background:#EFF6FF;border:1px solid #BFDBFE;border-radius:2px;padding:2px 8px;font-size:10px;color:#1B5FBF;font-weight:600;}}
.af-x{{cursor:pointer;font-size:12px;line-height:1;color:#1B5FBF;margin-left:2px;}}
.af-x:hover{{color:#DC2626;}}
tbody tr:nth-child(even){{background:var(--row-alt);}}
tbody tr:nth-child(odd){{background:var(--surface);}}
tbody tr:hover{{background:var(--row-hover)!important;}}
td{{border-bottom:1px solid var(--border-lt);border-right:1px solid var(--border-lt);padding:5px 9px;font-size:12px;color:var(--text);overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
td.col-num{{text-align:center;font-size:10px;color:var(--muted);}}
td.col-sel{{background:#EFF6FF!important;}}
.b-smp{{color:#1B5FBF;font-weight:700;font-family:'IBM Plex Mono',monospace;}}
.b-stfc{{color:#6D28D9;font-weight:700;font-family:'IBM Plex Mono',monospace;}}
#pagination{{display:flex;align-items:center;justify-content:space-between;padding:5px 12px;background:var(--surface);border-top:1px solid var(--border);flex-shrink:0;}}
#pag-info{{font-size:11px;color:var(--muted);}}
#pag-ctrl{{display:flex;gap:3px;}}
.pb{{min-width:26px;height:24px;padding:0 5px;border:1px solid var(--border);background:var(--surface);color:var(--muted);font-size:11px;cursor:pointer;font-family:inherit;}}
.pb:hover:not(:disabled){{border-color:var(--accent);color:var(--accent);}}
.pb.on{{background:var(--accent);color:#fff;border-color:var(--accent);font-weight:700;}}
.pb:disabled{{opacity:.35;cursor:default;}}
#export-bar{{display:flex;align-items:center;gap:8px;flex-shrink:0;padding:6px 12px;background:#F8F9FA;border-top:1px solid var(--border);}}
#export-label{{font-size:9px;font-weight:700;letter-spacing:.08em;text-transform:uppercase;color:var(--muted);}}
.exp-btn{{font-size:11px;font-weight:600;padding:4px 14px;border-radius:3px;cursor:pointer;font-family:inherit;border:1px solid var(--border);background:#fff;color:#374151;transition:background .12s,border-color .12s,color .12s;}}
.exp-btn:hover{{background:var(--row-hover);border-color:var(--accent);color:var(--accent);}}
</style>
</head>
<body>

<script>
(function(){{
  try {{
    var pd=window.parent.document;
    if(pd.getElementById('__itx_css')) return;
    var s=pd.createElement('style'); s.id='__itx_css';
    s.textContent='[data-testid="stComponentContainer"] [data-testid="stFileUploaderDropzone"]{{display:none!important}}[data-testid="stComponentContainer"] section{{display:none!important}}[data-testid="stComponentContainer"] iframe{{display:block!important}}';
    pd.head.appendChild(s);
  }}catch(e){{}}
}})();
</script>

<div id="toolbar">
  <span id="count"></span>
  <span id="hint">Clique no nome da coluna para ver o dashboard. Use o icone de filtro para filtrar.</span>
</div>
<div id="active-filters"></div>
<div id="wrap"><table><thead id="thead"></thead><tbody id="tbody"></tbody></table></div>
<div id="pagination"><span id="pag-info"></span><div id="pag-ctrl"></div></div>
<div id="export-bar">
  <span id="export-label">Exportar filtrado:</span>
  <button class="exp-btn" id="btn-csv">Baixar CSV</button>
  <button class="exp-btn" id="btn-xlsx">Gerar Excel</button>
</div>

<div class="fdd" id="fdd">
  <div class="fdd-head"><span class="fdd-title" id="fdd-col-name"></span><div class="fdd-actions"><button class="fdd-btn" id="fdd-selall">Marcar todos</button><button class="fdd-btn" id="fdd-clear">Limpar</button></div></div>
  <div class="fdd-search"><input type="text" id="fdd-search" placeholder="Pesquisar..."></div>
  <div class="fdd-list" id="fdd-list"></div>
  <div class="fdd-footer"><button class="fdd-apply" id="fdd-apply">Aplicar</button></div>
</div>

<script>
const ALL_ROWS={data_json};
const COLS={cols_json};
const PS={_PAGE_SIZE};
const DL_KEY={dl_key_js};
const DLC_KEY={dlc_key_js};
let selCol={sel_js},sortCol=null,sortDir='asc',page=0;
const activeFilters={{}};
const uniqueVals={{}};
COLS.forEach(c=>{{ uniqueVals[c.key]=[...new Set(ALL_ROWS.map(r=>String(r[c.key]??'')))].sort(); }});
let fddCol=null,fddSel=new Set();

function getFiltered(){{
  let rows=ALL_ROWS;
  for(const[col,vals]of Object.entries(activeFilters)){{
    if(vals.size===0)continue;
    rows=rows.filter(r=>vals.has(String(r[col]??'')));
  }}
  if(sortCol){{
    rows=[...rows].sort((a,b)=>{{
      const av=String(a[sortCol]||''),bv=String(b[sortCol]||'');
      return sortDir==='asc'?av.localeCompare(bv,'pt'):bv.localeCompare(av,'pt');
    }});
  }}
  return rows;
}}

function renderHead(){{
  const thead=document.getElementById('thead');
  thead.innerHTML='';
  const tr=document.createElement('tr');
  const thN=document.createElement('th');
  thN.className='col-num';
  thN.innerHTML='<div style="height:36px;display:flex;align-items:center;justify-content:center;font-size:10px;color:rgba(255,255,255,.25)">#</div>';
  tr.appendChild(thN);
  COLS.forEach(col=>{{
    const th=document.createElement('th');
    th.style.width=col.width+'px';
    if(col.key===selCol)th.className='col-sel';
    const hasF=activeFilters[col.key]&&activeFilters[col.key].size>0;
    const sico=col.key===sortCol
      ?(sortDir==='asc'
        ?'<svg class="sort-ico on" width="8" height="10" viewBox="0 0 8 10"><path d="M4 1v8M1 3l3-3 3 3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>'
        :'<svg class="sort-ico on" width="8" height="10" viewBox="0 0 8 10"><path d="M4 9V1M1 7l3 3 3-3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>')
      :'<svg class="sort-ico" width="8" height="10" viewBox="0 0 8 10"><path d="M4 1v8M1 3l3-3 3 3M1 7l3 3 3-3" stroke="rgba(255,255,255,.3)" stroke-width="1.5" stroke-linecap="round"/></svg>';
    th.innerHTML='<div class="th-wrap"><div class="th-name" data-col="'+esc(col.key)+'"><span class="th-label">'+esc(col.label)+'</span>'+sico+'</div><div class="th-filter '+(hasF?'active':'')+'" data-col="'+esc(col.key)+'">'+(hasF?'F':'f')+'</div></div>';
    th.querySelector('.th-name').addEventListener('click',e=>{{
      e.stopPropagation();
      if(col.key===sortCol){{sortDir=sortDir==='asc'?'desc':'asc';}}else{{sortCol=col.key;sortDir='asc';}}
      selCol=col.key;
      window.parent.postMessage({{type:'col_click',col:col.key}},'*');
      page=0;render();
    }});
    th.querySelector('.th-filter').addEventListener('click',e=>{{
      e.stopPropagation();
      openFilter(col.key,e.currentTarget);
    }});
    tr.appendChild(th);
  }});
  thead.appendChild(tr);
}}

function renderBody(){{
  const data=getFiltered();
  const total=data.length;
  const paged=data.slice(page*PS,(page+1)*PS);
  document.getElementById('count').textContent=total.toLocaleString('pt-BR')+' registros'+(total<ALL_ROWS.length?' (filtrado de '+ALL_ROWS.length.toLocaleString('pt-BR')+')':'');
  const tbody=document.getElementById('tbody');
  tbody.innerHTML='';
  paged.forEach((row,i)=>{{
    const tr=document.createElement('tr');
    const tdN=document.createElement('td');
    tdN.className='col-num';tdN.textContent=page*PS+i+1;tr.appendChild(tdN);
    COLS.forEach(col=>{{
      const td=document.createElement('td');
      if(col.key===selCol)td.classList.add('col-sel');
      const v=String(row[col.key]??'');
      if(col.key.toLowerCase()==='rede'){{td.innerHTML='<span class="'+(v==='VIVO-SMP'?'b-smp':'b-stfc')+'">'+esc(v)+'</span>';}}
      else{{td.textContent=v;}}
      tr.appendChild(td);
    }});
    tbody.appendChild(tr);
  }});
  renderPag(total);
}}

function renderChips(){{
  const bar=document.getElementById('active-filters');
  bar.innerHTML='';
  for(const[col,vals]of Object.entries(activeFilters)){{
    if(!vals||vals.size===0)continue;
    const lbl=COLS.find(c=>c.key===col)?.label||col;
    const chip=document.createElement('span');
    chip.className='af-chip';
    const vl=[...vals].slice(0,2).join(', ')+(vals.size>2?' +'+(vals.size-2):'');
    chip.innerHTML=esc(lbl)+': '+esc(vl)+'<span class="af-x" data-col="'+esc(col)+'">[x]</span>';
    chip.querySelector('.af-x').addEventListener('click',e=>{{delete activeFilters[e.target.dataset.col];page=0;render();}});
    bar.appendChild(chip);
  }}
}}

function renderPag(total){{
  const tp=Math.max(1,Math.ceil(total/PS));
  const s=total===0?0:page*PS+1,e=Math.min((page+1)*PS,total);
  document.getElementById('pag-info').textContent='Exibindo '+s+'-'+e+' de '+total.toLocaleString('pt-BR');
  const ctrl=document.getElementById('pag-ctrl');
  ctrl.innerHTML='';
  const btn=(lbl,cb,dis,on)=>{{const b=document.createElement('button');b.className='pb'+(on?' on':'');b.textContent=lbl;b.disabled=dis;b.onclick=cb;ctrl.appendChild(b);}};
  btn('|<',()=>{{page=0;render();}},page===0);
  btn('<',()=>{{page--;render();}},page===0);
  const w=5,lo=Math.max(0,Math.min(page-2,tp-w)),hi=Math.min(tp-1,lo+w-1);
  for(let i=lo;i<=hi;i++){{const pi=i;btn(String(i+1),()=>{{page=pi;render();}},false,i===page);}}
  btn('>',()=>{{page++;render();}},page>=tp-1);
  btn('>|',()=>{{page=tp-1;render();}},page>=tp-1);
}}

function openFilter(colKey,anchor){{
  const fdd=document.getElementById('fdd');
  if(fddCol===colKey&&fdd.classList.contains('open')){{closeFilter();return;}}
  fddCol=colKey;
  document.getElementById('fdd-col-name').textContent=COLS.find(c=>c.key===colKey)?.label||colKey;
  document.getElementById('fdd-search').value='';
  fddSel=activeFilters[colKey]?new Set(activeFilters[colKey]):new Set(uniqueVals[colKey]);
  buildList('');
  fdd.classList.add('open');
  const rect=anchor.getBoundingClientRect();
  fdd.style.top=(rect.bottom+2)+'px';
  fdd.style.left=Math.min(rect.left,window.innerWidth-290)+'px';
}}

function buildList(search){{
  const list=document.getElementById('fdd-list');
  list.innerHTML='';
  const vals=uniqueVals[fddCol]||[];
  const lower=search.toLowerCase();
  const shown=search?vals.filter(v=>v.toLowerCase().includes(lower)):vals;
  shown.forEach(v=>{{
    const label=v===''?'(Sem valor)':v;
    const item=document.createElement('div');item.className='fdd-item';
    const chk=document.createElement('input');chk.type='checkbox';chk.checked=fddSel.has(v);
    chk.addEventListener('change',()=>{{if(chk.checked)fddSel.add(v);else fddSel.delete(v);}});
    const lbl=document.createElement('label');lbl.textContent=label;
    lbl.addEventListener('click',()=>{{chk.checked=!chk.checked;if(chk.checked)fddSel.add(v);else fddSel.delete(v);}});
    item.appendChild(chk);item.appendChild(lbl);list.appendChild(item);
  }});
}}

function closeFilter(){{document.getElementById('fdd').classList.remove('open');fddCol=null;}}

function applyFilter(){{
  if(!fddCol)return;
  const allVals=new Set(uniqueVals[fddCol]||[]);
  if([...allVals].every(v=>fddSel.has(v))||fddSel.size===0){{delete activeFilters[fddCol];}}
  else{{activeFilters[fddCol]=new Set(fddSel);}}
  closeFilter();page=0;render();
}}

document.getElementById('fdd-apply').addEventListener('click',applyFilter);
document.getElementById('fdd-selall').addEventListener('click',()=>{{fddSel=new Set(uniqueVals[fddCol]||[]);buildList(document.getElementById('fdd-search').value);}});
document.getElementById('fdd-clear').addEventListener('click',()=>{{fddSel=new Set();buildList(document.getElementById('fdd-search').value);}});
document.getElementById('fdd-search').addEventListener('input',e=>{{buildList(e.target.value);}});
document.addEventListener('click',e=>{{
  const fdd=document.getElementById('fdd');
  if(!fdd.contains(e.target)&&!e.target.closest('.th-filter')){{if(fdd.classList.contains('open'))closeFilter();}}
}});

function esc(s){{return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;').replace(/"/g,'&quot;');}}

/* ── DOWNLOAD HELPER ─────────────────────────────────────────────────
   Usa window.parent.URL para criar o objectURL — funciona em iframe
   same-origin (localhost). Evita o bloqueio do browser em iframes.    */
function triggerDownload(blob, filename){{
  try{{
    const pURL = window.parent.URL || URL;
    const url  = pURL.createObjectURL(blob);
    const doc  = window.parent.document;
    const a    = doc.createElement('a');
    a.href = url; a.download = filename; a.style.display='none';
    doc.body.appendChild(a);
    a.click();
    setTimeout(function(){{doc.body.removeChild(a);pURL.revokeObjectURL(url);}},1000);
  }}catch(err){{
    console.error('Download error:',err);
  }}
}}

/* EXPORT CSV filtrado */
function exportCSV(){{
  const rows=getFiltered();
  const cols=COLS.map(c=>c.key);
  const lines=[cols.join(';')];
  rows.forEach(r=>{{
    lines.push(cols.map(k=>{{
      const v=String(r[k]??'').replace(/"/g,'""');
      return(v.includes(';')||v.includes('"')||v.includes('\n'))?'"'+v+'"':v;
    }}).join(';'));
  }});
  const csv='\uFEFF'+lines.join('\r\n');
  const ts=new Date().toISOString().slice(0,10);
  triggerDownload(new Blob([csv],{{type:'text/csv;charset=utf-8'}}),'tabela_filtrada_'+ts+'.csv');
}}

/* EXPORT EXCEL filtrado (SpreadsheetML — abre no Excel sem instalar nada) */
function exportExcel(){{
  const rows=getFiltered();
  const cols=COLS.map(c=>c.key);
  const enc=v=>String(v??'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  const toRow=(arr,i)=>'<Row ss:Index="'+(i+1)+'">'+arr.map(v=>'<Cell><Data ss:Type="String">'+enc(v)+'</Data></Cell>').join('')+'</Row>';
  const wsRows=[toRow(cols,0),...rows.map((r,i)=>toRow(cols.map(k=>r[k]),i+1))].join('');
  const xml='<?xml version="1.0" encoding="UTF-8"?><?mso-application progid="Excel.Sheet"?>'
    +'<Workbook xmlns="urn:schemas-microsoft-com:office:spreadsheet" xmlns:ss="urn:schemas-microsoft-com:office:spreadsheet">'
    +'<Worksheet ss:Name="Tabela"><Table>'+wsRows+'</Table></Worksheet></Workbook>';
  const ts=new Date().toISOString().slice(0,10);
  triggerDownload(new Blob([xml],{{type:'application/vnd.ms-excel'}}),'tabela_filtrada_'+ts+'.xls');
}}

document.getElementById('btn-csv').addEventListener('click',exportCSV);
document.getElementById('btn-xlsx').addEventListener('click',exportExcel);

function render(){{renderHead();renderBody();renderChips();}}
render();
</script>
</body>
</html>"""

    components.html(html, height=height + 46, scrolling=False)
