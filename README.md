<!doctype html>
<html lang="pt-br">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
  <title>{% block title %}PTI AUTOMATIZADO{% endblock %}</title>
  <meta name="theme-color" content="#6b09a6">
  <link rel="icon" href="{{ url_for('static', filename='favicon.ico') }}">
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" crossorigin="anonymous">
  <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css" rel="stylesheet" crossorigin="anonymous">
  <style>
    :root {
      --p:    #6b09a6;
      --p-lt: #f4eefa;
      --p-dk: #520781;
      --ink:  #111318;
      --sub:  #6b7280;
      --bdr:  #e5e7eb;
      --bg:   #f8f9fb;
      --surf: #ffffff;
      --r:    12px;
      --sh:   0 1px 3px rgba(0,0,0,.07), 0 4px 16px rgba(0,0,0,.05);
      --sh-lg:0 8px 32px rgba(0,0,0,.1);
      --err:  #dc2626;
      --ok:   #16a34a;
    }
    *,*::before,*::after { box-sizing: border-box; }
    html,body { height: 100%; }
    body {
      font-family: -apple-system,BlinkMacSystemFont,"Segoe UI",system-ui,sans-serif;
      background: var(--bg); color: var(--ink);
      -webkit-font-smoothing: antialiased; margin: 0;
    }
    body.plain { background: #fff !important; }

    /* ── Navbar ── */
    .v-nav {
      height: 52px; background: var(--surf);
      border-bottom: 1px solid var(--bdr);
      display: flex; align-items: center;
      padding: 0 1.5rem; gap: .75rem;
      position: sticky; top: 0; z-index: 100;
    }
    .v-brand {
      font-size: .85rem; font-weight: 700; letter-spacing: -.1px;
      color: var(--p); text-decoration: none;
      display: flex; align-items: center; gap: .4rem;
    }
    .v-brand-dot { width: 7px; height: 7px; border-radius: 50%; background: var(--p); }
    .v-nav-end { display: flex; align-items: center; gap: .4rem; margin-left: auto; }

    /* ── Botões ── */
    .btn-p, .btn-o, .btn-g {
      display: inline-flex; align-items: center; gap: .35rem;
      font-size: .8rem; font-weight: 600; border-radius: 8px;
      padding: .45rem .85rem; cursor: pointer; border: none;
      text-decoration: none; line-height: 1.2;
      transition: opacity .15s, background .15s, transform .12s;
      white-space: nowrap;
    }
    .btn-p { background: var(--p); color: #fff; }
    .btn-p:hover { opacity: .87; color: #fff; transform: translateY(-1px); }
    .btn-o { background: transparent; color: var(--p); border: 1px solid var(--bdr); }
    .btn-o:hover { background: var(--p-lt); border-color: var(--p); color: var(--p); }
    .btn-g { background: #f3f4f6; color: var(--ink); border: 1px solid var(--bdr); }
    .btn-g:hover { background: #e8eaed; color: var(--ink); }
    .btn-danger { background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }
    .btn-danger:hover { background: #fee2e2; color: #991b1b; }
    .btn-sm { padding: .3rem .6rem; font-size: .75rem; border-radius: 7px; }
    .btn-lg { padding: .65rem 1.2rem; font-size: .9rem; border-radius: 10px; }

    /* ── Card ── */
    .card {
      background: var(--surf); border: 1px solid var(--bdr);
      border-radius: var(--r); box-shadow: var(--sh);
    }

    /* ── Status badges ── */
    .badge-s {
      font-size: .7rem; font-weight: 600; padding: .18rem .5rem;
      border-radius: 999px; border: 1px solid transparent;
      white-space: nowrap;
    }
    .badge-s.draft  { background:#f3f4f6; color:#374151; border-color:#e5e7eb; }
    .badge-s.sent   { background:#dbeafe; color:#1d4ed8; border-color:#bfdbfe; }
    .badge-s.review { background:#fef3c7; color:#92400e; border-color:#fde68a; }
    .badge-s.done   { background:#d1fae5; color:#065f46; border-color:#a7f3d0; }
    .badge-s.fail   { background:#fee2e2; color:#991b1b; border-color:#fecaca; }

    /* ── Tables ── */
    .v-table { width: 100%; border-collapse: collapse; }
    .v-table th {
      text-align: left; font-size: .7rem; font-weight: 600;
      color: var(--sub); text-transform: uppercase; letter-spacing: .06em;
      padding: .6rem 1rem; border-bottom: 1px solid var(--bdr);
      white-space: nowrap;
    }
    .v-table td {
      padding: .7rem 1rem; border-bottom: 1px solid var(--bdr);
      font-size: .875rem; vertical-align: middle;
    }
    .v-table tbody tr:last-child td { border-bottom: none; }
    .v-table tbody tr:hover td { background: #fafbff; }

    /* ── Inputs ── */
    .v-input {
      width: 100%; padding: .5rem .75rem;
      border: 1px solid var(--bdr); border-radius: 8px;
      font-size: .875rem; background: var(--surf); color: var(--ink);
      transition: border-color .15s, box-shadow .15s; outline: none;
      font-family: inherit;
    }
    .v-input:focus { border-color: var(--p); box-shadow: 0 0 0 3px rgba(107,9,166,.1); }
    .v-input-sm { padding: .35rem .6rem; font-size: .8rem; border-radius: 7px; }
    .v-input[disabled], .v-input[readonly] { background: #f9fafb; color: var(--sub); }
    select.v-input {
      appearance: none;
      background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='10' viewBox='0 0 10 10'%3E%3Cpath fill='%236b7280' d='M5 7L0 2h10z'/%3E%3C/svg%3E");
      background-repeat: no-repeat; background-position: right .7rem center; padding-right: 2rem;
    }
    textarea.v-input { resize: vertical; min-height: 80px; }

    /* ── Flash ── */
    .flash-area {
      position: fixed; top: 60px; left: 50%; transform: translateX(-50%);
      z-index: 500; width: min(500px, 94vw);
      display: flex; flex-direction: column; gap: .4rem;
      pointer-events: none;
    }
    .flash-msg {
      display: flex; align-items: center; gap: .6rem;
      padding: .65rem .9rem; border-radius: 10px; pointer-events: all;
      font-size: .85rem; font-weight: 500; box-shadow: var(--sh-lg);
      animation: popIn .18s ease;
    }
    @keyframes popIn { from { opacity:0; transform:translateY(-6px); } to { opacity:1; transform:none; } }
    .flash-msg.success { background:#f0fdf4; color:#166534; border:1px solid #bbf7d0; }
    .flash-msg.danger  { background:#fef2f2; color:#991b1b; border:1px solid #fecaca; }
    .flash-msg.warning { background:#fffbeb; color:#92400e; border:1px solid #fde68a; }
    .flash-msg.info    { background:#eff6ff; color:#1e40af; border:1px solid #bfdbfe; }
    .flash-x { margin-left:auto; background:none; border:none; cursor:pointer; opacity:.45; padding:0; font-size:.8rem; line-height:1; }
    .flash-x:hover { opacity:1; }

    /* ── Layout ── */
    .page    { max-width: 1160px; margin: 0 auto; padding: 2rem 1.5rem; }
    .page-sm { max-width: 440px;  margin: 0 auto; padding: 3rem 1.5rem; }
    .page-md { max-width: 640px;  margin: 0 auto; padding: 2.5rem 1.5rem; }

    /* ── Type ── */
    .v-title { font-size: 1.3rem; font-weight: 700; letter-spacing: -.3px; margin: 0; }
    .v-sub   { font-size: .85rem; color: var(--sub); margin: .2rem 0 0; }
    .v-label { font-size: .8rem; font-weight: 600; display: block; margin-bottom: .3rem; }
    .v-hint  { font-size: .75rem; color: var(--sub); margin-top: .25rem; }
    .req::after { content:" *"; color: var(--err); }
    .divider { height: 1px; background: var(--bdr); }
    .kbd     { display:inline-block; padding:.1rem .38rem; border:1px solid var(--bdr); border-bottom-width:2px; border-radius:5px; font-size:.72rem; font-weight:600; background:var(--surf); }

    /* ── Empty state ── */
    .empty { text-align:center; padding:3rem 1rem; color:var(--sub); }
    .empty i { font-size:1.75rem; display:block; margin-bottom:.65rem; opacity:.35; }
    .empty p { font-size:.875rem; margin:0; }

    /* ── Chips filter ── */
    .chips { display:flex; flex-wrap:wrap; gap:.4rem; }
    .chip {
      display:inline-flex; align-items:center; gap:.3rem;
      padding:.28rem .65rem; border-radius:999px; font-size:.78rem; font-weight:600;
      border:1px solid var(--bdr); background:var(--surf); color:var(--sub);
      text-decoration:none; transition:all .15s; cursor:pointer;
    }
    .chip:hover { background:var(--p-lt); border-color:var(--p); color:var(--p); text-decoration:none; }
    .chip.on    { background:var(--p-lt); border-color:var(--p); color:var(--p); }

    @media (max-width:640px) {
      .page { padding:1.25rem 1rem; }
      .v-nav { padding:0 1rem; }
    }
    @media (prefers-reduced-motion:reduce) { *,*::before,*::after { transition:none !important; animation:none !important; } }
  </style>
  {% block extra_head %}{% endblock %}
</head>
<body class="{{ 'plain' if plain_bg else '' }}">

  <nav class="v-nav">
    {% if session.get('user_id') %}
    <a class="btn-g btn-sm" id="navBack" href="#" onclick="history.back();return false;"
       style="display:none">← Voltar</a>
    {% endif %}
    <a class="v-brand" href="#">
      <span class="v-brand-dot"></span>PTI AUTOMATIZADO
    </a>
    <div class="v-nav-end">
      {% if session.get('user_id') %}
        <a class="btn-o btn-sm" href="{{ url_for('auth.logout') }}">Sair</a>
      {% endif %}
    </div>
  </nav>
  <script>
  // Mostrar botão Voltar quando há histórico de navegação
  (function(){
    const b = document.getElementById('navBack');
    if(b && window.history.length > 1) b.style.display = 'inline-flex';
  })();
  </script>

  <div class="flash-area">
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% for category, msg in messages %}
        <div class="flash-msg {{ category }}">
          <span>{{ msg }}</span>
          <button class="flash-x" onclick="this.parentElement.remove()">✕</button>
        </div>
      {% endfor %}
    {% endwith %}
  </div>

  <main id="main">{% block content %}{% endblock %}</main>

  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" crossorigin="anonymous"></script>
  {% block extra_scripts %}{% endblock %}
</body>
</html>
