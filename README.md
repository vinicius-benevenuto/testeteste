<!doctype html>
<html lang="pt-br" data-color-scheme="auto">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
  <meta http-equiv="x-ua-compatible" content="ie=edge">

  <title>{% block title %}VIVOHUB{% endblock %}</title>
  <meta name="description" content="VIVOHUB — Gestão de Pré-PTI (Atacado/Engenharia)">
  <meta name="theme-color" content="#6b09a6">

  <link rel="icon" href="{{ url_for('static', filename='favicon.ico') }}">

  <!-- Bootstrap 5 + Icons (CDN) -->
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" crossorigin="anonymous">
  <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css" rel="stylesheet" crossorigin="anonymous">

  <!-- Design System — VIVO Purple -->
  <style>
    :root{
      /* Brand */
      --vh-primary:       #6b09a6;
      --vh-primary-600:   #5f0895;
      --vh-primary-700:   #520781;
      --vh-primary-050:   #f6effb;

      /* Semântica */
      --vh-success:  #17a673;
      --vh-warning:  #ffcc00;
      --vh-danger:   #d93c65;

      /* Superfícies e texto */
      --vh-bg:        #f6f7fb;
      --vh-surface-0: #ffffff;
      --vh-surface-1: #fafafa;
      --vh-text:      #1f2430;
      --vh-text-sub:  #656b78;
      --vh-border:    #e5e7eb;

      /* Aliases (usados nos sub-templates) */
      --vh-muted:  var(--vh-text-sub);
      --vh-ink:    var(--vh-text);

      /* Elevação e bordas */
      --vh-shadow-sm: 0 4px 12px rgba(17,17,26,.07);
      --vh-shadow-md: 0 10px 34px rgba(17,17,26,.12);
      --vh-shadow-lg: 0 20px 60px rgba(17,17,26,.18);
      --vh-radius:    16px;
    }

    /* Estrutura */
    html, body { height: 100%; }
    body {
      font-feature-settings: "cv11","ss01";
      background: linear-gradient(180deg, var(--vh-bg), #ffffff);
      color: var(--vh-text);
    }
    body.bg-plain { background: #fff !important; }

    /* Navbar */
    .vh-navbar {
      background: var(--vh-surface-0);
      border-bottom: 1px solid var(--vh-border);
    }
    .vh-brand {
      display: inline-flex; align-items: center; gap: .5rem;
      font-weight: 800; letter-spacing: .2px;
      color: var(--vh-primary) !important; text-decoration: none;
    }
    .vh-brand-dot {
      width: 10px; height: 10px; border-radius: 999px;
      background: var(--vh-primary);
      box-shadow: 0 0 0 3px var(--vh-primary-050);
    }

    /* Botões padrão */
    .btn-hub {
      --bs-btn-color: #fff;
      --bs-btn-bg: var(--vh-primary);
      --bs-btn-border-color: var(--vh-primary);
      --bs-btn-hover-color: #000;
      --bs-btn-hover-bg: #ffffff;
      --bs-btn-hover-border-color: var(--vh-primary);
      --bs-btn-focus-shadow-rgb: 107,9,166;
      font-weight: 700; border: 0; border-radius: .85rem; padding: .6rem 1.05rem;
      box-shadow: var(--vh-shadow-sm);
      transition: box-shadow .18s ease, transform .18s ease;
    }
    .btn-hub:hover { transform: translateY(-1px); box-shadow: var(--vh-shadow-md); }

    .btn-outline-hub {
      border-radius: .85rem; font-weight: 700;
      border: 1px solid var(--vh-primary); color: var(--vh-primary);
      background: transparent;
      transition: background .18s ease, color .18s ease, transform .18s ease;
    }
    .btn-outline-hub:hover {
      background: var(--vh-primary-050); color: #000; transform: translateY(-1px);
    }

    /* Chips */
    .chip {
      display: inline-flex; align-items: center; gap: .4rem;
      border-radius: 999px; padding: .35rem .7rem;
      font-weight: 600; line-height: 1;
      background: var(--vh-surface-0); border: 1px solid var(--vh-border);
    }
    .chip.active { background: #eef3ff; border-color: #bfd0ff; color: #0d6efd; }

    /* Badges de status */
    .badge-status { font-weight: 600; border: 1px solid transparent; }
    .badge-status.rascunho { background: #eef1f4; color: #2b2f32; border-color: #e1e6ea; }
    .badge-status.enviado  { background: #cfe2ff; color: #084298; border-color: #b6d0ff; }
    .badge-status.em-revisao { background: #fff3cd; color: #664d03; border-color: #ffe29a; }
    .badge-status.aprovado { background: #d1e7dd; color: #0f5132; border-color: #bcd9cc; }

    /* Tabelas e cartões */
    .table-hover tbody tr:hover { background: #f8fbff; }
    .table thead th { font-weight: 600; white-space: nowrap; }
    .vh-card {
      border: 1px solid var(--vh-border);
      border-radius: var(--vh-radius);
      box-shadow: var(--vh-shadow-sm);
      background: var(--vh-surface-0);
    }

    /* Utilitários */
    .vh-divider { height: 1px; background: var(--vh-border); margin: 1rem 0; }
    .vh-flash {
      position: fixed; top: 12px; left: 50%; transform: translateX(-50%);
      z-index: 1080; width: min(680px, 96vw);
    }

    @media (prefers-reduced-motion: reduce) {
      .btn, .btn-hub, .btn-outline-hub { transition: none !important; }
    }
  </style>

  {% block extra_head %}{% endblock %}
</head>

<body class="{{ 'bg-plain' if plain_bg else '' }}">

  <!-- Navbar -->
  <nav class="navbar navbar-expand-lg vh-navbar">
    <div class="container-fluid">

      <button class="navbar-toggler" type="button"
              data-bs-toggle="collapse" data-bs-target="#vhNav"
              aria-controls="vhNav" aria-expanded="false"
              aria-label="Alternar navegação">
        <span class="navbar-toggler-icon"></span>
      </button>

      <div class="collapse navbar-collapse" id="vhNav">
        <ul class="navbar-nav ms-auto align-items-lg-center">
          {% set _role = session.get('role') %}

          {% if _role == 'atacado' %}
            <li class="nav-item me-1">
              <a class="btn btn-outline-hub btn-sm" href="{{ url_for('central.central_atacado') }}">Central Atacado</a>
            </li>
            <li class="nav-item me-1">
              <a class="btn btn-outline-hub btn-sm" href="{{ url_for('atacado.form_list') }}">Formulários</a>
            </li>
          {% elif _role == 'engenharia' %}
            <li class="nav-item me-1">
              <a class="btn btn-outline-hub btn-sm" href="{{ url_for('central.central_engenharia') }}">Central Engenharia</a>
            </li>
            <li class="nav-item me-1">
              <a class="btn btn-outline-hub btn-sm" href="{{ url_for('engenharia.form_list') }}">Formulários</a>
            </li>
          {% endif %}

          {% if session.get('user_id') %}
            <li class="nav-item ms-lg-2">
              <a class="btn btn-hub btn-sm" href="{{ url_for('auth.logout') }}">
                <i class="bi bi-box-arrow-right"></i> Sair
              </a>
            </li>
          {% endif %}
        </ul>
      </div>
    </div>
  </nav>

  <!-- Flash messages -->
  <div class="vh-flash">
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, msg in messages %}
          <div class="alert alert-{{ category }} alert-dismissible fade show shadow-sm mb-2" role="alert">
            {{ msg }}
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Fechar"></button>
          </div>
        {% endfor %}
      {% endif %}
    {% endwith %}
  </div>

  <!-- Conteúdo -->
  <main id="main" class="min-vh-100">
    {% block content %}
    <div class="container py-5">
      <div class="text-center text-body-secondary">
        <p class="mb-0">Conteúdo não definido.</p>
      </div>
    </div>
    {% endblock %}
  </main>

  <!-- Footer -->
  <footer class="border-top mt-auto py-3">
    <div class="container d-flex justify-content-between small text-body-secondary">
      <span>VIVOHUB</span>
      <span>Gestão de PTI — Interno</span>
    </div>
  </footer>

  <!-- Bootstrap JS -->
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" crossorigin="anonymous"></script>

  <script>
    document.addEventListener('DOMContentLoaded', function(){
      document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(function(el){
        new bootstrap.Tooltip(el);
      });
    });
    // Link "pular para conteúdo"
    (function(){
      const skip = document.createElement('a');
      skip.href = '#main';
      skip.className = 'visually-hidden-focusable position-absolute top-0 start-0 m-2';
      skip.textContent = 'Pular para o conteúdo';
      document.body.prepend(skip);
    })();
  </script>

  {% block extra_scripts %}{% endblock %}
</body>
</html>
