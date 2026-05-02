{% extends "base.html" %}
{% block title %}Central Atacado — PTI AUTOMATIZADO{% endblock %}
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
      <p style="font-size:.72rem;font-weight:600;color:var(--p);letter-spacing:.08em;text-transform:uppercase;margin:0 0 .35rem">Atacado</p>
      <h1 class="v-title">Área de trabalho</h1>
    </div>
    <div style="display:flex;gap:.5rem;flex-wrap:wrap">
      <a href="{{ url_for('atacado.form_list') }}" class="btn-o">Meus formulários</a>
      <a href="{{ url_for('atacado.form_new') }}"  class="btn-p"><i class="bi bi-plus"></i> Novo PTI</a>
    </div>
  </div>

  <!-- Pesquisa rápida -->
  <div class="card" style="padding:.75rem 1rem;margin-bottom:1.5rem">
    <form method="get" action="{{ url_for('atacado.form_list') }}"
          style="display:flex;gap:.5rem;align-items:center">
      <input class="v-input v-input-sm" type="text" name="q"
             placeholder="Buscar PTI por operadora…" style="flex:1;max-width:360px">
      <button type="submit" class="btn-p btn-sm">
        <i class="bi bi-search"></i> Buscar
      </button>
      <a href="{{ url_for('atacado.form_new') }}" class="btn-g btn-sm">
        <i class="bi bi-plus"></i> Novo PTI
      </a>
    </form>
  </div>

  <!-- Ações -->
  <div style="display:grid;grid-template-columns:repeat(auto-fill,minmax(240px,1fr));gap:1rem">

    <a class="action-card" href="{{ url_for('atacado.form_new') }}">
      <div class="action-card-icon"><i class="bi bi-file-earmark-plus"></i></div>
      <h3>Criar Pré-PTI</h3>
      <p>Criar formulário de interligação</p>
    </a>

    <a class="action-card" href="{{ url_for('atacado.form_list') }}">
      <div class="action-card-icon"><i class="bi bi-list-task"></i></div>
      <h3>Meus PTIs</h3>
      <p>Consultar, editar e acompanhar</p>
    </a>

    <a class="action-card" href="{{ url_for('atacado.form_list') }}?status=aprovado">
      <div class="action-card-icon"><i class="bi bi-check2-circle"></i></div>
      <h3>Aprovados</h3>
      <p>PTIs validados pela Engenharia</p>
    </a>

  </div>

</div>
{% endblock %}
{% block extra_scripts %}{% endblock %}
