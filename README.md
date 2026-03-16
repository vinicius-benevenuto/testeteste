"""
ui/interactive_table.py — filtros estilo Excel + export filtrado
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
    out = []
    for c in df.columns:
        key = (c.lower()
               .replace("\u00f3","o").replace("\u00e7","c")
               .replace("\u00e3","a").replace("\u00e9","e")
               .replace("\u00f5","o").replace("\u00ea","e"))
        out.append({"key": c, "label": c, "width": widths.get(key, 115)})
    return json.dumps(out, ensure_ascii=False)

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
<style>
:root{{
  --bg:#F8F9FA;--surface:#FFFFFF;--border:#D1D5DB;--blt:#E9ECEF;
  --hdr:#0F172A;--hdr-h:#1E293B;--hdr-sel:#1B5FBF;
  --txt:#111827;--muted:#6B7280;--accent:#1B5FBF;
  --row-alt:#F9FAFB;--row-h:#EFF6FF;
}}
*{{box-sizing:border-box;margin:0;padding:0;}}
body{{font-family:'Segoe UI',Arial,sans-serif;font-size:12px;
  background:var(--bg);color:var(--txt);
  display:flex;flex-direction:column;height:{height}px;overflow:hidden;}}
#bar{{display:flex;align-items:center;justify-content:space-between;
  padding:5px 12px;background:var(--surface);
  border-bottom:1px solid var(--border);flex-shrink:0;gap:8px;}}
#count{{font-size:11px;color:var(--muted);font-weight:600;flex-shrink:0;}}
#hint{{font-size:11px;color:var(--accent);flex:1;}}
#xbtns{{display:flex;gap:6px;flex-shrink:0;}}
.xbtn{{font-size:11px;font-weight:600;padding:4px 12px;border-radius:3px;cursor:pointer;
  font-family:inherit;border:1px solid var(--border);background:var(--surface);
  color:#374151;transition:all .12s;white-space:nowrap;}}
.xbtn:hover{{border-color:var(--accent);color:var(--accent);background:var(--row-h);}}
#chips{{display:flex;align-items:center;flex-wrap:wrap;gap:4px;
  padding:4px 12px;background:var(--surface);
  border-bottom:1px solid var(--blt);flex-shrink:0;}}
#chips:empty{{display:none;padding:0;}}
.chip{{display:inline-flex;align-items:center;gap:4px;background:#EFF6FF;
  border:1px solid #BFDBFE;border-radius:2px;padding:2px 7px;
  font-size:10px;color:#1B5FBF;font-weight:600;}}
.chip-x{{cursor:pointer;font-size:13px;line-height:1;margin-left:2px;}}
.chip-x:hover{{color:#DC2626;}}
#wrap{{flex:1;overflow:auto;}}
table{{border-collapse:collapse;table-layout:fixed;width:max-content;min-width:100%;}}
thead tr{{position:sticky;top:0;z-index:20;}}
th{{background:var(--hdr);border-right:1px solid #1E293B;
  border-bottom:2px solid #1B5FBF;padding:0;white-space:nowrap;user-select:none;}}
th.col-num{{width:40px;min-width:40px;}}
th.col-sel{{background:var(--hdr-sel)!important;}}
.th-wrap{{display:flex;align-items:stretch;height:34px;}}
.th-name{{flex:1;display:flex;align-items:center;padding:0 8px;gap:5px;
  cursor:pointer;min-width:0;border:none;background:transparent;}}
.th-name:hover{{background:var(--hdr-h);}}
.th-lbl{{font-size:10px;font-weight:700;color:#FFF;letter-spacing:.05em;
  text-transform:uppercase;overflow:hidden;text-overflow:ellipsis;}}
.sort{{opacity:.25;flex-shrink:0;}}
.sort.on{{opacity:1;}}
.th-flt{{display:flex;align-items:center;justify-content:center;
  width:22px;min-width:22px;cursor:pointer;
  border:none;border-left:1px solid rgba(255,255,255,.07);
  background:transparent;font-size:9px;color:rgba(255,255,255,.4);transition:all .15s;}}
.th-flt:hover{{background:rgba(255,255,255,.12);color:#fff;}}
.th-flt.on{{color:#60A5FA;background:rgba(27,95,191,.3);}}
.fdd{{position:fixed;z-index:9999;background:#FFF;border:1px solid #D1D5DB;
  border-radius:4px;box-shadow:0 4px 20px rgba(0,0,0,.15);
  min-width:210px;max-width:280px;display:none;overflow:hidden;}}
.fdd.open{{display:block;}}
.fdd-head{{display:flex;align-items:center;justify-content:space-between;
  padding:8px 10px;border-bottom:1px solid #E9ECEF;background:#F8F9FA;}}
.fdd-title{{font-size:11px;font-weight:700;color:#0F172A;
  letter-spacing:.05em;text-transform:uppercase;}}
.fdd-acts{{display:flex;gap:8px;}}
.fdd-act{{font-size:10px;font-weight:700;color:#1B5FBF;
  background:none;border:none;cursor:pointer;font-family:inherit;padding:0;}}
.fdd-act:hover{{text-decoration:underline;}}
.fdd-search{{padding:6px 10px;border-bottom:1px solid #E9ECEF;}}
.fdd-search input{{width:100%;padding:4px 8px;font-size:11px;
  border:1px solid #D1D5DB;border-radius:3px;font-family:inherit;outline:none;color:#1C2536;}}
.fdd-search input:focus{{border-color:#1B5FBF;}}
.fdd-list{{max-height:200px;overflow-y:auto;padding:4px 0;}}
.fdd-item{{display:flex;align-items:center;gap:7px;padding:5px 10px;
  cursor:pointer;font-size:12px;color:#374151;}}
.fdd-item:hover{{background:#EFF6FF;}}
.fdd-item input[type=checkbox]{{width:13px;height:13px;accent-color:#1B5FBF;
  cursor:pointer;flex-shrink:0;}}
.fdd-item label{{cursor:pointer;flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
.fdd-foot{{padding:7px 10px;border-top:1px solid #E9ECEF;display:flex;justify-content:flex-end;}}
.fdd-ok{{font-size:11px;font-weight:700;background:#1B5FBF;color:#fff;
  border:none;border-radius:3px;padding:4px 16px;cursor:pointer;font-family:inherit;}}
.fdd-ok:hover{{background:#1549A0;}}
tbody tr:nth-child(even){{background:var(--row-alt);}}
tbody tr:nth-child(odd){{background:var(--surface);}}
tbody tr:hover{{background:var(--row-h)!important;}}
td{{border-bottom:1px solid var(--blt);border-right:1px solid var(--blt);
  padding:5px 9px;font-size:12px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
td.col-num{{text-align:center;font-size:10px;color:var(--muted);}}
td.col-sel{{background:#EFF6FF!important;}}
.smp{{color:#1B5FBF;font-weight:700;font-family:'Courier New',monospace;}}
.stfc{{color:#6D28D9;font-weight:700;font-family:'Courier New',monospace;}}
#pag{{display:flex;align-items:center;justify-content:space-between;
  padding:4px 12px;background:var(--surface);border-top:1px solid var(--border);flex-shrink:0;}}
#pag-info{{font-size:11px;color:var(--muted);}}
#pag-btns{{display:flex;gap:2px;}}
.pb{{min-width:26px;height:24px;padding:0 5px;border:1px solid var(--border);
  background:var(--surface);color:var(--muted);font-size:11px;
  cursor:pointer;font-family:inherit;}}
.pb:hover:not(:disabled){{border-color:var(--accent);color:var(--accent);}}
.pb.on{{background:var(--accent);color:#fff;border-color:var(--accent);font-weight:700;}}
.pb:disabled{{opacity:.35;cursor:default;}}
</style>
</head>
<body>
<script>
(function(){{
  try{{
    var pd=window.parent.document;
    if(pd.getElementById('_itx_dz'))return;
    var s=pd.createElement('style');s.id='_itx_dz';
    s.textContent='[data-testid="stComponentContainer"] section{{display:none!important;}}'
      +'[data-testid="stComponentContainer"] [data-testid="stFileUploaderDropzone"]{{display:none!important;}}'
      +'[data-testid="stComponentContainer"] iframe{{display:block!important;}}';
    pd.head.appendChild(s);
  }}catch(e){{}}
}})();
</script>

<div id="bar">
  <span id="count"></span>
  <span id="hint">Clique no icone de cada coluna para filtrar</span>
  <div id="xbtns">
    <button class="xbtn" id="btn-csv">Exportar CSV</button>
    <button class="xbtn" id="btn-xls">Exportar Excel</button>
  </div>
</div>
<div id="chips"></div>
<div id="wrap">
  <table><thead id="thead"></thead><tbody id="tbody"></tbody></table>
</div>
<div id="pag">
  <span id="pag-info"></span>
  <div id="pag-btns"></div>
</div>
<div class="fdd" id="fdd">
  <div class="fdd-head">
    <span class="fdd-title" id="fdd-col"></span>
    <div class="fdd-acts">
      <button class="fdd-act" id="fdd-all">Selecionar tudo</button>
      <button class="fdd-act" id="fdd-none">Limpar</button>
    </div>
  </div>
  <div class="fdd-search"><input type="text" id="fdd-search" placeholder="Pesquisar..."></div>
  <div class="fdd-list" id="fdd-list"></div>
  <div class="fdd-foot"><button class="fdd-ok" id="fdd-ok">Aplicar</button></div>
</div>

<script>
const ROWS={data_json};
const COLS={cols_json};
const PS={_PAGE_SIZE};
let selCol={sel_js};
let sortCol=null,sortDir='asc',page=0;
const filters={{}};
COLS.forEach(c=>{{filters[c.key]=null;}});
const uniques={{}};
COLS.forEach(c=>{{
  uniques[c.key]=[...new Set(ROWS.map(r=>String(r[c.key]??'')))].sort();
}});

function getFiltered(){{
  let rows=ROWS;
  COLS.forEach(c=>{{
    const s=filters[c.key];
    if(s!==null)rows=rows.filter(r=>s.has(String(r[c.key]??'')));
  }});
  if(sortCol){{
    rows=[...rows].sort((a,b)=>{{
      const av=String(a[sortCol]||''),bv=String(b[sortCol]||'');
      return sortDir==='asc'?av.localeCompare(bv,'pt'):bv.localeCompare(av,'pt');
    }});
  }}
  return rows;
}}

function renderHead(){{
  const tr=document.createElement('tr');
  const thN=document.createElement('th');thN.className='col-num';
  thN.innerHTML='<div class="th-wrap" style="justify-content:center;align-items:center;"><span style="font-size:10px;color:rgba(255,255,255,.2)">#</span></div>';
  tr.appendChild(thN);
  COLS.forEach(col=>{{
    const th=document.createElement('th');
    th.style.width=col.width+'px';
    const isActive=col.key===selCol,hasFlt=filters[col.key]!==null;
    if(isActive)th.classList.add('col-sel');
    const ico=col.key===sortCol
      ?(sortDir==='asc'
        ?'<svg class="sort on" width="8" height="9" viewBox="0 0 8 9" fill="none"><path d="M4 1v7M1 3l3-3 3 3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>'
        :'<svg class="sort on" width="8" height="9" viewBox="0 0 8 9" fill="none"><path d="M4 8V1M1 6l3 3 3-3" stroke="#fff" stroke-width="1.5" stroke-linecap="round"/></svg>')
      :'<svg class="sort" width="8" height="9" viewBox="0 0 8 9" fill="none"><path d="M4 1v7M1 3l3-3 3 3M1 6l3 3 3-3" stroke="rgba(255,255,255,.3)" stroke-width="1.5" stroke-linecap="round"/></svg>';
    th.innerHTML='<div class="th-wrap">'
      +'<div class="th-name" data-col="'+col.key+'">'
      +'<span class="th-lbl">'+col.label+'</span>'+ico+'</div>'
      +'<div class="th-flt'+(hasFlt?' on':'')+'" data-col="'+col.key+'">'+
      (hasFlt?'&#9660;':'&#9661;')+'</div></div>';
    th.querySelector('.th-name').addEventListener('click',()=>{{
      if(col.key===sortCol)sortDir=sortDir==='asc'?'desc':'asc';
      else{{sortCol=col.key;sortDir='asc';}}
      selCol=col.key;
      window.parent.postMessage({{type:'col_click',col:col.key}},'*');
      page=0;render();
    }});
    th.querySelector('.th-flt').addEventListener('click',e=>{{
      e.stopPropagation();openFilter(col.key,e.currentTarget);
    }});
    tr.appendChild(th);
  }});
  const thead=document.getElementById('thead');thead.innerHTML='';thead.appendChild(tr);
}}

function renderBody(){{
  const data=getFiltered(),total=data.length;
  const slice=data.slice(page*PS,(page+1)*PS);
  document.getElementById('count').textContent=total.toLocaleString('pt-BR')+' registros';
  const tbody=document.getElementById('tbody');tbody.innerHTML='';
  slice.forEach((row,i)=>{{
    const tr=document.createElement('tr');
    const tdN=document.createElement('td');tdN.className='col-num';tdN.textContent=page*PS+i+1;
    tr.appendChild(tdN);
    COLS.forEach(col=>{{
      const td=document.createElement('td');
      if(col.key===selCol)td.classList.add('col-sel');
      const v=String(row[col.key]??'');
      if(col.key.toLowerCase()==='rede'){{
        td.innerHTML='<span class="'+(v==='VIVO-SMP'?'smp':'stfc')+'">'+v+'</span>';
      }}else{{td.textContent=v;}}
      tr.appendChild(td);
    }});
    tbody.appendChild(tr);
  }});
  renderPag(total);renderChips();
}}

function renderChips(){{
  const bar=document.getElementById('chips');bar.innerHTML='';
  COLS.forEach(col=>{{
    const sel=filters[col.key];if(sel===null)return;
    const excl=uniques[col.key].filter(v=>!sel.has(v));
    if(!excl.length)return;
    const chip=document.createElement('span');chip.className='chip';
    const prev=excl.slice(0,2).join(', ')+(excl.length>2?' +'+(excl.length-2):'');
    chip.innerHTML=col.label+': <em style="font-weight:400;font-style:normal">'+prev+'</em>'
      +'<span class="chip-x" data-col="'+col.key+'">&times;</span>';
    chip.querySelector('.chip-x').addEventListener('click',e=>{{
      filters[e.target.dataset.col]=null;page=0;render();
    }});
    bar.appendChild(chip);
  }});
}}

function renderPag(total){{
  const tp=Math.max(1,Math.ceil(total/PS));
  const s=total===0?0:page*PS+1,e=Math.min((page+1)*PS,total);
  document.getElementById('pag-info').textContent=s+'\u2013'+e+' de '+total.toLocaleString('pt-BR');
  const btns=document.getElementById('pag-btns');btns.innerHTML='';
  const b=(lbl,cb,dis,on)=>{{
    const el=document.createElement('button');
    el.className='pb'+(on?' on':'');el.textContent=lbl;el.disabled=dis;el.onclick=cb;
    btns.appendChild(el);
  }};
  b('|<',()=>{{page=0;render();}},page===0);
  b('<', ()=>{{page--;render();}},page===0);
  const lo=Math.max(0,Math.min(page-2,tp-5)),hi=Math.min(tp-1,lo+4);
  for(let i=lo;i<=hi;i++){{const pi=i;b(String(i+1),()=>{{page=pi;render();}},false,i===page);}}
  b('>', ()=>{{page++;render();}},page>=tp-1);
  b('>|',()=>{{page=tp-1;render();}},page>=tp-1);
}}

let fddCol=null,fddTmp=new Set();
function openFilter(colKey,anchor){{
  const fdd=document.getElementById('fdd');
  if(fddCol===colKey&&fdd.classList.contains('open')){{closeFilter();return;}}
  fddCol=colKey;
  document.getElementById('fdd-col').textContent=colKey;
  document.getElementById('fdd-search').value='';
  fddTmp=filters[colKey]!==null?new Set(filters[colKey]):new Set(uniques[colKey]);
  buildList('');fdd.classList.add('open');
  const rect=anchor.getBoundingClientRect();
  fdd.style.top=(rect.bottom+2)+'px';
  fdd.style.left=Math.min(rect.left,window.innerWidth-290)+'px';
}}
function buildList(search){{
  const list=document.getElementById('fdd-list');list.innerHTML='';
  const vals=uniques[fddCol]||[],lower=search.toLowerCase();
  const shown=search?vals.filter(v=>v.toLowerCase().includes(lower)):vals;
  shown.forEach(v=>{{
    const lbl=v===''?'(Sem valor)':v;
    const item=document.createElement('div');item.className='fdd-item';
    const chk=document.createElement('input');chk.type='checkbox';chk.checked=fddTmp.has(v);
    chk.addEventListener('change',()=>{{chk.checked?fddTmp.add(v):fddTmp.delete(v);}});
    const lb=document.createElement('label');lb.textContent=lbl;
    lb.addEventListener('click',()=>{{chk.checked=!chk.checked;chk.checked?fddTmp.add(v):fddTmp.delete(v);}});
    item.appendChild(chk);item.appendChild(lb);list.appendChild(item);
  }});
}}
function closeFilter(){{document.getElementById('fdd').classList.remove('open');fddCol=null;}}
function applyFilter(){{
  if(!fddCol)return;
  const all=new Set(uniques[fddCol]);
  filters[fddCol]=(fddTmp.size===0||[...all].every(v=>fddTmp.has(v)))?null:new Set(fddTmp);
  closeFilter();page=0;render();
}}
document.getElementById('fdd-ok').addEventListener('click',applyFilter);
document.getElementById('fdd-all').addEventListener('click',()=>{{fddTmp=new Set(uniques[fddCol]);buildList(document.getElementById('fdd-search').value);}});
document.getElementById('fdd-none').addEventListener('click',()=>{{fddTmp=new Set();buildList(document.getElementById('fdd-search').value);}});
document.getElementById('fdd-search').addEventListener('input',e=>buildList(e.target.value));
document.addEventListener('click',e=>{{
  const fdd=document.getElementById('fdd');
  if(fdd.classList.contains('open')&&!fdd.contains(e.target)&&!e.target.closest('.th-flt'))closeFilter();
}});

function dlFile(blob,name){{
  try{{
    const url=(window.parent.URL||URL).createObjectURL(blob);
    const doc=window.parent.document;
    const a=doc.createElement('a');a.href=url;a.download=name;a.style.display='none';
    doc.body.appendChild(a);a.click();
    setTimeout(()=>{{doc.body.removeChild(a);(window.parent.URL||URL).revokeObjectURL(url);}},1000);
  }}catch(e){{
    const url=URL.createObjectURL(blob);const a=document.createElement('a');
    a.href=url;a.download=name;document.body.appendChild(a);a.click();
    setTimeout(()=>{{document.body.removeChild(a);URL.revokeObjectURL(url);}},1000);
  }}
}}
document.getElementById('btn-csv').addEventListener('click',()=>{{
  const rows=getFiltered(),cols=COLS.map(c=>c.key);
  const lines=[cols.join(';')];
  rows.forEach(r=>lines.push(cols.map(k=>{{
    const v=String(r[k]??'').replace(/"/g,'""');
    return(v.includes(';')||v.includes('"')||v.includes('\n'))?'"'+v+'"':v;
  }}).join(';')));
  const ts=new Date().toISOString().slice(0,10);
  dlFile(new Blob(['\uFEFF'+lines.join('\r\n')],{{type:'text/csv;charset=utf-8'}}),'tabela_'+ts+'.csv');
}});
document.getElementById('btn-xls').addEventListener('click',()=>{{
  const rows=getFiltered(),cols=COLS.map(c=>c.key);
  const enc=v=>String(v??'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  const toRow=(arr,i)=>'<Row ss:Index="'+(i+1)+'">'+arr.map(v=>'<Cell><Data ss:Type="String">'+enc(v)+'</Data></Cell>').join('')+'</Row>';
  const ws=[toRow(cols,0),...rows.map((r,i)=>toRow(cols.map(k=>r[k]),i+1))].join('');
  const xml='<?xml version="1.0" encoding="UTF-8"?><?mso-application progid="Excel.Sheet"?><Workbook xmlns="urn:schemas-microsoft-com:office:spreadsheet" xmlns:ss="urn:schemas-microsoft-com:office:spreadsheet"><Worksheet ss:Name="Tabela"><Table>'+ws+'</Table></Worksheet></Workbook>';
  const ts=new Date().toISOString().slice(0,10);
  dlFile(new Blob([xml],{{type:'application/vnd.ms-excel'}}),'tabela_'+ts+'.xls');
}});

function render(){{renderHead();renderBody();}}
render();
</script>
</body>
</html>"""

    components.html(html, height=height + 10, scrolling=False)
