{% extends "base.html" %}
{% block title %}Central Engenharia — VIVOHUB{% endblock %}
{% block extra_head %}
<style>
  .action-card {
    display:block; text-decoration:none; color:inherit;
    padding:1.5rem; border-radius:var(--r); border:1px solid var(--bdr);
    background:var(--surf); transition:box-shadow .2s, transform .15s;
  }
  .action-card:hover { box-shadow:var(--sh-lg); transform:translateY(-2px); text-decoration:none; color:inherit; }
  .action-card-icon {
    width:40px; height:40px; border-radius:10px;
    background:var(--p-lt); color:var(--p);
    display:flex; align-items:center; justify-content:center;
    font-size:1.1rem; margin-bottom:1rem;
  }
  .action-card h3 { font-size:.95rem; font-weight:700; margin:0 0 .35rem; }
  .action-card p  { font-size:.8rem; color:var(--sub); margin:0; }
</style>
{% endblock %}
{% block content %}
<div class="page">

  <!-- Header -->
  <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:2rem;gap:1rem;flex-wrap:wrap">
    <div>
      <p style="font-size:.72rem;font-weight:600;color:var(--p);letter-spacing:.08em;text-transform:uppercase;margin:0 0 .35rem">Engenharia</p>
      <h1 class="v-title">Área de trabalho</h1>
    </div>
    <div style="display:flex;gap:.5rem;flex-wrap:wrap">
      <a href="{{ url_for('engenharia.form_list') }}?show_files=1" class="btn-o"><i class="bi bi-folder2-open"></i> Exports</a>
      <a href="{{ url_for('engenharia.form_list') }}"              class="btn-p"><i class="bi bi-clipboard-check"></i> Validar</a>
    </div>
  </div>

  <!-- Filtros rápidos -->
  <div class="chips" style="margin-bottom:2rem">
    <a class="chip" href="{{ url_for('engenharia.form_list') }}">Todos</a>
    <a class="chip" href="{{ url_for('engenharia.form_list') }}?status=enviado">Enviados</a>
    <a class="chip" href="{{ url_for('engenharia.form_list') }}?status=em%20revis%C3%A3o">Em revisão</a>
    <a class="chip" href="{{ url_for('engenharia.form_list') }}?status=aprovado">Aprovados</a>
    <a class="chip" href="{{ url_for('engenharia.form_list') }}?status=rascunho">Rascunhos</a>
  </div>

  <!-- Ações -->
  <div style="display:grid;grid-template-columns:repeat(auto-fill,minmax(240px,1fr));gap:1rem">

    <a class="action-card" href="{{ url_for('engenharia.form_list') }}">
      <div class="action-card-icon"><i class="bi bi-clipboard-check"></i></div>
      <h3>Validar Pré-PTIs</h3>
      <p>Revisar e preencher Seção 9</p>
    </a>

    <a class="action-card" href="{{ url_for('engenharia.form_list') }}?show_files=1">
      <div class="action-card-icon"><i class="bi bi-file-earmark-spreadsheet"></i></div>
      <h3>Exports Excel</h3>
      <p>Baixar e gerenciar PTIs gerados</p>
    </a>

    <a class="action-card" href="{{ url_for('engenharia.form_list') }}?status=enviado">
      <div class="action-card-icon"><i class="bi bi-send"></i></div>
      <h3>Aguardando revisão</h3>
      <p>PTIs enviados pelo Atacado</p>
    </a>

  </div>

  <!-- Atalhos -->
  <div style="margin-top:2rem;padding:1rem 1.25rem;background:var(--p-lt);border-radius:var(--r);display:flex;gap:1.5rem;flex-wrap:wrap;align-items:center">
    <span style="font-size:.78rem;color:var(--sub);font-weight:500">Atalhos:</span>
    <span style="font-size:.78rem;color:var(--sub)"><kbd class="kbd">L</kbd> Lista</span>
    <span style="font-size:.78rem;color:var(--sub)"><kbd class="kbd">F</kbd> Exports</span>
    <span style="font-size:.78rem;color:var(--sub)"><kbd class="kbd">Ctrl S</kbd> Salvar validação</span>
  </div>

</div>
{% endblock %}
{% block extra_scripts %}
<script>
document.addEventListener('keydown',e=>{
  if(['INPUT','TEXTAREA','SELECT'].includes(e.target?.tagName)) return;
  if(e.key==='l'||e.key==='L'){ e.preventDefault(); location.href="{{ url_for('engenharia.form_list') }}"; }
  if(e.key==='f'||e.key==='F'){ e.preventDefault(); location.href="{{ url_for('engenharia.form_list') }}?show_files=1"; }
});
</script>
{% endblock %}
