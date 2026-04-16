{% extends "base.html" %}
{% block title %}Central Atacado — VIVOHUB{% endblock %}
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

  <!-- Filtros rápidos -->
  <div class="chips" style="margin-bottom:2rem">
    <a class="chip" href="{{ url_for('atacado.form_list') }}">Todos</a>
    <a class="chip" href="{{ url_for('atacado.form_list') }}?status=rascunho">Rascunhos</a>
    <a class="chip" href="{{ url_for('atacado.form_list') }}?status=enviado">Enviados</a>
    <a class="chip" href="{{ url_for('atacado.form_list') }}?status=em%20revis%C3%A3o">Em revisão</a>
    <a class="chip" href="{{ url_for('atacado.form_list') }}?status=aprovado">Aprovados</a>
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
      <h3>Formulários</h3>
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
