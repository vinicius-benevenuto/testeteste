"""
ui/interactive_table.py
Tabela com filtros estilo Excel e exports filtrados.
"""
from __future__ import annotations
import json
import pandas as pd
import streamlit.components.v1 as components


_PAGE_SIZE = 500


def render_interactive_table(
    df: pd.DataFrame,
    selected_col: str | None,
    height: int = 560,
    component_key: str = "itable",
) -> None:

    # Prepara dados
    safe = df.copy()
    for c in safe.columns:
        safe[c] = safe[c].astype(str).replace({"nan": "", "None": ""})

    # CRITICO: escapa </ para evitar que valores como </script> nos dados
    # encerrem prematuramente o tag <script> no HTML, quebrando todo o JS.
    data_json = json.dumps(
        safe.to_dict(orient="records"), ensure_ascii=False
    ).replace("</", "<\\/")

    widths = {
        "rótulos de linha": 200, "denominação": 180,
        "central": 145, "cluster": 120, "operadora": 130,
        "tipo de rota": 125, "uf": 70, "rede": 100,
    }
    cols_list = [{"key": c, "label": c, "width": widths.get(c.lower(), 115)}
                 for c in df.columns]
    cols_json = json.dumps(cols_list, ensure_ascii=False).replace("</", "<\\/")
    sel_js = json.dumps(selected_col or "")
    ps = _PAGE_SIZE

    html = f"""<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<style>
*{{box-sizing:border-box;margin:0;padding:0;}}
body{{font-family:'Segoe UI',Arial,sans-serif;font-size:12px;background:#F8F9FA;color:#111827;
  display:flex;flex-direction:column;height:{height}px;overflow:hidden;}}
#bar{{display:flex;align-items:center;gap:10px;padding:5px 12px;
  background:#fff;border-bottom:1px solid #D1D5DB;flex-shrink:0;}}
#count{{font-size:11px;color:#6B7280;font-weight:600;}}
#hint{{font-size:11px;color:#1B5FBF;flex:1;}}
.xbtn{{padding:4px 12px;font-size:11px;font-weight:600;cursor:pointer;
  border:1px solid #D1D5DB;background:#fff;border-radius:3px;font-family:inherit;}}
.xbtn:hover{{border-color:#1B5FBF;color:#1B5FBF;}}
#chips{{display:flex;flex-wrap:wrap;gap:4px;padding:3px 12px;
  background:#fff;border-bottom:1px solid #E9ECEF;flex-shrink:0;}}
#chips:empty{{display:none;}}
.chip{{display:inline-flex;align-items:center;gap:4px;background:#EFF6FF;
  border:1px solid #BFDBFE;border-radius:2px;padding:2px 7px;
  font-size:10px;color:#1B5FBF;font-weight:600;}}
.chipx{{cursor:pointer;font-size:14px;line-height:1;}}
.chipx:hover{{color:#DC2626;}}
#wrap{{flex:1;overflow:auto;}}
table{{border-collapse:collapse;table-layout:fixed;width:max-content;min-width:100%;}}
thead tr{{position:sticky;top:0;z-index:10;}}
th{{background:#0F172A;border-right:1px solid #1E293B;border-bottom:2px solid #1B5FBF;
  padding:0;white-space:nowrap;user-select:none;}}
th.cn{{width:40px;min-width:40px;}}
th.ca{{background:#1B5FBF!important;}}
.thw{{display:flex;align-items:stretch;height:34px;}}
.thn{{flex:1;display:flex;align-items:center;padding:0 8px;gap:5px;
  cursor:pointer;border:none;background:transparent;}}
.thn:hover{{background:#1E293B;}}
.thl{{font-size:10px;font-weight:700;color:#fff;letter-spacing:.05em;
  text-transform:uppercase;overflow:hidden;text-overflow:ellipsis;}}
.thf{{display:flex;align-items:center;justify-content:center;width:22px;
  min-width:22px;cursor:pointer;border:none;border-left:1px solid rgba(255,255,255,.1);
  background:transparent;color:rgba(255,255,255,.4);font-size:10px;}}
.thf:hover{{background:rgba(255,255,255,.1);color:#fff;}}
.thf.on{{color:#60A5FA;background:rgba(27,95,191,.3);}}
tbody tr:nth-child(even){{background:#F9FAFB;}}
tbody tr:nth-child(odd){{background:#fff;}}
tbody tr:hover{{background:#EFF6FF!important;}}
td{{border-bottom:1px solid #E9ECEF;border-right:1px solid #E9ECEF;
  padding:5px 9px;font-size:12px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
td.cn{{text-align:center;font-size:10px;color:#6B7280;}}
td.ca{{background:#EFF6FF!important;}}
.smp{{color:#1B5FBF;font-weight:700;}}
.stfc{{color:#6D28D9;font-weight:700;}}
#pag{{display:flex;align-items:center;justify-content:space-between;
  padding:4px 12px;background:#fff;border-top:1px solid #D1D5DB;flex-shrink:0;}}
#pinfo{{font-size:11px;color:#6B7280;}}
#pbtns{{display:flex;gap:2px;}}
.pb{{min-width:26px;height:24px;padding:0 5px;border:1px solid #D1D5DB;
  background:#fff;color:#6B7280;font-size:11px;cursor:pointer;font-family:inherit;}}
.pb:hover:not(:disabled){{border-color:#1B5FBF;color:#1B5FBF;}}
.pb.on{{background:#1B5FBF;color:#fff;border-color:#1B5FBF;font-weight:700;}}
.pb:disabled{{opacity:.4;cursor:default;}}
.fdd{{position:fixed;z-index:9999;background:#fff;border:1px solid #D1D5DB;
  border-radius:4px;box-shadow:0 4px 16px rgba(0,0,0,.15);
  width:240px;display:none;overflow:hidden;}}
.fdd.open{{display:block;}}
.fdh{{display:flex;align-items:center;justify-content:space-between;
  padding:8px 10px;border-bottom:1px solid #E9ECEF;background:#F8F9FA;}}
.fdt{{font-size:11px;font-weight:700;color:#0F172A;text-transform:uppercase;letter-spacing:.04em;}}
.fda{{display:flex;gap:8px;}}
.fdab{{font-size:10px;font-weight:700;color:#1B5FBF;background:none;border:none;
  cursor:pointer;font-family:inherit;padding:0;}}
.fdab:hover{{text-decoration:underline;}}
.fds{{padding:6px 10px;border-bottom:1px solid #E9ECEF;}}
.fds input{{width:100%;padding:4px 8px;font-size:11px;border:1px solid #D1D5DB;
  border-radius:3px;font-family:inherit;outline:none;}}
.fds input:focus{{border-color:#1B5FBF;}}
.fdl{{max-height:200px;overflow-y:auto;padding:3px 0;}}
.fdi{{display:flex;align-items:center;gap:7px;padding:5px 10px;cursor:pointer;
  font-size:12px;color:#374151;}}
.fdi:hover{{background:#EFF6FF;}}
.fdi input{{width:13px;height:13px;accent-color:#1B5FBF;cursor:pointer;flex-shrink:0;}}
.fdi label{{cursor:pointer;flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;}}
.fdf{{padding:7px 10px;border-top:1px solid #E9ECEF;display:flex;justify-content:flex-end;}}
.fdo{{font-size:11px;font-weight:700;background:#1B5FBF;color:#fff;border:none;
  border-radius:3px;padding:4px 14px;cursor:pointer;font-family:inherit;}}
.fdo:hover{{background:#1549A0;}}
</style>
</head>
<body>
<div id="bar">
  <span id="count"></span>
  <span id="hint">Clique no icone para filtrar a coluna</span>
  <button class="xbtn" id="bcsv">Exportar CSV</button>
  <button class="xbtn" id="bxls">Exportar Excel</button>
</div>
<div id="chips"></div>
<div id="wrap">
  <table><thead id="thead"></thead><tbody id="tbody"></tbody></table>
</div>
<div id="pag">
  <span id="pinfo"></span>
  <div id="pbtns"></div>
</div>
<div class="fdd" id="fdd">
  <div class="fdh">
    <span class="fdt" id="fdtitle"></span>
    <div class="fda">
      <button class="fdab" id="fdall">Selecionar tudo</button>
      <button class="fdab" id="fdnone">Limpar</button>
    </div>
  </div>
  <div class="fds"><input type="text" id="fdsearch" placeholder="Pesquisar..."></div>
  <div class="fdl" id="fdlist"></div>
  <div class="fdf"><button class="fdo" id="fdok">Aplicar</button></div>
</div>
<script>
const ROWS={data_json};
const COLS={cols_json};
const PS={ps};
let SC={sel_js};
let SCOL=null,SDIR='asc',PAGE=0;
const FLT={{}};
COLS.forEach(c=>{{FLT[c.key]=null;}});
const UNQ={{}};
COLS.forEach(c=>{{UNQ[c.key]=[...new Set(ROWS.map(r=>String(r[c.key]??'')))].sort();}});

function filtered(){{
  let r=ROWS;
  COLS.forEach(c=>{{if(FLT[c.key])r=r.filter(x=>FLT[c.key].has(String(x[c.key]??'')));}});
  if(SCOL){{
    r=[...r].sort((a,b)=>{{
      const av=String(a[SCOL]||''),bv=String(b[SCOL]||'');
      return SDIR==='asc'?av.localeCompare(bv,'pt'):bv.localeCompare(av,'pt');
    }});
  }}
  return r;
}}

function rHead(){{
  const tr=document.createElement('tr');
  const tn=document.createElement('th');tn.className='cn';
  tn.innerHTML='<div class="thw" style="justify-content:center;align-items:center;"><span style="color:rgba(255,255,255,.2);font-size:10px">#</span></div>';
  tr.appendChild(tn);
  COLS.forEach(col=>{{
    const th=document.createElement('th');th.style.width=col.width+'px';
    if(col.key===SC)th.classList.add('ca');
    const f=FLT[col.key]!==null;
    const ico=col.key===SCOL?(SDIR==='asc'?'&#8593;':'&#8595;'):'';
    th.innerHTML='<div class="thw"><div class="thn" data-k="'+col.key+'"><span class="thl">'+col.label+'</span><span style="opacity:.4;font-size:10px;">'+ico+'</span></div><div class="thf'+(f?' on':'')+'" data-k="'+col.key+'">'+(f?'&#9660;':'&#9661;')+'</div></div>';
    th.querySelector('.thn').onclick=()=>{{
      if(col.key===SCOL)SDIR=SDIR==='asc'?'desc':'asc';
      else{{SCOL=col.key;SDIR='asc';}}
      SC=col.key;
      window.parent.postMessage({{type:'col_click',col:col.key}},'*');
      PAGE=0;render();
    }};
    th.querySelector('.thf').onclick=e=>{{e.stopPropagation();openF(col.key,e.currentTarget);}};
    tr.appendChild(th);
  }});
  const thead=document.getElementById('thead');thead.innerHTML='';thead.appendChild(tr);
}}

function rBody(){{
  const data=filtered(),total=data.length;
  const sl=data.slice(PAGE*PS,(PAGE+1)*PS);
  document.getElementById('count').textContent=total.toLocaleString('pt-BR')+' registros';
  const tbody=document.getElementById('tbody');tbody.innerHTML='';
  sl.forEach((row,i)=>{{
    const tr=document.createElement('tr');
    const tn=document.createElement('td');tn.className='cn';tn.textContent=PAGE*PS+i+1;tr.appendChild(tn);
    COLS.forEach(col=>{{
      const td=document.createElement('td');if(col.key===SC)td.classList.add('ca');
      const v=String(row[col.key]??'');
      if(col.key.toLowerCase()==='rede')td.innerHTML='<span class="'+(v==='VIVO-SMP'?'smp':'stfc')+'">'+v+'</span>';
      else td.textContent=v;
      tr.appendChild(td);
    }});
    tbody.appendChild(tr);
  }});
  rPag(total);rChips();
}}

function rChips(){{
  const bar=document.getElementById('chips');bar.innerHTML='';
  COLS.forEach(col=>{{
    const s=FLT[col.key];if(!s)return;
    const excl=UNQ[col.key].filter(v=>!s.has(v));if(!excl.length)return;
    const chip=document.createElement('span');chip.className='chip';
    const pv=excl.slice(0,2).join(', ')+(excl.length>2?' +'+(excl.length-2):'');
    chip.innerHTML='<span>'+col.label+': '+pv+'</span><span class="chipx" data-k="'+col.key+'">&times;</span>';
    chip.querySelector('.chipx').onclick=e=>{{FLT[e.target.dataset.k]=null;PAGE=0;render();}};
    bar.appendChild(chip);
  }});
}}

function rPag(total){{
  const tp=Math.max(1,Math.ceil(total/PS));
  const s=total===0?0:PAGE*PS+1,e=Math.min((PAGE+1)*PS,total);
  document.getElementById('pinfo').textContent=s+'\u2013'+e+' de '+total.toLocaleString('pt-BR');
  const b=document.getElementById('pbtns');b.innerHTML='';
  const mk=(l,cb,d,on)=>{{
    const el=document.createElement('button');
    el.className='pb'+(on?' on':'');el.textContent=l;el.disabled=d;el.onclick=cb;
    b.appendChild(el);
  }};
  mk('|<',()=>{{PAGE=0;render();}},PAGE===0);
  mk('<',()=>{{PAGE--;render();}},PAGE===0);
  const lo=Math.max(0,Math.min(PAGE-2,tp-5)),hi=Math.min(tp-1,lo+4);
  for(let i=lo;i<=hi;i++){{const p=i;mk(''+(i+1),()=>{{PAGE=p;render();}},false,i===PAGE);}}
  mk('>',()=>{{PAGE++;render();}},PAGE>=tp-1);
  mk('>|',()=>{{PAGE=tp-1;render();}},PAGE>=tp-1);
}}

let FC=null,FT=new Set();
function openF(k,anchor){{
  const fdd=document.getElementById('fdd');
  if(FC===k&&fdd.classList.contains('open')){{closeF();return;}}
  FC=k;document.getElementById('fdtitle').textContent=k;
  document.getElementById('fdsearch').value='';
  FT=FLT[k]?new Set(FLT[k]):new Set(UNQ[k]);
  buildList('');fdd.classList.add('open');
  const r=anchor.getBoundingClientRect();
  fdd.style.top=(r.bottom+2)+'px';
  fdd.style.left=Math.min(r.left,window.innerWidth-250)+'px';
}}
function buildList(q){{
  const list=document.getElementById('fdlist');list.innerHTML='';
  const vals=UNQ[FC]||[];const lq=q.toLowerCase();
  const show=q?vals.filter(v=>v.toLowerCase().includes(lq)):vals;
  show.forEach(v=>{{
    const lbl=v===''?'(Sem valor)':v;
    const item=document.createElement('div');item.className='fdi';
    const chk=document.createElement('input');chk.type='checkbox';chk.checked=FT.has(v);
    chk.onchange=()=>{{chk.checked?FT.add(v):FT.delete(v);}};
    const lb=document.createElement('label');lb.textContent=lbl;
    lb.onclick=()=>{{chk.checked=!chk.checked;chk.checked?FT.add(v):FT.delete(v);}};
    item.appendChild(chk);item.appendChild(lb);list.appendChild(item);
  }});
}}
function closeF(){{document.getElementById('fdd').classList.remove('open');FC=null;}}
function applyF(){{
  if(!FC)return;
  const all=new Set(UNQ[FC]);
  FLT[FC]=(FT.size===0||[...all].every(v=>FT.has(v)))?null:new Set(FT);
  closeF();PAGE=0;render();
}}
document.getElementById('fdok').onclick=applyF;
document.getElementById('fdall').onclick=()=>{{FT=new Set(UNQ[FC]);buildList(document.getElementById('fdsearch').value);}};
document.getElementById('fdnone').onclick=()=>{{FT=new Set();buildList(document.getElementById('fdsearch').value);}};
document.getElementById('fdsearch').oninput=e=>buildList(e.target.value);
document.onclick=e=>{{
  const fdd=document.getElementById('fdd');
  if(fdd.classList.contains('open')&&!fdd.contains(e.target)&&!e.target.closest('.thf'))closeF();
}};

function dlFile(blob,name){{
  try{{
    const url=(window.parent.URL||URL).createObjectURL(blob);
    const a=window.parent.document.createElement('a');
    a.href=url;a.download=name;a.style.display='none';
    window.parent.document.body.appendChild(a);a.click();
    setTimeout(()=>{{window.parent.document.body.removeChild(a);(window.parent.URL||URL).revokeObjectURL(url);}},500);
  }}catch(e){{
    const url=URL.createObjectURL(blob);
    const a=document.createElement('a');a.href=url;a.download=name;
    document.body.appendChild(a);a.click();
    setTimeout(()=>{{document.body.removeChild(a);URL.revokeObjectURL(url);}},500);
  }}
}}
document.getElementById('bcsv').onclick=()=>{{
  const rows=filtered(),cols=COLS.map(c=>c.key);
  const lines=[cols.join(';')];
  rows.forEach(r=>lines.push(cols.map(k=>{{
    const v=String(r[k]??'').replace(/"/g,'""');
    return(v.includes(';')||v.includes('"'))?'"'+v+'"':v;
  }}).join(';')));
  const ts=new Date().toISOString().slice(0,10);
  dlFile(new Blob(['\uFEFF'+lines.join('\r\n')],{{type:'text/csv;charset=utf-8'}}),'tabela_'+ts+'.csv');
}};
document.getElementById('bxls').onclick=()=>{{
  const rows=filtered(),cols=COLS.map(c=>c.key);
  const enc=v=>String(v??'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
  const toRow=(arr,i)=>'<Row ss:Index="'+(i+1)+'">'+arr.map(v=>'<Cell><Data ss:Type="String">'+enc(v)+'</Data></Cell>').join('')+'</Row>';
  const xml='<?xml version="1.0" encoding="UTF-8"?><?mso-application progid="Excel.Sheet"?><Workbook xmlns="urn:schemas-microsoft-com:office:spreadsheet" xmlns:ss="urn:schemas-microsoft-com:office:spreadsheet"><Worksheet ss:Name="Tabela"><Table>'
    +[toRow(cols,0),...rows.map((r,i)=>toRow(cols.map(k=>r[k]),i+1))].join('')
    +'</Table></Worksheet></Workbook>';
  const ts=new Date().toISOString().slice(0,10);
  dlFile(new Blob([xml],{{type:'application/vnd.ms-excel'}}),'tabela_'+ts+'.xls');
}};

function render(){{rHead();rBody();}}
render();
</script>
</body>
</html>"""

    components.html(html, height=height + 10, scrolling=False)

