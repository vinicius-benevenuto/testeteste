{% extends "base.html" %}
{% block title %}VIVOHUB — Acesso do Administrador{% endblock %}

{% block extra_head %}
<style>
  .admin-hero{
    position:relative; isolation:isolate;
    background:
      radial-gradient(1200px 600px at -10% -20%, rgba(107,9,166,.14), transparent 60%),
      radial-gradient(800px 500px at 120% 10%, rgba(107,9,166,.12), transparent 60%),
      linear-gradient(180deg, #ffffff 0%, var(--vh-bg) 100%);
  }
  .admin-card{
    border:1px solid var(--vh-border);
    border-radius: var(--vh-radius);
    background: var(--vh-surface-0);
    box-shadow: var(--vh-shadow-md);
  }
  .admin-card .form-control:focus{
    border-color: var(--vh-primary);
    box-shadow: 0 0 0 .2rem rgba(107,9,166,.15);
  }
  .btn-admin{ padding:.7rem 1rem; font-weight:700; border-radius:.9rem; }
  .helper{ font-size:.9rem; color:var(--vh-muted); }
</style>
{% endblock %}

{% block content %}
<div class="admin-hero">
  <div class="container py-5 py-xl-6">
    <div class="row justify-content-center">
      <div class="col-lg-7 col-xl-6">
        <div class="admin-card p-4 p-lg-5">
          <div class="d-flex align-items-center justify-content-between mb-3">
            <div class="d-inline-flex align-items-center gap-2">
              <i class="bi bi-key-fill" style="color:var(--vh-primary);font-size:1.4rem"></i>
              <h1 class="h5 fw-bold mb-0">Acesso do Administrador</h1>
            </div>
            <a href="{{ url_for('login') }}" class="btn btn-outline-hub btn-sm">← Voltar</a>
          </div>

          <p class="helper mb-4">
            Use o <strong>código de administrador</strong> temporário para gerenciar cadastros.
            O acesso é auditável. Mantenha este código em sigilo.
          </p>

          <form method="post" action="{{ url_for('admin_login_post') }}" novalidate id="adminForm">
            <div class="mb-3">
              <label for="code" class="form-label fw-semibold">Código de administrador</label>
              <div class="input-group input-group-lg">
                <input
                  type="password"
                  class="form-control"
                  id="code"
                  name="code"
                  inputmode="text"
                  placeholder="Digite o código fornecido"
                  required
                  aria-describedby="codeHelp"
                >
                <button class="btn btn-outline-hub" type="button" id="toggleCode" aria-controls="code" aria-pressed="false">
                  <i class="bi bi-eye"></i>
                </button>
              </div>
              <div id="codeHelp" class="form-text">Solicite ao responsável pela plataforma, se necessário.</div>
              <div class="invalid-feedback">Informe o código para continuar.</div>
            </div>

            <button type="submit" class="btn btn-hub btn-admin w-100" id="btnSubmit">
              <span class="spinner-border spinner-border-sm me-2 d-none" role="status" aria-hidden="true"></span>
              Entrar como Admin
            </button>
          </form>

          <div class="mt-4 small text-body-secondary">
            <i class="bi bi-shield-lock me-1"></i>
            Ao acessar, você concorda em utilizar o sistema conforme as políticas internas.
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
    const form = document.getElementById('adminForm');
    const code = document.getElementById('code');
    const btnSubmit = document.getElementById('btnSubmit');
    const toggle = document.getElementById('toggleCode');

    // Mostrar/ocultar código
    toggle?.addEventListener('click', ()=>{
      const isText = code.type === 'text';
      code.type = isText ? 'password' : 'text';
      toggle.setAttribute('aria-pressed', String(!isText));
      toggle.innerHTML = isText ? '<i class="bi bi-eye"></i>' : '<i class="bi bi-eye-slash"></i>';
      code.focus({preventScroll:true});
    });

    // Submissão com validação e spinner
    form.addEventListener('submit', (e)=>{
      if(!form.checkValidity()){
        e.preventDefault(); e.stopPropagation();
        form.classList.add('was-validated');
        code.focus();
        return;
      }
      const spn = btnSubmit.querySelector('.spinner-border');
      btnSubmit.disabled = true;
      if (spn) spn.classList.remove('d-none');
    });

    // Foco inicial
    code?.focus();
  })();
</script>
{% endblock %}
