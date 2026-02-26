/* =========================================================
   TEMA VIVO — Paleta global
   ========================================================= */
   :root {
    /* Paleta base (Vivo) */
    --vivo-primary-700: #4b0073;
    --vivo-primary-600: #5a008a;
    --vivo-primary-500: #660099;   /* principal */
    --vivo-primary-400: #7B2CBF;   /* hover/acento */
    --vivo-primary-300: #9D4EDD;   /* tom claro */
    --vivo-primary-200: #c9a6f3;
  
    --vivo-bg-light: #F8F5FC;
    --vivo-bg-dark: #111217;
    --vivo-text-dark: #1A1A1A;
    --vivo-text-light: #e9e9ee;
  
    /* Injeção nas variáveis Bootstrap */
    --bs-primary: var(--vivo-primary-500);
    --bs-primary-rgb: 102, 0, 153;
    --bs-secondary: var(--vivo-primary-500);
    --bs-success: var(--vivo-primary-500);
    --bs-info: var(--vivo-primary-500);
    --bs-warning: var(--vivo-primary-500);
    --bs-danger: var(--vivo-primary-500);
    --bs-link-color: var(--vivo-primary-500);
    --bs-link-hover-color: var(--vivo-primary-400);
  }
  
  /* =========================================================
     Navbar
     ========================================================= */
  .navbar {
    background-color: var(--vivo-primary-500) !important;
    border-bottom: 3px solid rgba(157, 78, 221, .25);
  }
  .navbar .navbar-brand,
  .navbar .nav-link,
  .navbar .btn { color: #fff !important; }
  .navbar .nav-link:hover,
  .navbar .btn:hover { color: #f2e9ff !important; }
  
  /* =========================================================
     Botões — padroniza TUDO para o roxo Vivo
     ========================================================= */
  .btn,
  .btn-primary,
  .btn-success,
  .btn-info,
  .btn-warning,
  .btn-danger,
  .btn-secondary {
    background-color: var(--vivo-primary-500) !important;
    border-color: var(--vivo-primary-500) !important;
    color: #fff !important;
  }
  .btn:hover,
  .btn:focus {
    background-color: var(--vivo-primary-400) !important;
    border-color: var(--vivo-primary-400) !important;
    color: #fff !important;
  }
  .btn-outline-secondary,
  .btn-outline-primary {
    color: var(--vivo-primary-500) !important;
    border-color: var(--vivo-primary-500) !important;
    background: transparent !important;
  }
  .btn-outline-secondary:hover,
  .btn-outline-primary:hover {
    color: #fff !important;
    background-color: var(--vivo-primary-500) !important;
    border-color: var(--vivo-primary-500) !important;
  }
  
  /* =========================================================
     Cards & Títulos
     ========================================================= */
  .card { border-color: rgba(102, 0, 153, .20); }
  .card-title,
  h1,h2,h3,h4,h5,h6 { color: var(--vivo-primary-500); }
  
  /* =========================================================
     Links
     ========================================================= */
  a { color: var(--vivo-primary-500); }
  a:hover { color: var(--vivo-primary-400); }
  
  /* =========================================================
     Tabela — cabeçalho duplo "sticky" + rolagem
     ========================================================= */
  .table-responsive {
    max-height: calc(100vh - 280px);
    overflow: auto;
    border-top: 1px solid rgba(0,0,0,.06);
  }
  .table thead tr:first-child th {
    position: sticky;
    top: 0;
    z-index: 3;
    background: var(--bs-body-bg, #fff);
    box-shadow: inset 0 -1px 0 rgba(0,0,0,.08);
    text-transform: uppercase;
    font-weight: 600;
    color: var(--vivo-primary-600);
  }
  .table thead tr:nth-child(2) th {
    position: sticky;
    top: 42px;
    z-index: 2;
    background: var(--bs-body-bg, #fff);
    box-shadow: inset 0 -1px 0 rgba(0,0,0,.06);
  }
  .table td, .table th {
    white-space: nowrap;
    text-overflow: ellipsis;
    overflow: hidden;
    vertical-align: middle;
  }
  .table-hover > tbody > tr:hover {
    --bs-table-accent-bg: rgba(102, 0, 153, .06);
  }
  
  /* Filtros (inputs) */
  .filtro-coluna,
  .filtro-data-inicial,
  .filtro-data-final {
    min-width: 10rem;
  }
  
  /* Paginação */
  #paginacao .pagination .page-link {
    border-color: rgba(102, 0, 153, .25);
    color: var(--vivo-primary-500);
    background: transparent;
  }
  #paginacao .pagination .page-item.active .page-link {
    background-color: var(--vivo-primary-500);
    border-color: var(--vivo-primary-500);
    color: #fff;
  }
  
  /* =========================================================
     Dashboard — gráfico estável
     ========================================================= */
  .chart-wrap { position: relative; width: 100%; height: 240px; }
  #grafico-rede { display: block; width: 100% !important; height: 100% !important; }
  
  #contagem-stfc, #contagem-smp { font-variant-numeric: tabular-nums; }
  
  /* =========================================================
     Loading overlay
     ========================================================= */
  #carregando { background: rgba(0,0,0,.35); z-index: 1080; }
  
  /* =========================================================
     Tema escuro
     ========================================================= */
  html[data-bs-theme="dark"] {
    --bs-body-bg: var(--vivo-bg-dark);
    --bs-body-color: var(--vivo-text-light);
  }
  html[data-bs-theme="dark"] .navbar { border-bottom-color: rgba(255,255,255,.12); }
  html[data-bs-theme="dark"] .card {
    background-color: #171923;
    border-color: rgba(255,255,255,.08);
  }
  html[data-bs-theme="dark"] .table thead tr:first-child th,
  html[data-bs-theme="dark"] .table thead tr:nth-child(2) th {
    background: #171923;
    color: #e9d7ff;
    box-shadow: inset 0 -1px 0 rgba(255,255,255,.06);
  }
  html[data-bs-theme="dark"] #paginacao .pagination .page-link {
    color: #e9d7ff; border-color: rgba(255,255,255,.15);
  }
  html[data-bs-theme="dark"] #paginacao .pagination .page-item.active .page-link {
    background-color: var(--vivo-primary-400);
    border-color: var(--vivo-primary-400);
  }
  
  /* =========================================================
     Mensagem inicial
     ========================================================= */
  #mensagem-inicial.alert {
    border-color: rgba(102, 0, 153, .25);
    background: var(--vivo-bg-light);
    color: #3b0764;
  }
  
  /* Detalhes */
  #card-tabela .card-footer { border-top: 2px solid rgba(102,0,153,.12); }
  .sticky-top { top: 0; }
  
  /* =========================================================
     Agente IA — container e bolhas
     ========================================================= */
  #ia-mensagens {
    background: var(--bs-body-bg);
    border: 1px solid rgba(0,0,0,.06);
    border-radius: .5rem;
    padding: .5rem .75rem;
    min-height: 260px;
    max-height: 60vh;
    overflow: auto;
    scroll-behavior: auto;
  }
  html[data-bs-theme="dark"] #ia-mensagens {
    border-color: rgba(255,255,255,.10);
  }
  
  .ia-msg {
    background: transparent;
    margin-bottom: .5rem;
    line-height: 1.35;
  }
  
  .ia-msg.ia-user {
    text-align: right;
    background: rgba(102, 0, 153, .08);
    color: var(--vivo-primary-600);
    border: 1px solid rgba(102,0,153,.20);
    border-radius: .66rem;
    padding: .4rem .6rem;
    display: inline-block;
    max-width: 100%;
  }
  
  .ia-msg.ia-bot {
    background: rgba(0,0,0,.04);
    border: 1px solid rgba(0,0,0,.08);
    border-radius: .66rem;
    padding: .55rem .7rem;
    display: block;
  }
  html[data-bs-theme="dark"] .ia-msg.ia-bot {
    background: rgba(255,255,255,.04);
    border-color: rgba(255,255,255,.12);
  }
  
  /* Botões de sugestão do agente */
  .ia-sugestao.btn {
    padding: .2rem .55rem;
    border-width: 1px;
    font-size: .85rem;
  }
  
  /* =========================================================
     Responsividade
     ========================================================= */
  @media (max-width: 991.98px) {
    .table-responsive {
      max-height: calc(100vh - 360px);
    }
    #ia-mensagens {
      max-height: 50vh;
    }
  }
