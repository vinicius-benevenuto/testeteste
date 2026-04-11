{% extends "base.html" %}
{% block title %}Login — VIVOHUB{% endblock %}

{% block extra_head %}
<style>
  .vh-hero{
    position:relative; isolation:isolate;
    background:
      radial-gradient(1200px 600px at -10% -20%, rgba(107,9,166,.14), transparent 60%),
      radial-gradient(800px 500px at 120% 10%, rgba(107,9,166,.12), transparent 60%),
      linear-gradient(180deg, #ffffff 0%, var(--vh-bg) 100%);
  }
  .vh-hero::after{
    content:""; position:absolute; inset:0; z-index:-1;
    background: radial-gradient(300px 300px at 70% 10%, rgba(107,9,166,.08), transparent 60%);
  }
  .login-card{
    border:1px solid var(--vh-border);
    border-radius: var(--vh-radius);
    background: var(--vh-surface-0);
    box-shadow: var(--vh-shadow-md);
  }
  .login-card .form-control:focus{
    border-color: var(--vh-primary);
    box-shadow: 0 0 0 .2rem rgba(107,9,166,.15);
  }
  .btn-login{ padding:.7rem 1rem; font-weight:700; border-radius:.9rem; }
  .auth-links a{ text-decoration:none; }
  .auth-links a:hover{ text-decoration:underline; }
  .caps-hint{ font-size:.85rem; color:#c03548; display:none; }
</style>
{% endblock %}

{% block content %}
<div class="vh-hero">
  <div class="container py-5 py-xl-6">
    <div class="row align-items-center g-4">

      <!-- Lado de mensagem / marca -->
      <div class="col-lg-6">
        <div class="d-flex align-items-center gap-3 mb-3">
          <h1 class="h3 fw-bold mb-0" style="color:var(--vh-primary)">Bem-vindo</h1>
        </div>
        <p class="lead text-body-secondary mb-3">
          Plataforma para gestão de <strong>PTI</strong> entre <em>Atacado</em> e <em>Engenharia</em>.
        </p>
        <ul class="text-body-secondary small mb-0">
          <li>Design consistente com foco em velocidade e acessibilidade.</li>
          <li>Exportação de Excel (Índice, Versões e Diagrama) em um clique.</li>
          <li>Atalhos úteis e validações inteligentes durante o preenchimento.</li>
        </ul>
      </div>

      <!-- Formulário -->
      <div class="col-lg-6">
        <div class="login-card p-4 p-lg-5">
          <div class="mb-3 text-center">
            <div class="d-inline-flex align-items-center gap-2">
              <i class="bi bi-shield-lock-fill" style="color:var(--vh-primary);font-size:1.3rem"></i>
              <span class="fw-semibold">Acesso seguro</span>
            </div>
          </div>

          <form method="post" action="{{ url_for('auth.login_post') }}" novalidate autocomplete="on" id="loginForm">
            <div class="mb-3">
              <label for="email" class="form-label fw-semibold">E-mail</label>
              <input
                type="email"
                class="form-control form-control-lg"
                id="email"
                name="email"
                inputmode="email"
                placeholder="nome.sobrenome@empresa.com"
                required
                autofocus
              >
              <div class="invalid-feedback">Informe um e-mail válido.</div>
            </div>

            <div class="mb-2">
              <label for="password" class="form-label fw-semibold d-flex justify-content-between align-items-center">
                <span>Senha</span>
                <button class="btn btn-sm btn-outline-hub" type="button" id="togglePwd"
                        aria-controls="password" aria-pressed="false">
                  <i class="bi bi-eye"></i> Mostrar
                </button>
              </label>
              <div class="input-group input-group-lg">
                <input
                  type="password"
                  class="form-control"
                  id="password"
                  name="password"
                  placeholder="Sua senha"
                  required
                  aria-describedby="capsMsg"
                >
              </div>
              <div id="capsMsg" class="caps-hint mt-1">
                <i class="bi bi-exclamation-triangle-fill me-1"></i> Caps Lock está ativado.
              </div>
              <div class="invalid-feedback">Informe sua senha.</div>
            </div>

            <div class="d-flex justify-content-between align-items-center mb-4">
              <div class="form-check">
                <input class="form-check-input" type="checkbox" value="1" id="remember" disabled>
                <label class="form-check-label text-body-secondary" for="remember">
                  Lembrar sessão (em breve)
                </label>
              </div>
              <div class="auth-links small">
                <a class="text-body-secondary" href="{{ url_for('auth.admin_login') }}">Sou administrador</a>
              </div>
            </div>

            <button type="submit" class="btn btn-hub btn-login w-100" id="btnSubmit">
              <span class="spinner-border spinner-border-sm me-2 d-none" role="status" aria-hidden="true"></span>
              Entrar
            </button>
          </form>

          <div class="mt-4 small text-body-secondary text-center">
            Problemas para acessar? Entre em contato com o suporte da sua área.
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
    const form      = document.getElementById('loginForm');
    const email     = document.getElementById('email');
    const pwd       = document.getElementById('password');
    const capsMsg   = document.getElementById('capsMsg');
    const btnToggle = document.getElementById('togglePwd');
    const btnSubmit = document.getElementById('btnSubmit');

    btnToggle?.addEventListener('click', ()=>{
      const isText = pwd.type === 'text';
      pwd.type = isText ? 'password' : 'text';
      btnToggle.setAttribute('aria-pressed', String(!isText));
      btnToggle.innerHTML = isText
        ? '<i class="bi bi-eye"></i> Mostrar'
        : '<i class="bi bi-eye-slash"></i> Ocultar';
      pwd.focus({preventScroll:true});
    });

    function handleCaps(e){
      try{
        const on = e.getModifierState && e.getModifierState('CapsLock');
        if(capsMsg) capsMsg.style.display = on ? 'block' : 'none';
      }catch(_){}
    }
    pwd.addEventListener('keyup', handleCaps);
    pwd.addEventListener('keydown', handleCaps);

    form.addEventListener('submit', (ev)=>{
      if(!form.checkValidity()){
        ev.preventDefault(); ev.stopPropagation();
        form.classList.add('was-validated');
        (email.value ? pwd : email).focus();
        return;
      }
      const spn = btnSubmit.querySelector('.spinner-border');
      btnSubmit.disabled = true;
      if(spn) spn.classList.remove('d-none');
    });

    window.setTimeout(()=>{
      if(email?.value) pwd?.focus();
    }, 50);
  })();
</script>
{% endblock %}
