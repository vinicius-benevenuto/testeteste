{% extends "base.html" %}
{% block title %}VIVOHUB — Formulários (Engenharia){% endblock %}

{% block extra_head %}
<style>
  /* ===== CARTÕES ===== */
  .card-shell{
    border:1px solid var(--vh-border);
    border-radius: var(--vh-radius);
    background:var(--vh-surface-0);
    box-shadow: var(--vh-shadow-md);
  }

  /* ===== CHIPS DE STATUS ===== */
  .chip{
    display:inline-flex; align-items:center; gap:.45rem;
    border-radius:999px; padding:.38rem .75rem; font-weight:600; line-height:1;
    background:var(--vh-surface-0); border:1px solid var(--vh-border); color:var(--vh-muted);
    transition: background .18s ease, color .18s ease, border-color .18s ease, transform .18s ease;
  }
  .chip:hover{ background:var(--vh-surface-1); color:var(--vh-ink); text-decoration:none; }
  .chip.active{ background:rgba(107,9,166,.08); border-color:rgba(107,9,166,.25); color:var(--vh-primary); }

  /* ===== TABELA ===== */
  .table thead th{ white-space:nowrap; font-weight:600; }
  .table-hover tbody tr:hover{ background:#fbfbff; }
  .id-copy{ cursor:pointer; opacity:.65; }
  .id-copy:hover{ opacity:1; }
  .op-name{ max-width:520px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }

  /* ===== BADGES DE STATUS ===== */
  .badge-status{ font-weight:600; border:1px solid transparent; }
  .badge-status.rascunho{ background:#eef1f4; color:#2b2f32; border-color:#e1e6ea; }
  .badge-status.enviado{ background:#cfe2ff; color:#084298; border-color:#b6d0ff; }
  .badge-status.em-revisao{ background:#fff3cd; color:#664d03; border-color:#ffe29a; }
  .badge-status.aprovado{ background:#d1e7dd; color:#0f5132; border-color:#bcd9cc; }

  /* ===== BARRA DE AÇÕES ===== */
  .actions-wrap{ gap:.75rem; }
  @media (max-width: 576px){
    .actions-wrap{ flex-direction:column; align-items:stretch !important; }
    .actions-left,.actions-right{ width:100%; }
    .actions-right form{ width:100%; }
    .actions-right .input-group{ width:100%; }
  }

  /* ===== EXPORTS ===== */
  .exports-item{
    display:grid; grid-template-columns:1fr auto; gap:.25rem .75rem;
    border:1px dashed var(--vh-border); border-radius:.65rem; background:var(--vh-surface-0);
    padding:.65rem .75rem;
  }
  .exports-item + .exports-item{ margin-top:.5rem; }
  .exports-name{ font-weight:600; }
  .exports-meta{ font-size:.85rem; color:var(--vh-muted); }

  /* ===== DASHBOARD SBC ===== */
  .sbc-dash{
    border:1px solid var(--vh-border);
    border-radius: var(--vh-radius);
    background: linear-gradient(135deg, #f8f5ff 0%, #fefefe 100%);
    padding: 1.25rem 1.5rem;
    margin-bottom: 1.25rem;
  }
  .sbc-dash-header{
    display:flex; align-items:center; justify-content:space-between;
    margin-bottom: .75rem;
  }
  .sbc-dash-title{
    font-size: .95rem; font-weight: 700; color: var(--vh-primary);
    display:flex; align-items:center; gap:.5rem;
  }
  .sbc-dash-grid{
    display:grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
    gap: .75rem;
  }
  .sbc-stat{
    background: var(--vh-surface-0);
    border:1px solid var(--vh-border);
    border-radius: .5rem;
    padding: .6rem .75rem;
    text-align:center;
  }
  .sbc-stat-value{ font-size:1.35rem; font-weight:800; line-height:1.2; }
  .sbc-stat-label{ font-size:.72rem; color:var(--vh-muted); text-transform:uppercase; letter-spacing:.04em; font-weight:600; }
  .sbc-stat.ok .sbc-stat-value{ color:#0f5132; }
  .sbc-stat.warn .sbc-stat-value{ color:#664d03; }
  .sbc-stat.crit .sbc-stat-value{ color:#842029; }
  .sbc-stat.info .sbc-stat-value{ color:var(--vh-primary); }

  .sbc-source{
    font-size:.75rem; color:var(--vh-muted); margin-top:.6rem;
    display:flex; align-items:center; gap:.4rem;
  }
  .sbc-loading{
    display:flex; align-items:center; justify-content:center;
    padding:1rem; color:var(--vh-muted); gap:.5rem;
  }
  .sbc-error{
    padding:.6rem .75rem; background:#fff3cd; border:1px solid #ffe29a;
    border-radius:.5rem; font-size:.85rem; color:#664d03;
  }

  /* Animação de loading */
  @keyframes sbc-pulse{ 0%,100%{opacity:.4} 50%{opacity:1} }
  .sbc-loading .spinner{ animation: sbc-pulse 1.2s ease-in-out infinite; }
</style>
{% endblock %}

{% block content %}
<div class="container py-5" role="main">

  {# ============================================================
     DASHBOARD DE SBC — Visão geral rápida para a engenharia
     Carregado via AJAX do endpoint /api/sbc/overview
     ============================================================ #}
  <div id="sbcDashboard" class="sbc-dash" role="region" aria-label="Dashboard de SBCs">
    <div class="sbc-loading" id="sbcDashLoading">
      <span class="spinner">📡</span> Carregando dados de SBC...
    </div>
    <div id="sbcDashContent" style="display:none;">
      <div class="sbc-dash-header">
        <div class="sbc-dash-title">
          <i class="bi bi-broadcast-pin"></i> Panorama de SBCs
        </div>
        <div class="d-flex gap-2 align-items-center">
          <button class="btn btn-outline-hub btn-sm" id="sbcReloadBtn"
                  data-bs-toggle="tooltip" data-bs-title="Recarregar dados do CSV">
            <i class="bi bi-arrow-clockwise"></i> Atualizar
          </button>
        </div>
      </div>
      <div class="sbc-dash-grid" id="sbcDashGrid">
        <!-- Preenchido via JS -->
      </div>
      <div class="sbc-source" id="sbcDashSource">
        <!-- Preenchido via JS -->
      </div>
    </div>
    <div id="sbcDashError" class="sbc-error" style="display:none;"></div>
  </div>

  <div class="row g-4">
    <!-- LISTA DE FORMULÁRIOS -->
    <div class="col-12 {% if show_files %}col-xxl-8{% else %}col-xxl-12{% endif %}">
      <div class="card-shell">
        <div class="p-4 p-xl-5">

          <!-- Cabeçalho -->
          <header class="d-flex align-items-center justify-content-between mb-3">
            <div>
              <h1 class="h4 fw-bold mb-1">Formulários — Engenharia</h1>
              <div class="text-body-secondary small">PTIs submetidos pelo Atacado para leitura e validação (Seção 9).</div>
            </div>
            <div class="d-flex gap-2">
              <a href="{{ url_for('central_engenharia') }}"
                 class="btn btn-outline-hub btn-sm"
                 data-bs-toggle="tooltip" data-bs-title="Voltar para a Central"
                 aria-label="Voltar para a Central">← Central</a>
              <a href="{{ url_for('logout') }}"
                 class="btn btn-outline-hub btn-sm"
                 data-bs-toggle="tooltip" data-bs-title="Encerrar sessão"
                 aria-label="Encerrar sessão">
                <i class="bi bi-box-arrow-right"></i> Sair
              </a>
            </div>
          </header>

          <!-- Subheader com contadores -->
          {% if counters %}
          {% set q = request.args.get('q','') %}
          {% set sort = request.args.get('sort','-created_at') %}
          {% set s = request.args.get('status','') %}
          <section class="mb-3 small text-body-secondary" aria-label="Resumo por status">
            <span class="me-2">Total:</span>

            <a class="chip me-1 text-decoration-none {% if not s %}active{% endif %}"
               href="?q={{ q }}&sort={{ sort }}"
               aria-current="{{ 'true' if not s else 'false' }}">
              <i class="bi bi-collection"></i> {{ counters.total or 0 }} Todos
            </a>

            <a class="chip me-1 text-decoration-none {% if s=='em revisão' %}active{% endif %}"
               href="?status=em revisão&q={{ q }}&sort={{ sort }}">
              <i class="bi bi-hourglass-split"></i> {{ counters.em_revisao or 0 }} Em revisão
            </a>

            <a class="chip me-1 text-decoration-none {% if s=='enviado' %}active{% endif %}"
               href="?status=enviado&q={{ q }}&sort={{ sort }}">
              <i class="bi bi-send"></i> {{ counters.enviado or 0 }} Enviados
            </a>

            <a class="chip me-1 text-decoration-none {% if s=='aprovado' %}active{% endif %}"
               href="?status=aprovado&q={{ q }}&sort={{ sort }}">
              <i class="bi bi-check2-circle"></i> {{ counters.aprovado or 0 }} Aprovados
            </a>

            <a class="chip me-1 text-decoration-none {% if s=='rascunho' %}active{% endif %}"
               href="?status=rascunho&q={{ q }}&sort={{ sort }}">
              <i class="bi bi-pencil-square"></i> {{ counters.rascunho or 0 }} Rascunhos
            </a>
          </section>
          {% endif %}

          <!-- Ações: filtros/ordenar/busca -->
          <section class="d-flex align-items-center justify-content-between mb-4 actions-wrap">
            <div class="actions-left d-flex align-items-center">
              <span class="text-body-secondary small">Use filtros, ordene e busque por operadora.</span>
            </div>
            <div class="actions-right d-flex align-items-center gap-2">
              <!-- Filtros -->
              <form method="get" class="d-flex align-items-center gap-2" aria-label="Filtros">
                <input type="hidden" name="q" value="{{ q }}">
                <select name="status" class="form-select form-select-sm" aria-label="Filtro de status">
                  <option value=""  {{ 'selected' if s=='' else '' }}>Todos os status</option>
                  <option value="em revisão" {{ 'selected' if s=='em revisão' else '' }}>Em revisão</option>
                  <option value="enviado" {{ 'selected' if s=='enviado' else '' }}>Enviado</option>
                  <option value="aprovado" {{ 'selected' if s=='aprovado' else '' }}>Aprovado</option>
                  <option value="rascunho" {{ 'selected' if s=='rascunho' else '' }}>Rascunho</option>
                </select>

                <select name="sort" class="form-select form-select-sm" aria-label="Ordenação">
                  <option value="-created_at" {{ 'selected' if sort=='-created_at' else '' }}>Mais recentes</option>
                  <option value="created_at"  {{ 'selected' if sort=='created_at' else '' }}>Mais antigos</option>
                  <option value="nome_operadora" {{ 'selected' if sort=='nome_operadora' else '' }}>Operadora (A–Z)</option>
                  <option value="-nome_operadora" {{ 'selected' if sort=='-nome_operadora' else '' }}>Operadora (Z–A)</option>
                  <option value="-id" {{ 'selected' if sort=='-id' else '' }}>ID (maior→menor)</option>
                  <option value="id" {{ 'selected' if sort=='id' else '' }}>ID (menor→maior)</option>
                </select>

                <button type="submit" class="btn btn-outline-hub btn-sm"
                        data-bs-toggle="tooltip" data-bs-title="Aplicar filtros">
                  <i class="bi bi-funnel"></i>
                </button>
                <a class="btn btn-outline-hub btn-sm"
                   href="{{ url_for('engenharia_form_list') }}"
                   data-bs-toggle="tooltip" data-bs-title="Limpar filtros">
                  Limpar
                </a>
              </form>

              <!-- Busca -->
              <form method="get" class="d-flex" style="max-width:280px;" aria-label="Buscar por Operadora">
                <input type="hidden" name="status" value="{{ s }}">
                <input type="hidden" name="sort" value="{{ sort }}">
                <div class="input-group input-group-sm">
                     <span class="input-group-text"><i class="bi bi-search"></i></span>
                     <input type="text"
                            name="q"
                            value="{{ q }}"
                            class="form-control"
                            placeholder="Buscar por operadora…"
                            aria-label="Buscar por operadora">
                     <button type="submit" class="btn btn-outline-hub">Buscar</button>
                </div>
              </form>

              <!-- Alternar biblioteca -->
              <a class="btn btn-hub btn-sm"
                 href="{{ url_for('engenharia_form_list') }}?show_files={{ 0 if show_files else 1 }}{% if q %}&q={{ q }}{% endif %}{% if s %}&status={{ s }}{% endif %}{% if sort %}&sort={{ sort }}{% endif %}">
                <i class="bi {{ 'bi-view-list' if show_files else 'bi-folder2-open' }}"></i>
                {{ 'Ocultar exports' if show_files else 'Ver exports' }}
              </a>
            </div>
          </section>

          <!-- Lista -->
          {% if forms and forms|length %}
            <div class="table-responsive">
              <table class="table align-middle table-hover">
                <thead class="table-light">
                  <tr>
                    <th class="text-nowrap">ID</th>
                    <th>Operadora</th>
                    <th>Status</th>
                    <th class="text-nowrap">Criado em</th>
                    <th class="text-end">Ações</th>
                  </tr>
                </thead>
                <tbody id="resultsBody">
                  {% for f in forms %}
                    {% set status_lbl = (f.status or 'rascunho')|lower %}
                    <tr>
                      <td class="fw-semibold">
                        #{{ f.id }}
                        <i class="bi bi-clipboard-check id-copy ms-1"
                           data-id="{{ f.id }}"
                           data-bs-toggle="tooltip" data-bs-title="Copiar ID"
                           aria-label="Copiar ID"></i>
                      </td>
                      <td class="op-name" title="{{ f.nome_operadora or '—' }}">{{ f.nome_operadora or '—' }}</td>
                      <td>
                        <span class="badge badge-status
                          {% if status_lbl=='aprovado' %}aprovado
                          {% elif status_lbl=='em revisão' %}em-revisao
                          {% elif status_lbl=='enviado' %}enviado
                          {% else %}rascunho{% endif %}">
                          {{ status_lbl|capitalize }}
                        </span>
                      </td>
                      <td class="text-body-secondary">
                        {{ f.created_at | date_br }}
                      </td>
                      <td class="text-end">
                        <div class="btn-group btn-group-sm" role="group" aria-label="Ações do formulário {{ f.id }}">
                          <a class="btn btn-outline-hub"
                             href="{{ url_for('engenharia_form_view', form_id=f.id) }}"
                             data-bs-toggle="tooltip" data-bs-title="Abrir para leitura/validação (Seção 9)">
                            <i class="bi bi-eye"></i> Abrir
                          </a>
                          <a class="btn btn-hub"
                             href="{{ url_for('exportar_form_excel_index', form_id=f.id) }}"
                             data-bs-toggle="tooltip" data-bs-title="Exportar Excel (Índice/Versões/Diagrama)">
                            <i class="bi bi-file-earmark-spreadsheet"></i> Gerar PTI
                          </a>
                        </div>
                      </td>
                    </tr>
                  {% endfor %}
                </tbody>
              </table>
            </div>
          {% else %}
            <!-- Estado vazio -->
            <section class="text-center py-5">
              <div class="mb-3"><i class="bi bi-inbox fs-1 text-secondary" aria-hidden="true"></i></div>
              <h2 class="h5">Nenhum formulário disponível</h2>
              {% if q %}
                <p class="text-body-secondary mb-0">Sua busca por <strong>{{ q }}</strong> não retornou resultados.</p>
              {% else %}
                <p class="text-body-secondary mb-0">Assim que o Atacado enviar um Pré-PTI, ele aparecerá aqui para validação.</p>
              {% endif %}
            </section>
          {% endif %}

          {# Paginação — opcional #}
          {% if page and pages and pages>1 %}
          <nav class="mt-3" aria-label="Paginação">
            <ul class="pagination justify-content-end">
              <li class="page-item {% if page<=1 %}disabled{% endif %}">
                <a class="page-link"
                   href="?q={{ q }}&status={{ s }}&sort={{ sort }}&page={{ page-1 }}"
                   aria-label="Anterior">Anterior</a>
              </li>
              {% for p in range(1, pages+1) %}
              <li class="page-item {% if p==page %}active{% endif %}">
                <a class="page-link"
                   href="?q={{ q }}&status={{ s }}&sort={{ sort }}&page={{ p }}"
                   aria-current="{{ 'page' if p==page else 'false' }}">{{ p }}</a>
              </li>
              {% endfor %}
              <li class="page-item {% if page>=pages %}disabled{% endif %}">
                <a class="page-link"
                   href="?q={{ q }}&status={{ s }}&sort={{ sort }}&page={{ page+1 }}"
                   aria-label="Próxima">Próxima</a>
              </li>
            </ul>
          </nav>
          {% endif %}

        </div>
      </div>
    </div>

    <!-- BIBLIOTECA DE EXPORTS (Persistidos) -->
    {% if show_files %}
    <div class="col-12 col-xxl-4">
      <div class="card-shell h-100">
        <div class="p-4">
          <div class="d-flex align-items-center justify-content-between mb-2">
            <h3 class="h6 fw-bold mb-0">
              <i class="bi bi-folder2-open me-1"></i> Biblioteca de Exports
            </h3>
            <a class="btn btn-outline-hub btn-sm"
               href="{{ url_for('engenharia_form_list') }}?show_files=1">
              Atualizar
            </a>
          </div>
          <p class="text-body-secondary small mb-3">
            Downloads persistidos no servidor. Você pode filtrar por Formulário (ID) e limpar itens.
          </p>

          <!-- Filtro por Form -->
          <form method="get" class="mb-3 d-flex align-items-center gap-2">
            <input type="hidden" name="show_files" value="1">
            <div class="input-group input-group-sm">
              <span class="input-group-text"><i class="bi bi-hash"></i></span>
              <input type="number" name="form" value="{{ form_filter or '' }}" class="form-control" placeholder="Filtrar por ID do formulário">
              <button type="submit" class="btn btn-outline-hub">Aplicar</button>
              <a class="btn btn-outline-hub" href="{{ url_for('engenharia_form_list') }}?show_files=1">Limpar</a>
            </div>
          </form>

          <!-- Lista de arquivos -->
          {% if exports and exports|length %}
            <div>
              {% for e in exports %}
                <div class="exports-item">
                  <div>
                    <div class="exports-name">{{ e.filename }}</div>
                    <div class="exports-meta">Form #{{ e.form_id }} • {{ e.nome_operadora or '—' }} • {{ e.created_at|date_br }} • {{ (e.size_bytes or 0) // 1024 }} KB</div>
                  </div>
                  <div class="d-flex gap-1">
                    <a class="btn btn-sm btn-hub" href="{{ url_for('engenharia_export_download', export_id=e.id) }}" title="Baixar">
                      <i class="bi bi-download"></i>
                    </a>
                    <form method="post" action="{{ url_for('engenharia_export_delete', export_id=e.id) }}"
                          onsubmit="return confirm('Remover este arquivo do servidor?');">
                      <button class="btn btn-sm btn-outline-hub" type="submit" title="Excluir">
                        <i class="bi bi-trash"></i>
                      </button>
                    </form>
                  </div>
                </div>
              {% endfor %}
            </div>
          {% else %}
            <div class="text-center text-body-secondary small py-4">
              <i class="bi bi-file-earmark-spreadsheet fs-3 d-block mb-2"></i>
              Nenhum arquivo encontrado.
            </div>
          {% endif %}
        </div>
      </div>
    </div>
    {% endif %}
  </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script>
  // =====================================================================
  // TOOLTIPS
  // =====================================================================
  document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(el => {
    new bootstrap.Tooltip(el);
  });

  // =====================================================================
  // ACESSIBILIDADE — live region para feedback
  // =====================================================================
  const live = document.createElement('div');
  live.className = 'visually-hidden';
  live.setAttribute('aria-live', 'polite');
  document.body.appendChild(live);
  const speak = (m) => { live.textContent = m || ''; };

  // =====================================================================
  // COPIAR ID
  // =====================================================================
  document.querySelectorAll('.id-copy').forEach(el => {
    el.addEventListener('click', async () => {
      const id = el.getAttribute('data-id');
      try {
        await navigator.clipboard.writeText(id);
        el.setAttribute('data-bs-title', 'Copiado!');
        bootstrap.Tooltip.getInstance(el)?.show();
        speak(`ID ${id} copiado.`);
        setTimeout(() => {
          el.setAttribute('data-bs-title', 'Copiar ID');
          bootstrap.Tooltip.getInstance(el)?.hide();
        }, 900);
      } catch (_) { speak('Não foi possível copiar o ID.'); }
    });
  });

  // =====================================================================
  // REALCE DO TERMO BUSCADO
  // =====================================================================
  (function () {
    const q = new URLSearchParams(location.search).get('q');
    if (!q) return;
    const regex = new RegExp('(' + q.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + ')', 'gi');
    document.querySelectorAll('#resultsBody .op-name').forEach(td => {
      const txt = td.textContent;
      td.innerHTML = txt.replace(regex, '<mark>$1</mark>');
    });
  })();

  // =====================================================================
  // DASHBOARD DE SBC — Carregamento via AJAX
  // =====================================================================
  (function () {
    const dashEl     = document.getElementById('sbcDashboard');
    const loadingEl  = document.getElementById('sbcDashLoading');
    const contentEl  = document.getElementById('sbcDashContent');
    const gridEl     = document.getElementById('sbcDashGrid');
    const sourceEl   = document.getElementById('sbcDashSource');
    const errorEl    = document.getElementById('sbcDashError');
    const reloadBtn  = document.getElementById('sbcReloadBtn');

    // Ícone e classe CSS por nível de saúde
    const HEALTH_CONFIG = {
      disponivel: { icon: '✅', label: 'Disponível', css: 'ok' },
      moderado:   { icon: '⚠️', label: 'Moderado',   css: 'warn' },
      atencao:    { icon: '🟠', label: 'Atenção',     css: 'warn' },
      critico:    { icon: '🔴', label: 'Crítico',     css: 'crit' },
    };

    function showError(msg) {
      loadingEl.style.display = 'none';
      contentEl.style.display = 'none';
      errorEl.style.display = 'block';
      errorEl.innerHTML = `<i class="bi bi-exclamation-triangle me-1"></i> ${msg}`;
    }

    function renderDashboard(data) {
      loadingEl.style.display = 'none';
      errorEl.style.display = 'none';
      contentEl.style.display = 'block';

      // Limpa grid anterior
      gridEl.innerHTML = '';

      // Total de SBCs
      gridEl.innerHTML += `
        <div class="sbc-stat info">
          <div class="sbc-stat-value">${data.total_sbcs || 0}</div>
          <div class="sbc-stat-label">Total SBCs</div>
        </div>`;

      // Estatísticas por saúde
      const healthOrder = ['disponivel', 'moderado', 'atencao', 'critico'];
      for (const level of healthOrder) {
        const count = (data.por_saude || {})[level] || 0;
        if (count === 0 && level !== 'disponivel') continue; // Omite zeros (exceto disponível)
        const cfg = HEALTH_CONFIG[level] || { icon: '❓', label: level, css: 'info' };
        gridEl.innerHTML += `
          <div class="sbc-stat ${cfg.css}">
            <div class="sbc-stat-value">${cfg.icon} ${count}</div>
            <div class="sbc-stat-label">${cfg.label}</div>
          </div>`;
      }

      // Por regional (top 3)
      const regionais = Object.entries(data.por_regional || {})
        .sort((a, b) => b[1] - a[1])
        .slice(0, 4);
      for (const [reg, count] of regionais) {
        gridEl.innerHTML += `
          <div class="sbc-stat info">
            <div class="sbc-stat-value">${count}</div>
            <div class="sbc-stat-label">${reg}</div>
          </div>`;
      }

      // Dados agregados do CSV (linhas sem SBC individual)
      if (data.dados_agregados_csv > 0) {
        gridEl.innerHTML += `
          <div class="sbc-stat">
            <div class="sbc-stat-value" style="color:var(--vh-muted);">${data.dados_agregados_csv}</div>
            <div class="sbc-stat-label">Linhas Agregadas</div>
          </div>`;
      }

      // Fonte do CSV
      const info = data.csv_info || {};
      if (info.filename) {
        sourceEl.innerHTML = `
          <i class="bi bi-file-earmark-text"></i>
          Fonte: <strong>${info.filename}</strong>
          &nbsp;·&nbsp; Modificado: ${info.modified_at || '—'}
          &nbsp;·&nbsp; ${info.total_measurements || 0} medições
          &nbsp;·&nbsp; Cache: ${info.cache_age_seconds || 0}s`;
      } else {
        sourceEl.innerHTML = `<i class="bi bi-exclamation-circle"></i> Nenhum CSV carregado.`;
      }
    }

    async function loadOverview(reload = false) {
      loadingEl.style.display = 'flex';
      contentEl.style.display = 'none';
      errorEl.style.display = 'none';

      try {
        const url = reload ? '/api/sbc/reload' : '/api/sbc/overview';
        const resp = await fetch(url);
        if (!resp.ok) {
          if (resp.status === 403 || resp.status === 401) {
            // Usuário não autenticado ou sem permissão
            dashEl.style.display = 'none';
            return;
          }
          throw new Error(`HTTP ${resp.status}`);
        }
        const data = await resp.json();

        // Se foi reload, busca o overview atualizado
        if (reload) {
          const resp2 = await fetch('/api/sbc/overview');
          if (resp2.ok) {
            renderDashboard(await resp2.json());
          }
          speak('Dados de SBC recarregados.');
        } else {
          renderDashboard(data);
        }
      } catch (err) {
        console.warn('SBC dashboard error:', err);
        // Não mostra erro se diretório não existe — apenas oculta
        if (err.message.includes('404') || err.message.includes('500')) {
          showError('Não foi possível carregar dados de SBC. Verifique se o diretório de CSVs está configurado.');
        } else {
          dashEl.style.display = 'none';
        }
      }
    }

    // Reload manual
    if (reloadBtn) {
      reloadBtn.addEventListener('click', () => {
        reloadBtn.disabled = true;
        reloadBtn.innerHTML = '<i class="bi bi-arrow-clockwise"></i> Recarregando...';
        loadOverview(true).finally(() => {
          reloadBtn.disabled = false;
          reloadBtn.innerHTML = '<i class="bi bi-arrow-clockwise"></i> Atualizar';
        });
      });
    }

    // Carregamento inicial
    loadOverview();
  })();

  // =====================================================================
  // PREFETCH
  // =====================================================================
  try {
    const linkA = document.createElement('link');
    linkA.rel = 'prefetch';
    linkA.href = "{{ url_for('engenharia_form_list') }}";
    document.head.appendChild(linkA);
  } catch (_) {}
</script>
{% endblock %}
