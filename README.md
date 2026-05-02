{% extends "base.html" %}
{% block title %}Formulários — PTI AUTOMATIZADO{% endblock %}
{% block extra_head %}
<style>
  .op-cell { max-width:280px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .id-btn  { background:none; border:none; cursor:pointer; color:var(--sub); padding:0 .2rem; font-size:.8rem; }
  .id-btn:hover { color:var(--p); }
</style>
{% endblock %}
{% block content %}
<div class="page">

  <!-- Header -->
  <div style="display:flex;align-items:center;gap:.75rem;margin-bottom:1.5rem;flex-wrap:wrap">
    <a href="{{ url_for('central.central_engenharia') }}" class="btn-g btn-sm">← Voltar</a>
    <div>
      <h1 class="v-title">Formulários</h1>
      <p class="v-sub">PTIs submetidos pelo Atacado para validação</p>
    </div>
  </div>

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
          <option value="nome_operadora" {{ 'selected' if sort=='nome_operadora' }}>Operadora A-Z</option>
          <option value="-nome_operadora"{{ 'selected' if sort=='-nome_operadora' }}>Operadora Z-A</option>
          <option value="-id"            {{ 'selected' if sort=='-id' }}>ID mais recente</option>
          <option value="id"             {{ 'selected' if sort=='id' }}>ID mais antigo</option>
        </select>
        <button type="submit" class="btn-g btn-sm"><i class="bi bi-funnel"></i></button>
      </form>
      <form method="get" style="display:flex;gap:.4rem;margin-left:auto">
        <input type="hidden" name="status" value="{{ s }}">
        <input type="hidden" name="sort"   value="{{ sort }}">
        <input class="v-input v-input-sm" type="text" name="q" value="{{ q }}"
               placeholder="Buscar operadora..." style="width:200px">
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
            <th>Status</th>
            <th>Criado</th>
            <th style="text-align:right">Acoes</th>
          </tr>
        </thead>
        <tbody id="resultsBody">
          {% for f in forms %}
            {% set st = (f.status or 'rascunho')|lower %}
            <tr>
              <td style="font-weight:600;color:var(--sub);font-size:.8rem">
                #{{ f.id }}
                <button class="id-btn" data-id="{{ f.id }}" title="Copiar ID"><i class="bi bi-clipboard"></i></button>
              </td>
              <td class="op-cell" title="{{ f.nome_operadora or '' }}">{{ f.nome_operadora or '' }}</td>
              <td>
                <span class="badge-s {% if st=='aprovado' %}done{% elif st=='em revisao' %}review{% elif st=='enviado' %}sent{% else %}draft{% endif %}">
                  {{ st|capitalize }}
                </span>
              </td>
              <td style="color:var(--sub);font-size:.8rem">{{ f.created_at|date_br }}</td>
              <td style="text-align:right">
                <div style="display:flex;gap:.35rem;justify-content:flex-end">
                  <a href="{{ url_for('engenharia.form_view', form_id=f.id) }}" class="btn-g btn-sm">
                    <i class="bi bi-eye"></i> Abrir
                  </a>
                  <a href="{{ url_for('engenharia.exportar_excel', form_id=f.id) }}" class="btn-o btn-sm">
                    <i class="bi bi-file-earmark-spreadsheet"></i> Excel
                  </a>
                  {% if st != 'aprovado' %}
                  <form method="post" action="{{ url_for('engenharia.validar', form_id=f.id) }}"
                        onsubmit="return confirm('Aprovar PTI #{{ f.id }}?')">
                    <button type="submit" class="btn-p btn-sm">
                      <i class="bi bi-check-circle"></i> Validar
                    </button>
                  </form>
                  {% else %}
                  <span class="badge-s done" style="padding:.3rem .6rem;font-size:.72rem">
                    <i class="bi bi-check-circle-fill"></i> Aprovado
                  </span>
                  {% endif %}
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
      <p>Nenhum formulario disponivel.</p>
    {% endif %}
  </div>
  {% endif %}

</div>
{% endblock %}
{% block extra_scripts %}
<script>
document.querySelectorAll('.id-btn').forEach(btn=>{
  btn.addEventListener('click',async()=>{
    try{
      await navigator.clipboard.writeText(btn.dataset.id);
      btn.innerHTML='<i class="bi bi-clipboard-check"></i>';
      setTimeout(()=>{ btn.innerHTML='<i class="bi bi-clipboard"></i>'; },1200);
    }catch(_){}
  });
});
(function(){
  const q = new URLSearchParams(location.search).get('q');
  if(!q) return;
  const re = new RegExp('('+q.replace(/[.*+?^${}()|[\]\\]/g,'\\$&')+')','gi');
  document.querySelectorAll('#resultsBody .op-cell').forEach(td=>{
    td.innerHTML = td.textContent.replace(re,'<mark>$1</mark>');
  });
})();
</script>
{% endblock %}
