{% extends "base.html" %}
{% block title %}Formulários — VIVOHUB{% endblock %}
{% block extra_head %}
<style>
  .op-group { margin-bottom:1.5rem; }
  .op-header {
    display:flex; justify-content:space-between; align-items:center;
    padding:.65rem 1rem; background:var(--p-lt);
    border:1px solid var(--bdr); border-radius:var(--r) var(--r) 0 0;
    cursor:pointer;
  }
  .op-header:hover { background:#ede8f5; }
  .op-name-label { font-weight:700; font-size:.9rem; color:var(--p); }
  .op-count      { font-size:.75rem; color:var(--sub); }
  .op-body       { border:1px solid var(--bdr); border-top:none; border-radius:0 0 var(--r) var(--r); overflow:hidden; }
  .op-body.collapsed { display:none; }
  .v-badge {
    font-size:.68rem; font-weight:700; padding:.12rem .45rem;
    border-radius:999px; background:var(--p); color:#fff; margin-left:.4rem;
  }
  .op-cell { max-width:260px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .filter-bar { display:flex; gap:.5rem; flex-wrap:wrap; align-items:center; }
</style>
{% endblock %}
{% block content %}
<div class="page">

  <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:1.5rem;gap:1rem;flex-wrap:wrap">
    <div>
      <h1 class="v-title">Formulários</h1>
      <p class="v-sub">Organizados por operadora · <kbd class="kbd">N</kbd> novo &nbsp;<kbd class="kbd">/</kbd> buscar</p>
    </div>
    <a href="{{ url_for('atacado.form_new') }}" class="btn-p"><i class="bi bi-plus"></i> Novo</a>
  </div>

  <!-- Contadores -->
  {% if counters %}
    {% set q    = request.args.get('q','') %}
    {% set sort = request.args.get('sort','-created_at') %}
    {% set s    = request.args.get('status','') %}
    <div class="chips" style="margin-bottom:1.25rem">
      <a class="chip {% if not s %}on{% endif %}" href="?q={{ q }}&sort={{ sort }}">Todos · {{ counters.total or 0 }}</a>
      <a class="chip {% if s=='rascunho'   %}on{% endif %}" href="?status=rascunho&q={{ q }}&sort={{ sort }}">Rascunhos · {{ counters.rascunho or 0 }}</a>
      <a class="chip {% if s=='enviado'    %}on{% endif %}" href="?status=enviado&q={{ q }}&sort={{ sort }}">Enviados · {{ counters.enviado or 0 }}</a>
      <a class="chip {% if s=='em revisão' %}on{% endif %}" href="?status=em revisão&q={{ q }}&sort={{ sort }}">Em revisão · {{ counters.em_revisao or 0 }}</a>
      <a class="chip {% if s=='aprovado'   %}on{% endif %}" href="?status=aprovado&q={{ q }}&sort={{ sort }}">Aprovados · {{ counters.aprovado or 0 }}</a>
    </div>
  {% endif %}

  <!-- Filtros -->
  {% set q    = request.args.get('q','') %}
  {% set s    = request.args.get('status','') %}
  {% set sort = request.args.get('sort','-created_at') %}
  <div class="card" style="padding:.75rem;margin-bottom:1.25rem">
    <div class="filter-bar">
      <form method="get" style="display:contents">
        <input type="hidden" name="q" value="{{ q }}">
        <select class="v-input v-input-sm" name="status" style="width:auto;min-width:140px">
          <option value=""           {{ 'selected' if s=='' }}>Todos os status</option>
          <option value="rascunho"   {{ 'selected' if s=='rascunho' }}>Rascunho</option>
          <option value="enviado"    {{ 'selected' if s=='enviado' }}>Enviado</option>
          <option value="em revisão" {{ 'selected' if s=='em revisão' }}>Em revisão</option>
          <option value="aprovado"   {{ 'selected' if s=='aprovado' }}>Aprovado</option>
        </select>
        <select class="v-input v-input-sm" name="sort" style="width:auto;min-width:150px">
          <option value="-created_at"    {{ 'selected' if sort=='-created_at' }}>Mais recentes</option>
          <option value="created_at"     {{ 'selected' if sort=='created_at' }}>Mais antigos</option>
          <option value="nome_operadora" {{ 'selected' if sort=='nome_operadora' }}>Operadora A→Z</option>
          <option value="-id"            {{ 'selected' if sort=='-id' }}>ID ↓</option>
        </select>
        <button type="submit" class="btn-g btn-sm"><i class="bi bi-funnel"></i></button>
      </form>
      <form method="get" style="display:flex;gap:.4rem;margin-left:auto">
        <input type="hidden" name="status" value="{{ s }}">
        <input type="hidden" name="sort"   value="{{ sort }}">
        <input class="v-input v-input-sm" type="text" id="searchInput" name="q"
               value="{{ q }}" placeholder="Buscar operadora…" style="width:200px">
        <button type="submit" class="btn-g btn-sm"><i class="bi bi-search"></i></button>
        {% if q or s %}<a href="{{ url_for('atacado.form_list') }}" class="btn-g btn-sm">Limpar</a>{% endif %}
      </form>
    </div>
  </div>

  <!-- Grupos por operadora -->
  {% if grupos %}
    {% for op_name, forms in grupos.items()|sort %}
    <div class="op-group">
      <div class="op-header" onclick="toggleGroup(this)">
        <div>
          <span class="op-name-label">{{ op_name }}</span>
          <span class="op-count">{{ forms|length }} formulário(s)</span>
        </div>
        <i class="bi bi-chevron-down" style="color:var(--sub);font-size:.8rem"></i>
      </div>
      <div class="op-body">
        <table class="v-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Versão</th>
              <th>RN1</th>
              <th>Status</th>
              <th>Criado</th>
              <th style="text-align:right">Ações</th>
            </tr>
          </thead>
          <tbody>
            {% for f in forms %}
              {% set st  = (f.status or 'rascunho')|lower %}
              {% set ver = f.version if f.version else 1 %}
              <tr>
                <td style="font-weight:600;color:var(--sub);font-size:.8rem">#{{ f.id }}</td>
                <td>
                  <span class="v-badge">v{{ ver }}</span>
                </td>
                <td style="font-size:.82rem">{{ f.rn1 or '—' }}</td>
                <td>
                  <span class="badge-s {% if st=='aprovado' %}done{% elif st=='em revisão' %}review{% elif st=='enviado' %}sent{% else %}draft{% endif %}">
                    {{ st|capitalize }}
                  </span>
                </td>
                <td style="color:var(--sub);font-size:.8rem">{{ (f.created_at or '')|date_br }}</td>
                <td style="text-align:right">
                  <div style="display:flex;gap:.35rem;justify-content:flex-end">
                    <a href="{{ url_for('atacado.form_edit', form_id=f.id) }}" class="btn-g btn-sm">
                      <i class="bi bi-pencil"></i> Editar
                    </a>
                    <form method="post" action="{{ url_for('atacado.form_new_version', form_id=f.id) }}"
                          style="display:inline"
                          onsubmit="return confirm('Criar nova versão a partir do formulário #{{ f.id }}?')">
                      <button type="submit" class="btn-o btn-sm" title="Nova versão">
                        <i class="bi bi-plus-circle"></i> v{{ ver + 1 }}
                      </button>
                    </form>
                    <button class="btn-sm del-form-btn"
                            data-id="{{ f.id }}"
                            data-name="{{ f.nome_operadora | e or '—' }}"
                            style="border-radius:7px;border:1px solid #fecaca;background:#fef2f2;color:#991b1b;cursor:pointer;font-size:.75rem;font-weight:600;padding:.3rem .6rem">
                      <i class="bi bi-trash"></i>
                    </button>
                  </div>
                </td>
              </tr>
            {% endfor %}
          </tbody>
        </table>
      </div>
    </div>
    {% endfor %}
  {% else %}
    <div class="card empty">
      <i class="bi bi-inbox"></i>
      {% if q %}
        <p>Sem resultados para <strong>{{ q }}</strong></p>
      {% else %}
        <p>Nenhum formulário criado ainda.</p>
      {% endif %}
      <a href="{{ url_for('atacado.form_new') }}" class="btn-p btn-sm" style="margin-top:.75rem">
        <i class="bi bi-plus"></i> Criar primeiro PTI
      </a>
    </div>
  {% endif %}

</div>

<form method="post" id="deleteForm" action="#" style="display:none"></form>
{% endblock %}

{% block extra_scripts %}
<script>
function toggleGroup(header){
  const body = header.nextElementSibling;
  const icon = header.querySelector('.bi');
  body.classList.toggle('collapsed');
  icon.className = body.classList.contains('collapsed')
    ? 'bi bi-chevron-right' : 'bi bi-chevron-down';
  icon.style.color = 'var(--sub)';
  icon.style.fontSize = '.8rem';
}

// Manter grupos abertos na carga inicial
document.querySelectorAll('.op-body').forEach(b => b.classList.remove('collapsed'));

document.querySelectorAll('.del-form-btn').forEach(btn => {
  btn.addEventListener('click', () => {
    const id   = btn.dataset.id;
    const name = btn.dataset.name || '—';
    if(!confirm(`Excluir formulário #${id} (${name})?\n\nEsta ação não pode ser desfeita.`)) return;
    const f = document.getElementById('deleteForm');
    f.action = "{{ url_for('atacado.form_delete', form_id=0) }}".replace('/0/', '/' + id + '/');
    f.submit();
  });
});

document.addEventListener('keydown', e => {
  if(['INPUT','TEXTAREA','SELECT'].includes(e.target?.tagName)) return;
  if(e.key === 'n' || e.key === 'N'){ e.preventDefault(); location.href = "{{ url_for('atacado.form_new') }}"; }
  if(e.key === '/'){ e.preventDefault(); document.getElementById('searchInput')?.focus(); }
});
</script>
{% endblock %}
