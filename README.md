{% extends "base.html" %}
{% block title %}VIVOHUB — Formulários do Atacado{% endblock %}

{% block extra_head %}
<style>
  /* Cartão padrão */
  .card-shell{
    border:1px solid var(--vh-border);
    border-radius: var(--vh-radius);
    background:var(--vh-surface-0);
    box-shadow: var(--vh-shadow-md);
  }

  /* Chips de estado/contagem */
  .chip{
    display:inline-flex; align-items:center; gap:.45rem;
    border-radius:999px; padding:.38rem .75rem; font-weight:600; line-height:1;
    background:var(--vh-surface-0); border:1px solid var(--vh-border); color:var(--vh-muted);
    transition: background .18s ease, color .18s ease, border-color .18s ease, transform .18s ease;
  }
  .chip:hover{ background:var(--vh-surface-1); color:var(--vh-ink); text-decoration:none; }
  .chip.active{ background:rgba(107,9,166,.08); border-color:rgba(107,9,166,.25); color:var(--vh-primary); }

  /* Tabela */
  .table thead th{ white-space:nowrap; font-weight:600; }
  .table-hover tbody tr:hover{ background:#fbfbff; }
  .op-name{ max-width: 520px; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .id-copy{ cursor:pointer; opacity:.65; }
  .id-copy:hover{ opacity:1; }

  /* Badges de status (padrão do produto) */
  .badge-status{ font-weight:600; border:1px solid transparent; }
  .badge-status.rascunho{ background:#eef1f4; color:#2b2f32; border-color:#e1e6ea; }
  .badge-status.enviado{ background:#cfe2ff; color:#084298; border-color:#b6d0ff; }
  .badge-status.em-revisao{ background:#fff3cd; color:#664d03; border-color:#ffe29a; }
  .badge-status.aprovado{ background:#d1e7dd; color:#0f5132; border-color:#bcd9cc; }

  /* Barra de ações responsiva */
  .actions-wrap{ gap:.75rem; }
  @media (max-width: 576px){
    .actions-wrap{ flex-direction: column; align-items: stretch !important; }
    .actions-left, .actions-right{ width:100%; }
    .actions-right form{ width:100%; }
    .actions-right .input-group{ width:100%; }
  }

  /* Modal */
  .modal-content{ border-radius: var(--vh-radius); box-shadow: var(--vh-shadow-lg); }
</style>
{% endblock %}

{% block content %}
<div class="container py-5" role="main">
  <div class="card-shell">
    <div class="p-4 p-xl-5">

      <!-- Cabeçalho -->
      <header class="d-flex align-items-center justify-content-between mb-3">
        <div>
          <h1 class="h4 fw-bold mb-1">Formulários — Atacado</h1>
          <div class="text-body-secondary small">
            Crie, edite e acompanhe seus Pré-PTIs.
            <span class="ms-1">Atalhos: <kbd>N</kbd> novo • <kbd>/</kbd> buscar.</span>
          </div>
        </div>
        <div class="d-flex gap-2">
          <a href="{{ url_for('central_atacado') }}"
             class="btn btn-outline-hub btn-sm"
             data-bs-toggle="tooltip" data-bs-title="Voltar para a Central"
             aria-label="Voltar para a Central">
            ← Central
          </a>
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

          <a class="chip me-1 text-decoration-none {% if s=='rascunho' %}active{% endif %}"
             href="?status=rascunho&q={{ q }}&sort={{ sort }}">
            <i class="bi bi-pencil-square"></i> {{ counters.rascunho or 0 }} Rascunhos
          </a>

          <a class="chip me-1 text-decoration-none {% if s=='enviado' %}active{% endif %}"
             href="?status=enviado&q={{ q }}&sort={{ sort }}">
            <i class="bi bi-send"></i> {{ counters.enviado or 0 }} Enviados
          </a>

          <a class="chip me-1 text-decoration-none {% if s=='em revisão' %}active{% endif %}"
             href="?status=em revisão&q={{ q }}&sort={{ sort }}">
            <i class="bi bi-hourglass-split"></i> {{ counters.em_revisao or 0 }} Em revisão
          </a>

          <a class="chip me-1 text-decoration-none {% if s=='aprovado' %}active{% endif %}"
             href="?status=aprovado&q={{ q }}&sort={{ sort }}">
            <i class="bi bi-check2-circle"></i> {{ counters.aprovado or 0 }} Aprovados
          </a>
        </section>
      {% endif %}

      <!-- Ações: criar | filtros | busca -->
      <section class="d-flex align-items-center justify-content-between mb-4 actions-wrap">
        <div class="actions-left d-flex align-items-center">
          <a href="{{ url_for('atacado_form_new') }}"
             class="btn-hub"
             data-bs-toggle="tooltip"
             data-bs-title="Criar um novo formulário de Pré-PTI (atalho: N)"
             aria-label="Criar um novo formulário de Pré-PTI">
            <i class="bi bi-file-earmark-plus me-2"></i> Novo formulário
          </a>
        </div>

        <div class="actions-right d-flex align-items-center gap-2">
          <!-- Filtros -->
          <form method="get" class="d-flex align-items-center gap-2" aria-label="Filtros">
            <input type="hidden" name="q" value="{{ request.args.get('q','') }}">
            {% set s = request.args.get('status','') %}
            {% set sort = request.args.get('sort','-created_at') %}

            <select name="status" class="form-select form-select-sm" aria-label="Filtro de status">
              <option value=""  {{ 'selected' if s=='' else '' }}>Todos os status</option>
              <option value="rascunho" {{ 'selected' if s=='rascunho' else '' }}>Rascunho</option>
              <option value="enviado" {{ 'selected' if s=='enviado' else '' }}>Enviado</option>
              <option value="em revisão" {{ 'selected' if s=='em revisão' else '' }}>Em revisão</option>
              <option value="aprovado" {{ 'selected' if s=='aprovado' else '' }}>Aprovado</option>
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
               href="{{ url_for('atacado_form_list') }}"
               data-bs-toggle="tooltip" data-bs-title="Limpar filtros">
              Limpar
            </a>
          </form>

          <!-- Busca -->
          <form method="get" class="d-flex" style="max-width:280px;" aria-label="Busca por Operadora">
            <input type="hidden" name="status" value="{{ s }}">
            <input type="hidden" name="sort" value="{{ sort }}">
            <div class="input-group input-group-sm">
              <span class="input-group-text"><i class="bi bi-search"></i></span>
              <input type="text"
                     id="searchInput"
                     name="q"
                     value="{{ request.args.get('q','') }}"
                     class="form-control"
                     placeholder="Buscar por operadora… (atalho: /)"
                     aria-label="Buscar por operadora">
              <button type="submit" class="btn btn-outline-hub">Buscar</button>
            </div>
          </form>
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
                {% set st = (f.status or 'rascunho')|lower %}
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
                      {% if st=='aprovado' %}aprovado
                      {% elif st=='em revisão' %}em-revisao
                      {% elif st=='enviado' %}enviado
                      {% else %}rascunho{% endif %}">
                      {{ st|capitalize }}
                    </span>
                  </td>
                  <td class="text-body-secondary">
                    {{ (f.created_at or '')|date_br }}
                  </td>
                  <td class="text-end">
                    <div class="btn-group btn-group-sm" role="group" aria-label="Ações do formulário {{ f.id }}">
                      <a class="btn btn-outline-hub"
                         href="{{ url_for('atacado_form_edit', form_id=f.id) }}"
                         data-bs-toggle="tooltip" data-bs-title="Editar formulário">
                        <i class="bi bi-pencil-square"></i> Editar
                      </a>

                      <button type="button"
                              class="btn btn-outline-hub"
                              data-bs-toggle="modal"
                              data-bs-target="#confirmDeleteModal"
                              data-form-id="{{ f.id }}"
                              data-op-name="{{ f.nome_operadora or '—' }}">
                        <i class="bi bi-trash"></i> Excluir
                      </button>
                    </div>
                  </td>
                </tr>
              {% endfor %}
            </tbody>
          </table>
        </div>

        <!-- Paginação -->
        {% if page and pages and pages > 1 %}
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

      {% else %}
        <!-- Estado vazio -->
        <section class="text-center py-5">
          <div class="mb-3"><i class="bi bi-inbox fs-1 text-secondary" aria-hidden="true"></i></div>
          <h2 class="h5">Nenhum formulário encontrado</h2>
          {% if request.args.get('q') %}
            <p class="text-body-secondary mb-3">Sua busca por <strong>{{ request.args.get('q') }}</strong> não retornou resultados.</p>
          {% else %}
            <p class="text-body-secondary mb-3">Você ainda não criou formulários de Pré-PTI.</p>
          {% endif %}
          <a href="{{ url_for('atacado_form_new') }}" class="btn-hub">
            <i class="bi bi-file-earmark-plus me-2"></i> Criar primeiro formulário
          </a>
        </section>
      {% endif %}

    </div>
  </div>
</div>

<!-- Modal de confirmação de exclusão -->
<div class="modal fade" id="confirmDeleteModal" tabindex="-1" aria-labelledby="confirmDeleteLabel" aria-hidden="true">
  <div class="modal-dialog modal-dialog-centered">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title d-flex align-items-center gap-2" id="confirmDeleteLabel">
          <i class="bi bi-trash3 text-danger"></i>
          Excluir formulário
        </h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Fechar"></button>
      </div>
      <div class="modal-body">
        <p class="mb-0">Tem certeza de que deseja excluir o formulário <span class="fw-semibold" id="delFormId">#—</span> (<span id="delOpName">—</span>)?</p>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-outline-hub" data-bs-dismiss="modal">Cancelar</button>
        <form method="post" id="deleteRealForm" action="#">
          <button type="submit" class="btn btn-danger">
            <i class="bi bi-trash"></i> Excluir
          </button>
        </form>
      </div>
    </div>
  </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script>
  // Tooltips
  document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(el=>{ new bootstrap.Tooltip(el); });

  // Live region discreto (acessibilidade)
  const live = document.createElement('div');
  live.className = 'visually-hidden';
  live.setAttribute('aria-live','polite');
  document.body.appendChild(live);
  const speak = (msg)=>{ live.textContent = msg || ''; };

  // Copiar ID para a área de transferência
  document.querySelectorAll('.id-copy').forEach(el=>{
    el.addEventListener('click', async ()=>{
      const id = el.getAttribute('data-id');
      try{
        await navigator.clipboard.writeText(id);
        el.setAttribute('data-bs-title','Copiado!');
        bootstrap.Tooltip.getInstance(el)?.show();
        speak(`ID ${id} copiado.`);
        setTimeout(()=>{
          el.setAttribute('data-bs-title','Copiar ID');
          bootstrap.Tooltip.getInstance(el)?.hide();
        }, 900);
      }catch(_){ speak('Não foi possível copiar o ID.'); }
    });
  });

  // Realçar termo buscado no nome da operadora
  (function(){
    const q = new URLSearchParams(window.location.search).get('q');
    if(!q) return;
    const regex = new RegExp('(' + q.replace(/[.*+?^${}()|[\\]\\\\]/g,'\\\\$&') + ')','gi');
    document.querySelectorAll('#resultsBody .op-name').forEach(td=>{
      const txt = td.textContent;
      td.innerHTML = txt.replace(regex, '<mark>$1</mark>');
    });
  })();

  // Atalhos: N (novo), / (buscar)
  document.addEventListener('keydown', (e)=>{
    const tag = (e.target && e.target.tagName) || '';
    const isTyping = ['INPUT','TEXTAREA','SELECT'].includes(tag);

    if (!isTyping && (e.key === 'n' || e.key === 'N')){
      e.preventDefault();
      window.location.href = "{{ url_for('atacado_form_new') }}";
    }
    if (e.key === '/'){
      e.preventDefault();
      const si = document.getElementById('searchInput');
      if(si){ si.focus(); si.select(); }
    }
  });

  // Modal de exclusão: preenche formulário e labels
  const modalEl = document.getElementById('confirmDeleteModal');
  if(modalEl){
    modalEl.addEventListener('show.bs.modal', (ev)=>{
      const btn = ev.relatedTarget;
      const id = btn?.getAttribute('data-form-id');
      const op = btn?.getAttribute('data-op-name') || '—';
      document.getElementById('delFormId').textContent = `#${id}`;
      document.getElementById('delOpName').textContent = op;
      const form = document.getElementById('deleteRealForm');
      form.setAttribute('action', "{{ url_for('atacado_form_delete', form_id=0) }}".replace('0', id));
    });
  }

  // Prefetch leve para navegação mais fluida
  try{
    const linkNew = document.createElement('link'); linkNew.rel='prefetch'; linkNew.href="{{ url_for('atacado_form_new') }}"; document.head.appendChild(linkNew);
    const linkList = document.createElement('link'); linkList.rel='prefetch'; linkList.href="{{ url_for('atacado_form_list') }}"; document.head.appendChild(linkList);
  }catch(_){}
</script>
{% endblock %}
