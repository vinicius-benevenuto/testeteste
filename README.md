{% extends "base.html" %}
{% block title %}Formulários Engenharia — VIVOHUB{% endblock %}
{% block extra_head %}
<style>
  .op-cell { max-width:280px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .id-btn  { background:none; border:none; cursor:pointer; color:var(--sub); padding:0 .2rem; font-size:.8rem; }
  .id-btn:hover { color:var(--p); }
  .v-badge { font-size:.68rem; font-weight:700; padding:.12rem .45rem; border-radius:999px; background:var(--p); color:#fff; }
</style>
{% endblock %}
{% block content %}
<div class="page">

  <!-- Header -->
  <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:1.5rem;gap:1rem;flex-wrap:wrap">
    <div>
      <h1 class="v-title">Formulários</h1>
      <p class="v-sub">PTIs submetidos pelo Atacado para validação</p>
    </div>
  </div>

  <!-- Contadores -->
  {% if counters %}
    {% set q    = request.args.get('q','') %}
    {% set sort = request.args.get('sort','-created_at') %}
    {% set s    = request.args.get('status','') %}
    <div class="chips" style="margin-bottom:1.25rem">
      <a class="chip {% if not s %}on{% endif %}"           href="?q={{ q }}&sort={{ sort }}">Todos · {{ counters.total or 0 }}</a>
      <a class="chip {% if s=='enviado'    %}on{% endif %}" href="?status=enviado&q={{ q }}&sort={{ sort }}">Enviados · {{ counters.enviado or 0 }}</a>
      <a class="chip {% if s=='em revisão' %}on{% endif %}" href="?status=em revisão&q={{ q }}&sort={{ sort }}">Em revisão · {{ counters.em_revisao or 0 }}</a>
      <a class="chip {% if s=='aprovado'   %}on{% endif %}" href="?status=aprovado&q={{ q }}&sort={{ sort }}">Aprovados · {{ counters.aprovado or 0 }}</a>
      <a class="chip {% if s=='rascunho'   %}on{% endif %}" href="?status=rascunho&q={{ q }}&sort={{ sort }}">Rascunhos · {{ counters.rascunho or 0 }}</a>
    </div>
  {% endif %}

  <!-- Filtros -->
  {% set q    = request.args.get('q','') %}
  {% set s    = request.args.get('status','') %}
  {% set sort = request.args.get('sort','-created_at') %}
  <div class="card" style="padding:.75rem;margin-bottom:1rem">
    <div style="display:flex;gap:.5rem;flex-wrap:wrap">
      <form method="get" style="display:contents">
        <input type="hidden" name="q" value="{{ q }}">
        <select class="v-input v-input-sm" name="status" style="width:auto;min-width:140px">
          <option value=""           {{ 'selected' if s=='' }}>Todos</option>
          <option value="enviado"    {{ 'selected' if s=='enviado' }}>Enviado</option>
          <option value="em revisão" {{ 'selected' if s=='em revisão' }}>Em revisão</option>
          <option value="aprovado"   {{ 'selected' if s=='aprovado' }}>Aprovado</option>
          <option value="rascunho"   {{ 'selected' if s=='rascunho' }}>Rascunho</option>
        </select>
        <select class="v-input v-input-sm" name="sort" style="width:auto;min-width:140px">
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
        <input class="v-input v-input-sm" type="text" name="q" value="{{ q }}"
               placeholder="Buscar operadora…" style="width:180px">
        <button type="submit" class="btn-g btn-sm"><i class="bi bi-search"></i></button>
        {% if q or s %}<a href="{{ url_for('engenharia.form_list') }}" class="btn-g btn-sm">Limpar</a>{% endif %}
      </form>
    </div>
  </div>

  <!-- Tabela -->
  {% if forms and forms|length %}
  <div class="card" style="overflow:hidden">
    <div style="overflow-x:auto">
      <table class="v-table">
        <thead>
          <tr>
            <th>ID</th>
            <th>Operadora</th>
            <th>Versão</th>
            <th>RN1</th>
            <th>Status</th>
            <th>Criado</th>
            <th style="text-align:right">Ações</th>
          </tr>
        </thead>
        <tbody id="resultsBody">
          {% for f in forms %}
            {% set st  = (f.status or 'rascunho')|lower %}
            {% set ver = f.version if f.version else 1 %}
            <tr>
              <td style="font-weight:600;color:var(--sub);font-size:.8rem">
                #{{ f.id }}
                <button class="id-btn" data-id="{{ f.id }}" title="Copiar ID">
                  <i class="bi bi-clipboard"></i>
                </button>
              </td>
              <td class="op-cell" title="{{ f.nome_operadora or '—' }}">{{ f.nome_operadora or '—' }}</td>
              <td><span class="v-badge">v{{ ver }}</span></td>
              <td style="font-size:.82rem">{{ f.rn1 or '—' }}</td>
              <td>
                <span class="badge-s {% if st=='aprovado' %}done{% elif st=='em revisão' %}review{% elif st=='enviado' %}sent{% else %}draft{% endif %}">
                  {{ st|capitalize }}
                </span>
              </td>
              <td style="color:var(--sub);font-size:.8rem">{{ f.created_at|date_br }}</td>
              <td style="text-align:right">
                <div style="display:flex;gap:.35rem;justify-content:flex-end">
                  <a href="{{ url_for('engenharia.form_view', form_id=f.id) }}" class="btn-g btn-sm">
                    <i class="bi bi-eye"></i> Abrir
                  </a>
                  <a href="{{ url_for('engenharia.exportar_excel', form_id=f.id) }}" class="btn-p btn-sm">
                    <i class="bi bi-file-earmark-spreadsheet"></i> Excel
                  </a>
                </div>
              </td>
            </tr>
          {% endfor %}
        </tbody>
      </table>
    </div>
  </div>
  {% else %}
    <div class="card empty">
      <i class="bi bi-inbox"></i>
      {% if q %}
        <p>Sem resultados para <strong>{{ q }}</strong></p>
      {% else %}
        <p>Nenhum formulário disponível.</p>
      {% endif %}
    </div>
  {% endif %}

</div>
{% endblock %}
{% block extra_scripts %}
<script>
document.querySelectorAll('.id-btn').forEach(btn => {
  btn.addEventListener('click', async () => {
    try{
      await navigator.clipboard.writeText(btn.dataset.id);
      btn.innerHTML = '<i class="bi bi-clipboard-check"></i>';
      setTimeout(() => { btn.innerHTML = '<i class="bi bi-clipboard"></i>'; }, 1200);
    }catch(_){}
  });
});
(function(){
  const q = new URLSearchParams(location.search).get('q');
  if(!q) return;
  const re = new RegExp('(' + q.replace(/[.*+?^${}()|[\]\\]/g,'\\$&') + ')','gi');
  document.querySelectorAll('#resultsBody .op-cell').forEach(td => {
    td.innerHTML = td.textContent.replace(re, '<mark>$1</mark>');
  });
})();
</script>
{% endblock %}
