form.addEventListener('submit', e=>{
  submitting = true;

  // Serializar tabelas
  const vh = document.getElementById('dados_vivo_json');
  const oh = document.getElementById('dados_operadora_json');

  if (ENG && document.getElementById('tableVivo')) {
    vh.value = JSON.stringify(serVivo());
  }

  if (!ENG && document.getElementById('tableOperadora')) {
    oh.value = JSON.stringify(serOp());
  }

  // Engenharia → validar parâmetros obrigatórios
  if (ENG) {
    const d = {};

    document.querySelectorAll('#eng-section .eng-flag[data-key]').forEach(c=>{
      d[c.getAttribute('data-key')] = !!c.checked;
    });

    const n = document.getElementById('eng_notes');
    if (n && n.value.trim()) {
      d.notes = n.value.trim();
    }

    document.getElementById('engenharia_params_json').value = JSON.stringify(d);

    let ok = true;

    document.querySelectorAll('#eng-section .eng-card[data-eng-group]').forEach(card=>{
      card.classList.remove('invalid');

      if (!card.querySelectorAll('.eng-flag:checked').length) {
        card.classList.add('invalid');
        ok = false;
      }
    });

    if (!ok) {
      e.preventDefault();
      e.stopPropagation();

      alert('Preencha todos os grupos de parâmetros da Seção 9.');

      document.querySelector('.eng-card.invalid')?.scrollIntoView({
        behavior:'smooth',
        block:'center'
      });

      submitting = false;
      return;
    }
  }

  // Feedback visual botões
  if (ENG) {
    document.getElementById('spin')?.classList.remove('d-none');
  } else {
    const status = document.getElementById('status')?.value;

    if (status === 'rascunho') {
      document.getElementById('spinSave')?.classList.remove('d-none');
    } else {
      document.getElementById('spinFinish')?.classList.remove('d-none');
    }
  }
});