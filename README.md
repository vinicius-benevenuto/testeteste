"""
ui/interactive_table.py
=======================
Tabela HTML/JS embutida via components.html.
- Ordenacao por clique no cabecalho
- Paginacao client-side
- Clique no nome da coluna notifica Streamlit (para dashboard)
- Sem filtros internos nem exports — gerenciados pelo app.py
"""
from __future__ import annotations

import json
import pandas as pd
import streamlit as st
import streamlit.components.v1 as components

_PAGE_SIZE = 500


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
    cols = []
    for c in df.columns:
        key = c.lower().replace("\u00f3","o").replace("\u00e7","c").replace("\u00e3","a").replace("\u00e9","e")
        cols.append({"key": c, "label": c, "width": widths.get(key, 115)})
    return json.dumps(cols, ensure_ascii=False)


def render_interactive_table(
    df: pd.DataFrame,
    selected_col: str | None,
    height: int = 520,
    component_key: str = "itable",
) -> None:
    data_json = _df_to_json(df)
    cols_json = _cols_to_json(df)
    sel_js    = json.dumps(selected_col or "")

    html = """<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<style>
:root {
  --bg:#F8F9FA; --surface:#FFFFFF; --border:#D1D5DB; --blt:#E9ECEF;
  --hdr:#0F172A; --hdr-h:#1E293B; --hdr-sel:#1B5FBF;
  --txt:#111827; --muted:#6B7280; --accent:#1B5FBF;
  --row-alt:#F9FAFB; --row-h:#EFF6FF;
}
*{box-sizing:border-box;margin:0;padding:0;}
body{font-family:'Segoe UI',Arial,sans-serif;font-size:12px;background:var(--bg);
  color:var(--txt);display:flex;flex-direction:column;height:HEIGHTpx;overflow:hidden;}
#bar{display:flex;align-items:center;justify-content:space-between;
  padding:5px 12px;background:var(--surface);border-bottom:1px solid var(--border);flex-shrink:0;}
#count{font-size:11px;color:var(--muted);font-weight:600;}
#hint{font-size:11px;color:var(--accent);}
#wrap{flex:1;overflow:auto;}
table{border-collapse:collapse;table-layout:fixed;width:max-content;min-width:100%;}
thead tr{position:sticky;top:0;z-index:10;}
th{background:var(--hdr);border-right:1px solid #1E293B;border-bottom:2px solid #1B5FBF;
  padding:0;white-space:nowrap;user-select:none;cursor:pointer;}
th:hover{background:var(--hdr-h);}
th.active{background:var(--hdr-sel)!important;}
th.num{width:40px;min-width:40px;cursor:default;}
th.num:hover{background:var(--hdr)!important;}
.th-inner{display:flex;align-items:center;height:34px;padding:0 10px;gap:5px;}
.th-lbl{font-size:10px;font-weight:700;color:#FFF;letter-spacing:.05em;
  text-transform:uppercase;overflow:hidden;text-overflow:ellipsis;}
.sort{opacity:.25;flex-shrink:0;}
.sort.on{opacity:1;}
tbody tr:nth-child(even){background:var(--row-alt);}
tbody tr:nth-child(odd){background:var(--surface);}
tbody tr:hover{background:var(--row-h)!important;}
td{border-bottom:1px solid var(--blt);border-right:1px solid var(--blt);
  padding:5px 9px;font-size:12px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}
td.num{text-align:center;font-size:10px;color:var(--muted);}
td.active{background:#EFF6FF!important;}
.smp{color:#1B5FBF;font-weight:700;font-family:'Courier New',monospace;}
.stfc{color:#6D28D9;font-weight:700;font-family:'Courier New',monospace;}
#pag{display:flex;align-items:center;justify-content:space-between;
  padding:4px 12px;background:var(--surface);border-top:1px solid var(--border);flex-shrink:0;}
#pag-info{font-size:11px;color:var(--muted);}
#pag-btns{display:flex;gap:2px;}
.pb{min-width:26px;height:24px;padding:0 5px;border:1px solid var(--border);
  background:var(--surface);color:var(--muted);font-size:11px;cursor:pointer;font-family:inherit;}
.pb:hover:not(:disabled){border-color:var(--accent);color:var(--accent);}
.pb.on{background:var(--accent);color:#fff;border-color:var(--accent);font-weight:700;}
.pb:disabled{opacity:.35;cursor:default;}
</style>
</head>
<body>
<script>
(function(){
  try{
    var pd=window.parent.document;
    if(pd.getElementById('_itx_dz'))return;
    var s=pd.createElement('style');s.id='_itx_dz';
    s.textContent='[data-testid="stComponentContainer"] section{display:none!important;}'
      +'[data-testid="stComponentContainer"] [data-testid="stFileUploaderDropzone"]{display:none!important;}'
      +'[data-testid="stComponentContainer"] iframe{display:block!important;}';
    pd.head.appendChild(s);
  }catch(e){}
})();
</script>
<div id="bar">
  <span id="count"></span>
  <span id="hint">Clique no cabecalho para ordenar · seleciona a coluna para o dashboard</span>
</div>
<div id="wrap">
  <table><thead id="thead"></thead><tbody id="tbody"></tbody></table>
</div>
<div id="pag">
  <span id="pag-info"></span>
  <div id="pag-btns"></div>
</div>
<script>
const ROWS=DATA_JSON_PLACEHOLDER;
const COLS=COLS_JSON_PLACEHOLDER;
const PS=500;
let selCol=SEL_JS_PLACEHOLDER;
let sortCol=null,sortDir='asc',page=0;

function sorted(arr){
  if(!sortCol)return arr;
  return[...arr].sort((a,b)=>{
    const av=String(a[sortCol]||''),bv=String(b[sortCol]||'');
    return sortDir==='asc'?av.localeCompare(bv,'pt'):bv.localeCompare(av,'pt');
  });
}

function renderHead(){
  const tr=document.createElement('tr');
  const thN=document.createElement('th');thN.className='num';
  thN.innerHTML='<div class="th-inner" style="justify-content:center"><span style="font-size:10px;color:rgba(255,255,255,.2)">#</span></div>';
  tr.appendChild(thN);
  COLS.forEach(col=>{
    const th=document.createElement('th');
    th.style.width=col.width+'px';
    if(col.key===selCol)th.classList.add('active');
    const ico=col.key===sortCol
      ?(sortDir==='asc'
        ?'<svg class="sort on" width="8" height="9" viewBox="0 0 8 9"><path d="M4 1v7M1 3l3-3 3 3" stroke="#fff" stroke-width="1.5" stroke-linecap="round" fill="none"/></svg>'
        :'<svg class="sort on" width="8" height="9" viewBox="0 0 8 9"><path d="M4 8V1M1 6l3 3 3-3" stroke="#fff" stroke-width="1.5" stroke-linecap="round" fill="none"/></svg>')
      :'<svg class="sort" width="8" height="9" viewBox="0 0 8 9"><path d="M4 1v7M1 3l3-3 3 3M1 6l3 3 3-3" stroke="rgba(255,255,255,.3)" stroke-width="1.5" stroke-linecap="round" fill="none"/></svg>';
    th.innerHTML='<div class="th-inner"><span class="th-lbl">'+col.label+'</span>'+ico+'</div>';
    th.addEventListener('click',()=>{
      if(col.key===sortCol)sortDir=sortDir==='asc'?'desc':'asc';
      else{sortCol=col.key;sortDir='asc';}
      selCol=col.key;
      window.parent.postMessage({type:'col_click',col:col.key},'*');
      page=0;render();
    });
    tr.appendChild(th);
  });
  const thead=document.getElementById('thead');
  thead.innerHTML='';thead.appendChild(tr);
}

function renderBody(){
  const data=sorted(ROWS);const total=data.length;
  const slice=data.slice(page*PS,(page+1)*PS);
  document.getElementById('count').textContent=total.toLocaleString('pt-BR')+' registros';
  const tbody=document.getElementById('tbody');tbody.innerHTML='';
  slice.forEach((row,i)=>{
    const tr=document.createElement('tr');
    const tdN=document.createElement('td');tdN.className='num';
    tdN.textContent=page*PS+i+1;tr.appendChild(tdN);
    COLS.forEach(col=>{
      const td=document.createElement('td');
      if(col.key===selCol)td.classList.add('active');
      const v=String(row[col.key]??'');
      if(col.key.toLowerCase()==='rede'){
        td.innerHTML='<span class="'+(v==='VIVO-SMP'?'smp':'stfc')+'">'+v+'</span>';
      }else{td.textContent=v;}
      tr.appendChild(td);
    });
    tbody.appendChild(tr);
  });
  renderPag(total);
}

function renderPag(total){
  const tp=Math.max(1,Math.ceil(total/PS));
  const s=total===0?0:page*PS+1,e=Math.min((page+1)*PS,total);
  document.getElementById('pag-info').textContent=s+'\u2013'+e+' de '+total.toLocaleString('pt-BR');
  const btns=document.getElementById('pag-btns');btns.innerHTML='';
  const b=(lbl,cb,dis,on)=>{
    const el=document.createElement('button');
    el.className='pb'+(on?' on':'');el.textContent=lbl;el.disabled=dis;
    el.onclick=cb;btns.appendChild(el);
  };
  b('|<',()=>{page=0;render();},page===0);
  b('<', ()=>{page--;render();},page===0);
  const lo=Math.max(0,Math.min(page-2,tp-5)),hi=Math.min(tp-1,lo+4);
  for(let i=lo;i<=hi;i++){const pi=i;b(String(i+1),()=>{page=pi;render();},false,i===page);}
  b('>', ()=>{page++;render();},page>=tp-1);
  b('>|',()=>{page=tp-1;render();},page>=tp-1);
}

function render(){renderHead();renderBody();}
render();
</script>
</body>
</html>"""

    html = (html
        .replace("HEIGHTpx", f"{height}px")
        .replace("DATA_JSON_PLACEHOLDER", data_json)
        .replace("COLS_JSON_PLACEHOLDER", cols_json)
        .replace("SEL_JS_PLACEHOLDER", sel_js))

    components.html(html, height=height + 10, scrolling=False)
