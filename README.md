/* =============================
 * Boot seguro (não duplica)
 * ============================= */
(function hardSafeInit() {
  if (window.__appBooted) return;
  window.__appBooted = true;

  try {
    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', boot);
    } else {
      boot();
    }
  } catch (e) {
    console.error('[BOOT] falhou:', e);
  }

  window.addEventListener('error', (ev) => {
    console.error('[GLOBAL ERROR]', ev.error || ev.message || ev);
  });
})();

/* =============================
 * Helpers HTTP/DOM
 * ============================= */
function csrfHeader() {
  const token = document.querySelector('meta[name="csrf-token"]')?.content;
  return token ? { 'X-CSRF-Token': token } : {};
}

async function fetchJSON(url, options = {}) {
  const res = await fetch(url, options);
  const ct = res.headers.get('content-type') || '';
  if (!res.ok) {
    if (ct.includes('application/json')) {
      const data = await res.json().catch(() => ({}));
      throw new Error(data.error || `HTTP ${res.status}`);
    }
    const t = await res.text().catch(() => '');
    throw new Error(t || `HTTP ${res.status}`);
  }
  if (!ct.includes('application/json')) throw new Error('Resposta não é JSON.');
  return res.json();
}

async function fetchBlobOrJSON(url, options = {}) {
  const res = await fetch(url, options);
  const ct = res.headers.get('content-type') || '';
  if (!res.ok) {
    if (ct.includes('application/json')) {
      const data = await res.json().catch(() => ({}));
      throw new Error(data.error || `HTTP ${res.status}`);
    }
    const t = await res.text().catch(() => '');
    throw new Error(t || `HTTP ${res.status}`);
  }
  if (ct.includes('application/json')) {
    const data = await res.json();
    throw new Error(data.error || 'Falha ao gerar arquivo.');
  }
  return res.blob();
}

function setLoading(on) {
  const overlay = document.getElementById('carregando');
  if (!overlay) return;
  overlay.classList.toggle('d-none', !on);
}

function bindClick(id, handler) {
  const el = document.getElementById(id);
  if (!el) {
    console.warn(`[UI] #${id} não encontrado.`);
    return null;
  }
  el.addEventListener('click', (ev) => {
    try { handler(ev); }
    catch (e) {
      console.error(`[UI] Erro no handler de #${id}:`, e);
      alert('Ocorreu um erro nesta ação.');
    }
  });
  return el;
}

function getModalById(id) {
  const el = document.getElementById(id);
  if (!el) return null;
  if (window.bootstrap && bootstrap.Modal) {
    return bootstrap.Modal.getOrCreateInstance(el);
  }
  // Fallback simples se Bootstrap não estiver disponível
  return {
    show() {
      el.classList.add('show');
      el.style.display = 'block';
      el.removeAttribute('aria-hidden');
      if (!document.getElementById('bd-fallback')) {
        const bd = document.createElement('div');
        bd.id = 'bd-fallback';
        bd.style.cssText = 'position:fixed;inset:0;background:rgba(0,0,0,.5);z-index:1040;';
        document.body.appendChild(bd);
      }
    },
    hide() {
      el.classList.remove('show');
      el.style.display = 'none';
      el.setAttribute('aria-hidden', 'true');
      const bd = document.getElementById('bd-fallback');
      if (bd) bd.remove();
    }
  };
}

/* =============================
 * Estado global
 * ============================= */
let nomesColunas = [];
let dadosAtuais = [];
let paginaAtual = 1;
const linhasPorPagina = 100;
let totalLinhas = 0;
let grafico = null;
let ultimoResumo = { STFC: 0, SMP: 0 };

/* =============================
 * Filtros (thead) helpers
 * ============================= */
function criarInputFiltro(col, index) {
  const th = document.createElement('th');
  const nome = (col || '').toLowerCase();
  if (nome.includes('data')) {
    const wrap = document.createElement('div');
    wrap.className = 'd-flex gap-1 flex-nowrap';
    const d1 = document.createElement('input');
    d1.type = 'date';
    d1.className = 'form-control form-control-sm filtro-data-inicial';
    d1.dataset.coluna = col;
    const d2 = document.createElement('input');
    d2.type = 'date';
    d2.className = 'form-control form-control-sm filtro-data-final';
    d2.dataset.coluna = col;
    wrap.append(d1, d2);
    th.appendChild(wrap);
  } else {
    const input = document.createElement('input');
    input.type = 'text';
    input.className = 'form-control form-control-sm filtro-coluna';
    input.dataset.index = index;
    input.placeholder = 'Filtrar...';
    th.appendChild(input);
  }
  return th;
}

function coletarFiltrosDeCabecalho() {
  const filtrosColuna = {};
  document.querySelectorAll('.filtro-coluna').forEach(input => {
    const index = input.dataset.index;
    const valor = input.value.trim();
    if (valor) filtrosColuna[nomesColunas[index]] = valor;
  });
  document.querySelectorAll('.filtro-data-inicial').forEach(input => {
    const valor = input.value;
    if (valor) filtrosColuna[`${input.dataset.coluna}__inicio`] = valor;
  });
  document.querySelectorAll('.filtro-data-final').forEach(input => {
    const valor = input.value;
    if (valor) filtrosColuna[`${input.dataset.coluna}__fim`] = valor;
  });
  return filtrosColuna;
}

function limparFiltros() {
  const campo = document.getElementById('campo-pesquisa');
  if (campo) campo.value = '';
  const filtroSTFC = document.getElementById('filtro-stfc');
  const filtroSMP  = document.getElementById('filtro-smp');
  if (filtroSTFC) filtroSTFC.checked = true;
  if (filtroSMP)  filtroSMP.checked  = true;

  const di = document.getElementById('data-inicio');
  const df = document.getElementById('data-fim');
  const bs = document.getElementById('base-data-aplicacao');
  if (di) di.value = '';
  if (df) df.value = '';
  if (bs) bs.value = '';

  document.querySelectorAll('.filtro-coluna').forEach(i => { i.value = ''; });
  document.querySelectorAll('.filtro-data-inicial').forEach(i => { i.value = ''; });
  document.querySelectorAll('.filtro-data-final').forEach(i => { i.value = ''; });
}

/* =============================
 * Tabela + paginação
 * ============================= */
function arraysIguais(a, b) {
  if (!Array.isArray(a) || !Array.isArray(b)) return false;
  if (a.length !== b.length) return false;
  for (let i = 0; i < a.length; i++) if (a[i] !== b[i]) return false;
  return true;
}

function preencherTabela(colunas, dados) {
  const cardTabela   = document.getElementById('card-tabela');
  const cabecalho    = document.getElementById('cabecalho');
  const linhaFiltros = document.getElementById('filtros');
  const corpoTabela  = document.getElementById('corpo-tabela');

  const precisaRebuild = !arraysIguais(nomesColunas, colunas);
  if (precisaRebuild) {
    cabecalho.innerHTML = '';
    linhaFiltros.innerHTML = '';
    colunas.forEach((col, index) => {
      const th = document.createElement('th');
      th.textContent = (col || '').toUpperCase();
      cabecalho.appendChild(th);
      linhaFiltros.appendChild(criarInputFiltro(col, index));
    });

    linhaFiltros.querySelectorAll('input').forEach((el) => {
      el.addEventListener('keydown', (e) => {
        if (e.key === 'Enter') {
          e.preventDefault();
          aplicarFiltrosAgora();
        }
      });
    });

    nomesColunas = [...colunas];
  }

  corpoTabela.innerHTML = '';
  dados.forEach(linha => {
    const tr = document.createElement('tr');
    colunas.forEach(col => {
      const td = document.createElement('td');
      td.textContent = linha[col] ?? '—';
      tr.appendChild(td);
    });
    corpoTabela.appendChild(tr);
  });

  const temDados = dados.length > 0;
  const msg = document.getElementById('mensagem-inicial');
  if (msg) msg.classList.toggle('d-none', temDados);
  if (cardTabela) cardTabela.hidden = !temDados;
}

function criarControlesPaginacao(total, paginaAtualLocal, linhasPorPaginaLocal, onChange) {
  const nav = document.getElementById('paginacao');
  if (!nav) return;
  nav.innerHTML = '';
  const totalPaginas = Math.max(1, Math.ceil(total / linhasPorPaginaLocal));

  const ul = document.createElement('ul');
  ul.className = 'pagination pagination-sm mb-0 flex-wrap';

  function addItem(label, page, disabled = false, active = false, aria = '') {
    const li = document.createElement('li');
    li.className = 'page-item' + (disabled ? ' disabled' : '') + (active ? ' active' : '');
    const a = document.createElement('a');
    a.className = 'page-link';
    a.href = '#';
    a.textContent = label;
    if (aria) a.setAttribute('aria-label', aria);
    if (active) a.setAttribute('aria-current', 'page');
    a.onclick = (e) => {
      e.preventDefault();
      if (!disabled && !active) onChange(page);
    };
    li.appendChild(a);
    ul.appendChild(li);
  }

  addItem('«', Math.max(1, paginaAtualLocal - 1), paginaAtualLocal === 1, false, 'Página anterior');

  const start = Math.max(1, paginaAtualLocal - 2);
  const end   = Math.min(totalPaginas, paginaAtualLocal + 2);
  for (let p = start; p <= end; p++) addItem(String(p), p, false, p === paginaAtualLocal);

  addItem('»', Math.min(totalPaginas, paginaAtualLocal + 1), paginaAtualLocal === totalPaginas, false, 'Próxima página');

  nav.appendChild(ul);
}

/* =============================
 * Dashboard (resumo + Chart.js)
 * ============================= */
function atualizarResumoEGraph(resumo) {
  const stfc = Number(resumo?.STFC || 0);
  const smp  = Number(resumo?.SMP  || 0);
  ultimoResumo = { STFC: stfc, SMP: smp };

  const elS = document.getElementById('contagem-stfc');
  const elM = document.getElementById('contagem-smp');
  if (elS) elS.textContent = stfc.toLocaleString('pt-BR');
  if (elM) elM.textContent = smp.toLocaleString('pt-BR');

  atualizarGrafico(stfc, smp);
}

function atualizarGrafico(stfc, smp) {
  const canvas = document.getElementById('grafico-rede');
  if (!canvas || typeof Chart === 'undefined') return;

  const vSTFC = (typeof stfc === 'number') ? stfc : (ultimoResumo.STFC || 0);
  const vSMP  = (typeof smp  === 'number') ? smp  : (ultimoResumo.SMP  || 0);

  if (grafico && typeof grafico.destroy === 'function') {
    try { grafico.destroy(); } catch (_) {}
    grafico = null;
  }
  canvas.height = 220;

  const ctx = canvas.getContext('2d');
  const isDark = document.documentElement.getAttribute('data-bs-theme') === 'dark';
  const legendColor = isDark ? '#e9d7ff' : '#333';

  const styles = getComputedStyle(document.documentElement);
  const VIVO_MAIN  = styles.getPropertyValue('--vivo-primary-500')?.trim() || '#660099';
  const VIVO_LIGHT = styles.getPropertyValue('--vivo-primary-300')?.trim() || '#9D4EDD';

  Chart.defaults.color = legendColor;
  Chart.defaults.font.family = "'Inter','Segoe UI',system-ui,-apple-system,Arial,sans-serif'";
  Chart.defaults.plugins.legend.labels.color = legendColor;
  Chart.defaults.plugins.title.color = legendColor;

  grafico = new Chart(ctx, {
    type: 'doughnut',
    data: {
      labels: ['STFC', 'SMP'],
      datasets: [{
        data: [vSTFC, vSMP],
        backgroundColor: [VIVO_MAIN, VIVO_LIGHT],
        borderColor: isDark ? '#171923' : '#ffffff',
        borderWidth: 2,
        hoverOffset: 6
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      cutout: '62%',
      animation: { duration: 250, easing: 'easeOutCubic' },
      plugins: {
        title: { display: true, text: 'Distribuição de Tráfego' },
        legend: { position: 'bottom', labels: { boxWidth: 10 } },
        tooltip: {
          callbacks: {
            label: (ctx) => {
              const v = ctx.raw ?? 0;
              const total = (vSTFC + vSMP) || 1;
              const pct = (v * 100 / total).toFixed(1);
              return `${ctx.label}: ${v.toLocaleString('pt-BR')} (${pct}%)`;
            }
          }
        }
      }
    }
  });
}

/* =============================
 * Carregar página (consulta)
 * ============================= */
function aplicarFiltrosAgora() {
  paginaAtual = 1;
  carregarPagina();
}

async function carregarPagina() {
  const campoPesquisa = document.getElementById('campo-pesquisa');
  const filtroSTFC = document.getElementById('filtro-stfc');
  const filtroSMP  = document.getElementById('filtro-smp');

  const dataInicio = document.getElementById('data-inicio')?.value || '';
  const dataFim    = document.getElementById('data-fim')?.value || '';
  const baseDatas  = document.getElementById('base-data-aplicacao')?.value || '';

  setLoading(true);
  try {
    const payload = {
      page: paginaAtual,
      page_size: linhasPorPagina,
      termo_geral: (campoPesquisa?.value || '').trim(),
      trafego: (filtroSTFC?.checked && !filtroSMP?.checked) ? 'STFC'
            : (!filtroSTFC?.checked && filtroSMP?.checked) ? 'SMP'
            : null,
      filtros_coluna: coletarFiltrosDeCabecalho(),
      data_inicio: dataInicio || undefined,
      data_fim: dataFim || undefined,
      aplicar_data_somente_science: baseDatas === 'science'
    };

    const res = await fetchJSON('/vivo_crc/api/dados-v2', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...csrfHeader() },
      body: JSON.stringify(payload)
    });

    totalLinhas  = res.total || 0;
    dadosAtuais  = res.dados || [];
    const novasColunas = res.colunas || [];

    preencherTabela(novasColunas, dadosAtuais);
    criarControlesPaginacao(totalLinhas, paginaAtual, linhasPorPagina, (p) => {
      paginaAtual = p;
      carregarPagina();
    });

    const tot = document.getElementById('total-resultados');
    if (tot) tot.textContent = (res.total || 0).toLocaleString('pt-BR');

    atualizarResumoEGraph(res.resumo || {});
  } catch (e) {
    console.error('[API] /vivo_crc/api/dados-v2 error:', e);
    alert('Erro ao aplicar filtros: ' + e.message);
  } finally {
    setLoading(false);
  }
}

/* =============================
 * Upload (Science / Portal)
 * ============================= */
function wireUpload() {
  const inputScience = document.getElementById('input-csv-science');
  const inputPortal  = document.getElementById('input-csv-portal');
  const origemUpload = document.getElementById('origem-upload');
  const formUpload   = document.getElementById('form-upload');

  async function enviarArquivo(file, origem) {
    if (!file) return;
    if (!/\.csv$/i.test(file.name)) {
      alert('Apenas arquivos .csv são aceitos.');
      return;
    }

    if (origemUpload) origemUpload.value = origem;
    const fd = new FormData(formUpload || undefined);
    fd.set('arquivo', file, file.name);
    fd.set('origem', origem);

    setLoading(true);
    try {
      const res = await fetchJSON('/vivo_crc/api/upload-csv', {
        method: 'POST',
        body: fd,
        headers: csrfHeader()
      });
      alert(res.message || 'Arquivo processado.');
      const modal = getModalById('modal-upload');
      if (modal) modal.hide();
      paginaAtual = 1;
      await carregarPagina();
    } catch (err) {
      console.error('[Upload] error:', err);
      alert('Erro ao enviar o arquivo: ' + err.message);
    } finally {
      setLoading(false);
      try { if (file && file.value !== undefined) file.value = ''; } catch (_) {}
      if (inputScience) inputScience.value = '';
      if (inputPortal)  inputPortal.value = '';
    }
  }

  if (inputScience) inputScience.addEventListener('change', () => enviarArquivo(inputScience.files[0], 'science'));
  if (inputPortal)  inputPortal .addEventListener('change', () => enviarArquivo(inputPortal.files[0],  'portal'));
}

/* =============================
 * Exportação
 * ============================= */
async function exportarFiltrado() {
  const campoPesquisa = document.getElementById('campo-pesquisa');
  const filtroSTFC = document.getElementById('filtro-stfc');
  const filtroSMP  = document.getElementById('filtro-smp');

  const dataInicio = document.getElementById('data-inicio')?.value || '';
  const dataFim    = document.getElementById('data-fim')?.value || '';
  const baseDatas  = document.getElementById('base-data-aplicacao')?.value || '';

  try {
    setLoading(true);
    const payload = {
      ignorar_filtros: false,
      termo_geral: (campoPesquisa?.value || '').trim(),
      trafego: (filtroSTFC?.checked && !filtroSMP?.checked) ? 'STFC'
            : (!filtroSTFC?.checked && filtroSMP?.checked) ? 'SMP'
            : null,
      filtros_coluna: coletarFiltrosDeCabecalho(),
      data_inicio: dataInicio || undefined,
      data_fim: dataFim || undefined,
      aplicar_data_somente_science: baseDatas === 'science'
    };

    const blob = await fetchBlobOrJSON('/vivo_crc/api/exportar', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...csrfHeader() },
      body: JSON.stringify(payload)
    });

    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    const isCSV = (blob.type || '').includes('csv') || (blob.type || '').includes('text');
    a.download = 'dados_filtrados.' + (isCSV ? 'csv' : 'xlsx');
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);

    const modal = getModalById('modal-envio');
    if (modal) modal.hide();
  } catch (e) {
    console.error('[Export filtrado] error:', e);
    alert('Erro ao gerar o arquivo: ' + e.message);
  } finally {
    setLoading(false);
  }
}

async function exportarTudoTurbo() {
  try {
    setLoading(true);
    const payload = { ignorar_filtros: true };

    const blob = await fetchBlobOrJSON('/vivo_crc/api/exportar', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...csrfHeader() },
      body: JSON.stringify(payload)
    });

    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    const isGZ = (blob.type || '').includes('gzip');
    a.download = 'dados_completos.' + (isGZ ? 'csv.gz' : 'csv');
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  } catch (e) {
    console.error('[Export tudo] error:', e);
    alert('Erro ao baixar o arquivo completo: ' + e.message);
  } finally {
    setLoading(false);
  }
}

/* =============================
 * Tema
 * ============================= */
function setTheme(mode) {
  const root = document.documentElement;
  const btnTema = document.getElementById('alternar-tema');
  root.setAttribute('data-bs-theme', mode);
  if (btnTema) {
    btnTema.textContent = (mode === 'dark') ? '☀️' : '🌙';
    btnTema.setAttribute('aria-pressed', String(mode === 'dark'));
  }
  localStorage.setItem('theme', mode);
  atualizarGrafico(); // re-render com cores do tema
}

/* =============================
 * Agente de IA — UI helpers
 * ============================= */
let iaBusy = false;
let iaAbortCtrl = null;

function iaScrollToBottom() {
  const box = document.getElementById('ia-mensagens');
  if (!box) return;
  if (box.__scrollLock) return;
  box.__scrollLock = true;
  box.style.scrollBehavior = 'auto';
  box.scrollTop = box.scrollHeight;
  setTimeout(() => { box.__scrollLock = false; }, 30);
}

function iaAppendUser(texto) {
  const box = document.getElementById('ia-mensagens');
  if (!box) return;
  const div = document.createElement('div');
  div.className = 'ia-msg ia-user';
  div.textContent = texto;
  box.appendChild(div);
  iaScrollToBottom();
}

function iaAppendBot(html) {
  const box = document.getElementById('ia-mensagens');
  if (!box) return;
  const div = document.createElement('div');
  div.className = 'ia-msg ia-bot';
  div.innerHTML = html;
  box.appendChild(div);
  iaScrollToBottom();
}

/* =============================
 * Agente de IA — execução
 * ============================= */
async function iaPerguntar(pergunta) {
  if (!pergunta || iaBusy) return;
  iaBusy = true;

  const btn = document.getElementById('ia-enviar');
  const input = document.getElementById('ia-pergunta');

  if (btn) { btn.disabled = true; btn.innerText = 'Perguntando...'; }
  if (input) input.disabled = true;

  iaAppendUser(pergunta);
  iaAppendBot('<span class="text-secondary">Pensando…</span>');

  try {
    const campoPesquisa = document.getElementById('campo-pesquisa');
    const filtroSTFC = document.getElementById('filtro-stfc');
    const filtroSMP  = document.getElementById('filtro-smp');
    const dataInicio = document.getElementById('data-inicio')?.value || '';
    const dataFim    = document.getElementById('data-fim')?.value || '';
    const baseDatas  = document.getElementById('base-data-aplicacao')?.value || '';

    const contexto = {
      termo_geral: (campoPesquisa?.value || '').trim(),
      trafego: (filtroSTFC?.checked && !filtroSMP?.checked) ? 'STFC'
            : (!filtroSTFC?.checked && filtroSMP?.checked) ? 'SMP'
            : null,
      filtros_coluna: coletarFiltrosDeCabecalho(),
      data_inicio: dataInicio || undefined,
      data_fim: dataFim || undefined,
      aplicar_data_somente_science: baseDatas === 'science'
    };

    if (iaAbortCtrl) iaAbortCtrl.abort();
    iaAbortCtrl = new AbortController();

    const res = await fetch('/vivo_crc/api/ask', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...csrfHeader() },
      body: JSON.stringify({ question: pergunta, contexto }),
      signal: iaAbortCtrl.signal
    });

    const ct = res.headers.get('content-type') || '';
    if (!res.ok) {
      let msg = `HTTP ${res.status}`;
      if (ct.includes('application/json')) {
        const j = await res.json().catch(() => ({}));
        if (j?.error) msg = j.error;
      }
      throw new Error(msg);
    }
    const data = ct.includes('application/json') ? await res.json() : {};

    const box = document.getElementById('ia-mensagens');
    if (box && box.lastElementChild && box.lastElementChild.classList.contains('ia-bot')) {
      box.removeChild(box.lastElementChild);
    }

    let html = '';
    if (data.answer) {
      html += `<div>${data.answer}</div>`;
    }
    if (data.rows && data.rows.length) {
      const cols = data.columns || Object.keys(data.rows[0]);
      const header = cols.map(c => `<th>${c}</th>`).join('');
      const body = data.rows.map(r =>
        `<tr>${cols.map(c => `<td>${(r[c] ?? '—')}</td>`).join('')}</tr>`
      ).join('');
      html += `
        <div class="table-responsive mt-2">
          <table class="table table-sm">
            <thead><tr>${header}</tr></thead>
            <tbody>${body}</tbody>
          </table>
        </div>`;
    }
    if (data.chart && Array.isArray(data.chart.labels) && Array.isArray(data.chart.data)) {
      const cid = 'ia-chart-' + Date.now();
      html += `<div class="mt-2" style="max-width:680px">
                 <canvas id="${cid}" height="180"></canvas>
               </div>`;
      iaAppendBot(html);
      try {
        const ctx = document.getElementById(cid)?.getContext('2d');
        if (ctx && typeof Chart !== 'undefined') {
          new Chart(ctx, {
            type: (data.chart.type === 'bar' ? 'bar' : 'doughnut'),
            data: { labels: data.chart.labels, datasets: [{ data: data.chart.data }] },
            options: {
              responsive: true, maintainAspectRatio: false,
              plugins: { title: { display: !!data.chart.title, text: data.chart.title || '' } }
            }
          });
        }
      } catch (e) {
        console.warn('[IA] chart render error:', e);
      }
      iaScrollToBottom();
    } else {
      iaAppendBot(html || '<span class="text-secondary">Sem dados para exibir.</span>');
    }
  } catch (err) {
    const box = document.getElementById('ia-mensagens');
    if (box && box.lastElementChild && box.lastElementChild.classList.contains('ia-bot')) {
      box.removeChild(box.lastElementChild);
    }
    iaAppendBot(`<span class="text-danger">Erro: ${err.message}</span>`);
  } finally {
    iaBusy = false;
    if (btn) { btn.disabled = false; btn.innerText = 'Perguntar'; }
    if (input) { input.disabled = false; input.value = ''; input.focus(); }
  }
}

/* =============================
 * Boot (liga tudo)
 * ============================= */
function boot() {
  const saved = localStorage.getItem('theme') || 'light';
  setTheme(saved);

  const btnTema = document.getElementById('alternar-tema');
  if (btnTema) {
    btnTema.addEventListener('click', () => {
      const current = document.documentElement.getAttribute('data-bs-theme') === 'dark' ? 'dark' : 'light';
      setTheme(current === 'dark' ? 'light' : 'dark');
    });
  }

  wireUpload();

  bindClick('botao-aplicar-filtros', () => aplicarFiltrosAgora());
  bindClick('botao-limpar-filtros',  () => { limparFiltros(); aplicarFiltrosAgora(); });

  bindClick('abrir-modal-upload',   () => { const m = getModalById('modal-upload'); if (m) m.show(); });
  bindClick('abrir-modal-exportar', () => { const m = getModalById('modal-envio'); if (m) m.show(); });

  bindClick('botao-baixar-excel', () => exportarFiltrado());
  bindClick('botao-baixar-tudo',  () => exportarTudoTurbo());

  const campoPesquisa = document.getElementById('campo-pesquisa');
  if (campoPesquisa) {
    campoPesquisa.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') {
        e.preventDefault();
        aplicarFiltrosAgora();
      }
    });
  }

  const iaBtn = document.getElementById('ia-enviar');
  const iaInput = document.getElementById('ia-pergunta');

  if (iaBtn) {
    iaBtn.replaceWith(iaBtn.cloneNode(true));
  }
  const iaBtnFresh = document.getElementById('ia-enviar');
  if (iaBtnFresh) {
    iaBtnFresh.addEventListener('click', () => {
      const q = iaInput?.value?.trim();
      if (q) iaPerguntar(q);
    }, { once: false });
  }

  if (iaInput) {
    iaInput.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') {
        e.preventDefault();
        const q = iaInput.value.trim();
        if (q) iaPerguntar(q);
      }
    }, { once: false });
  }

  document.querySelectorAll('.ia-sugestao').forEach(b => {
    b.addEventListener('click', () => {
      const q = b.getAttribute('data-q');
      if (q) iaPerguntar(q);
    }, { once: false });
  });

  const cardTabela = document.getElementById('card-tabela');
  if (cardTabela) cardTabela.hidden = true;
  const msg = document.getElementById('mensagem-inicial');
  if (msg) msg.classList.remove('d-none');

  document.querySelectorAll('.modal .btn-close,[data-bs-dismiss="modal"]').forEach((b) => {
    b.addEventListener('click', () => {
      const modalEl = b.closest('.modal');
      if (!modalEl) return;
      if (!(window.bootstrap && bootstrap.Modal)) {
        const m = getModalById(modalEl.id);
        if (m) m.hide();
      }
    });
  });

  carregarPagina();
}
