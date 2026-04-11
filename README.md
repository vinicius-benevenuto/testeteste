{% extends "base.html" %}
{% block title %}VIVOHUB — Criar usuário{% endblock %}

{% block extra_head %}
<style>
  .reg-hero{
    position:relative; isolation:isolate;
    background:
      radial-gradient(1200px 600px at -10% -20%, rgba(107,9,166,.14), transparent 60%),
      radial-gradient(900px 500px at 120% 10%, rgba(107,9,166,.10), transparent 60%),
      linear-gradient(180deg, #ffffff 0%, var(--vh-bg) 100%);
  }
  .reg-card{
    border:1px solid var(--vh-border);
    border-radius: var(--vh-radius);
    background: var(--vh-surface-0);
    box-shadow: var(--vh-shadow-md);
  }
  .reg-card .form-control:focus, .reg-card .form-select:focus{
    border-color: var(--vh-primary);
    box-shadow: 0 0 0 .2rem rgba(107,9,166,.15);
  }
  .btn-create{ padding:.75rem 1rem; font-weight:700; border-radius:.9rem; }
  .hint{ font-size:.9rem; color:var(--vh-muted); }
  .pw-meter{ height:.375rem; border-radius:999px; background:#e9ecef; overflow:hidden; }
  .pw-meter > span{ display:block; height:100%; width:0%; transition:width .25s ease; background:var(--vh-primary); }
  .req-list li{ margin:.15rem 0; }
</style>
{% endblock %}

{% block content %}
<div class="reg-hero">
  <div class="container py-5 py-xl-6">
    <div class="row justify-content-center">
      <div class="col-lg-8 col-xl-7">
        <div class="reg-card p-4 p-lg-5">
          <div class="d-flex align-items-center justify-content-between mb-3">
            <div class="d-inline-flex align-items-center gap-2">
              <i class="bi bi-person-plus-fill" style="color:var(--vh-primary);font-size:1.4rem"></i>
              <h1 class="h5 fw-bold mb-0">Criar novo usuário</h1>
            </div>
            <div class="d-flex gap-2">
              <a href="{{ url_for('central.central_engenharia') }}" class="btn btn-outline-hub btn-sm d-none d-md-inline">Ir para Engenharia</a>
              <a href="{{ url_for('central.central_atacado') }}"    class="btn btn-outline-hub btn-sm d-none d-md-inline">Ir para Atacado</a>
              <a href="{{ url_for('auth.admin_login') }}"           class="btn btn-outline-hub btn-sm">Sair do modo Admin</a>
            </div>
          </div>

          <p class="hint mb-4">
            Preencha os dados abaixo para criar um usuário. Apenas administradores autenticados podem acessar esta página.
          </p>

          <form method="post" action="{{ url_for('auth.register_post') }}" novalidate id="regForm" autocomplete="off">
            <div class="row g-3">

              <div class="col-12">
                <label for="email" class="form-label fw-semibold">E-mail corporativo</label>
                <input
                  type="email"
                  class="form-control form-control-lg"
                  id="email"
                  name="email"
                  placeholder="nome.sobrenome@empresa.com.br"
                  required
                  autocomplete="off"
                  inputmode="email"
                >
                <div class="invalid-feedback">Informe um e-mail válido.</div>
              </div>

              <div class="col-12">
                <label for="password"
                       class="form-label fw-semibold d-flex align-items-center justify-content-between">
                  <span>Senha</span>
                  <button class="btn btn-outline-hub btn-sm" type="button" id="togglePw"
                          aria-controls="password" aria-pressed="false">
                    <i class="bi bi-eye"></i>
                  </button>
                </label>
                <input
                  type="password"
                  class="form-control form-control-lg"
                  id="password"
                  name="password"
                  placeholder="Crie uma senha forte"
                  required
                  minlength="8"
                  autocomplete="new-password"
                >
                <div class="pw-meter mt-2" aria-hidden="true"><span id="pwBar"></span></div>
                <div class="form-text">Use no mínimo 8 caracteres. Recomenda-se combinar letras, números e símbolos.</div>
                <div class="invalid-feedback">Crie uma senha com pelo menos 8 caracteres.</div>
              </div>

              <div class="col-md-6">
                <label for="role" class="form-label fw-semibold">Perfil de acesso</label>
                <select id="role" name="role" class="form-select form-select-lg" required>
                  <option value="" selected disabled>Selecione…</option>
                  <option value="engenharia">Engenharia</option>
                  <option value="atacado">Atacado</option>
                </select>
                <div class="invalid-feedback">Selecione um perfil.</div>
              </div>

              <div class="col-md-6">
                <label class="form-label fw-semibold">Requisitos recomendados</label>
                <ul class="req-list small text-body-secondary mb-0">
                  <li>Senha única (não reutilizar de outros sistemas)</li>
                  <li>Ativar MFA no e-mail (se disponível)</li>
                  <li>Uso pessoal e intransferível</li>
                </ul>
              </div>

              <div class="col-12 d-flex gap-2 mt-2">
                <a href="{{ url_for('auth.register') }}" class="btn btn-outline-hub">Limpar</a>
                <button type="submit" class="btn btn-hub btn-create flex-grow-1" id="btnSubmit">
                  <span class="spinner-border spinner-border-sm me-2 d-none" role="status" aria-hidden="true"></span>
                  Criar usuário
                </button>
              </div>

            </div>
          </form>

          <div class="mt-4 small text-body-secondary">
            <i class="bi bi-shield-lock me-1"></i>
            O cadastro é registrado para auditoria. Ao criar usuários, você concorda com as políticas internas da plataforma.
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
{% endblock %}

{% block extra_scripts %}
<script>
  (function(){
    const form   = document.getElementById('regForm');
    const email  = document.getElementById('email');
    const pw     = document.getElementById('password');
    const role   = document.getElementById('role');
    const btn    = document.getElementById('btnSubmit');
    const bar    = document.getElementById('pwBar');
    const toggle = document.getElementById('togglePw');

    toggle?.addEventListener('click', ()=>{
      const show = pw.type === 'password';
      pw.type = show ? 'text' : 'password';
      toggle.setAttribute('aria-pressed', String(show));
      toggle.innerHTML = show ? '<i class="bi bi-eye-slash"></i>' : '<i class="bi bi-eye"></i>';
      pw.focus({preventScroll:true});
    });

    function score(s){
      if(!s) return 0;
      let n = 0;
      if(s.length>=8) n++;
      if(/[A-Z]/.test(s)) n++;
      if(/[a-z]/.test(s)) n++;
      if(/\d/.test(s)) n++;
      if(/[^A-Za-z0-9]/.test(s)) n++;
      return Math.min(n, 5);
    }
    function updateMeter(){
      const perc = [0,20,40,60,80,100][score(pw.value)] || 0;
      bar.style.width = perc + '%';
    }
    pw.addEventListener('input', updateMeter);
    updateMeter();

    form.addEventListener('submit', (e)=>{
      if(!form.checkValidity()){
        e.preventDefault(); e.stopPropagation();
        form.classList.add('was-validated');
        (email.value ? (pw.value ? role : pw) : email).focus();
        return;
      }
      const spn = btn.querySelector('.spinner-border');
      btn.disabled = true;
      if(spn) spn.classList.remove('d-none');
    });

    email?.focus();
  })();
</script>
{% endblock %}
