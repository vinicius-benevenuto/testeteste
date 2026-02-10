{% extends "base.html" %}
{% set engineer_mode = (session.get('role') == 'engenharia') %}
{% set is_edit = form is not none %}
{% set st = ((form.status if form else 'rascunho') or 'rascunho')|lower %}
{% block title %}VIVOHUB — {{ 'Pré-PTI (Engenharia)' if engineer_mode else 'Formulário Pré-PTI' }}{% endblock %}

{% block extra_head %}
<style>
  body.bg-plain, html { background:#fff !important; }
  .nav-white { background:#fff; border-bottom:1px solid var(--vh-border); }
  .section-card{ border:1px solid var(--vh-border); border-radius: var(--vh-radius); overflow:hidden; background:var(--vh-surface-0); }
  .section-card .card-header{
    background: linear-gradient(180deg, #fafafa, #fff);
    border-bottom:1px solid var(--vh-border);
    font-weight:600; letter-spacing:.2px;
  }
  .form-label{ font-weight:500; }
  .required::after{ content:" *"; color:var(--vh-danger); font-weight:700; margin-left:2px; }
  .table thead th{ white-space:nowrap; font-weight:600; }
  .table-sm td, .table-sm th{ vertical-align:middle; }
  .table-hover tbody tr:hover { background:#fbfbff; }
  .table-actions button{ width:2.2rem; height:2.2rem; padding:0; }
  .cap-counter{ font-size:.85rem; color:var(--vh-muted); }
  .action-bar{
    position:sticky; bottom:0; z-index:1020;
    background:#fff; border-top:1px solid var(--vh-border);
    box-shadow:0 -4px 18px rgba(0,0,0,.04);
  }
  .action-bar .btn{ min-width: 120px; }
  .status-badge{ font-weight:600; }
  .status-badge.st-rascunho{ background:#e9ecef; color:#343a40; }
  .status-badge.st-enviado{ background:#cfe2ff; color:#084298; }
  .status-badge.st-em-revisao{ background:#fff3cd; color:#664d03; }
  .status-badge.st-aprovado{ background:#d1e7dd; color:#0f5132; }
  .help-text{ font-size:.85rem; color:var(--vh-muted); }
  .scope-chips .form-check{ margin-right: .75rem; }
  .scope-chips .form-check-input{ cursor:pointer; }
  .scope-chips .form-check-label{ cursor:pointer; font-weight:500; }
  .readonly-mask input:not([type="hidden"]), .readonly-mask select, .readonly-mask textarea{
    background:#fbfbfb !important;
  }

  /* ===== PAINEL SBC ===== */
  .sbc-panel{
    border:1px solid #d4c5f9; border-radius:.65rem;
    background:linear-gradient(135deg,#faf8ff 0%,#fff 100%);
    padding:1rem 1.25rem; margin-top:.75rem;
  }
  .sbc-panel-header{
    display:flex; align-items:center; justify-content:space-between;
    margin-bottom:.65rem; gap:.5rem;
  }
  .sbc-panel-title{
    font-weight:700; font-size:.9rem; color:var(--vh-primary,#6b09a6);
    display:flex; align-items:center; gap:.4rem;
  }
  .sbc-panel-meta{ font-size:.75rem; color:var(--vh-muted,#6c757d); }
  #sbcCardList{
    max-height:340px; overflow-y:auto;
    scrollbar-width:thin; scrollbar-color:#d4c5f9 transparent;
  }
  #sbcCardList::-webkit-scrollbar{ width:5px; }
  #sbcCardList::-webkit-scrollbar-thumb{ background:#d4c5f9; border-radius:3px; }
  .sbc-card{
    border:1px solid var(--vh-border,#dee2e6); border-radius:.5rem;
    background:#fff; padding:.65rem .75rem; margin-bottom:.5rem;
    transition: border-color .15s, box-shadow .15s;
  }
  .sbc-card:hover{ border-color:#b39ddb; box-shadow:0 2px 8px rgba(107,9,166,.08); }
  .sbc-card.recommended{ border-color:#198754; border-width:2px; }
  .sbc-card-header{
    display:flex; align-items:center; justify-content:space-between;
    gap:.5rem; margin-bottom:.35rem;
  }
  .sbc-card-name{ font-weight:700; font-size:.88rem; }
  .sbc-card-cidade{ font-size:.78rem; color:#666; font-weight:400; margin-left:.35rem; }
  .sbc-card-badge{
    font-size:.7rem; font-weight:600; padding:.15rem .5rem;
    border-radius:999px; line-height:1.2;
  }
  .sbc-badge-disponivel{ background:#d1e7dd; color:#0f5132; }
  .sbc-badge-moderado{ background:#fff3cd; color:#664d03; }
  .sbc-badge-critico{ background:#f8d7da; color:#842029; }
  .sbc-card-details{
    display:grid; grid-template-columns:1fr 1fr; gap:.2rem .75rem;
    font-size:.78rem; color:#555;
  }
  .sbc-caps-bar-wrap{
    height:6px; background:#e9ecef; border-radius:3px;
    overflow:hidden; margin:.3rem 0;
  }
  .sbc-caps-bar{
    height:100%; border-radius:3px; transition: width .3s ease;
  }
  .sbc-caps-bar.low{ background:#198754; }
  .sbc-caps-bar.mid{ background:#ffc107; }
  .sbc-caps-bar.crit{ background:#dc3545; }
  .sbc-tendencia{
    font-size:.72rem; font-weight:600; display:inline-flex; align-items:center; gap:.2rem;
  }
  .sbc-tendencia.subindo{ color:#dc3545; }
  .sbc-tendencia.descendo{ color:#198754; }
  .sbc-tendencia.estavel{ color:#6c757d; }
  .sbc-card-meta{
    display:flex; gap:.5rem; flex-wrap:wrap;
    font-size:.72rem; color:var(--vh-muted); margin-top:.25rem;
  }
  .sbc-card-meta span{ display:inline-flex; align-items:center; gap:.2rem; }
  .sbc-card-footer{
    display:flex; align-items:center; justify-content:space-between;
    margin-top:.4rem; gap:.5rem;
  }
  .sbc-score{ font-weight:700; font-size:.85rem; }
  .sbc-reason{ font-size:.72rem; color:var(--vh-muted); flex:1; }
  .sbc-use-btn{ font-size:.75rem; padding:.2rem .6rem; white-space:nowrap; }
  .sbc-loading{ text-align:center; padding:.75rem; color:var(--vh-muted); font-size:.85rem; }
  .sbc-empty{ text-align:center; padding:.75rem; color:var(--vh-muted); font-size:.85rem; }
  .sbc-fallback-note{
    font-size:.75rem; padding:.35rem .6rem; margin-bottom:.5rem;
    background:#fff3cd; border:1px solid #ffe29a; border-radius:.4rem; color:#664d03;
  }

  @media (prefers-reduced-motion: reduce){
    .btn, .btn-hub, .btn-outline-hub { transition:none !important; }
  }
</style>
{% endblock %}

{% block content %}
{% set plain_bg = true %}

<datalist id="cnList">
  {% for cn in CN_FULL %}
    <option value="{{ cn.codigo }}">{{ cn.codigo }} — {{ cn.nome }}{% if cn.uf %}/{{ cn.uf }}{% endif %}</option>
  {% endfor %}
</datalist>

<nav class="navbar nav-white">
  <div class="container-fluid py-2">
    <div class="d-flex align-items-center gap-2">
      {% if engineer_mode %}
        <a class="btn btn-outline-hub btn-sm" href="{{ url_for('engenharia_form_list') }}">
          <i class="bi bi-arrow-left"></i> Voltar
        </a>
        <span class="fw-semibold">Pré-PTI <span class="badge text-bg-primary">Engenharia</span></span>
      {% else %}
        <a class="btn btn-outline-hub btn-sm" href="{{ url_for('atacado_form_list') }}">
          <i class="bi bi-arrow-left"></i> Voltar
        </a>
        <span class="fw-semibold">Formulário Pré-PTI</span>
      {% endif %}

      {% if is_edit %}
        <span class="ms-2 badge status-badge
          {% if st=='aprovado' %}st-aprovado{% elif st=='em revisão' %}st-em-revisao{% elif st=='enviado' %}st-enviado{% else %}st-rascunho{% endif %}">
          {{ st|capitalize }}
        </span>
        <span class="ms-1 small text-body-secondary">ID {{ form.id }}</span>
      {% endif %}
    </div>
    <div class="d-flex align-items-center gap-2">
      {% if is_edit %}
        <a class="btn btn-outline-hub btn-sm"
           href="{{ url_for('exportar_form_excel_index', form_id=form.id) }}"
           data-bs-toggle="tooltip" data-bs-title="Exportar Excel (Índice, Versões, Diagrama)">
          <i class="bi bi-file-earmark-spreadsheet"></i> Gerar PTI
        </a>
      {% endif %}
      <a href="{{ url_for('logout') }}" class="btn btn-outline-hub btn-sm">
        <i class="bi bi-box-arrow-right"></i> Sair
      </a>
    </div>
  </div>
</nav>

<main class="container py-4">
  <form
    id="formTecnico"
    method="post"
    action="{% if form and not engineer_mode %}{{ url_for('atacado_form_update', form_id=form.id) }}{% elif form and engineer_mode %}{{ url_for('engenharia_form_view', form_id=form.id) }}{% else %}{{ url_for('atacado_form_create') }}{% endif %}"
    novalidate
    data-form-id="{{ form.id if form else 'new' }}"
    data-vivo-json="{{ form.dados_vivo_json if form else '[]' }}"
    data-operadora-json="{{ form.dados_operadora_json if form else '[]' }}"
    data-engenharia-json="{{ form.engenharia_params_json if form else '{}' }}"
    class="{% if engineer_mode %}readonly-mask{% endif %}"
  >
    <!-- Hiddens -->
    <input type="hidden" id="dados_vivo_json" name="dados_vivo_json" value="{{ form.dados_vivo_json if form and form.dados_vivo_json else '[]' }}">
    <input type="hidden" id="dados_operadora_json" name="dados_operadora_json" value="{{ form.dados_operadora_json if form and form.dados_operadora_json else '[]' }}">
    <input type="hidden" id="engenharia_params_json" name="engenharia_params_json" value="{{ form.engenharia_params_json if form and form.engenharia_params_json else '{}' }}">
    {% if not engineer_mode %}
      <input type="hidden" id="status" name="status" value="{{ (form.status if form else '') or 'rascunho' }}">
      <input type="hidden" id="escopo_flags_json" name="escopo_flags_json" value="{{ form.escopo_flags_json if form and form.escopo_flags_json else '[]' }}">
    {% endif %}

    <!-- Meta topo -->
    <div class="d-flex flex-wrap align-items-center justify-content-between mb-3">
      <div class="small text-body-secondary">
        {% if is_edit %}
          <i class="bi bi-clock-history me-1"></i>
          Atualizado: {{ (form.updated_at or form.created_at or '')|date_br }}
        {% endif %}
      </div>
      <div class="small">
        <span class="text-body-secondary">Campos marcados com <span class="text-danger">*</span> são obrigatórios.</span>
      </div>
    </div>

    <!-- ============================================================= -->
    <!-- 1) Identificação Operadora                                    -->
    <!-- ============================================================= -->
    <section class="card section-card mb-3" aria-labelledby="sec-ident">
      <div class="card-header" id="sec-ident">1) Identificação da Operadora</div>
      <div class="card-body">
        <div class="row g-3">
          <div class="col-md-6">
            <label class="form-label required" for="nome_operadora">Nome da Operadora</label>
            <input type="text" class="form-control form-control-sm" id="nome_operadora" name="nome_operadora"
                   value="{{ form.nome_operadora if form else '' }}" required>
            <div class="invalid-feedback">Informe o nome da operadora.</div>
          </div>
          <div class="col-md-6">
            <label class="form-label" for="rn1">RN1</label>
            <input type="text" class="form-control form-control-sm" id="rn1" name="rn1"
                   value="{{ form.rn1 if form else '' }}" placeholder="Ex.: RN1-123">
          </div>
          <div class="col-md-6">
            <div class="d-flex flex-wrap gap-3">
              <div class="form-check"><input class="form-check-input" type="checkbox" id="csp" name="csp" {% if form and form.csp %}checked{% endif %}><label class="form-check-label" for="csp">CSP</label></div>
              <div class="form-check"><input class="form-check-input" type="checkbox" id="servicos_especiais" name="servicos_especiais" {% if form and form.servicos_especiais %}checked{% endif %}><label class="form-check-label" for="servicos_especiais">Serviços Especiais</label></div>
              <div class="form-check"><input class="form-check-input" type="checkbox" id="cng" name="cng" {% if form and form.cng %}checked{% endif %}><label class="form-check-label" for="cng">CNG</label></div>
            </div>
            <label class="form-label mt-3" for="atendimento">Atendimento</label>
            {% set atend = (form.atendimento if form else '') %}
            <select id="atendimento" name="atendimento" class="form-select form-select-sm">
              <option value="" {% if not atend %}selected{% endif %}>Selecione</option>
              <option value="UF"     {% if atend=='UF' %}selected{% endif %}>UF</option>
              <option value="CN"     {% if atend=='CN' %}selected{% endif %}>CN</option>
              <option value="REG I"  {% if atend=='REG I' %}selected{% endif %}>REG I</option>
              <option value="REG II" {% if atend=='REG II' %}selected{% endif %}>REG II</option>
              <option value="REG III"{% if atend=='REG III' %}selected{% endif %}>REG III</option>
            </select>
            <div class="help-text">Selecione REG I/II/III para exibir o campo "Qual?".</div>
          </div>
          <div class="col-md-6">
            <label class="form-label" for="redes">Redes</label>
            {% set redesv = (form.redes if form else '') %}
            <select id="redes" name="redes" class="form-select form-select-sm mb-2">
              <option value="" {% if not redesv %}selected{% endif %}>Selecione</option>
              <option value="STFC" {% if redesv=='STFC' %}selected{% endif %}>STFC</option>
              <option value="SMP"  {% if redesv=='SMP'  %}selected{% endif %}>SMP</option>
            </select>
            {% set qualv = (form.qual if form else '') %}
            <div id="qualGroup" class="mb-2 {% if atend and atend.startswith('REG') %}{% else %}d-none{% endif %}">
              <label class="form-label" for="qual">Qual? (se aplicável)</label>
              <input type="text" class="form-control form-control-sm" id="qual" name="qual" value="{{ qualv }}" placeholder="Ex.: Região II — Sul">
            </div>
            <label class="form-label" for="tmr">TMR</label>
            <input type="text" class="form-control form-control-sm" id="tmr" name="tmr"
                   value="{{ form.tmr if form else '' }}" placeholder="Ex.: 15s">
          </div>
        </div>
      </div>
    </section>

    <!-- ============================================================= -->
    <!-- 2) Contatos e Responsáveis                                    -->
    <!-- ============================================================= -->
    <section class="card section-card mb-3" aria-labelledby="sec-contatos">
      <div class="card-header" id="sec-contatos">2) Contatos e Responsáveis</div>
      <div class="card-body">
        <div class="row g-3">
          <div class="col-md-6">
            <label class="form-label required" for="responsavel_operadora">Responsável Operadora</label>
            <input type="text" class="form-control form-control-sm" id="responsavel_operadora" name="responsavel_operadora"
                   value="{{ form.responsavel_operadora if form else '' }}" required>
            <div class="invalid-feedback">Informe o responsável da operadora.</div>
          </div>
          <div class="col-md-6">
            <label class="form-label required" for="responsavel_vivo">Responsável Vivo (Atacado)</label>
            <input type="text" class="form-control form-control-sm" id="responsavel_vivo" name="responsavel_vivo"
                   value="{{ form.responsavel_vivo if form else '' }}" required>
            <div class="invalid-feedback">Informe o responsável da Vivo.</div>
          </div>
          <div class="col-md-6">
            <div class="d-flex flex-wrap gap-3">
              <div class="form-check"><input class="form-check-input" type="checkbox" id="sbc_ativo" name="sbc_ativo" {% if form and form.sbc_ativo %}checked{% endif %}><label class="form-check-label" for="sbc_ativo">Possui SBC ativo?</label></div>
              <div class="form-check"><input class="form-check-input" type="checkbox" id="ip_reservado" name="ip_reservado" {% if form and form.ip_reservado %}checked{% endif %}><label class="form-check-label" for="ip_reservado">Possui IP reservado?</label></div>
              <div class="form-check"><input class="form-check-input" type="checkbox" id="vivo_reserva" name="vivo_reserva" {% if form and form.vivo_reserva %}checked{% endif %}><label class="form-check-label" for="vivo_reserva">Deseja que a Vivo reserve?</label></div>
            </div>
          </div>
          <div class="col-md-6">
            <label class="form-label" for="asn">ASN (Operadora)</label>
            <input type="text" class="form-control form-control-sm" id="asn" name="asn"
                   value="{{ form.asn if form else '' }}" placeholder="Ex.: ASXXXXX">
            <div class="help-text">Informe o número AS do parceiro, se aplicável.</div>
          </div>
          <div class="col-md-6">
            <label class="form-label required" for="responsavel_itx_gestao">Responsável Gestão de ITX (Atacado)</label>
            <input type="text" class="form-control form-control-sm" id="responsavel_itx_gestao" name="responsavel_atacado"
                   value="{{ form.responsavel_atacado if form else preset_responsavel_atacado or '' }}" required>
            <div class="invalid-feedback">Informe o responsável de gestão de ITX.</div>
          </div>
          <div class="col-md-6">
            <label class="form-label" for="responsavel_itx_eng">Responsável Eng de ITX (Engenharia)</label>
            <input type="text" class="form-control form-control-sm" id="responsavel_itx_eng" name="responsavel_engenharia"
                   value="{{ form.responsavel_engenharia if form else '' }}">
            <div class="help-text">Campo preenchido pela Engenharia durante a validação.</div>
          </div>
        </div>
      </div>
    </section>

    <!-- ============================================================= -->
    <!-- 3) Escopo (Atacado) OU  3) Dados VIVO (Engenharia)            -->
    <!-- ============================================================= -->
    {% if not engineer_mode %}
    <section class="card section-card mb-3" aria-labelledby="sec-escopo">
      <div class="card-header" id="sec-escopo">3) Escopo</div>
      <div class="card-body">
        <label for="escopo_texto" class="form-label required">Detalhar o Escopo do Tráfego:</label>
        <textarea class="form-control" id="escopo_texto" name="escopo_text" rows="3" required
          placeholder="Ex.: Considerar LC nos CNs 31/11; LD15 + CNG nacional…">{{ form.escopo_text if form else '' }}</textarea>
        <div class="invalid-feedback">Descreva o escopo do tráfego.</div>
        <div class="mt-3">
          <div class="form-label mb-1">Selecione os tipos de tráfego incluídos:</div>
          <div class="scope-chips d-flex flex-wrap">
            <div class="form-check me-3"><input class="form-check-input esc-flag" type="checkbox" value="LC" id="esc_lc" data-flag="LC"><label class="form-check-label" for="esc_lc">LC</label></div>
            <div class="form-check me-3"><input class="form-check-input esc-flag" type="checkbox" value="LD15 + CNG" id="esc_ld15" data-flag="LD15 + CNG"><label class="form-check-label" for="esc_ld15">LD15 + CNG</label></div>
            <div class="form-check me-3"><input class="form-check-input esc-flag" type="checkbox" value="LDS/CSP + CNG" id="esc_ldscsp" data-flag="LDS/CSP + CNG"><label class="form-check-label" for="esc_ldscsp">LDS/CSP + CNG</label></div>
            <div class="form-check me-3"><input class="form-check-input esc-flag" type="checkbox" value="Transporte" id="esc_transp" data-flag="Transporte"><label class="form-check-label" for="esc_transp">Transporte</label></div>
            <div class="form-check me-3"><input class="form-check-input esc-flag" type="checkbox" value="VC1" id="esc_vc1" data-flag="VC1"><label class="form-check-label" for="esc_vc1">VC1</label></div>
            <div class="form-check me-3"><input class="form-check-input esc-flag" type="checkbox" value="Concentração" id="esc_conc" data-flag="Concentração"><label class="form-check-label" for="esc_conc">Concentração</label></div>
          </div>
          <div class="help-text mt-1">As seleções alimentam a "Ponta B — Tráfego" do diagrama no Excel.</div>
        </div>
      </div>
    </section>
    {% else %}
    <section class="card section-card mb-3" aria-labelledby="sec-vivo">
      <div class="card-header d-flex align-items-center justify-content-between" id="sec-vivo">
        <span>3) Dados VIVO</span>
        <div class="d-flex gap-2 align-items-center">
          <span class="cap-counter"><span id="vivoCount">0</span>/10</span>
          <button type="button" class="btn btn-sm btn-outline-hub" id="addRowVivo"><i class="bi bi-plus-lg"></i> Linha</button>
          <button type="button" class="btn btn-sm btn-outline-hub" id="clearRowsVivo">Limpar</button>
        </div>
      </div>
      <div class="card-body">
        <div class="table-responsive">
          <table class="table table-bordered table-sm table-hover align-middle" id="tableVivo">
            <thead class="table-light">
              <tr>
                <th>Ref</th><th>Data</th><th>Escopo</th><th>Localidade</th><th>CN</th><th>SBC</th><th>Mask</th>
                <th>Endereço LINK</th><th>Cidade</th><th>UF</th><th>LAT.</th><th>LONG</th>
                <th class="text-center">Ação</th>
              </tr>
            </thead>
            <tbody></tbody>
          </table>
          <div class="help-text">Máximo de 10 linhas. O backend aplica o mesmo limite.</div>
        </div>

        <!-- ===== PAINEL DE SUGESTÃO DE SBC ===== -->
        <div id="sbcSuggestionPanel" class="sbc-panel" style="display:none;" role="region" aria-label="Sugestão de SBC">
          <div class="sbc-panel-header">
            <div class="sbc-panel-title">
              <i class="bi bi-broadcast-pin"></i>
              <span id="sbcPanelTitle">SBCs sugeridos</span>
            </div>
            <div class="sbc-panel-meta" id="sbcPanelMeta"></div>
          </div>
          <div id="sbcFallbackNote" class="sbc-fallback-note" style="display:none;"></div>
          <div id="sbcCardList"></div>
          <div id="sbcLoading" class="sbc-loading" style="display:none;">Buscando SBCs disponíveis...</div>
          <div id="sbcEmpty" class="sbc-empty" style="display:none;"></div>
        </div>
      </div>
    </section>
    {% endif %}

    <!-- ============================================================= -->
    <!-- 4) Dados Operadora                                            -->
    <!-- ============================================================= -->
    <section class="card section-card mb-3" aria-labelledby="sec-op">
      <div class="card-header d-flex align-items-center justify-content-between" id="sec-op">
        <span>4) Dados Operadora</span>
        {% if not engineer_mode %}
        <div class="d-flex gap-2 align-items-center">
          <span class="cap-counter"><span id="opCount">0</span>/10</span>
          <button type="button" class="btn btn-sm btn-outline-hub" id="addRowOperadora"><i class="bi bi-plus-lg"></i> Linha</button>
          <button type="button" class="btn btn-sm btn-outline-hub" id="clearRowsOperadora">Limpar</button>
        </div>
        {% endif %}
      </div>
      <div class="card-body">
        <div class="table-responsive">
          <table class="table table-bordered table-sm table-hover align-middle" id="tableOperadora">
            <thead class="table-light">
              <tr>
                <th>Ref</th><th>Localidade</th><th>ETO LC</th><th>EOT LD</th><th>CN</th><th>SBC</th><th>Faixa de IP</th>
                <th>Concentração?</th><th>Endereço LINK</th><th>Cidade</th><th>UF</th><th>LAT.</th><th>LONG</th>
                {% if not engineer_mode %}<th class="text-center">Ação</th>{% endif %}
              </tr>
            </thead>
            <tbody></tbody>
          </table>
          <div class="help-text">Máximo de 10 linhas.</div>
        </div>
      </div>
    </section>

    <!-- ============================================================= -->
    <!-- 5) Infraestrutura                                             -->
    <!-- ============================================================= -->
    <section class="card section-card mb-3" aria-labelledby="sec-infra">
      <div class="card-header" id="sec-infra">5) Infraestrutura</div>
      <div class="card-body">
        <p class="mb-3"><strong>A Operadora será responsável pela construção da infraestrutura até a caixa Zero do prédio da VIVO.</strong></p>
        <div class="row g-3">
          <div class="col-md-4">
            <div class="form-check">
              <input class="form-check-input" type="checkbox" id="operadora_ciente" name="operadora_ciente"
                     {% if form and form.operadora_ciente %}checked{% endif %}>
              <label class="form-check-label" for="operadora_ciente">Operadora ciente?</label>
            </div>
          </div>
          <div class="col-md-8">
            <label class="form-label" for="responsavel_infra">Nome do responsável</label>
            <input type="text" class="form-control form-control-sm" id="responsavel_infra" name="responsavel_infra"
                   value="{{ form.responsavel_infra if form else '' }}">
          </div>
        </div>
      </div>
    </section>

    <!-- ============================================================= -->
    <!-- 6) LCR Nacional                                               -->
    <!-- ============================================================= -->
    <section class="card section-card mb-3" aria-labelledby="sec-lcr">
      <div class="card-header" id="sec-lcr">6) Transporte de Chamadas via LCR Nacional</div>
      <div class="card-body">
        <div class="form-check">
          <input class="form-check-input" type="checkbox" id="lcr_nacional" name="lcr_nacional"
                 {% if form and form.lcr_nacional %}checked{% endif %}>
          <label class="form-check-label" for="lcr_nacional">LCR Nacional?</label>
        </div>
        <div class="form-check mt-2">
          <input class="form-check-input" type="checkbox" id="white_list" name="white_list"
                 {% if form and form.white_list %}checked{% endif %}>
          <label class="form-check-label" for="white_list">Cadastro White List?</label>
        </div>
      </div>
    </section>

    <!-- ============================================================= -->
    <!-- 7) Faixa de Numeração                                         -->
    <!-- ============================================================= -->
    <section class="card section-card mb-3" aria-labelledby="sec-faixa">
      <div class="card-header" id="sec-faixa">7) Faixa de Numeração da Operadora</div>
      <div class="card-body">
        <div class="form-check">
          <input class="form-check-input" type="checkbox" id="prefixos_liberados_abr" name="prefixos_liberados_abr"
                 {% if form and form.prefixos_liberados_abr %}checked{% endif %}>
          <label class="form-check-label" for="prefixos_liberados_abr">Prefixos liberados na ABR?</label>
        </div>
      </div>
    </section>

    <!-- ============================================================= -->
    <!-- 8) Premissas de Abordagem                                     -->
    <!-- ============================================================= -->
    <section class="card section-card mb-3" aria-labelledby="sec-premissas">
      <div class="card-header" id="sec-premissas">8) Premissas de Abordagem</div>
      <div class="card-body">
        <div class="row g-3 align-items-center mb-3">
          <div class="col-md-8 d-flex align-items-center">
            <input class="form-check-input me-2" type="checkbox" id="premissas_ok" name="premissas_ok"
                   {% if form and form.premissas_ok %}checked{% endif %}>
            <label class="form-check-label fw-semibold mb-0" for="premissas_ok">
              A aprovação está de acordo com as informações apresentadas, bem como às 29 Pendências de Programação.
            </label>
          </div>
          <div class="col-md-4 d-flex align-items-center">
            <label class="form-label fw-semibold me-2 mb-0" for="aprovado_por">Aprovado por:</label>
            <input type="text" class="form-control form-control-sm" id="aprovado_por" name="aprovado_por"
                   value="{{ form.aprovado_por if form else '' }}" placeholder="Nome do responsável">
          </div>
        </div>
        <p class="mb-2"><strong>Premissas SIP:</strong> link (ex.: 1Gb com banda 100Mb e QoS 80%); ~100kb por canal; com dois links de 100Mb a soma deve ser 400 canais.</p>
        <p class="mb-2">Na Reunião de PTI a Operadora deverá informar a faixa de IP para as rotas do projeto, ou solicitar à Telefônica a designação de uma faixa de IP /29.</p>
        <p class="mb-3">A Operadora é responsável pela construção da infraestrutura até a caixa Zero do prédio da VIVO.</p>
        <div class="table-responsive">
          <table class="table table-bordered table-sm">
            <thead class="table-light"><tr><th>Tráfego</th><th>Descrição</th></tr></thead>
            <tbody>
              <tr><td>LC / TR LC (CNs1X)</td><td>Chamadas entre operadoras dentro da mesma Área Local; devoluções na mesma AL.</td></tr>
              <tr><td>LD 15 / CNG VIVO / SE</td><td>Encaminhamentos entre Vivo-STFC e operadora contratada (LD15, SE e CNG).</td></tr>
              <tr><td>LD Oper CSP XY / LD s/ CSP / CNG</td><td>Tráfego LD com/sem CSP e CNG conforme acordado.</td></tr>
              <tr><td>Transporte</td><td>Chamadas entregues pela operadora contratada de qualquer CN para qualquer operadora.</td></tr>
              <tr><td>VC1</td><td>Chamadas entre AL e VIVO-SMP no CN, destinadas aos prefixos da operadora.</td></tr>
            </tbody>
          </table>
        </div>
      </div>
    </section>

    <!-- ============================================================= -->
    <!-- 9) Parâmetros Técnicos — Engenharia                           -->
    <!-- ============================================================= -->
    {% if engineer_mode %}
    <section class="card section-card mb-3" id="eng-section" aria-labelledby="sec-eng">
      <div class="card-header d-flex align-items-center justify-content-between" id="sec-eng">
        <span>9) Parâmetros Técnicos — Engenharia</span>
        <div class="toolbar d-flex gap-2 flex-wrap">
          <button type="button" class="btn btn-outline-hub btn-sm" id="engCheckAll"><i class="bi bi-check2-square"></i> Marcar tudo</button>
          <button type="button" class="btn btn-outline-hub btn-sm" id="engUncheckAll"><i class="bi bi-x-square"></i> Limpar tudo</button>
        </div>
      </div>
      <div class="card-body eng-checklist">

        <div class="group-title fw-semibold mb-2">Dados do Tratamento das chamadas</div>
        {% for key, text in [
          ("tratamento.reinvite_obrigatorio", "É mandatório a utilização de re-invites pela Origem para definição de codecs quando o destino não define um único codec (envia lista)."),
          ("tratamento.ptime_maxptime_20ms", "<strong>Ptime/Maxptime</strong>: no SDP answer, devem ser múltiplos inteiros de 20 para codecs móveis (3GPP 26.114)."),
          ("tratamento.qos_marcacoes_rtp", "Para RTP em rede IP, espera-se marcação <strong>CS5 / DSCP 46 / EF</strong>.")
        ] %}
        <div class="row-line d-grid mt-2" style="grid-template-columns:36px 1fr;">
          <div class="d-grid place-items-center border-end"><input class="form-check-input eng-flag" type="checkbox" data-key="{{ key }}"></div>
          <div class="p-2"><p class="mb-0">{{ text|safe }}</p></div>
        </div>
        {% endfor %}

        <div class="group-title fw-semibold mt-3 mb-2">Negociação de CODEC / Modo</div>
        {% for key, text in [
          ("codec.sipi_sem_rn3_portado", "Chamadas <strong>SIP-I</strong> trocadas <em>sem RN3 (060)</em> para tráfego portado."),
          ("codec.oferta_g711a_g729_stfc", "SIP-I STFC/STFC e STFC/SMP: oferta mandatória dos codecs <strong>G.711A</strong> e <strong>G.729</strong>."),
          ("codec.amr_g711a_smp", "SIP-I SMP/SMP: <strong>AMR</strong> e <strong>G.711A</strong>."),
          ("codec.g711u_nao_suportado", "Codec <strong>G.711u</strong> não é suportado.")
        ] %}
        <div class="row-line d-grid mt-2" style="grid-template-columns:36px 1fr;">
          <div class="d-grid place-items-center border-end"><input class="form-check-input eng-flag" type="checkbox" data-key="{{ key }}"></div>
          <div class="p-2"><p class="mb-0">{{ text|safe }}</p></div>
        </div>
        {% endfor %}

        <div class="group-title fw-semibold mt-3 mb-2">Identificação e Interconexão</div>
        {% for key, text in [
          ("id.originador_fixo_isupbr_regiao3", "Originador válido (Rede Fixa): <strong>ISUPBR</strong> (Região III)."),
          ("id.originador_movel_isupbr_itu92_opcional", "Originador válido (Rede Móvel): <strong>ISUPBR</strong> (opcional <em>ITU-92</em>)."),
          ("id.fax_t38_nao_suportado", "<strong>FAX T.38</strong>: protocolo não suportado."),
          ("id.bgp_ip30_mpls_contingencia", "Configuração BGP em <strong>IP/30</strong> com <strong>MPLS</strong> e link de contingência.")
        ] %}
        <div class="row-line d-grid mt-2" style="grid-template-columns:36px 1fr;">
          <div class="d-grid place-items-center border-end"><input class="form-check-input eng-flag" type="checkbox" data-key="{{ key }}"></div>
          <div class="p-2"><p class="mb-0">{{ text|safe }}</p></div>
        </div>
        {% endfor %}

        <div class="group-title fw-semibold mt-3 mb-2">Principais RFC e SIP headers</div>
        {% for key, text in [
          ("rfc.3261_base", "<strong>RFC 3261</strong> — SIP (base)."),
          ("rfc.3262_100rel_prack", "<strong>RFC 3262</strong> — 100rel / PRACK."),
          ("rfc.sdp_3264_3266_2327_4566", "<strong>RFC 3264 / 3266 / 2327 / 4566</strong> — SDP."),
          ("rfc.4028_timers", "<strong>RFC 4028</strong> — SIP Timers.")
        ] %}
        <div class="row-line d-grid mt-2" style="grid-template-columns:36px 1fr;">
          <div class="d-grid place-items-center border-end"><input class="form-check-input eng-flag" type="checkbox" data-key="{{ key }}"></div>
          <div class="p-2"><p class="mb-0">{{ text|safe }}</p></div>
        </div>
        {% endfor %}

        <div class="group-title fw-semibold mt-3 mb-2">RFCs habilitadas/suportadas</div>
        {% for key, text in [
          ("habilitadas.3389_cn", "<strong>RFC 3389</strong> — Comfort Noise."),
          ("habilitadas.2833_dtmf", "<strong>RFC 2833</strong> — Transporte DTMF."),
          ("habilitadas.3264_hold", "<strong>RFC 3264</strong> — Suporte à função Hold.")
        ] %}
        <div class="row-line d-grid mt-2" style="grid-template-columns:36px 1fr;">
          <div class="d-grid place-items-center border-end"><input class="form-check-input eng-flag" type="checkbox" data-key="{{ key }}"></div>
          <div class="p-2"><p class="mb-0">{{ text|safe }}</p></div>
        </div>
        {% endfor %}

        <div class="group-title fw-semibold mt-3 mb-2">Early Media / RBT / Offer-Answer</div>
        {% for key, text in [
          ("early.3960_early_media", "<strong>RFC 3960</strong> — Early Media e Ring Tone Generation."),
          ("early.180ring_proxy_rbt_local", "<strong>OBS:</strong> Em <em>SIP 180 Ring</em> puro, o Proxy deve gerar o RBT localmente."),
          ("early.6337_offer_answer", "<strong>RFC 6337</strong> — Modelo de oferta-resposta (offer–answer).")
        ] %}
        <div class="row-line d-grid mt-2" style="grid-template-columns:36px 1fr;">
          <div class="d-grid place-items-center border-end"><input class="form-check-input eng-flag" type="checkbox" data-key="{{ key }}"></div>
          <div class="p-2"><p class="mb-0">{{ text|safe }}</p></div>
        </div>
        {% endfor %}

        <div class="group-title fw-semibold mt-3 mb-2">Observações da Engenharia (opcional)</div>
        <textarea class="form-control" id="eng_notes" placeholder="Ex.: particularidades do parceiro, exceções de RFC…" rows="3">{{ (form.engenharia_params_json or '{}') }}</textarea>
        <small class="text-body-secondary">Este texto também é salvo em <code>engenharia_params_json</code> como <code>notes</code>.</small>
      </div>
    </section>
    {% endif %}

    <!-- Barra de ações -->
    <div class="action-bar py-3">
      <div class="container d-flex justify-content-end gap-2">
        {% if engineer_mode %}
          <a class="btn btn-outline-hub" href="{{ url_for('engenharia_form_list') }}">Voltar</a>
          <button type="submit" id="btnSaveEng" class="btn btn-primary">
            <span class="spinner-border spinner-border-sm me-2 d-none" aria-hidden="true"></span>Salvar
          </button>
        {% else %}
          <button type="button" class="btn btn-outline-hub" id="resetForm">Limpar</button>
          <button type="submit" id="btnSave" class="btn btn-outline-hub" onclick="setStatus('rascunho')">
            <span class="spinner-border spinner-border-sm me-2 d-none" aria-hidden="true"></span>Salvar
          </button>
          <button type="submit" id="btnFinish" class="btn btn-success" onclick="return finalizeSubmit();">
            <span class="spinner-border spinner-border-sm me-2 d-none" aria-hidden="true"></span>Finalizar
          </button>
        {% endif %}
      </div>
    </div>
  </form>
</main>

<!-- ================================================================= -->
<!-- JAVASCRIPT                                                        -->
<!-- ================================================================= -->
<script data-engineer="{{ 'true' if engineer_mode else 'false' }}">
/* ===== GLOBALS ===== */
const engineerMode = document.currentScript.getAttribute('data-engineer') === 'true';
const ROW_CAP = 10;

/* ===== CN MAP (via API) ===== */
let CN_FULL = [];
let CN_MAP  = {};

fetch('/api/cns')
  .then(r => r.json())
  .then(data => {
    CN_FULL = data;
    data.forEach(({codigo, nome, uf}) => {
      CN_MAP[(codigo||'').toString().padStart(2,'0')] = { cidade: nome||'', uf: uf||'' };
    });
  })
  .catch(err => console.error('Erro ao carregar CNs:', err));

function resolveCityUF(cn){
  const code = String(cn||'').trim().padStart(2,'0');
  return CN_MAP[code] || { cidade:'', uf:'' };
}

/* ===== HELPERS ===== */
let isDirty = false, submitting = false;
function setDirty(){ isDirty = true; }

const mkInput = (ph='', attrs={}) => {
  const i = document.createElement('input');
  i.type='text'; i.placeholder=ph; i.className='form-control form-control-sm';
  Object.entries(attrs||{}).forEach(([k,v])=>{ if(v!=null) i.setAttribute(k,v); });
  i.addEventListener('input', setDirty);
  i.addEventListener('change', setDirty);
  return i;
};
const mkCheck = () => {
  const w=document.createElement('div'); w.className='form-check d-flex justify-content-center mb-0';
  const c=document.createElement('input'); c.type='checkbox'; c.className='form-check-input';
  c.addEventListener('change', setDirty); w.appendChild(c); return w;
};
const mkRemoveBtn = () => {
  const b=document.createElement('button'); b.type='button'; b.className='btn btn-outline-danger btn-sm';
  b.innerHTML='<i class="bi bi-dash-lg"></i>';
  b.addEventListener('click', e=>{ e.currentTarget.closest('tr')?.remove(); updateCounters(); setDirty(); });
  return b;
};

function updateCounters(){
  const tv=document.querySelector('#tableVivo tbody');
  const to=document.querySelector('#tableOperadora tbody');
  const vEl=document.getElementById('vivoCount'); if(vEl) vEl.textContent = tv ? tv.children.length : 0;
  const oEl=document.getElementById('opCount');   if(oEl) oEl.textContent = to ? to.children.length : 0;
}

/* ===== SERIALIZAÇÃO ===== */
function serializeTableVivo(){
  const rows=[];
  document.querySelectorAll('#tableVivo tbody tr').forEach(tr=>{
    const c=tr.querySelectorAll('td');
    rows.push({
      ref:c[0]?.querySelector('input')?.value||'', data:c[1]?.querySelector('input')?.value||'',
      escopo:c[2]?.querySelector('input')?.value||'', localidade:c[3]?.querySelector('input')?.value||'',
      cn:c[4]?.querySelector('input')?.value||'', sbc:c[5]?.querySelector('input')?.value||'',
      mask:c[6]?.querySelector('input')?.value||'', endereco_link:c[7]?.querySelector('input')?.value||'',
      cidade:c[8]?.querySelector('input')?.value||'', uf:c[9]?.querySelector('input')?.value||'',
      lat:c[10]?.querySelector('input')?.value||'', long:c[11]?.querySelector('input')?.value||''
    });
  });
  return rows;
}
function serializeTableOperadora(){
  const rows=[];
  document.querySelectorAll('#tableOperadora tbody tr').forEach(tr=>{
    const c=tr.querySelectorAll('td');
    rows.push({
      ref:c[0]?.querySelector('input')?.value||'', localidade:c[1]?.querySelector('input')?.value||'',
      eto_lc:c[2]?.querySelector('input')?.value||'', eot_ld:c[3]?.querySelector('input')?.value||'',
      cn:c[4]?.querySelector('input')?.value||'', sbc:c[5]?.querySelector('input')?.value||'',
      faixa_ip:c[6]?.querySelector('input')?.value||'',
      concentracao:c[7]?.querySelector('input')?.checked||false,
      endereco_link:c[8]?.querySelector('input')?.value||'',
      cidade:c[9]?.querySelector('input')?.value||'', uf:c[10]?.querySelector('input')?.value||'',
      lat:c[11]?.querySelector('input')?.value||'', long:c[12]?.querySelector('input')?.value||''
    });
  });
  return rows;
}

/* ================================================================= */
/* SBC SUGGESTION PANEL  (só ativo quando engineerMode=true)         */
/* Dados: CAPS + STATUS (fonte XLSX)                                 */
/* ================================================================= */
const SBC = (function(){
  const panel   = document.getElementById('sbcSuggestionPanel');
  const titleEl = document.getElementById('sbcPanelTitle');
  const metaEl  = document.getElementById('sbcPanelMeta');
  const fallEl  = document.getElementById('sbcFallbackNote');
  const listEl  = document.getElementById('sbcCardList');
  const loadEl  = document.getElementById('sbcLoading');
  const emptyEl = document.getElementById('sbcEmpty');

  if(!panel || !engineerMode) return { fetch:()=>{}, fillSbc:()=>{} };

  let _activeRowIndex = -1, _lastCn = '', _fetchTimeout = null;

  /* Barra visual baseada no score (0-100) */
  function scoreBarClass(score){
    if(score>=70) return 'low';    /* verde */
    if(score>=40) return 'mid';    /* amarelo */
    return 'crit';                 /* vermelho */
  }
  function badgeClass(saude){ return 'sbc-badge-'+(saude||'disponivel'); }
  function saudeLabel(s){
    return {disponivel:'Disponível',moderado:'Moderado',critico:'Crítico'}[s]||s;
  }
  function tendenciaHTML(t){
    if(!t || t==='estavel') return '<span class="sbc-tendencia estavel">→ estável</span>';
    if(t==='subindo')       return '<span class="sbc-tendencia subindo">↗ subindo</span>';
    return '<span class="sbc-tendencia descendo">↘ descendo</span>';
  }

  function renderCards(data){
    listEl.innerHTML=''; loadEl.style.display='none'; emptyEl.style.display='none';
    titleEl.textContent = `SBCs para ${data.uf} (${data.cidade}) — CN ${data.cn}`;
    if(data.source_file) metaEl.innerHTML = `Fonte: <strong>${data.source_file}</strong> · ${data.source_modificado_em||''}`;
    if(data.fallback_usado){
      fallEl.style.display='block';
      fallEl.innerHTML = `<i class="bi bi-info-circle me-1"></i> Sem SBCs na UF ${data.uf}. Usando dados de <strong>${data.fallback_origem}</strong>.`;
    } else { fallEl.style.display='none'; }

    if(!data.sbcs || data.sbcs.length===0){
      emptyEl.style.display='block';
      emptyEl.textContent = data.mensagem || 'Nenhum SBC encontrado.';
      panel.style.display='block'; return;
    }

    data.sbcs.forEach((sbc)=>{
      const card = document.createElement('div');
      card.className = 'sbc-card' + (sbc.recomendado?' recommended':'');
      const recBadge = sbc.recomendado ? '<span class="sbc-card-badge sbc-badge-disponivel">✅ RECOMENDADO</span>' : '';
      const scoreVal = sbc.score || 0;

      /* Montar bloco de responsável/prazo */
      let metaItems = '';
      if(sbc.responsavel) metaItems += `<span><i class="bi bi-person"></i> ${sbc.responsavel}</span>`;
      if(sbc.prazo) metaItems += `<span><i class="bi bi-calendar-event"></i> Prazo: ${sbc.prazo}</span>`;
      const metaBlock = metaItems ? `<div class="sbc-card-meta">${metaItems}</div>` : '';

      card.innerHTML = `
        <div class="sbc-card-header">
          <span class="sbc-card-name">${sbc.nome}<span class="sbc-card-cidade">${sbc.cidade||''}</span></span>
          <div class="d-flex gap-1 align-items-center">
            <span class="sbc-card-badge ${badgeClass(sbc.saude)}">${saudeLabel(sbc.saude)}</span>
            ${recBadge}
          </div>
        </div>
        <div class="sbc-caps-bar-wrap">
          <div class="sbc-caps-bar ${scoreBarClass(scoreVal)}" style="width:${Math.min(100,scoreVal)}%"></div>
        </div>
        <div class="sbc-card-details">
          <span>Modelo: <strong>${sbc.modelo||'—'}</strong></span>
          <span>CAPS avg: <strong>${sbc.caps_avg}</strong> · máx: ${sbc.caps_max} · mín: ${sbc.caps_min}</span>
          <span>Serviços: ${(sbc.servicos||[]).join(', ')||'—'}</span>
          <span>Medições: ${sbc.total_medicoes} · Status: <strong>${sbc.status_fonte}</strong> ${tendenciaHTML(sbc.caps_tendencia)}</span>
        </div>
        ${metaBlock}
        <div class="sbc-card-footer">
          <span class="sbc-score">Score: ${scoreVal}/100</span>
          <span class="sbc-reason">${sbc.motivo||''}</span>
          <button type="button" class="btn btn-sm btn-outline-hub sbc-use-btn" data-sbc-name="${sbc.nome}">
            Usar na linha ${_activeRowIndex+1}
          </button>
        </div>`;
      card.querySelector('.sbc-use-btn').addEventListener('click', ()=> fillSbc(sbc.nome, _activeRowIndex));
      listEl.appendChild(card);
    });
    panel.style.display='block';
  }

  function fillSbc(sbcName, rowIdx){
    const rows = document.querySelectorAll('#tableVivo tbody tr');
    if(rowIdx>=0 && rowIdx<rows.length){
      const inp = rows[rowIdx].querySelectorAll('td')[5]?.querySelector('input');
      if(inp){
        inp.value = sbcName;
        inp.style.transition='background .3s'; inp.style.background='#d1e7dd';
        setTimeout(()=>{ inp.style.background=''; }, 1200);
        setDirty();
      }
    }
  }

  async function fetchSuggestion(cn, rowIndex){
    cn = String(cn||'').trim().replace(/\D/g,'').slice(0,2);
    if(cn.length<2){ panel.style.display='none'; return; }
    if(cn===_lastCn && _activeRowIndex===rowIndex) return;
    _lastCn=cn; _activeRowIndex=rowIndex;

    if(_fetchTimeout) clearTimeout(_fetchTimeout);
    _fetchTimeout = setTimeout(async ()=>{
      panel.style.display='block'; loadEl.style.display='block';
      listEl.innerHTML=''; emptyEl.style.display='none'; fallEl.style.display='none';
      try{
        const resp = await fetch(`/api/sbc/suggest?cn=${encodeURIComponent(cn)}`);
        if(!resp.ok) throw new Error(`HTTP ${resp.status}`);
        renderCards(await resp.json());
      } catch(err){
        loadEl.style.display='none'; emptyEl.style.display='block';
        emptyEl.textContent='Erro ao buscar SBCs. Verifique o diretório de dados.';
        console.warn('SBC fetch error:', err);
      }
    }, 350);
  }

  return { fetch: fetchSuggestion, fillSbc };
})();

/* ===== AUTO-PREENCHER Cidade/UF + DISPARAR SBC ===== */
function wireCnAutoFill(tr){
  const cnInp    = tr.querySelector('input[data-field="cn"]');
  const ufInp    = tr.querySelector('input[data-field="uf"]');
  const cidInp   = tr.querySelector('input[data-field="cidade"]');
  if(!cnInp || !ufInp || !cidInp) return;

  const handler = ()=>{
    const v = (cnInp.value||'').replace(/\D/g,'').slice(0,2);
    cnInp.value = v;
    const info = resolveCityUF(v);
    if(info.cidade) cidInp.value = info.cidade;
    if(info.uf) ufInp.value = info.uf;
    setDirty();
    if(engineerMode && v.length===2){
      const rowIdx = Array.from(tr.parentElement.children).indexOf(tr);
      SBC.fetch(v, rowIdx);
    }
  };
  cnInp.addEventListener('change', handler);
  cnInp.addEventListener('blur', handler);
  if((cnInp.value||'').trim().length>=2) handler();
}

/* ===== TABELAS DINÂMICAS ===== */
(function(){
  let vivoInit=[], opInit=[];
  const fe = document.getElementById('formTecnico');
  if(fe){
    try{ const d=fe.getAttribute('data-vivo-json');      if(d) vivoInit=JSON.parse(d); } catch(_){}
    try{ const d=fe.getAttribute('data-operadora-json'); if(d) opInit=JSON.parse(d);   } catch(_){}
  }
  const tv=document.querySelector('#tableVivo tbody');
  const to=document.querySelector('#tableOperadora tbody');

  function addRowVivo(v={}){
    if(!tv) return;
    if(tv.children.length>=ROW_CAP){ alert('Máximo de '+ROW_CAP+' linhas.'); return; }
    const tr=document.createElement('tr');
    const cols=[
      {k:'ref'},{k:'data'},{k:'escopo'},{k:'localidade'},
      {k:'cn',attrs:{'data-field':'cn',list:'cnList',inputmode:'numeric',pattern:'\\d{2}',maxlength:'2'}},
      {k:'sbc'},{k:'mask'},{k:'endereco_link'},
      {k:'cidade',attrs:{'data-field':'cidade'}},
      {k:'uf',attrs:{'data-field':'uf',maxlength:'2'}},
      {k:'lat'},{k:'long'}
    ];
    cols.forEach(({k,attrs})=>{
      const td=document.createElement('td');
      const inp=mkInput(k.toUpperCase(),attrs||{});
      inp.value=(v[k]??''); td.appendChild(inp); tr.appendChild(td);
    });
    const a=document.createElement('td'); a.className='text-center table-actions'; a.appendChild(mkRemoveBtn()); tr.appendChild(a);
    tv.appendChild(tr); wireCnAutoFill(tr); updateCounters(); setDirty();
  }

  function addRowOperadora(v={}){
    if(!to) return;
    if(to.children.length>=ROW_CAP){ alert('Máximo de '+ROW_CAP+' linhas.'); return; }
    const tr=document.createElement('tr');
    ['ref','localidade','eto_lc','eot_ld'].forEach(k=>{
      const td=document.createElement('td'); const inp=mkInput(k.toUpperCase()); inp.value=v[k]??''; td.appendChild(inp); tr.appendChild(td);
    });
    {const td=document.createElement('td');const inp=mkInput('CN',{'data-field':'cn',list:'cnList',inputmode:'numeric',pattern:'\\d{2}',maxlength:'2'});inp.value=v['cn']??'';td.appendChild(inp);tr.appendChild(td);}
    {const td=document.createElement('td');const inp=mkInput('SBC');inp.value=v['sbc']??'';td.appendChild(inp);tr.appendChild(td);}
    {const td=document.createElement('td');const inp=mkInput('Faixa de IP');inp.value=v['faixa_ip']??'';td.appendChild(inp);tr.appendChild(td);}
    const tdC=document.createElement('td');tdC.appendChild(mkCheck());tdC.querySelector('input').checked=!!v.concentracao;tr.appendChild(tdC);
    [{k:'endereco_link'},{k:'cidade',attrs:{'data-field':'cidade'}},{k:'uf',attrs:{'data-field':'uf',maxlength:'2'}},{k:'lat'},{k:'long'}].forEach(({k,attrs})=>{
      const td=document.createElement('td');const inp=mkInput(k.toUpperCase(),attrs||{});inp.value=v[k]??'';td.appendChild(inp);tr.appendChild(td);
    });
    if(!engineerMode){const a=document.createElement('td');a.className='text-center table-actions';a.appendChild(mkRemoveBtn());tr.appendChild(a);}
    to.appendChild(tr); wireCnAutoFill(tr); updateCounters(); setDirty();
  }

  (Array.isArray(vivoInit)?vivoInit:[]).forEach(addRowVivo);
  (Array.isArray(opInit)?opInit:[]).forEach(addRowOperadora);

  document.getElementById('addRowVivo')?.addEventListener('click',()=>addRowVivo({}));
  document.getElementById('clearRowsVivo')?.addEventListener('click',()=>{if(tv){tv.innerHTML='';updateCounters();setDirty();}});
  document.getElementById('addRowOperadora')?.addEventListener('click',()=>addRowOperadora({}));
  document.getElementById('clearRowsOperadora')?.addEventListener('click',()=>{if(to){to.innerHTML='';updateCounters();setDirty();}});
  updateCounters();
})();

/* ===== ENGENHARIA CHECKLIST HYDRATE ===== */
(function(){
  if(!engineerMode) return;
  let saved={};
  const fe=document.getElementById('formTecnico');
  if(fe){ try{ const d=fe.getAttribute('data-engenharia-json'); if(d) saved=JSON.parse(d); } catch(_){} }
  document.querySelectorAll('#eng-section .eng-flag[data-key]').forEach(chk=>{
    const key=chk.getAttribute('data-key');
    if(Object.prototype.hasOwnProperty.call(saved,key)) chk.checked=!!saved[key];
  });
  if(typeof saved.notes==='string'){
    const n=document.getElementById('eng_notes'); if(n) n.value=saved.notes;
  }
  document.getElementById('engCheckAll')?.addEventListener('click',()=>{document.querySelectorAll('#eng-section .eng-flag').forEach(c=>c.checked=true);setDirty();});
  document.getElementById('engUncheckAll')?.addEventListener('click',()=>{document.querySelectorAll('#eng-section .eng-flag').forEach(c=>c.checked=false);setDirty();});
})();

/* ===== QUAL? TOGGLE ===== */
(function(){
  const at=document.getElementById('atendimento'), qg=document.getElementById('qualGroup');
  function toggle(){ if(qg) qg.classList.toggle('d-none',!(at?.value||'').startsWith('REG')); setDirty(); }
  if(at){ at.addEventListener('change',toggle); toggle(); }
})();

/* ===== ESCOPO FLAGS (Atacado) ===== */
(function(){
  if(engineerMode) return;
  const h=document.getElementById('escopo_flags_json'); if(!h) return;
  const flags=(function(){try{return JSON.parse(h.value||'[]');}catch(_){return[];}})();
  document.querySelectorAll('.esc-flag').forEach(chk=>{
    if(flags.includes(chk.getAttribute('data-flag'))) chk.checked=true;
    chk.addEventListener('change',()=>{
      const cur=[]; document.querySelectorAll('.esc-flag:checked').forEach(c=>cur.push(c.getAttribute('data-flag')));
      h.value=JSON.stringify(cur); setDirty();
    });
  });
})();

/* ===== VALIDAÇÃO E SUBMIT ===== */
const form=document.getElementById('formTecnico');
function setStatus(v){ const s=document.getElementById('status'); if(s) s.value=v; }
function finalizeSubmit(){
  if(!confirm('Confirmar finalização do Pré-PTI? Após enviar, ele seguirá para revisão da Engenharia.')) return false;
  setStatus('enviado'); return true;
}
function showSpinner(id,show){
  const b=document.getElementById(id),s=b?.querySelector('.spinner-border');
  if(b) b.disabled=!!show; if(s) s.classList.toggle('d-none',!show);
}

window.addEventListener('beforeunload',(e)=>{if(isDirty&&!submitting){e.preventDefault();e.returnValue='';}});

document.getElementById('resetForm')?.addEventListener('click',()=>{
  form.reset(); form.classList.remove('was-validated');
  const tv=document.querySelector('#tableVivo tbody');if(tv) tv.innerHTML='';
  const to=document.querySelector('#tableOperadora tbody');if(to) to.innerHTML='';
  const qg=document.getElementById('qualGroup');if(qg) qg.classList.add('d-none');
  const at=document.getElementById('atendimento');if(at) at.value='';
  updateCounters(); setDirty(); window.scrollTo({top:0,behavior:'smooth'});
});

document.querySelectorAll('#formTecnico input:not([type="hidden"]),#formTecnico select,#formTecnico textarea').forEach(el=>{
  el.addEventListener('input',setDirty); el.addEventListener('change',setDirty);
});

form.addEventListener('submit',(e)=>{
  const vivoH=document.getElementById('dados_vivo_json');
  const opH=document.getElementById('dados_operadora_json');
  if(engineerMode && document.getElementById('tableVivo'))  vivoH.value=JSON.stringify(serializeTableVivo());
  if(!engineerMode && document.getElementById('tableOperadora')) opH.value=JSON.stringify(serializeTableOperadora());

  if(engineerMode){
    const data={};
    document.querySelectorAll('#eng-section .eng-flag[data-key]').forEach(chk=>{ data[chk.getAttribute('data-key')]=!!chk.checked; });
    const n=document.getElementById('eng_notes'); if(n&&n.value.trim()) data.notes=n.value.trim();
    document.getElementById('engenharia_params_json').value=JSON.stringify(data);
  }

  if(!form.checkValidity()){
    e.preventDefault(); e.stopPropagation(); form.classList.add('was-validated');
    const inv=form.querySelector(':invalid');
    if(inv){inv.scrollIntoView({behavior:'smooth',block:'center'});inv.focus({preventScroll:true});}
    return;
  }
  submitting=true; form.classList.add('was-validated');
  if(engineerMode) showSpinner('btnSaveEng',true);
  else if(document.activeElement?.id==='btnFinish') showSpinner('btnFinish',true);
  else showSpinner('btnSave',true);
});

/* ===== HOTKEYS ===== */
document.addEventListener('keydown',(e)=>{
  if(e.target&&['INPUT','TEXTAREA','SELECT'].includes(e.target.tagName)){
    if((e.ctrlKey||e.metaKey)&&e.key.toLowerCase()==='s'){
      e.preventDefault();
      if(engineerMode) document.getElementById('btnSaveEng')?.click();
      else{setStatus('rascunho');document.getElementById('btnSave')?.click();}
    }
    return;
  }
  if(e.key==='g'||e.key==='G') window.scrollTo({top:0,behavior:'smooth'});
});

/* ===== TOOLTIPS ===== */
document.querySelectorAll('[data-bs-toggle="tooltip"]').forEach(el=>new bootstrap.Tooltip(el));
</script>
{% endblock %}

