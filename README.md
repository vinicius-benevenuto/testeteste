{% extends "base.html" %}
{% block title %}VIVOHUB — Formulários (Engenharia){% endblock %}

{% block extra_head %}
<style>
  .card-shell{
    border:1px solid var(--vh-border);
    border-radius: var(--vh-radius);
    background:var(--vh-surface-0);
    box-shadow: var(--vh-shadow-md);
  }
  .chip{
    display:inline-flex; align-items:center; gap:.45rem;
    border-radius:999px; padding:.38rem .75rem; font-weight:600; line-height:1;
    background:var(--vh-surface-0); border:1px solid var(--vh-border); color:var(--vh-muted);
    transition: background .18s ease, color .18s ease, border-color .18s ease;
  }
  .chip:hover{ background:var(--vh-surface-1); color:var(--vh-ink); text-decoration:none; }
  .chip.active{ background:rgba(107,9,166,.08); border-color:rgba(107,9,166,.25); color:var(--vh-primary); }
  .table thead th{ white-space:nowrap; font-weight:600; }
  .table-hover tbody tr:hover{ background:#fbfbff; }
  .id-copy{ cursor:pointer; opacity:.65; }
  .id-copy:hover{ opacity:1; }
  .op-name{ max-width:520px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .badge-status{ font-weight:600; border:1px solid transparent; }
  .badge-status.rascunho{ background:#eef1f4; color:#2b2f32; border-color:#e1e6ea; }
  .badge-status.enviado{ background:#cfe2ff; color:#084298; border-color:#b6d0ff; }
  .badge-status.em-revisao{ background:#fff3cd; color:#664d03; border-color:#ffe29a; }
  .badge-status.aprovado{ background:#d1e7dd; color:#0f5132; border-color:#bcd9cc; }
  .actions-wrap{ gap:.75rem; }
  @media (max-width: 576px){
    .actions-wrap{ flex-direction:column; align-items:stretch !important; }
    .actions-left,.actions-right{ width:100%; }
    .actions-right form{ width:100%; }
    .actions-right .input-group{ width:100%; }
  }
  .exports-item{
    display:grid; grid-template-columns:1fr auto; gap:.25rem .75rem;
    border:1px dashed var(--vh-border); border-radius:.65rem; background:var(--vh-surface-0);
    padding:.65rem .75rem;
  }
  .exports-item + .exports-item{ margin-top:.5rem; }
  .exports-name{ font-weight:600; }
  .exports-meta{ font-size:.85rem; color:var(--vh-muted); }
</style>
{% endblock %}

{% block content %}
<div class="container py-5" role="main">
  <div class="row g-4">
    <div class="col-12 {% if show_files %}col-xxl-8{% else %}col-xxl-12{% endif %}">
      <div class="card-shell">
        <div class="p-4 p-xl-5">

          <header class="d-flex align-items-center justify-content-between mb-3">
            <div>
              <h1 class="h4 fw-bold mb-1">Formulários — Engenharia</h1>
              <div class="text-body-secondary small">PTIs submetidos pelo Atacado para leitura e validação (Seção 9).</div>
            </div>
            <div class="d-flex gap-2">
              <a href="{{ url_for('central_engenharia') }}" class="btn btn-outline-hub btn-sm" data-bs-toggle="tooltip" data-bs-title="Voltar para a Central">← Central</a>
              <a href="{{ url_for('logout') }}" class="btn btn-outline-hub btn-sm" data-bs-toggle="tooltip" data-bs-title="Encerrar sessão"><i class="bi bi-box-arrow-right"></i> Sair</a>
            </div>
          </header>

          {% if counters %}
          {% set q = request.args.get('q','') %}
          {% set sort = request.args.get('sort','-created_at') %}
          {% set s = request.args.get('status','') %}
          <section class="mb-3 small text-body-secondary" aria-label="Resumo por status">
            <span class="me-2">Total:</span>
            <a class="chip me-1 text-decoration-none {% if not s %}active{% endif %}" href="?q={{ q }}&sort={{ sort }}"><i class="bi bi-collection"></i> {{ counters.total or 0 }} Todos</a>
            <a class="chip me-1 text-decoration-none {% if s=='em revisão' %}active{% endif %}" href="?status=em revisão&q={{ q }}&sort={{ sort }}"><i class="bi bi-hourglass-split"></i> {{ counters.em_revisao or 0 }} Em revisão</a>
            <a class="chip me-1 text-decoration-none {% if s=='enviado' %}active{% endif %}" href="?status=enviado&q={{ q }}&sort={{ sort }}"><i class="bi bi-send"></i> {{ counters.enviado or 0 }} Enviados</a>
            <a class="chip me-1 text-decoration-none {% if s=='aprovado' %}active{% endif %}" href="?status=aprovado&q={{ q }}&sort={{ sort }}"><i class="bi bi-check2-circle"></i> {{ counters.aprovado or 0 }} Aprovados</a>
            <a class="chip me-1 text-decoration-none {% if s=='rascunho' %}active{% endif %}" href="?status=rascunho&q={{ q }}&sort={{ sort }}"><i class="bi bi-pencil-square"></i> {{ counters.rascunho or 0 }} Rascunhos</a>
          </section>
          {% endif %}

          <section class="d-flex align-items-center justify-content-between mb-4 actions-wrap">
            <div class="actions-left d-flex align-items-center">
              <span class="text-body-secondary small">Use filtros, ordene e busque por operadora.</span>
            </div>
            <div class="actions-right d-flex align-items-center gap-2">
              <form method="get" class="d-flex align-items-center gap-2" aria-label="Filtros">
                <input type="hidden" name="q" value="{{ q }}">
                <select name="status" class="form-select form-select-sm" aria-label="Filtro de status">
                  <option value="" {{ 'selected' if s=='' else '' }}>Todos os status</option>
                  <option value="em revisão" {{ 'selected' if s=='em revisão' else '' }}>Em revisão</option>
                  <option value="enviado" {{ 'selected' if s=='enviado' else '' }}>Enviado</option>
                  <option value="aprovado" {{ 'selected' if s=='aprovado' else '' }}>Aprovado</option>
                  <option value="rascunho" {{ 'selected' if s=='rascunho' else '' }}>Rascunho</option>
                </select>
                <select name="sort" class="form-select form-select-sm" aria-label="Ordenação">
                  <option value="-created_at" {{ 'selected' if sort=='-created_at' else '' }}>Mais recentes</option>
                  <option value="created_at" {{ 'selected' if sort=='created_at' else '' }}>Mais antigos</option>
                  <option value="nome_operadora" {{ 'selected' if sort=='nome_operadora' else '' }}>Operadora (A–Z)</option>
                  <option value="-nome_operadora" {{ 'selected' if sort=='-nome_operadora' else '' }}>Operadora (Z–A)</option>
                  <option value="-id" {{ 'selected' if sort=='-id' else '' }}>ID (maior→menor)</option>
                  <option value="id" {{ 'selected' if sort=='id' else '' }}>ID (menor→maior)</option>
                </select>
                <button type="submit" class="btn btn-outline-hub btn-sm" data-bs-toggle="tooltip" data-bs-title="Aplicar filtros"><i class="bi bi-funnel"></i></button>
                <a class="btn btn-outline-hub btn-sm" href="{{ url_for('engenharia_form_list') }}" data-bs-toggle="tooltip" data-bs-title="Limpar filtros">Limpar</a>
              </form>
              <form method="get" class="d-flex" style="max-width:280px;" aria-label="Buscar por Operadora">
                <input type="hidden" name="status" value="{{ s }}">
                <input type="hidden" name="sort" value="{{ sort }}">
                <div class="input-group input-group-sm">
                  <span class="input-group-text"><i class="bi bi-search"></i></span>
                  <input type="text" name="q" value="{{ q }}" class="form-control" placeholder="Buscar por operadora…" aria-label="Buscar por operadora">
                  <button type="submit" class="btn btn-outline-hub">Buscar</button>
                </div>
              </form>
              <a class="btn btn-hub btn-sm" href="{{ url_for('engenharia_form_list') }}?show_files={{ 0 if show_files else 1 }}{% if q %}&q={{ q }}{% endif %}{% if s %}&status={{ s }}{% endif %}{% if sort %}&sort={{ sort }}{% endif %}">
                <i class="bi {{ 'bi-view-list' if show_files else 'bi-folder2-open' }}"></i>
                {{ 'Ocultar exports' if show_files else 'Ver exports' }}
              </a>
            </div>
          </section>

          {% if forms and forms|length %}
            <div class="table-responsive">
              <table class="table align-middle table-hover">
                <thead class="table-light">
                  <tr><th class="text-nowrap">ID</th><th>Operadora</th><th>Status</th><th class="text-nowrap">Criado em</th><th class="text-end">Ações</th></tr>
                </thead>
                <tbody id="resultsBody">
                  {% for f in forms %}
                  {% set status_lbl = (f.status or 'rascunho')|lower %}
                  <tr>
                    <td class="fw-semibold">#{{ f.id }} <i class="bi bi-clipboard-check id-copy ms-1" data-id="{{ f.id }}" data-bs-toggle="tooltip" data-bs-title="Copiar ID"></i></td>
                    <td class="op-name" title="{{ f.nome_operadora or '—' }}">{{ f.nome_operadora or '—' }}</td>
                    <td><span class="badge badge-status {% if status_lbl=='aprovado' %}aprovado{% elif status_lbl=='em revisão' %}em-revisao{% elif status_lbl=='enviado' %}enviado{% else %}rascunho{% endif %}">{{ status_lbl|capitalize }}</span></td>
                    <td class="text-body-secondary">{{ f.created_at | date_br }}</td>
                    <td class="text-end">
                      <div class="btn-group btn-group-sm" role="group">
                        <a class="btn btn-outline-hub" href="{{ url_for('engenharia_form_view', form_id=f.id) }}" data-bs-toggle="tooltip" data-bs-title="Abrir para leitura/validação (Seção 9)"><i class="bi bi-eye"></i> Abrir</a>
                        <a class="btn btn-hub" href="{{ url_for('exportar_form_excel_index', form_id=f.id) }}" data-bs-toggle="tooltip" data-bs-title="Exportar Excel"><i class="bi bi-file-earmark-spreadsheet"></i> Gerar PTI</a>
                      </div>
                    </td>
                  </tr>
                  {% endfor %}
                </tbody>
              </table>
            </div>
          {% else %}
            <section class="text-center py-5">
              <div class="mb-3"><i class="bi bi-inbox fs-1 text-secondary"></i></div>
              <h2 class="h5">Nenhum formulário disponível</h2>
              {% if q %}<p class="text-body-secondary mb-0">Sua busca por <strong>{{ q }}</strong> não retornou resultados.</p>
              {% else %}<p class="text-body-secondary mb-0">Assim que o Atacado enviar um Pré-PTI, ele aparecerá aqui para validação.</p>{% endif %}
            </section>
          {% endif %}

        </div>
      </div>
    </div>

    {% if show_files %}
    <div class="col-12 col-xxl-4">
      <div class="card-shell h-100">
        <div class="p-4">
          <div class="d-flex align-items-center justify-content-between mb-2">
            <h3 class="h6 fw-bold mb-0"><i class="bi bi-folder2-open me-1"></i> Biblioteca de Exports</h3>
            <a class="btn btn-outline-hub btn-sm" href="{{ url_for('engenharia_form_list') }}?show_files=1">Atualizar</a>
          </div>
          <p class="text-body-secondary small mb-3">Downloads persistidos no servidor.</p>
          <form method="get" class="mb-3 d-flex align-items-center gap-2">
            <input type="hidden" name="show_files" value="1">
            <div class="input-group input-group-sm">
              <span class="input-group-text"><i class="bi bi-hash"></i></span>
              <input type="number" name="form" value="{{ form_filter or '' }}" class="form-control" placeholder="Filtrar por ID">
              <button type="submit" class="btn btn-outline-hub">Aplicar</button>
              <a class="btn btn-outline-hub" href="{{ url_for('engenharia_form_list') }}?show_files=1">Limpar</a>
            </div>
          </form>
          {% if exports and exports|length %}
            <div>
              {% for e in exports %}
              <div class="exports-item">
                <div>
                  <div class="exports-name">{{ e.filename }}</div>
                  <div class="exports-meta">Form #{{ e.form_id }} • {{ e.nome_operadora or '—' }} • {{ e.created_at|date_br }} • {{ (e.size_bytes or 0) // 1024 }} KB</div>
                </div>
                <div class="d-flex gap-1">
                  <a class="btn btn-sm btn-hub" href="{{ url_for('engenharia_export_download', export_id=e.id) }}" title="Baixar"><i class="bi bi-download"></i></a>
                  <form method="post" action="{{ url_for('engenharia_export_delete', export_id=e.id) }}" onsubmit="return confirm('Remover este arquivo do servidor?');">
                    <button class="btn btn-sm btn-outline-hub" type="submit" title="Excluir"><i class="bi bi-trash"></i></button>
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
  document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(el => new bootstrap.Tooltip(el));

  const live = document.createElement('div');
  live.className = 'visually-hidden';
  live.setAttribute('aria-live', 'polite');
  document.body.appendChild(live);
  const speak = (m) => { live.textContent = m || ''; };

  document.querySelectorAll('.id-copy').forEach(el => {
    el.addEventListener('click', async () => {
      const id = el.getAttribute('data-id');
      try {
        await navigator.clipboard.writeText(id);
        el.setAttribute('data-bs-title', 'Copiado!');
        bootstrap.Tooltip.getInstance(el)?.show();
        speak(`ID ${id} copiado.`);
        setTimeout(() => { el.setAttribute('data-bs-title', 'Copiar ID'); bootstrap.Tooltip.getInstance(el)?.hide(); }, 900);
      } catch (_) { speak('Não foi possível copiar o ID.'); }
    });
  });

  (function () {
    const q = new URLSearchParams(location.search).get('q');
    if (!q) return;
    const regex = new RegExp('(' + q.replace(/[.*+?^${}()|[\]\\]/g, '\\$&') + ')', 'gi');
    document.querySelectorAll('#resultsBody .op-name').forEach(td => {
      td.innerHTML = td.textContent.replace(regex, '<mark>$1</mark>');
    });
  })();
</script>
{% endblock %}
