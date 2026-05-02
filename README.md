{% extends "base.html" %}
{% block title %}Formulários — PTI AUTOMATIZADO{% endblock %}
{% block extra_head %}
<style>
  .op-cell { max-width:300px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
  .id-btn  { background:none; border:none; cursor:pointer; color:var(--sub); padding:0 .2rem; font-size:.8rem; }
  .id-btn:hover { color:var(--p); }
  .op-group { margin-bottom:1.5rem; }
  .op-group-header {
    display:flex; align-items:center; justify-content:space-between;
    padding:.6rem 1rem; background:var(--p-lt); border:1px solid var(--bdr);
    border-radius:var(--r) var(--r) 0 0; cursor:pointer;
  }
  .op-group-header h3 { font-size:.88rem; font-weight:700; color:var(--p); margin:0; }
  .op-group-body { border:1px solid var(--bdr); border-top:none;
    border-radius:0 0 var(--r) var(--r); overflow:hidden; }
  .v-badge {
    font-size:.68rem; font-weight:700; padding:.15rem .4rem;
    border-radius:5px; background:var(--p); color:#fff; white-space:nowrap;
  }
</style>
{% endblock %}
{% block content %}
<div class="page">

  <!-- Header -->
  <div style="display:flex;align-items:center;gap:.75rem;margin-bottom:1.5rem;flex-wrap:wrap">
    <a href="{{ url_for('central.central_engenharia') }}" class="btn-g btn-sm">← Voltar</a>
    <div>
      <h1 class="v-title">Formulários</h1>
      <p class="v-sub">PTIs agrupados por operadora · todas as versões</p>
    </div>
  </div>

  <!-- Filtros -->
  {% set q      = q or '' %}
  {% set status = status or '' %}
  <div class="card" style="padding:.75rem;margin-bottom:1rem">
    <div style="display:flex;gap:.5rem;flex-wrap:wrap">
      <form method="get" style="display:contents">
        <input type="hidden" name="q" value="{{ q }}">
        <select class="v-input v-input-sm" name="status" style="width:auto;min-width:140px">
          <option value=""           {{ 'selected' if not status }}>Todos</option>
          <option value="enviado"    {{ 'selected' if status=='enviado' }}>Enviado</option>
          <option value="em revisão" {{ 'selected' if status=='em revisão' }}>Em revisão</option>
          <option value="aprovado"   {{ 'selected' if status=='aprovado' }}>Aprovado</option>
          <option value="rascunho"   {{ 'selected' if status=='rascunho' }}>Rascunho</option>
        </select>
        <button type="submit" class="btn-g btn-sm"><i class="bi bi-funnel"></i></button>
      </form>
      <form method="get" style="display:flex;gap:.4rem;margin-left:auto">
        <input type="hidden" name="status" value="{{ status }}">
        <input class="v-input v-input-sm" type="text" name="q" value="{{ q }}"
               placeholder="Buscar operadora..." style="width:200px">
        <button type="submit" class="btn-g btn-sm"><i class="bi bi-search"></i></button>
        {% if q or status %}<a href="{{ url_for('engenharia.form_list') }}" class="btn-g btn-sm">Limpar</a>{% endif %}
      </form>
    </div>
  </div>

  <!-- Grupos por operadora -->
  {% if grupos %}
    {% for nome_op, versoes in grupos.items() %}
    <div class="op-group">
      <div class="op-group-header" onclick="this.nextElementSibling.style.display=this.nextElementSibling.style.display==='none'?'':'none'">
        <h3><i class="bi bi-building" style="margin-right:.4rem"></i>{{ nome_op }}</h3>
        <span style="font-size:.75rem;color:var(--sub)">{{ versoes|length }} versão{{ 'ões' if versoes|length > 1 else '' }}</span>
      </div>
      <div class="op-group-body">
        <table class="v-table">
          <thead>
            <tr>
              <th>ID</th>
              <th>Versão</th>
              <th>Status</th>
              <th>RN1</th>
              <th>Atualizado</th>
              <th style="text-align:right">Ações</th>
            </tr>
          </thead>
          <tbody id="resultsBody">
            {% for f in versoes %}
              {% set st = (f.status or 'rascunho')|lower %}
              <tr>
                <td style="font-weight:600;color:var(--sub);font-size:.8rem">
                  #{{ f.id }}
                  <button class="id-btn" data-id="{{ f.id }}" title="Copiar ID"><i class="bi bi-clipboard"></i></button>
                </td>
                <td><span class="v-badge">v{{ f.version or 1 }}</span></td>
                <td>
                  <span class="badge-s {% if st=='aprovado' %}done{% elif st=='em revisão' %}review{% elif st=='enviado' %}sent{% else %}draft{% endif %}">
                    {{ st|capitalize }}
                  </span>
                </td>
                <td style="font-size:.8rem;color:var(--sub)">{{ f.rn1 or '—' }}</td>
                <td style="color:var(--sub);font-size:.8rem">{{ (f.updated_at or f.created_at or '')|date_br }}</td>
                <td style="text-align:right">
                  <div style="display:flex;gap:.35rem;justify-content:flex-end">
                    <a href="{{ url_for('engenharia.form_view', form_id=f.id) }}" class="btn-g btn-sm">
                      <i class="bi bi-eye"></i> Abrir
                    </a>
                    <a href="{{ url_for('engenharia.exportar_excel', form_id=f.id) }}" class="btn-o btn-sm">
                      <i class="bi bi-file-earmark-spreadsheet"></i> Excel v{{ f.version or 1 }}
                    </a>
                    {% if st != 'aprovado' %}
                    <form method="post" action="{{ url_for('engenharia.validar', form_id=f.id) }}"
                          onsubmit="return confirm('Aprovar PTI #{{ f.id }} v{{ f.version or 1 }} — {{ f.nome_operadora }}?')">
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
    {% endfor %}
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
