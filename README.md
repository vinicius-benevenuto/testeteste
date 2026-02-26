from flask import Flask
import uuid, os
from flask_cors import CORS

def create_app():
    app = Flask(__name__)

    app.secret_key = str(uuid.uuid4())##para deploy
    ###app.secret_key = 'DEV'

    UPLOAD_FOLDER = 'uploads'
    ALLOWED_EXTENSIONS = {'csv'}
    app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
    app.config.from_mapping(
        DATABASE=os.path.join(app.instance_path, 'vivo_crc.sqlite'),
    )

    CORS(app)
    os.makedirs(UPLOAD_FOLDER, exist_ok=True)

    from . import vivo_crc
    app.register_blueprint(vivo_crc.bp)

    from . import sqlite_db
    sqlite_db.init_app(app)

    return app

HTML

<!DOCTYPE html>
<html lang="pt-BR" data-bs-theme="light">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Controle de Rotas Cadastradas</title>

  <!-- CSRF injetado pelo Flask -->
  <meta name="csrf-token" content="{{ csrf_token }}"/>

  <!-- Bootstrap CSS -->
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" />

  <!-- Estilos (tema Vivo + tabelas sticky + ajustes gerais) -->
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}" />

  <!-- Chart.js -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-body-tertiary">

  <!-- Navbar -->
  <nav class="navbar navbar-expand-lg sticky-top">
    <div class="container-fluid">
      <a class="navbar-brand fw-semibold" href="#">Controle de Rotas Cadastradas</a>
      <div class="d-flex gap-2">
        <button id="alternar-tema" class="btn btn-outline-secondary" type="button" aria-pressed="false" title="Alternar tema">🌙</button>
      </div>
    </div>
  </nav>

  <div class="container-fluid py-3">
    <div class="row g-3">
      <!-- Sidebar / Dashboard -->
      <aside class="col-12 col-lg-3 col-xxl-2">
        <div class="card shadow-sm">
          <div class="card-body">
            <h6 class="card-title mb-3">Resumo</h6>
            <div class="d-flex justify-content-between mb-1">
              <span class="text-secondary">STFC:</span>
              <span id="contagem-stfc" class="fw-semibold">0</span>
            </div>
            <div class="d-flex justify-content-between">
              <span class="text-secondary">SMP:</span>
              <span id="contagem-smp" class="fw-semibold">0</span>
            </div>
            <div class="chart-wrap mt-3" style="height:220px;">
              <canvas id="grafico-rede" aria-label="Distribuição de Tráfego" role="img"></canvas>
            </div>
          </div>
        </div>
      </aside>

      <!-- Main -->
      <main class="col-12 col-lg-9 col-xxl-10">
        <!-- Toolbar de filtros e ações -->
        <div class="card shadow-sm mb-3">
          <div class="card-body">
            <div class="row g-3 align-items-end">
              <!-- Busca geral -->
              <div class="col-12 col-xl-4">
                <label for="campo-pesquisa" class="form-label mb-1">Busca geral</label>
                <input id="campo-pesquisa" class="form-control" placeholder="Operadora, EOT, CN, descrição..." autocomplete="off" />
              </div>

              <!-- Tráfego -->
              <div class="col-12 col-sm-4 col-xl-2">
                <span class="form-label mb-1 d-block">Tipo de tráfego</span>
                <div class="d-flex align-items-center gap-3">
                  <div class="form-check">
                    <input class="form-check-input" type="checkbox" id="filtro-stfc" checked>
                    <label class="form-check-label" for="filtro-stfc">STFC</label>
                  </div>
                  <div class="form-check">
                    <input class="form-check-input" type="checkbox" id="filtro-smp" checked>
                    <label class="form-check-label" for="filtro-smp">SMP</label>
                  </div>
                </div>
              </div>

              <!-- Período (datas globais) + Base onde aplicar -->
              <div class="col-12 col-sm-8 col-xl-6">
                <span class="form-label mb-1 d-block">Período (Data_Desativacao) &nbsp;•&nbsp; aplicar datas em:</span>
                <div class="row g-2">
                  <div class="col-12 col-md-4">
                    <input id="data-inicio" type="date" class="form-control" placeholder="Início (YYYY-MM-DD)">
                  </div>
                  <div class="col-12 col-md-4">
                    <input id="data-fim" type="date" class="form-control" placeholder="Fim (YYYY-MM-DD)">
                  </div>
                  <div class="col-12 col-md-4">
                    <select id="base-data-aplicacao" class="form-select">
                      <option value="">Todas as bases</option>
                      <option value="science">Somente Science</option>
                    </select>
                  </div>
                </div>
                <small class="text-secondary d-block mt-1">
                  Ex.: selecione um intervalo e “Somente Science” para que o filtro atinja apenas os dados de Science; os dados do Portal permanecem como estão.
                </small>
              </div>

              <!-- Ações -->
              <div class="col-12">
                <div class="d-flex flex-wrap gap-2">
                  <button id="botao-aplicar-filtros" class="btn btn-primary" title="Aplicar filtros e atualizar a tabela">
                    Aplicar filtros
                  </button>
                  <button id="botao-limpar-filtros" class="btn btn-outline-primary" title="Limpar todos os filtros">
                    Limpar
                  </button>

                  <button id="abrir-modal-upload" class="btn btn-primary ms-auto" title="Carregar novas bases CSV">
                    Carregar bases
                  </button>

                  <!-- Dropdown Exportar -->
                  <div class="dropdown">
                    <button class="btn btn-primary dropdown-toggle" type="button"
                            data-bs-toggle="dropdown" data-bs-display="static" aria-expanded="false">
                      Exportar
                    </button>
                    <ul class="dropdown-menu dropdown-menu-end shadow">
                      <li>
                        <button class="dropdown-item" id="abrir-modal-exportar" type="button">
                          Exportar filtrado (XLSX/CSV)
                        </button>
                      </li>
                      <li><hr class="dropdown-divider"></li>
                      <li>
                        <button class="dropdown-item" id="botao-baixar-tudo" type="button">
                          Exportar tudo (CSV)
                        </button>
                      </li>
                    </ul>
                  </div>
                </div>
                <small class="text-secondary d-block mt-1">
                  Dica: além do período global, você pode usar os filtros por coluna na tabela para refinar ainda mais.
                </small>
              </div>
            </div>
          </div>
        </div>

        <!-- Mensagem inicial -->
        <div id="mensagem-inicial" class="alert alert-secondary">
          Informe os critérios e clique em <strong>Aplicar filtros</strong>.
        </div>

        <!-- Tabela -->
        <div class="card shadow-sm" id="card-tabela" hidden>
          <div class="table-responsive">
            <table id="tabela" class="table table-hover table-sm align-middle">
              <thead class="table-light">
                <tr id="cabecalho"></tr>
                <tr id="filtros"></tr>
              </thead>
              <tbody id="corpo-tabela"></tbody>
            </table>
          </div>

          <!-- Rodapé com total + paginação embaixo -->
          <div class="card-footer d-flex flex-wrap gap-2 justify-content-between align-items-center">
            <div>
              <strong>Resultados:</strong> <span id="total-resultados">0</span>
            </div>
            <nav id="paginacao" aria-label="Paginação"></nav>
          </div>
        </div>

        <!-- Agente de IA em largura total (abaixo da tabela) -->
        <div class="card shadow-sm mt-3">
          <div class="card-body d-flex flex-column">
            <h6 class="card-title mb-3">Agente de IA</h6>

            <div id="ia-mensagens" class="flex-grow-1 border rounded p-2 overflow-auto bg-body-tertiary"
                 style="min-height: 260px; max-height: 52vh;">
              <div class="ia-msg ia-bot">
                Olá! Posso responder perguntas como:
                <br><em>“quantos registros temos?”</em>,
                <em>“distribuição por operadora”</em>,
                <em>“top 10 EOTs”</em>,
                <em>“entre 2024-01-01 e 2024-03-31 só STFC”</em>.
              </div>
            </div>

            <div class="input-group mt-2">
              <input id="ia-pergunta" class="form-control" placeholder="Pergunte sobre os dados..." autocomplete="off">
              <button id="ia-enviar" class="btn btn-primary" type="button" title="Perguntar">Perguntar</button>
            </div>

            <div class="d-flex gap-2 flex-wrap mt-2">
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="quantos registros?">quantos registros?</button>
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="distribuição por operadora">por operadora</button>
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="top 10 operadoras">top 10 operadoras</button>
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="top 10 EOTs">top 10 EOTs</button>
            </div>
          </div>
        </div>
        <!-- /Agente de IA -->
      </main>
    </div>
  </div>

  <!-- Modal Upload -->
  <div class="modal fade" id="modal-upload" tabindex="-1" aria-labelledby="titulo-upload" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" id="titulo-upload">Carregar Base</h1>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Fechar"></button>
        </div>
        <div class="modal-body">
          <div class="d-grid gap-2 mb-2">
            <label class="btn btn-primary">
              CARREGAR BASE SCIENCE <input type="file" id="input-csv-science" accept=".csv" hidden>
            </label>
            <label class="btn btn-outline-primary">
              CARREGAR BASE PORTAL ITX <input type="file" id="input-csv-portal" accept=".csv" hidden>
            </label>
          </div>
          <form id="form-upload" action="{{ url_for('vivo_crc.upload_csv') }}" method="POST" enctype="multipart/form-data">
            <input type="hidden" name="origem" id="origem-upload" />
            <button id="enviar-upload" type="submit" class="visually-hidden">Enviar</button>
          </form>
          <small class="text-secondary">Apenas arquivos <code>.csv</code> são aceitos.</small>
        </div>
      </div>
    </div>
  </div>

  <!-- Modal Export (filtrado) -->
  <div class="modal fade" id="modal-envio" tabindex="-1" aria-labelledby="titulo-exportar" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" id="titulo-exportar">Exportar filtrado</h1>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Fechar"></button>
        </div>
        <div class="modal-body">
          <p>Baixe o arquivo considerando os filtros atuais (o sistema escolherá o melhor formato — XLSX em múltiplas abas ou CSV).</p>
          <div class="d-grid">
            <button id="botao-baixar-excel" class="btn btn-primary">Baixar agora</button>
          </div>
          <small class="text-secondary d-block mt-2">
            Dica: Para baixar todos os registros sem filtros, use <em>Exportar → Exportar tudo (CSV)</em>.
          </small>
        </div>
      </div>
    </div>
  </div>

  <!-- Loading -->
  <div id="carregando" class="position-fixed top-0 start-0 w-100 h-100 d-none" style="background:rgba(0,0,0,.35); z-index: 1080;">
    <div class="d-flex h-100 justify-content-center align-items-center">
      <div class="spinner-border text-light" role="status" aria-label="Carregando"></div>
    </div>
  </div>

  <!-- Scripts: Bootstrap primeiro; seu script por último -->
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
  <script src="{{ url_for('static', filename='script.js') }}"></script>
</body>
</html>

JAVASCRIPT

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

  // Novos campos (datas globais + base)
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

  // Filtros globais de data/base
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
 * Agente de IA — UI helpers (scroll/bolhas)
 * ============================= */
let iaBusy = false;
let iaAbortCtrl = null;

function iaScrollToBottom() {
  const box = document.getElementById('ia-mensagens');
  if (!box) return;
  if (box.__scrollLock) return;      // evita reentrância
  box.__scrollLock = true;
  box.style.scrollBehavior = 'auto'; // evita smooth loop
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

  // trava UI
  if (btn) { btn.disabled = true; btn.innerText = 'Perguntando...'; }
  if (input) input.disabled = true;

  iaAppendUser(pergunta);
  iaAppendBot('<span class="text-secondary">Pensando…</span>');

  try {
    // contexto dos filtros atuais
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

    // aborta requisição anterior, se existir
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

    // remove o "Pensando…" (a última bolha do bot)
    const box = document.getElementById('ia-mensagens');
    if (box && box.lastElementChild && box.lastElementChild.classList.contains('ia-bot')) {
      box.removeChild(box.lastElementChild);
    }

    // monta resposta
    let html = '';
    if (data.answer) {
      html += `<div>${data.answer}</div>`;
    }
    if (data.rows && data.rows.length) {
      // tabela simples
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
      // renderiza um pequeno gráfico inline
      const cid = 'ia-chart-' + Date.now();
      html += `<div class="mt-2" style="max-width:680px">
                 <canvas id="${cid}" height="180"></canvas>
               </div>`;
      iaAppendBot(html);
      // desenha o gráfico após inserir no DOM
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
    // remove o "Pensando…" se ainda estiver lá
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
  // Tema inicial
  const saved = localStorage.getItem('theme') || 'light';
  setTheme(saved);

  // Alternância de tema
  const btnTema = document.getElementById('alternar-tema');
  if (btnTema) {
    btnTema.addEventListener('click', () => {
      const current = document.documentElement.getAttribute('data-bs-theme') === 'dark' ? 'dark' : 'light';
      setTheme(current === 'dark' ? 'light' : 'dark');
    });
  }

  // Upload
  wireUpload();

  // Binds: toolbar e modais
  bindClick('botao-aplicar-filtros', () => aplicarFiltrosAgora());
  bindClick('botao-limpar-filtros',  () => { limparFiltros(); aplicarFiltrosAgora(); });

  bindClick('abrir-modal-upload',   () => { const m = getModalById('modal-upload'); if (m) m.show(); });
  bindClick('abrir-modal-exportar', () => { const m = getModalById('modal-envio'); if (m) m.show(); });

  bindClick('botao-baixar-excel', () => exportarFiltrado());
  bindClick('botao-baixar-tudo',  () => exportarTudoTurbo());

  // Enter ativa filtros na busca geral
  const campoPesquisa = document.getElementById('campo-pesquisa');
  if (campoPesquisa) {
    campoPesquisa.addEventListener('keydown', (e) => {
      if (e.key === 'Enter') {
        e.preventDefault();
        aplicarFiltrosAgora();
      }
    });
  }

  // Agente de IA — binds (garantindo 1 listener)
  const iaBtn = document.getElementById('ia-enviar');
  const iaInput = document.getElementById('ia-pergunta');

  if (iaBtn) {
    // limpa listeners antigos (proteção contra hot-reload/duplicatas)
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

  // Sugestões do agente
  document.querySelectorAll('.ia-sugestao').forEach(b => {
    b.addEventListener('click', () => {
      const q = b.getAttribute('data-q');
      if (q) iaPerguntar(q);
    }, { once: false });
  });

  // Estado inicial visual
  const cardTabela = document.getElementById('card-tabela');
  if (cardTabela) cardTabela.hidden = true;
  const msg = document.getElementById('mensagem-inicial');
  if (msg) msg.classList.remove('d-none');

  // Fecha modais no fallback quando clicar no .btn-close
  document.querySelectorAll('.modal .btn-close,[data-bs-dismiss="modal"]').forEach((b) => {
    b.addEventListener('click', () => {
      const modalEl = b.closest('.modal');
      if (!modalEl) return;
      // Se for bootstrap, o botão já funciona; no fallback fechamos manualmente
      if (!(window.bootstrap && bootstrap.Modal)) {
        const m = getModalById(modalEl.id);
        if (m) m.hide();
      }
    });
  });

  // Carrega a primeira página
  carregarPagina();
}


CSS

/* =========================================================
   TEMA VIVO — Paleta global
   ========================================================= */
   :root {
    /* Paleta base (Vivo) */
    --vivo-primary-700: #4b0073;
    --vivo-primary-600: #5a008a;
    --vivo-primary-500: #660099;   /* principal */
    --vivo-primary-400: #7B2CBF;   /* hover/acento */
    --vivo-primary-300: #9D4EDD;   /* tom claro */
    --vivo-primary-200: #c9a6f3;
  
    --vivo-bg-light: #F8F5FC;
    --vivo-bg-dark: #111217;
    --vivo-text-dark: #1A1A1A;
    --vivo-text-light: #e9e9ee;
  
    /* Injeção nas variáveis Bootstrap */
    --bs-primary: var(--vivo-primary-500);
    --bs-primary-rgb: 102, 0, 153;
    --bs-secondary: var(--vivo-primary-500);
    --bs-success: var(--vivo-primary-500);
    --bs-info: var(--vivo-primary-500);
    --bs-warning: var(--vivo-primary-500);
    --bs-danger: var(--vivo-primary-500);
    --bs-link-color: var(--vivo-primary-500);
    --bs-link-hover-color: var(--vivo-primary-400);
  }
  
  /* =========================================================
     Navbar
     ========================================================= */
  .navbar {
    background-color: var(--vivo-primary-500) !important;
    border-bottom: 3px solid rgba(157, 78, 221, .25);
  }
  .navbar .navbar-brand,
  .navbar .nav-link,
  .navbar .btn { color: #fff !important; }
  .navbar .nav-link:hover,
  .navbar .btn:hover { color: #f2e9ff !important; }
  
  /* =========================================================
     Botões — padroniza TUDO para o roxo Vivo
     ========================================================= */
  .btn,
  .btn-primary,
  .btn-success,
  .btn-info,
  .btn-warning,
  .btn-danger,
  .btn-secondary {
    background-color: var(--vivo-primary-500) !important;
    border-color: var(--vivo-primary-500) !important;
    color: #fff !important;
  }
  .btn:hover,
  .btn:focus {
    background-color: var(--vivo-primary-400) !important;
    border-color: var(--vivo-primary-400) !important;
    color: #fff !important;
  }
  .btn-outline-secondary,
  .btn-outline-primary {
    color: var(--vivo-primary-500) !important;
    border-color: var(--vivo-primary-500) !important;
    background: transparent !important;
  }
  .btn-outline-secondary:hover,
  .btn-outline-primary:hover {
    color: #fff !important;
    background-color: var(--vivo-primary-500) !important;
    border-color: var(--vivo-primary-500) !important;
  }
  
  /* =========================================================
     Cards & Títulos
     ========================================================= */
  .card { border-color: rgba(102, 0, 153, .20); }
  .card-title,
  h1,h2,h3,h4,h5,h6 { color: var(--vivo-primary-500); }
  
  /* =========================================================
     Links
     ========================================================= */
  a { color: var(--vivo-primary-500); }
  a:hover { color: var(--vivo-primary-400); }
  
  /* =========================================================
     Tabela — cabeçalho duplo “sticky” + rolagem
     ========================================================= */
  .table-responsive {
    max-height: calc(100vh - 280px);
    overflow: auto;
    border-top: 1px solid rgba(0,0,0,.06);
  }
  .table thead tr:first-child th {
    position: sticky;
    top: 0;
    z-index: 3;
    background: var(--bs-body-bg, #fff);
    box-shadow: inset 0 -1px 0 rgba(0,0,0,.08);
    text-transform: uppercase;
    font-weight: 600;
    color: var(--vivo-primary-600);
  }
  .table thead tr:nth-child(2) th {
    position: sticky;
    top: 42px;
    z-index: 2;
    background: var(--bs-body-bg, #fff);
    box-shadow: inset 0 -1px 0 rgba(0,0,0,.06);
  }
  .table td, .table th {
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
    vertical-align: middle;
  }
  .table-hover > tbody > tr:hover {
    --bs-table-accent-bg: rgba(102, 0, 153, .06);
  }
  
  /* Filtros (inputs) */
  .filtro-coluna,
  .filtro-data-inicial,
  .filtro-data-final {
    min-width: 10rem;
  }
  
  /* Paginação */
  #paginacao .pagination .page-link {
    border-color: rgba(102, 0, 153, .25);
    color: var(--vivo-primary-500);
    background: transparent;
  }
  #paginacao .pagination .page-item.active .page-link {
    background-color: var(--vivo-primary-500);
    border-color: var(--vivo-primary-500);
    color: #fff;
  }
  
  /* =========================================================
     Dashboard — gráfico estável
     ========================================================= */
  .chart-wrap { position: relative; width: 100%; height: 240px; }
  #grafico-rede { display: block; width: 100% !important; height: 100% !important; }
  
  #contagem-stfc, #contagem-smp { font-variant-numeric: tabular-nums; }
  
  /* =========================================================
     Loading overlay
     ========================================================= */
  #carregando { background: rgba(0,0,0,.35); z-index: 1080; }
  
  /* =========================================================
     Tema escuro
     ========================================================= */
  html[data-bs-theme="dark"] {
    --bs-body-bg: var(--vivo-bg-dark);
    --bs-body-color: var(--vivo-text-light);
  }
  html[data-bs-theme="dark"] .navbar { border-bottom-color: rgba(255,255,255,.12); }
  html[data-bs-theme="dark"] .card {
    background-color: #171923;
    border-color: rgba(255,255,255,.08);
  }
  html[data-bs-theme="dark"] .table thead tr:first-child th,
  html[data-bs-theme="dark"] .table thead tr:nth-child(2) th {
    background: #171923;
    color: #e9d7ff;
    box-shadow: inset 0 -1px 0 rgba(255,255,255,.06);
  }
  html[data-bs-theme="dark"] #paginacao .pagination .page-link {
    color: #e9d7ff; border-color: rgba(255,255,255,.15);
  }
  html[data-bs-theme="dark"] #paginacao .pagination .page-item.active .page-link {
    background-color: var(--vivo-primary-400);
    border-color: var(--vivo-primary-400);
  }
  
  /* =========================================================
     Mensagem inicial
     ========================================================= */
  #mensagem-inicial.alert {
    border-color: rgba(102, 0, 153, .25);
    background: var(--vivo-bg-light);
    color: #3b0764;
  }
  
  /* Detalhes */
  #card-tabela .card-footer { border-top: 2px solid rgba(102,0,153,.12); }
  .sticky-top { top: 0; }
  
  /* =========================================================
     Agente IA — container e bolhas (largura total abaixo da tabela)
     ========================================================= */
  #ia-mensagens {
    background: var(--bs-body-bg);
    border: 1px solid rgba(0,0,0,.06);
    border-radius: .5rem;
    padding: .5rem .75rem;
    min-height: 260px;            /* altura confortável mínima */
    max-height: 60vh;             /* limita altura para caber na tela */
    overflow: auto;
    scroll-behavior: auto;        /* evita smooth/loops de scroll */
  }
  html[data-bs-theme="dark"] #ia-mensagens {
    border-color: rgba(255,255,255,.10);
  }
  
  .ia-msg {
    background: transparent;
    margin-bottom: .5rem;
    line-height: 1.35;
  }
  
  .ia-msg.ia-user {
    text-align: right;
    background: rgba(102, 0, 153, .08);
    color: var(--vivo-primary-600);
    border: 1px solid rgba(102,0,153,.20);
    border-radius: .66rem;
    padding: .4rem .6rem;
    display: inline-block;
    max-width: 100%;
  }
  
  .ia-msg.ia-bot {
    background: rgba(0,0,0,.04);
    border: 1px solid rgba(0,0,0,.08);
    border-radius: .66rem;
    padding: .55rem .7rem;
    display: block;
  }
  html[data-bs-theme="dark"] .ia-msg.ia-bot {
    background: rgba(255,255,255,.04);
    border-color: rgba(255,255,255,.12);
  }
  
  /* Botões de sugestão do agente */
  .ia-sugestao.btn {
    padding: .2rem .55rem;
    border-width: 1px;
    font-size: .85rem;
  }
  
  /* =========================================================
     Responsividade
     ========================================================= */
  @media (max-width: 991.98px) {
    .table-responsive {
      max-height: calc(100vh - 360px);
    }
    #ia-mensagens {
      max-height: 50vh;
    }
  }





