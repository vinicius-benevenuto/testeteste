{% extends "base.html" %}
{% block title %}Central Engenharia{% endblock %}

{% block extra_head %}
<style>
  /* Hero com identidade VIVOHUB */
  .eng-hero{
    position:relative; isolation:isolate; overflow:hidden;
    background:
      radial-gradient(1200px 600px at -10% -20%, rgba(107,9,166,.14), transparent 60%),
      radial-gradient(900px 500px at 120% 10%, rgba(107,9,166,.10), transparent 60%),
      linear-gradient(180deg, #ffffff 0%, var(--vh-bg) 100%);
  }
  .eng-badge{
    display:inline-flex; align-items:center; gap:.4rem;
    padding:.35rem .6rem; border-radius:999px;
    border:1px solid var(--vh-border); background:var(--vh-surface-0);
    color:var(--vh-muted); font-weight:600; font-size:.85rem;
  }

  /* Cards */
  .eng-card{
    border:1px solid var(--vh-border);
    background:var(--vh-surface-0);
    border-radius: var(--vh-radius);
    box-shadow: var(--vh-shadow-md);
    transition: transform .18s ease, box-shadow .18s ease, border-color .18s ease;
    height:100%;
  }
  .eng-card:hover{
    transform: translateY(-2px);
    box-shadow: var(--vh-shadow-lg);
  }
  .eng-card .icon-wrap{
    width:44px; height:44px; border-radius:12px;
    display:grid; place-items:center;
    background: rgba(107,9,166,.08);
    color: var(--vh-primary);
  }
  .eng-card .list-inline{ margin:0; padding:0; list-style:none; }
  .eng-card .list-inline li{
    display:inline-flex; align-items:center; gap:.35rem;
    color:var(--vh-muted); font-size:.9rem;
  }

  /* Chips (atalhos) */
  .eng-chip{
    display:inline-flex; align-items:center; gap:.45rem;
    border:1px solid var(--vh-border); background:var(--vh-surface-0);
    border-radius:999px; padding:.42rem .75rem; font-weight:600;
  }
  .eng-chip:hover{ background:var(--vh-surface-1); text-decoration:none; }

  /* Tabela (se usada futuramente) */
  .eng-table th{ white-space:nowrap; font-weight:600; }
  .eng-table tbody tr:hover{ background:#fbfbff; }

  /* Preferências de movimento */
  @media (prefers-reduced-motion: reduce){
    .eng-card{ transition:none; }
  }
</style>
{% endblock %}

{% block content %}
<section class="eng-hero border-bottom">
  <div class="container py-5 py-xl-6">
    <div class="d-flex align-items-start justify-content-between flex-wrap gap-3">
      <div>
        <span class="eng-badge mb-2" aria-label="Área da Engenharia">
          <i class="bi bi-cpu"></i> Engenharia
        </span>
        <h1 class="display-6 fw-bold mb-2">Central Engenharia</h1>
        <p class="lead text-body-secondary mb-0">
          Valide <strong>Pré-PTIs</strong>, gere o <strong>Excel (Índice/Versões/Diagrama)</strong> e acompanhe os arquivos gerados — tudo padronizado.
        </p>
      </div>
      <div class="d-flex gap-2 align-items-center">
        <a class="btn btn-outline-hub" href="{{ url_for('logout') }}">
          <i class="bi bi-box-arrow-right me-1"></i> Sair
        </a>
      </div>
    </div>

    <!-- CTAs principais -->
    <div class="mt-4 d-flex flex-wrap gap-2">
      <a href="{{ url_for('engenharia_form_list') }}" class="btn btn-hub btn-lg">
        <i class="bi bi-journal-check me-2"></i>Formulários para validar
      </a>
      <a href="{{ url_for('engenharia_form_list') }}?show_files=1" class="btn btn-outline-hub btn-lg">
        <i class="bi bi-folder2-open me-2"></i>Biblioteca de exports
      </a>
    </div>

    <!-- Chips de filtro rápido -->
    <div class="mt-3">
      <a class="eng-chip me-1" href="{{ url_for('engenharia_form_list') }}">
        <i class="bi bi-collection"></i> Todos
      </a>
      <a class="eng-chip me-1" href="{{ url_for('engenharia_form_list') }}?status=enviado">
        <i class="bi bi-send"></i> Enviados
      </a>
      <a class="eng-chip me-1" href="{{ url_for('engenharia_form_list') }}?status=em%20revis%C3%A3o">
        <i class="bi bi-hourglass-split"></i> Em revisão
      </a>
      <a class="eng-chip me-1" href="{{ url_for('engenharia_form_list') }}?status=aprovado">
        <i class="bi bi-check2-circle"></i> Aprovados
      </a>
      <a class="eng-chip" href="{{ url_for('engenharia_form_list') }}?status=rascunho">
        <i class="bi bi-pencil-square"></i> Rascunhos (em elaboração)
      </a>
    </div>
  </div>
</section>

<!-- Painéis de ação -->
<section class="py-5">
  <div class="container">
    <div class="row g-4">
      <!-- Lista/Validação -->
      <div class="col-12 col-md-6 col-xl-4">
        <a class="text-decoration-none" href="{{ url_for('engenharia_form_list') }}" aria-label="Abrir lista de Pré-PTIs para validação">
          <div class="eng-card p-4 h-100">
            <div class="d-flex align-items-start gap-3">
              <div class="icon-wrap"><i class="bi bi-clipboard-check"></i></div>
              <div class="flex-grow-1">
                <h2 class="h5 fw-bold mb-1">Validar Pré-PTIs</h2>
                <p class="mb-3 text-body-secondary">Abra o formulário, preencha a <strong>Seção 9</strong> e salve a validação.</p>
                <ul class="list-inline d-flex flex-wrap gap-3">
                  <li><i class="bi bi-check2-square"></i> Checklist técnico</li>
                  <li><i class="bi bi-lock"></i> Demais seções em leitura</li>
                </ul>
              </div>
            </div>
          </div>
        </a>
      </div>

      <!-- Exports -->
      <div class="col-12 col-md-6 col-xl-4">
        <a class="text-decoration-none" href="{{ url_for('engenharia_form_list') }}?show_files=1" aria-label="Consultar a biblioteca de arquivos Excel gerados">
          <div class="eng-card p-4 h-100">
            <div class="d-flex align-items-start gap-3">
              <div class="icon-wrap"><i class="bi bi-file-earmark-spreadsheet"></i></div>
              <div class="flex-grow-1">
                <h2 class="h5 fw-bold mb-1">Biblioteca de Exports</h2>
                <p class="mb-3 text-body-secondary">Baixe, filtre por formulário e limpe o histórico, quando necessário.</p>
                <ul class="list-inline d-flex flex-wrap gap-3">
                  <li><i class="bi bi-folder2"></i> Registro persistente</li>
                  <li><i class="bi bi-cloud-arrow-down"></i> Download rápido</li>
                </ul>
              </div>
            </div>
          </div>
        </a>
      </div>

      <!-- Diretrizes técnicas -->
      <div class="col-12 col-md-12 col-xl-4">
        <div class="eng-card p-4 h-100">
          <div class="d-flex align-items-start gap-3">
            <div class="icon-wrap"><i class="bi bi-sliders2"></i></div>
            <div class="flex-grow-1">
              <h2 class="h5 fw-bold mb-1">Diretrizes técnicas</h2>
              <p class="mb-3 text-body-secondary">Lembretes essenciais aplicados na Seção 9 (já no checklist do sistema):</p>
              <ul class="mb-0 text-body-secondary">
                <li class="mb-1"><strong>Codec</strong>: STFC/STFC e STFC/SMP — G.711A e G.729; SMP/SMP — AMR e G.711A. G.711u não suportado.</li>
                <li class="mb-1"><strong>QoS RTP</strong>: CS5 / DSCP 46 / EF. <strong>Ptime/Maxptime</strong> múltiplos de 20 ms.</li>
                <li class="mb-1"><strong>Identificação</strong>: ISUPBR em rede fixa; móvel ISUPBR (ITU-92 opcional).</li>
                <li class="mb-0"><strong>SIP/RFCs</strong>: 3261, 3262, 3264/3266/2327/4566, 4028, 3389 (CN), 2833 (DTMF), 3960 (Early Media), 6337.</li>
              </ul>
            </div>
          </div>
        </div>
      </div>

    </div><!-- /row -->
  </div>
</section>

<!-- Ajuda & atalhos -->
<section class="pb-5">
  <div class="container">
    <div class="eng-card p-4">
      <div class="d-flex flex-wrap align-items-center justify-content-between gap-3">
        <div class="d-flex align-items-center gap-3">
          <div class="icon-wrap"><i class="bi bi-keyboard"></i></div>
          <div>
            <h3 class="h6 fw-bold mb-1">Atalhos úteis</h3>
            <div class="text-body-secondary small">
              <span class="me-3"><kbd>L</kbd> Abrir lista</span>
              <span class="me-3"><kbd>F</kbd> Abrir biblioteca de exports</span>
              <span><kbd>Ctrl</kbd> + <kbd>S</kbd> Salvar validação (na Seção 9)</span>
            </div>
          </div>
        </div>
        <div class="d-flex gap-2">
          <a href="{{ url_for('engenharia_form_list') }}" class="btn btn-outline-hub">
            <i class="bi bi-search me-1"></i> Abrir lista
          </a>
          <a href="{{ url_for('engenharia_form_list') }}?show_files=1" class="btn btn-hub">
            <i class="bi bi-folder-symlink me-1"></i> Ir para exports
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
    // Prefetch para navegação fluida
    try{
      const prefetch = (href)=>{
        const l = document.createElement('link');
        l.rel = 'prefetch'; l.href = href; document.head.appendChild(l);
      };
      prefetch("{{ url_for('engenharia_form_list') }}");
      prefetch("{{ url_for('engenharia_form_list') }}?show_files=1");
    }catch(_){}

    // Atalhos globais (apenas quando não estiver digitando)
    document.addEventListener('keydown', (e)=>{
      const tag = (e.target && e.target.tagName) || '';
      const typing = ['INPUT','TEXTAREA','SELECT'].includes(tag);
      if (typing) return;

      if (e.key === 'l' || e.key === 'L'){
        e.preventDefault();
        window.location.href = "{{ url_for('engenharia_form_list') }}";
      }
      if (e.key === 'f' || e.key === 'F'){
        e.preventDefault();
        window.location.href = "{{ url_for('engenharia_form_list') }}?show_files=1";
      }
    });
  })();
</script>
{% endblock %}
