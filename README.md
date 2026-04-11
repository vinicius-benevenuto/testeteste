{% extends "base.html" %}
{% block title %}Central Atacado{% endblock %}

{% block extra_head %}
<style>
  /* Hero com identidade VIVOHUB */
  .atk-hero{
    position:relative; isolation:isolate; overflow:hidden;
    background:
      radial-gradient(1200px 600px at -10% -20%, rgba(107,9,166,.14), transparent 60%),
      radial-gradient(900px 500px at 120% 10%, rgba(107,9,166,.10), transparent 60%),
      linear-gradient(180deg, #ffffff 0%, var(--vh-bg) 100%);
  }
  .atk-badge{
    display:inline-flex; align-items:center; gap:.4rem;
    padding:.35rem .6rem; border-radius:999px;
    border:1px solid var(--vh-border); background:var(--vh-surface-0);
    color:var(--vh-muted); font-weight:600; font-size:.85rem;
  }

  /* Cards de ação */
  .atk-card{
    border:1px solid var(--vh-border);
    background:var(--vh-surface-0);
    border-radius: var(--vh-radius);
    box-shadow: var(--vh-shadow-md);
    transition: transform .18s ease, box-shadow .18s ease;
    height:100%;
  }
  .atk-card:hover{
    transform: translateY(-2px);
    box-shadow: var(--vh-shadow-lg);
  }
  .atk-card .icon-wrap{
    width:44px; height:44px; border-radius:12px;
    display:grid; place-items:center;
    background: rgba(107,9,166,.08);
    color: var(--vh-primary);
  }
  .atk-card .list-inline{
    margin:0; padding:0; list-style:none;
  }
  .atk-card .list-inline li{
    display:inline-flex; align-items:center; gap:.35rem;
    color:var(--vh-muted); font-size:.9rem;
  }

  /* Quick links (chips) */
  .atk-chip{
    display:inline-flex; align-items:center; gap:.4rem;
    border:1px solid var(--vh-border); background:var(--vh-surface-0);
    border-radius:999px; padding:.4rem .7rem; font-weight:600;
  }
  .atk-chip:hover{ background:var(--vh-surface-1); text-decoration:none; }

  /* Tabela “últimas ações” (opcional futura) */
  .atk-table th{ white-space:nowrap; font-weight:600; }
  .atk-table tbody tr:hover{ background:#fbfbff; }

  /* Reduz animações para quem prefere */
  @media (prefers-reduced-motion: reduce){
    .atk-card{ transition:none; }
  }
</style>
{% endblock %}

{% block content %}
<section class="atk-hero border-bottom">
  <div class="container py-5 py-xl-6">
    <div class="d-flex align-items-start justify-content-between flex-wrap gap-3">
      <div>
        <h1 class="display-6 fw-bold mb-2">Central Atacado</h1>
        <p class="lead text-body-secondary mb-0">
          Crie e acompanhe seus <strong>Pré-PTIs</strong>. Tudo padronizado, rápido e com foco na melhor experiência.
        </p>
      </div>
      <div class="d-flex gap-2 align-items-center">
        <a class="btn btn-outline-hub" href="{{ url_for('logout') }}">
          <i class="bi bi-box-arrow-right me-1"></i> Sair
        </a>
      </div>
    </div>

    <!-- CTA principal -->
    <div class="mt-4 d-flex flex-wrap gap-2">
      <a href="{{ url_for('atacado_form_new') }}" class="btn btn-hub btn-lg">
        <i class="bi bi-file-earmark-plus me-2"></i>Novo Pré-PTI
      </a>
      <a href="{{ url_for('atacado_form_list') }}" class="btn btn-outline-hub btn-lg">
        <i class="bi bi-journal-text me-2"></i>Meus Formulários
      </a>
    </div>

    <!-- Atalhos de filtro (chips) -->
    <div class="mt-3">
      <a class="atk-chip me-1" href="{{ url_for('atacado_form_list') }}">
        <i class="bi bi-collection"></i> Todos
      </a>
      <a class="atk-chip me-1" href="{{ url_for('atacado_form_list') }}?status=rascunho">
        <i class="bi bi-pencil-square"></i> Rascunhos
      </a>
      <a class="atk-chip me-1" href="{{ url_for('atacado_form_list') }}?status=enviado">
        <i class="bi bi-send"></i> Enviados
      </a>
      <a class="atk-chip me-1" href="{{ url_for('atacado_form_list') }}?status=em%20revis%C3%A3o">
        <i class="bi bi-hourglass-split"></i> Em revisão
      </a>
      <a class="atk-chip" href="{{ url_for('atacado_form_list') }}?status=aprovado">
        <i class="bi bi-check2-circle"></i> Aprovados
      </a>
    </div>
  </div>
</section>

<!-- Seções de ação rápida -->
<section class="py-5">
  <div class="container">
    <div class="row g-4">
      <!-- Criar Pré-PTI -->
      <div class="col-12 col-md-6 col-xl-4">
        <a class="text-decoration-none" href="{{ url_for('atacado_form_new') }}" aria-label="Criar um novo Pré-PTI">
          <div class="atk-card p-4 h-100">
            <div class="d-flex align-items-start gap-3">
              <div class="icon-wrap"><i class="bi bi-plus-lg"></i></div>
              <div class="flex-grow-1">
                <h2 class="h5 fw-bold mb-1">Criar Pré-PTI</h2>
                <p class="mb-3 text-body-secondary">Formulário claro, com validações e preenchimentos assistidos.</p>
                <ul class="list-inline d-flex flex-wrap gap-3">
                  <li><i class="bi bi-lightning-charge"></i> Atalhos <kbd>N</kbd> / <kbd>/</kbd></li>
                  <li><i class="bi bi-shield-check"></i> Campos obrigatórios</li>
                </ul>
              </div>
            </div>
          </div>
        </a>
      </div>

      <!-- Meus Formulários -->
      <div class="col-12 col-md-6 col-xl-4">
        <a class="text-decoration-none" href="{{ url_for('atacado_form_list') }}" aria-label="Acessar a lista de formulários">
          <div class="atk-card p-4 h-100">
            <div class="d-flex align-items-start gap-3">
              <div class="icon-wrap"><i class="bi bi-list-task"></i></div>
              <div class="flex-grow-1">
                <h2 class="h5 fw-bold mb-1">Meus Formulários</h2>
                <p class="mb-3 text-body-secondary">Filtre por status, ordene por data ou operadora, busque rapidamente.</p>
                <ul class="list-inline d-flex flex-wrap gap-3">
                  <li><i class="bi bi-funnel"></i> Filtros</li>
                  <li><i class="bi bi-sort-alpha-down"></i> Ordenação</li>
                  <li><i class="bi bi-search"></i> Busca</li>
                </ul>
              </div>
            </div>
          </div>
        </a>
      </div>

      <!-- Boas práticas -->
      <div class="col-12 col-md-12 col-xl-4">
        <div class="atk-card p-4 h-100">
          <div class="d-flex align-items-start gap-3">
            <div class="icon-wrap"><i class="bi bi-stars"></i></div>
            <div class="flex-grow-1">
              <h2 class="h5 fw-bold mb-1">Boas práticas</h2>
              <p class="mb-3 text-body-secondary">Algumas dicas para agilizar sua rotina e evitar retrabalho.</p>
              <ul class="mb-0 text-body-secondary">
                <li class="mb-1">Use o <strong>RN1</strong> correto e preencha <strong>Contatos</strong> completos.</li>
                <li class="mb-1">Na seção <strong>Escopo</strong>, marque os tipos de tráfego que se aplicam.</li>
                <li class="mb-1">Finalize quando estiver seguro: o status muda para <em>Enviado</em> e segue para Engenharia.</li>
                <li class="mb-0">Exportações Excel ficam registradas (lado Engenharia) e o download local é facilitado.</li>
              </ul>
            </div>
          </div>
        </div>
      </div>

    </div><!-- row -->
  </div>
</section>

<!-- Ajuda e atalhos -->
<section class="pb-5">
  <div class="container">
    <div class="atk-card p-4">
      <div class="d-flex flex-wrap align-items-center justify-content-between gap-3">
        <div class="d-flex align-items-center gap-3">
          <div class="icon-wrap"><i class="bi bi-keyboard"></i></div>
          <div>
            <h3 class="h6 fw-bold mb-1">Atalhos úteis</h3>
            <div class="text-body-secondary small">
              <span class="me-3"><kbd>N</kbd> Novo Pré-PTI</span>
              <span class="me-3"><kbd>/</kbd> Ir para busca (na lista)</span>
              <span><kbd>Ctrl</kbd> + <kbd>S</kbd> Salvar no formulário</span>
            </div>
          </div>
        </div>
        <div class="d-flex gap-2">
          <a href="{{ url_for('atacado_form_list') }}" class="btn btn-outline-hub">
            <i class="bi bi-search me-1"></i> Abrir lista
          </a>
          <a href="{{ url_for('atacado_form_new') }}" class="btn btn-hub">
            <i class="bi bi-rocket-takeoff me-1"></i> Começar agora
          </a>
        </div>
      </div>
    </div>
  </div>
</section>
{% endblock %}

{% block extra_scripts %}
<script>
  (function(){
    // Prefetch de páginas frequentes para navegação fluida
    try{
      const prefetch = (href)=>{
        const l = document.createElement('link');
        l.rel = 'prefetch'; l.href = href; document.head.appendChild(l);
      };
      prefetch("{{ url_for('atacado_form_new') }}");
      prefetch("{{ url_for('atacado_form_list') }}");
    }catch(_){}

    // Atalhos globais
    document.addEventListener('keydown', (e)=>{
      const tag = (e.target && e.target.tagName) || '';
      const typing = ['INPUT','TEXTAREA','SELECT'].includes(tag);
      if (!typing && (e.key === 'n' || e.key === 'N')){
        e.preventDefault();
        window.location.href = "{{ url_for('atacado_form_new') }}";
      }
      if (!typing && e.key === '/'){
        e.preventDefault();
        window.location.href = "{{ url_for('atacado_form_list') }}?q="; // foca busca ao chegar
      }
    });
  })();
</script>
{% endblock %}
