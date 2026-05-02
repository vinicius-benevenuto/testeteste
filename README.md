{% extends "base.html" %}
{% block title %}PTI AUTOMATIZADO — Login{% endblock %}
{% block content %}
<div class="page-sm" style="padding-top:4rem">
  <div class="card" style="padding:2rem 2rem 1.75rem">

    <div style="margin-bottom:1.5rem">
      <p style="font-size:.75rem;font-weight:600;color:var(--p);letter-spacing:.08em;text-transform:uppercase;margin:0 0 .5rem">PTI AUTOMATIZADO</p>
      <h1 style="font-size:1.4rem;font-weight:700;letter-spacing:-.3px;margin:0">Entrar</h1>
    </div>

    <form method="post" action="{{ url_for('auth.login_post') }}" novalidate id="loginForm">
      <div style="margin-bottom:1rem">
        <label class="v-label" for="email">E-mail</label>
        <input class="v-input" type="email" id="email" name="email"
               placeholder="nome@empresa.com" required autofocus autocomplete="email">
      </div>

      <div style="margin-bottom:1.5rem">
        <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:.3rem">
          <label class="v-label" for="password" style="margin:0">Senha</label>
          <button type="button" id="togglePwd" style="font-size:.75rem;color:var(--sub);background:none;border:none;cursor:pointer;padding:0">Mostrar</button>
        </div>
        <input class="v-input" type="password" id="password" name="password"
               placeholder="••••••••" required autocomplete="current-password">
        <p id="capsWarn" style="display:none;font-size:.72rem;color:#92400e;margin:.3rem 0 0">
          <i class="bi bi-exclamation-triangle"></i> Caps Lock ativo
        </p>
      </div>

      <button type="submit" class="btn-p" id="btnLogin"
              style="width:100%;justify-content:center;padding:.65rem">
        <span class="spinner-border spinner-border-sm d-none me-1" id="spin"></span>
        Entrar
      </button>
    </form>

    <div style="margin-top:1.25rem;padding-top:1.25rem;border-top:1px solid var(--bdr);text-align:center">
      <a href="{{ url_for('auth.admin_login') }}" style="font-size:.78rem;color:var(--sub);text-decoration:none">
        Acesso de administrador
      </a>
    </div>
  </div>
</div>
{% endblock %}
{% block extra_scripts %}
<script>
(function(){
  const form  = document.getElementById('loginForm');
  const email = document.getElementById('email');
  const pwd   = document.getElementById('password');
  const btn   = document.getElementById('btnLogin');
  const spin  = document.getElementById('spin');
  const toggle= document.getElementById('togglePwd');
  const caps  = document.getElementById('capsWarn');

  toggle.addEventListener('click',()=>{
    const show = pwd.type==='password';
    pwd.type = show?'text':'password';
    toggle.textContent = show?'Ocultar':'Mostrar';
    pwd.focus();
  });

  pwd.addEventListener('keydown', e=>{
    try{ caps.style.display = e.getModifierState('CapsLock')?'block':'none'; }catch(_){}
  });

  form.addEventListener('submit',e=>{
    if(!form.checkValidity()){ e.preventDefault(); return; }
    btn.disabled=true; spin.classList.remove('d-none');
  });

  if(email.value) pwd.focus();
})();
</script>
{% endblock %}