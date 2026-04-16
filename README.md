{% extends "base.html" %}
{% set engineer_mode = (session.get('role') == 'engenharia') %}
{% set is_edit = form is not none %}
{% set st = ((form.status if form else 'rascunho') or 'rascunho')|lower %}
{% block title %}{{ 'Validar PTI' if engineer_mode else 'Pré-PTI' }} — VIVOHUB{% endblock %}

{% block extra_head %}
<style>
  /* ── Layout ── */
  body.plain { background:#fff; }
  .form-nav {
    position:sticky; top:52px; z-index:90;
    background:#fff; border-bottom:1px solid var(--bdr);
    display:flex; align-items:center; justify-content:space-between;
    padding:0 1.5rem; height:48px; gap:1rem;
  }
  .form-main { max-width:960px; margin:0 auto; padding:1.5rem 1.5rem 6rem; }

  /* ── Seções ── */
  .sec {
    background:var(--surf); border:1px solid var(--bdr);
    border-radius:var(--r); margin-bottom:1rem; overflow:hidden;
  }
  .sec-header {
    display:flex; align-items:center; justify-content:space-between;
    padding:.75rem 1.25rem; border-bottom:1px solid var(--bdr);
    background:#fafafa;
  }
  .sec-title { font-size:.82rem; font-weight:700; letter-spacing:-.1px; margin:0; }
  .sec-body  { padding:1.25rem; }

  /* ── Campos ── */
  .field-grid { display:grid; gap:1rem; }
  .col2 { grid-template-columns:1fr 1fr; }
  .col3 { grid-template-columns:1fr 1fr 1fr; }
  @media(max-width:640px){ .col2,.col3 { grid-template-columns:1fr; } }
  .v-label { font-size:.78rem; font-weight:600; display:block; margin-bottom:.3rem; color:var(--ink); }
  .req::after { content:" *"; color:var(--err); }
  .hint { font-size:.72rem; color:var(--sub); margin-top:.25rem; }

  /* Overrides bootstrap form */
  .form-check-input:checked { background-color:var(--p); border-color:var(--p); }
  .form-check-label { font-size:.8rem; }

  /* ── Tabelas de dados ── */
  .data-sec { overflow-x:auto; }
  .data-sec table { width:100%; border-collapse:collapse; font-size:.78rem; }
  .data-sec thead th {
    padding:.45rem .5rem; background:#f9fafb;
    border-bottom:1px solid var(--bdr); font-weight:600;
    font-size:.7rem; color:var(--sub); white-space:nowrap; text-align:left;
  }
  .data-sec tbody td { padding:.35rem .4rem; border-bottom:1px solid var(--bdr); }
  .data-sec tbody tr:last-child td { border-bottom:none; }
  .data-sec input { width:100%; padding:.3rem .45rem; border:1px solid var(--bdr); border-radius:6px; font-size:.75rem; outline:none; }
  .data-sec input:focus { border-color:var(--p); box-shadow:0 0 0 2px rgba(107,9,166,.08); }
  .cap-badge { font-size:.72rem; color:var(--sub); }
  .del-btn {
    background:none; border:1px solid var(--bdr); border-radius:6px;
    padding:.25rem .45rem; cursor:pointer; color:var(--sub); font-size:.7rem;
  }
  .del-btn:hover { border-color:#fecaca; color:#dc2626; background:#fef2f2; }

  /* ── Action bar ── */
  .action-bar {
    position:fixed; bottom:0; left:0; right:0; z-index:90;
    background:#fff; border-top:1px solid var(--bdr);
    display:flex; align-items:center; justify-content:flex-end;
    gap:.5rem; padding:.75rem 1.5rem;
    box-shadow:0 -4px 20px rgba(0,0,0,.06);
  }

  /* ── Painel Siprouter ── */
  #sipPanel table td, #sipPanel table th { white-space:nowrap; }

  /* ── Eng grid ── */
  .eng-grid { display:grid; grid-template-columns:repeat(3,1fr); gap:.75rem; }
  @media(max-width:900px){ .eng-grid { grid-template-columns:1fr 1fr; } }
  @media(max-width:560px){ .eng-grid { grid-template-columns:1fr; } }
  .eng-card { border:1px solid var(--bdr); border-radius:8px; overflow:hidden; }
  .eng-card-title {
    font-size:.75rem; font-weight:700; padding:.45rem .65rem;
    background:var(--p); color:#fff; display:flex; align-items:center; min-height:38px;
  }
  .eng-card-body { padding:.5rem .6rem; }
  .eng-row { display:grid; grid-template-columns:24px 1fr; gap:.2rem .5rem; margin-bottom:.3rem; align-items:start; }
  .eng-row input[type="checkbox"] { margin-top:.25rem; width:13px; height:13px; accent-color:var(--p); }
  .eng-row p { margin:0; font-size:.72rem; line-height:1.35; }
  .eng-card.invalid { border-color:#dc2626; }
  .eng-card.invalid .eng-card-title { background:#dc2626; }

  /* ── Readonly overlay (seções 1–8 bloqueadas para Engenharia) ── */
  .ro input:not([type="hidden"]):not([type="checkbox"]),
  .ro select, .ro textarea { background:#f9fafb !important; pointer-events:none; }

  /* Exceções: áreas editáveis pelo Engenheiro */
  #tableVivo input,
  #tableVivo select,
  #tableVivo textarea,
  #eng-section input[type="checkbox"],
  #eng_notes {
    background: var(--surf) !important;
    pointer-events: auto !important;
    cursor: text;
  }
  #eng-section input[type="checkbox"] { cursor: pointer; }

  @media (prefers-reduced-motion:reduce){ * { transition:none !important; } }
</style>
{% endblock %}

{% block content %}
{% set plain_bg = true %}

<datalist id="cnList">
  {% for cn in CN_FULL %}
    <option value="{{ cn.codigo }}">{{ cn.codigo }} — {{ cn.nome }}{% if cn.uf %}/{{ cn.uf }}{% endif %}</option>
  {% endfor %}
</datalist>

<!-- Nav do formulário -->
<div class="form-nav">
  <div style="display:flex;align-items:center;gap:.75rem">
    {% if engineer_mode %}
      <a href="{{ url_for('engenharia.form_list') }}" class="btn-g btn-sm">← Voltar</a>
      <span style="font-size:.82rem;font-weight:600">Pré-PTI</span>
      <span style="font-size:.72rem;padding:.15rem .45rem;border-radius:5px;background:var(--p);color:#fff;font-weight:700">Engenharia</span>
    {% else %}
      <a href="{{ url_for('atacado.form_list') }}" class="btn-g btn-sm">← Voltar</a>
      <span style="font-size:.82rem;font-weight:600">Pré-PTI</span>
    {% endif %}
    {% if is_edit %}
      <span class="badge-s {% if st=='aprovado' %}done{% elif st=='em revisão' %}review{% elif st=='enviado' %}sent{% else %}draft{% endif %}">{{ st|capitalize }}</span>
      <span style="font-size:.72rem;color:var(--sub)">#{{ form.id }}</span>
    {% endif %}
  </div>
  <div style="display:flex;gap:.4rem">
    {% if is_edit %}
      <a href="{{ url_for('engenharia.exportar_excel', form_id=form.id) }}" class="btn-o btn-sm">
        <i class="bi bi-file-earmark-spreadsheet"></i> Excel
      </a>
    {% endif %}
    <a href="{{ url_for('auth.logout') }}" class="btn-g btn-sm">Sair</a>
  </div>
</div>

<div class="form-main">
  <form
    id="formTecnico" method="post"
    action="{% if form and not engineer_mode %}{{ url_for('atacado.form_update', form_id=form.id) }}{% elif form and engineer_mode %}{{ url_for('engenharia.form_view', form_id=form.id) }}{% else %}{{ url_for('atacado.form_create') }}{% endif %}"
    novalidate
    data-form-id="{{ form.id if form else 'new' }}"
    data-vivo-json="{{ form.dados_vivo_json if form else '[]' }}"
    data-operadora-json="{{ form.dados_operadora_json if form else '[]' }}"
    data-engenharia-json="{{ form.engenharia_params_json if form else '{}' }}"
    class="{{ 'ro' if engineer_mode else '' }}"
  >
    <input type="hidden" id="dados_vivo_json"       name="dados_vivo_json"       value="{{ form.dados_vivo_json if form and form.dados_vivo_json else '[]' }}">
    <input type="hidden" id="dados_operadora_json"  name="dados_operadora_json"  value="{{ form.dados_operadora_json if form and form.dados_operadora_json else '[]' }}">
    <input type="hidden" id="engenharia_params_json" name="engenharia_params_json" value="{{ form.engenharia_params_json if form and form.engenharia_params_json else '{}' }}">
    {% if not engineer_mode %}
      <input type="hidden" id="status"            name="status"            value="{{ (form.status if form else '') or 'rascunho' }}">
      <input type="hidden" id="escopo_flags_json" name="escopo_flags_json" value="{{ form.escopo_flags_json if form and form.escopo_flags_json else '[]' }}">
    {% endif %}

    {% if is_edit %}
    <p style="font-size:.72rem;color:var(--sub);margin:0 0 1rem;text-align:right">
      Atualizado: {{ (form.updated_at or form.created_at or '')|date_br }}
    </p>
    {% endif %}

    <!-- 1 | Identificação -->
    <div class="sec">
      <div class="sec-header"><h2 class="sec-title">1 · Identificação da Operadora</h2></div>
      <div class="sec-body">
        <div class="field-grid col2">
          <div>
            <label class="v-label req" for="nome_operadora">Nome da Operadora</label>
            <input class="v-input" type="text" id="nome_operadora" name="nome_operadora"
                   value="{{ form.nome_operadora if form else '' }}" required placeholder="Nome completo">
          </div>
          <div>
            <label class="v-label" for="rn1">RN1</label>
            <input class="v-input" type="text" id="rn1" name="rn1"
                   value="{{ form.rn1 if form else '' }}" placeholder="Ex.: RN1-123">
          </div>
          <div>
            <label class="v-label" for="atendimento">Atendimento</label>
            {% set atend = (form.atendimento if form else '') %}
            <select id="atendimento" name="atendimento" class="v-input">
              <option value="" {{ 'selected' if not atend }}>—</option>
              <option value="UF"      {{ 'selected' if atend=='UF' }}>UF</option>
              <option value="CN"      {{ 'selected' if atend=='CN' }}>CN</option>
              <option value="REG I"   {{ 'selected' if atend=='REG I' }}>REG I</option>
              <option value="REG II"  {{ 'selected' if atend=='REG II' }}>REG II</option>
              <option value="REG III" {{ 'selected' if atend=='REG III' }}>REG III</option>
            </select>
          </div>
          <div id="qualGroup" class="{{ '' if atend and atend.startswith('REG') else 'd-none' }}">
            <label class="v-label" for="qual">Qual?</label>
            <input class="v-input" type="text" id="qual" name="qual"
                   value="{{ form.qual if form else '' }}" placeholder="Ex.: Região II — Sul">
          </div>
          <div>
            <label class="v-label" for="redes">Redes</label>
            {% set redesv = (form.redes if form else '') %}
            <select id="redes" name="redes" class="v-input">
              <option value=""     {{ 'selected' if not redesv }}>—</option>
              <option value="STFC" {{ 'selected' if redesv=='STFC' }}>STFC</option>
              <option value="SMP"  {{ 'selected' if redesv=='SMP' }}>SMP</option>
            </select>
          </div>
          <div>
            <label class="v-label" for="tmr">TMR</label>
            <input class="v-input" type="text" id="tmr" name="tmr"
                   value="{{ form.tmr if form else '' }}" placeholder="Ex.: 15s">
          </div>
        </div>
        <div style="margin-top:1rem;display:flex;flex-wrap:wrap;gap:.75rem">
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="csp" name="csp" class="form-check-input" {% if form and form.csp %}checked{% endif %}> CSP
          </label>
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="servicos_especiais" name="servicos_especiais" class="form-check-input" {% if form and form.servicos_especiais %}checked{% endif %}> Serviços Especiais
          </label>
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="cng" name="cng" class="form-check-input" {% if form and form.cng %}checked{% endif %}> CNG
          </label>
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer;padding:.25rem .6rem;border:1px solid var(--bdr);border-radius:999px;transition:all .15s" id="lbl_scm">
            <input type="checkbox" id="scm" name="scm" class="form-check-input sip-flag" {% if form and form.scm %}checked{% endif %}> SCM
          </label>
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer;padding:.25rem .6rem;border:1px solid var(--bdr);border-radius:999px;transition:all .15s" id="lbl_av">
            <input type="checkbox" id="av" name="av" class="form-check-input sip-flag" {% if form and form.av %}checked{% endif %}> AV
          </label>
        </div>
      </div>
    </div>

    <!-- 2 | Contatos -->
    <div class="sec">
      <div class="sec-header"><h2 class="sec-title">2 · Contatos e Responsáveis</h2></div>
      <div class="sec-body">
        <div class="field-grid col2">
          <div>
            <label class="v-label req" for="responsavel_operadora">Responsável Operadora</label>
            <input class="v-input" type="text" id="responsavel_operadora" name="responsavel_operadora"
                   value="{{ form.responsavel_operadora if form else '' }}" required>
          </div>
          <div>
            <label class="v-label req" for="responsavel_vivo">Responsável Vivo (Atacado)</label>
            <input class="v-input" type="text" id="responsavel_vivo" name="responsavel_vivo"
                   value="{{ form.responsavel_vivo if form else '' }}" required>
          </div>
          <div>
            <label class="v-label" for="asn">ASN</label>
            <input class="v-input" type="text" id="asn" name="asn"
                   value="{{ form.asn if form else '' }}" placeholder="Ex.: ASXXXXX">
          </div>
          <div>
            <label class="v-label req" for="responsavel_itx_gestao">Gestão ITX (Atacado)</label>
            <input class="v-input" type="text" id="responsavel_itx_gestao" name="responsavel_atacado"
                   value="{{ form.responsavel_atacado if form else preset_responsavel_atacado or '' }}" required>
          </div>
          <div>
            <label class="v-label" for="responsavel_itx_eng">Eng. de ITX (Engenharia)</label>
            <input class="v-input" type="text" id="responsavel_itx_eng" name="responsavel_engenharia"
                   value="{{ form.responsavel_engenharia if form else '' }}"
                   placeholder="Preenchido pela Engenharia">
          </div>
        </div>
        <div style="margin-top:1rem;display:flex;flex-wrap:wrap;gap:.75rem">
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="sbc_ativo"    name="sbc_ativo"    class="form-check-input" {% if form and form.sbc_ativo %}checked{% endif %}> SBC ativo
          </label>
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="ip_reservado" name="ip_reservado" class="form-check-input" {% if form and form.ip_reservado %}checked{% endif %}> IP reservado
          </label>
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="vivo_reserva" name="vivo_reserva" class="form-check-input" {% if form and form.vivo_reserva %}checked{% endif %}> Vivo reserva IP
          </label>
        </div>
      </div>
    </div>

    <!-- 3 | Escopo (Atacado) ou Dados VIVO (Engenharia) -->
    {% if not engineer_mode %}
    <div class="sec">
      <div class="sec-header"><h2 class="sec-title">3 · Escopo</h2></div>
      <div class="sec-body">
        <label class="v-label req" for="escopo_texto">Descrição do tráfego</label>
        <textarea class="v-input" id="escopo_texto" name="escopo_text" rows="3" required
                  placeholder="Ex.: LC nos CNs 31/11; LD15 + CNG nacional…">{{ form.escopo_text if form else '' }}</textarea>
        <div style="margin-top:.9rem">
          <p style="font-size:.75rem;font-weight:600;margin:0 0 .5rem">Tipos de tráfego</p>
          <div style="display:flex;flex-wrap:wrap;gap:.6rem">
            {% for val, lbl in [('LC','LC'),('LD15 + CNG','LD15 + CNG'),('LDS/CSP + CNG','LDS/CSP + CNG'),('Transporte','Transporte'),('VC1','VC1'),('Concentração','Concentração')] %}
            <label style="display:flex;align-items:center;gap:.35rem;font-size:.8rem;cursor:pointer;padding:.3rem .6rem;border:1px solid var(--bdr);border-radius:999px;transition:all .15s"
                   class="esc-chip-label">
              <input type="checkbox" class="form-check-input esc-flag" value="{{ val }}" data-flag="{{ val }}" style="margin:0"> {{ lbl }}
            </label>
            {% endfor %}
          </div>
        </div>
      </div>
    </div>
    {% else %}
    <!-- Dados VIVO (somente Engenharia) -->
    <div class="sec">
      <div class="sec-header">
        <h2 class="sec-title">3 · Dados VIVO</h2>
        <div style="display:flex;align-items:center;gap:.5rem">
          <span class="cap-badge"><span id="vivoCount">0</span>/10</span>
          <button type="button" class="btn-g btn-sm" id="addRowVivo"><i class="bi bi-plus"></i></button>
          <button type="button" class="btn-g btn-sm" id="clearRowsVivo" style="font-size:.7rem">Limpar</button>
        </div>
      </div>
      <div class="sec-body" style="padding:.75rem">
        <div class="data-sec">
          <table id="tableVivo">
            <thead><tr>
              <th>Ref</th><th>Data</th><th>Escopo</th><th>Localidade</th>
              <th>CN</th><th>CDSIP_SPO_PL / CDSIP_SPO_JG</th><th>Mask</th><th>Endereço Link</th>
              <th>Cidade</th><th>UF</th><th>Lat.</th><th>Long.</th><th></th>
            </tr></thead>
            <tbody></tbody>
          </table>
        </div>

        <!-- Painel Siprouter -->
        <div id="sipPanel" style="display:none;margin-top:.5rem">
          <div id="sipLoading" style="display:none;font-size:.78rem;color:var(--sub);text-align:center;padding:.5rem">Consultando siprouter…</div>
          <div id="sipResult"></div>
          <div id="sipEmpty"   style="display:none;font-size:.78rem;color:var(--sub);text-align:center;padding:.5rem"></div>
          <span id="sipPanelMeta" style="display:none"></span>
        </div>
      </div>
    </div>
    {% endif %}

    <!-- 4 | Dados Operadora -->
    <div class="sec">
      <div class="sec-header">
        <h2 class="sec-title">4 · Dados Operadora</h2>
        {% if not engineer_mode %}
        <div style="display:flex;align-items:center;gap:.5rem">
          <span class="cap-badge"><span id="opCount">0</span>/10</span>
          <button type="button" class="btn-g btn-sm" id="addRowOperadora"><i class="bi bi-plus"></i></button>
          <button type="button" class="btn-g btn-sm" id="clearRowsOperadora" style="font-size:.7rem">Limpar</button>
        </div>
        {% endif %}
      </div>
      <div class="sec-body" style="padding:.75rem">
        <div class="data-sec">
          <table id="tableOperadora">
            <thead><tr>
              <th>Ref</th><th>Localidade</th><th>ETO LC</th><th>EOT LD</th>
              <th>CN</th><th>SBC</th><th>Faixa IP</th><th>Conc.?</th>
              <th>Endereço Link</th><th>Cidade</th><th>UF</th><th>Lat.</th><th>Long.</th>
              {% if not engineer_mode %}<th></th>{% endif %}
            </tr></thead>
            <tbody></tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- 5 | Infraestrutura -->
    <div class="sec">
      <div class="sec-header"><h2 class="sec-title">5 · Infraestrutura</h2></div>
      <div class="sec-body">
        <div class="field-grid col2">
          <div>
            <label style="display:flex;align-items:center;gap:.5rem;font-size:.8rem;cursor:pointer">
              <input type="checkbox" id="operadora_ciente" name="operadora_ciente" class="form-check-input"
                     {% if form and form.operadora_ciente %}checked{% endif %}>
              Operadora ciente da responsabilidade de infraestrutura?
            </label>
          </div>
          <div>
            <label class="v-label" for="responsavel_infra">Responsável</label>
            <input class="v-input" type="text" id="responsavel_infra" name="responsavel_infra"
                   value="{{ form.responsavel_infra if form else '' }}">
          </div>
        </div>
      </div>
    </div>

    <!-- 6 | LCR Nacional -->
    <div class="sec">
      <div class="sec-header"><h2 class="sec-title">6 · LCR Nacional</h2></div>
      <div class="sec-body">
        <div style="display:flex;flex-wrap:wrap;gap:1rem">
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="lcr_nacional" name="lcr_nacional" class="form-check-input"
                   {% if form and form.lcr_nacional %}checked{% endif %}> LCR Nacional
          </label>
          <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
            <input type="checkbox" id="white_list" name="white_list" class="form-check-input"
                   {% if form and form.white_list %}checked{% endif %}> White List
          </label>
        </div>
      </div>
    </div>

    <!-- 7 | Faixa de Numeração -->
    <div class="sec">
      <div class="sec-header"><h2 class="sec-title">7 · Faixa de Numeração</h2></div>
      <div class="sec-body">
        <label style="display:flex;align-items:center;gap:.4rem;font-size:.8rem;cursor:pointer">
          <input type="checkbox" id="prefixos_liberados_abr" name="prefixos_liberados_abr" class="form-check-input"
                 {% if form and form.prefixos_liberados_abr %}checked{% endif %}> Prefixos liberados na ABR
        </label>
      </div>
    </div>

    <!-- 8 | Premissas -->
    <div class="sec">
      <div class="sec-header"><h2 class="sec-title">8 · Premissas de Abordagem</h2></div>
      <div class="sec-body">
        <div class="field-grid col2" style="margin-bottom:1rem">
          <div style="grid-column:1/-1">
            <label style="display:flex;align-items:flex-start;gap:.5rem;font-size:.8rem;cursor:pointer">
              <input type="checkbox" id="premissas_ok" name="premissas_ok" class="form-check-input" style="margin-top:.15rem"
                     {% if form and form.premissas_ok %}checked{% endif %}>
              <span>Aprovação em acordo com as informações apresentadas e às 29 Pendências de Programação.</span>
            </label>
          </div>
          <div>
            <label class="v-label" for="aprovado_por">Aprovado por</label>
            <input class="v-input" type="text" id="aprovado_por" name="aprovado_por"
                   value="{{ form.aprovado_por if form else '' }}">
          </div>
        </div>
        <div style="overflow-x:auto">
          <table style="width:100%;border-collapse:collapse;font-size:.78rem">
            <thead><tr style="background:#f9fafb;border-bottom:1px solid var(--bdr)">
              <th style="padding:.45rem .75rem;font-weight:600;font-size:.7rem;color:var(--sub)">Tráfego</th>
              <th style="padding:.45rem .75rem;font-weight:600;font-size:.7rem;color:var(--sub)">Descrição</th>
            </tr></thead>
            <tbody>
              <tr style="border-bottom:1px solid var(--bdr)">
                <td style="padding:.4rem .75rem;font-weight:600;white-space:nowrap">LC / TR LC</td>
                <td style="padding:.4rem .75rem;color:var(--sub)">Chamadas entre operadoras dentro da mesma Área Local.</td>
              </tr>
              <tr style="border-bottom:1px solid var(--bdr)">
                <td style="padding:.4rem .75rem;font-weight:600;white-space:nowrap">LD 15 / CNG VIVO</td>
                <td style="padding:.4rem .75rem;color:var(--sub)">Encaminhamentos entre Vivo-STFC e operadora (LD15, SE e CNG).</td>
              </tr>
              <tr style="border-bottom:1px solid var(--bdr)">
                <td style="padding:.4rem .75rem;font-weight:600;white-space:nowrap">LD Oper CSP XY</td>
                <td style="padding:.4rem .75rem;color:var(--sub)">Tráfego LD com/sem CSP e CNG conforme acordado.</td>
              </tr>
              <tr style="border-bottom:1px solid var(--bdr)">
                <td style="padding:.4rem .75rem;font-weight:600;white-space:nowrap">Transporte</td>
                <td style="padding:.4rem .75rem;color:var(--sub)">Chamadas entregues pela operadora de qualquer CN para qualquer destino.</td>
              </tr>
              <tr>
                <td style="padding:.4rem .75rem;font-weight:600;white-space:nowrap">VC1</td>
                <td style="padding:.4rem .75rem;color:var(--sub)">Chamadas entre AL e VIVO-SMP no CN.</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- 9 | Parâmetros Técnicos (somente Engenharia) -->
    {% if engineer_mode %}
    <div class="sec" id="eng-section">
      <div class="sec-header">
        <h2 class="sec-title">9 · Parâmetros Técnicos — Engenharia</h2>
        <div style="display:flex;gap:.4rem">
          <button type="button" id="engCheckAll"   class="btn-g btn-sm"><i class="bi bi-check2-square"></i> Marcar tudo</button>
          <button type="button" id="engUncheckAll" class="btn-g btn-sm"><i class="bi bi-x-square"></i> Limpar</button>
        </div>
      </div>
      <div class="sec-body">
        <div class="eng-grid">

          <div class="eng-card" data-eng-group="tratamento">
            <div class="eng-card-title">Tratamento de Chamadas</div>
            <div class="eng-card-body">
              {% for key, text in [
                ("tratamento.reinvite_obrigatorio", "Re-invites obrigatórios pela Origem para definição de codecs."),
                ("tratamento.ptime_maxptime_20ms",  "Ptime/Maxptime: múltiplos de 20 ms para codecs móveis (3GPP 26.114)."),
                ("tratamento.qos_marcacoes_rtp",    "RTP: marcação CS5 / DSCP 46 / EF.")
              ] %}<div class="eng-row"><input type="checkbox" class="eng-flag" data-key="{{ key }}"><p>{{ text }}</p></div>{% endfor %}
            </div>
          </div>

          <div class="eng-card" data-eng-group="codec">
            <div class="eng-card-title">CODEC / Modo</div>
            <div class="eng-card-body">
              {% for key, text in [
                ("codec.sipi_sem_rn3_portado",  "SIP-I sem RN3 (060) para tráfego portado."),
                ("codec.oferta_g711a_g729_stfc", "STFC/STFC e STFC/SMP: G.711A e G.729."),
                ("codec.amr_g711a_smp",          "SMP/SMP: AMR e G.711A."),
                ("codec.g711u_nao_suportado",    "G.711u não suportado.")
              ] %}<div class="eng-row"><input type="checkbox" class="eng-flag" data-key="{{ key }}"><p>{{ text }}</p></div>{% endfor %}
            </div>
          </div>

          <div class="eng-card" data-eng-group="id">
            <div class="eng-card-title">Identificação</div>
            <div class="eng-card-body">
              {% for key, text in [
                ("id.originador_fixo_isupbr_regiao3",         "Rede Fixa: ISUPBR (Região III)."),
                ("id.originador_movel_isupbr_itu92_opcional", "Rede Móvel: ISUPBR (ITU-92 opcional)."),
                ("id.fax_t38_nao_suportado",                  "FAX T.38 não suportado."),
                ("id.bgp_ip30_mpls_contingencia",             "BGP IP/30 com MPLS e link de contingência.")
              ] %}<div class="eng-row"><input type="checkbox" class="eng-flag" data-key="{{ key }}"><p>{{ text }}</p></div>{% endfor %}
            </div>
          </div>

          <div class="eng-card" data-eng-group="rfc">
            <div class="eng-card-title">RFC Principais</div>
            <div class="eng-card-body">
              {% for key, text in [
                ("rfc.3261_base",           "RFC 3261 — SIP base."),
                ("rfc.3262_100rel_prack",   "RFC 3262 — 100rel / PRACK."),
                ("rfc.sdp_3264_3266_2327_4566","RFC 3264/3266/2327/4566 — SDP."),
                ("rfc.4028_timers",         "RFC 4028 — SIP Timers.")
              ] %}<div class="eng-row"><input type="checkbox" class="eng-flag" data-key="{{ key }}"><p>{{ text }}</p></div>{% endfor %}
            </div>
          </div>

          <div class="eng-card" data-eng-group="habilitadas">
            <div class="eng-card-title">RFC Habilitadas</div>
            <div class="eng-card-body">
              {% for key, text in [
                ("habilitadas.3389_cn",   "RFC 3389 — Comfort Noise."),
                ("habilitadas.2833_dtmf", "RFC 2833 — DTMF."),
                ("habilitadas.3264_hold", "RFC 3264 — Hold.")
              ] %}<div class="eng-row"><input type="checkbox" class="eng-flag" data-key="{{ key }}"><p>{{ text }}</p></div>{% endfor %}
            </div>
          </div>

          <div class="eng-card" data-eng-group="early">
            <div class="eng-card-title">Early Media / RBT</div>
            <div class="eng-card-body">
              {% for key, text in [
                ("early.3960_early_media",       "RFC 3960 — Early Media e Ring Tone."),
                ("early.180ring_proxy_rbt_local", "SIP 180 Ring puro: Proxy gera RBT localmente."),
                ("early.6337_offer_answer",      "RFC 6337 — Offer-Answer.")
              ] %}<div class="eng-row"><input type="checkbox" class="eng-flag" data-key="{{ key }}"><p>{{ text }}</p></div>{% endfor %}
            </div>
          </div>

        </div>

        <div style="margin-top:1rem">
          <label class="v-label" for="eng_notes">Observações</label>
          <textarea class="v-input" id="eng_notes" rows="2" placeholder="Particularidades, exceções…">{{ (form.engenharia_params_json or '{}') }}</textarea>
        </div>
      </div>
    </div>
    {% endif %}

  </form>
</div>

<!-- Barra de ações -->
<div class="action-bar">
  {% if engineer_mode %}
    <a href="{{ url_for('engenharia.form_list') }}" class="btn-g">Cancelar</a>
    <button type="button" id="btnSaveEng" onclick="document.getElementById('formTecnico').requestSubmit()" class="btn-p">
      <span class="spinner-border spinner-border-sm d-none me-1" id="spin"></span> Salvar validação
    </button>
  {% else %}
    <button type="button" class="btn-g" id="resetForm">Limpar</button>
    <button type="button" id="btnSave" class="btn-o" onclick="setStatus('rascunho');document.getElementById('formTecnico').requestSubmit()">
      <span class="spinner-border spinner-border-sm d-none me-1" id="spinSave"></span> Salvar rascunho
    </button>
    <button type="button" id="btnFinish" class="btn-p" onclick="finalizeSubmit()">
      <span class="spinner-border spinner-border-sm d-none me-1" id="spinFinish"></span> Finalizar
    </button>
  {% endif %}
</div>

<!-- =====================================================================
     JAVASCRIPT
     ===================================================================== -->
<script data-engineer="{{ 'true' if engineer_mode else 'false' }}">
const ENG = document.currentScript.getAttribute('data-engineer') === 'true';
const CAP = 10;

/* ── CN map ── */
let CN_MAP = {};
fetch('/api/cns').then(r=>r.json()).then(data=>{
  data.forEach(({codigo,nome,uf})=>{ CN_MAP[String(codigo||'').padStart(2,'0')]={cidade:nome||'',uf:uf||''}; });
}).catch(()=>{});
const cn2meta = cn => CN_MAP[String(cn||'').trim().padStart(2,'0')] || {cidade:'',uf:''};

/* ── Estado ── */
let dirty=false, submitting=false;
const setDirty=()=>{ dirty=true; };

/* ── Helpers de tabela ── */
const mkInput=(ph,attrs={})=>{
  const i=document.createElement('input');
  i.placeholder=ph; i.style.cssText='width:100%;padding:.3rem .4rem;border:1px solid var(--bdr);border-radius:6px;font-size:.75rem;outline:none';
  Object.entries(attrs).forEach(([k,v])=>{ if(v!=null) i.setAttribute(k,v); });
  i.addEventListener('input',setDirty); i.addEventListener('change',setDirty);
  i.addEventListener('focus',()=>{ i.style.borderColor='var(--p)'; i.style.boxShadow='0 0 0 2px rgba(107,9,166,.08)'; });
  i.addEventListener('blur', ()=>{ i.style.borderColor='var(--bdr)'; i.style.boxShadow='none'; });
  return i;
};
const mkCheck=()=>{
  const w=document.createElement('div'); w.style='display:flex;justify-content:center;padding:.2rem 0';
  const c=document.createElement('input'); c.type='checkbox'; c.style='accent-color:var(--p);cursor:pointer';
  c.addEventListener('change',setDirty); w.appendChild(c); return w;
};
const mkDel=()=>{
  const b=document.createElement('button'); b.type='button'; b.className='del-btn'; b.innerHTML='<i class="bi bi-dash"></i>';
  b.addEventListener('click',e=>{ e.currentTarget.closest('tr').remove(); upd(); setDirty(); });
  return b;
};
const upd=()=>{
  const vc=document.getElementById('vivoCount'); if(vc) vc.textContent=document.querySelectorAll('#tableVivo tbody tr').length;
  const oc=document.getElementById('opCount');   if(oc) oc.textContent=document.querySelectorAll('#tableOperadora tbody tr').length;
};

/* ── Serialização ── */
function serVivo(){
  return Array.from(document.querySelectorAll('#tableVivo tbody tr')).map(tr=>{
    const c=tr.querySelectorAll('td');
    return {ref:c[0]?.querySelector('input')?.value||'',data:c[1]?.querySelector('input')?.value||'',escopo:c[2]?.querySelector('input')?.value||'',localidade:c[3]?.querySelector('input')?.value||'',cn:c[4]?.querySelector('input')?.value||'',bloco_ip:c[5]?.querySelector('input')?.value||'',mask:c[6]?.querySelector('input')?.value||'',endereco_link:c[7]?.querySelector('input')?.value||'',cidade:c[8]?.querySelector('input')?.value||'',uf:c[9]?.querySelector('input')?.value||'',lat:c[10]?.querySelector('input')?.value||'',long:c[11]?.querySelector('input')?.value||''};
  });
}
function serOp(){
  return Array.from(document.querySelectorAll('#tableOperadora tbody tr')).map(tr=>{
    const c=tr.querySelectorAll('td');
    return {ref:c[0]?.querySelector('input')?.value||'',localidade:c[1]?.querySelector('input')?.value||'',eto_lc:c[2]?.querySelector('input')?.value||'',eot_ld:c[3]?.querySelector('input')?.value||'',cn:c[4]?.querySelector('input')?.value||'',sbc:c[5]?.querySelector('input')?.value||'',faixa_ip:c[6]?.querySelector('input')?.value||'',concentracao:c[7]?.querySelector('input')?.checked||false,endereco_link:c[8]?.querySelector('input')?.value||'',cidade:c[9]?.querySelector('input')?.value||'',uf:c[10]?.querySelector('input')?.value||'',lat:c[11]?.querySelector('input')?.value||'',long:c[12]?.querySelector('input')?.value||''};
  });
}

/* ── Siprouter SP Query ── */
const SIP = (function(){
  const panel    = document.getElementById('sipPanel');
  const metaEl   = document.getElementById('sipPanelMeta');
  const loadEl   = document.getElementById('sipLoading');
  const resultEl = document.getElementById('sipResult');
  const emptyEl  = document.getElementById('sipEmpty');
  if(!panel) return { query:()=>{} };

  let _last='', _tmr=null;

  function getFlags(){
    const scm = document.getElementById('scm')?.checked ? '1' : '0';
    const av  = document.getElementById('av')?.checked  ? '1' : '0';
    return {scm, av};
  }

  function renderResult(data, rowIdx){
    loadEl.style.display='none'; emptyEl.style.display='none'; resultEl.innerHTML='';

    if(!data.found){
      emptyEl.style.display='block';
      emptyEl.textContent = data.mensagem || 'Nenhum bloco IP encontrado.';
      panel.style.display='block';
      return;
    }

    // Auto-preencher o campo — apenas isso, sem card extra
    const pl    = data.pl_bloco || '';
    const jg    = data.jg_bloco || '';
    const valor = pl && jg ? `PL:${pl} / JG:${jg}` : (pl || jg);
    fillBlocoIP(valor, rowIdx);

    // Painel discreto apenas com a mensagem de confirmação
    metaEl.textContent = `CN ${data.cn} · ${data.descricao || ''} · ${data.mensagem || ''}`;
    panel.style.display='block';
  }

  function fillBlocoIP(bloco, rowIdx){
    const rows=document.querySelectorAll('#tableVivo tbody tr');
    if(rowIdx<0||rowIdx>=rows.length) return;
    const inp=rows[rowIdx].querySelectorAll('td')[5]?.querySelector('input');
    if(inp){
      inp.value=bloco;
      inp.style.background='#d1fae5';
      setTimeout(()=>{ inp.style.background=''; },1200);
      setDirty();
    }
  }

  async function query(cn, rowIdx){
    const rn1 = (document.getElementById('rn1')?.value||'').trim();
    if(!rn1 || !cn || String(cn).replace(/\D/g,'').length < 2) return;

    const {scm, av} = getFlags();
    const ck = `${cn}:${rn1}:${scm}:${av}:${rowIdx}`;
    if(ck===_last) return;
    _last=ck;

    clearTimeout(_tmr);
    _tmr=setTimeout(async()=>{
      panel.style.display='block';
      loadEl.style.display='block';
      resultEl.innerHTML='';
      emptyEl.style.display='none';

      try{
        const r = await fetch(`/api/siprouter/query?cn=${encodeURIComponent(cn)}&rn1=${encodeURIComponent(rn1)}&scm=${scm}&av=${av}`);
        if(!r.ok) throw new Error(r.status);
        renderResult(await r.json(), rowIdx);
      }catch(e){
        loadEl.style.display='none';
        emptyEl.style.display='block';
        emptyEl.textContent='Erro ao consultar siprouter. Verifique CN e RN1.';
      }
    }, 320);
  }

  return { query };
})();

// Re-disparar consulta quando SCM/AV mudar
document.querySelectorAll('.sip-flag').forEach(chk=>{
  chk.addEventListener('change',()=>{
    // Invalidar cache forçando nova consulta
    const firstRow = document.querySelector('#tableVivo tbody tr');
    if(!firstRow) return;
    const cnI = firstRow.querySelector('[data-field="cn"]');
    if(cnI && cnI.value.length>=2){
      const ri = 0;
      SIP.query(cnI.value, ri);
    }
  });
  // Estilo visual do chip SCM/AV
  const lbl = chk.closest('label');
  const syncChip=()=>{
    if(!lbl) return;
    lbl.style.borderColor = chk.checked ? 'var(--p)' : 'var(--bdr)';
    lbl.style.background  = chk.checked ? 'var(--p-lt)' : 'transparent';
    lbl.style.color       = chk.checked ? 'var(--p)' : 'inherit';
  };
  chk.addEventListener('change', syncChip); syncChip();
});

/* ── Wire CN autofill ── */
function wireCN(tr){
  const cnI=tr.querySelector('[data-field="cn"]');
  const ufI=tr.querySelector('[data-field="uf"]');
  const ciI=tr.querySelector('[data-field="cidade"]');
  if(!cnI) return;
  let t1=null;
  const cnH=()=>{
    const v=(cnI.value||'').replace(/\D/g,'').slice(0,2); cnI.value=v;
    const m=cn2meta(v);
    if(ciI&&m.cidade) ciI.value=m.cidade;
    if(ufI&&m.uf) ufI.value=m.uf;
    setDirty();
    if(v.length===2){
      clearTimeout(t1);
      t1=setTimeout(()=>{
        const ri=Array.from(tr.parentElement.children).indexOf(tr);
        SIP.query(v, ri);
      },200);
    }
  };
  ['input','change','blur'].forEach(ev=>{ cnI.addEventListener(ev,cnH); });
  if(ufI) ['input','change','blur'].forEach(ev=>{ ufI.addEventListener('change',setDirty); });
  if((cnI.value||'').trim().length>=2) cnH();
}

// Re-disparar consulta quando RN1 do formulário mudar
(function(){
  const rn1I = document.getElementById('rn1');
  if(!rn1I) return;
  let t=null;
  rn1I.addEventListener('input',()=>{
    clearTimeout(t);
    t=setTimeout(()=>{
      const firstRow=document.querySelector('#tableVivo tbody tr');
      if(!firstRow) return;
      const cnI=firstRow.querySelector('[data-field="cn"]');
      if(cnI&&cnI.value.length>=2){
        SIP.query(cnI.value, 0);
      }
    },400);
  });
})();

/* ── Tabelas dinâmicas ── */
(function(){
  let vi=[],oi=[];
  const fe=document.getElementById('formTecnico');
  try{ vi=JSON.parse(fe.getAttribute('data-vivo-json')||'[]'); }catch(_){}
  try{ oi=JSON.parse(fe.getAttribute('data-operadora-json')||'[]'); }catch(_){}
  const tv=document.querySelector('#tableVivo tbody');
  const to=document.querySelector('#tableOperadora tbody');

  function addVivo(v={}){
    if(!tv) return;
    if(tv.children.length>=CAP){ alert('Máximo '+CAP+' linhas.'); return; }
    const tr=document.createElement('tr');
    [['ref',{}],['data',{}],['escopo',{}],['localidade',{}],
     ['cn',{'data-field':'cn',list:'cnList',inputmode:'numeric',maxlength:'2'}],
     ['sbc',{}],['mask',{}],['endereco_link',{}],
     ['cidade',{'data-field':'cidade'}],
     ['uf',{'data-field':'uf',maxlength:'2'}],
     ['lat',{}],['long',{}]
    ].forEach(([k,a])=>{ const td=document.createElement('td'); const i=mkInput(k,a); i.value=v[k]??''; td.appendChild(i); tr.appendChild(td); });
    const td=document.createElement('td'); td.appendChild(mkDel()); tr.appendChild(td);
    tv.appendChild(tr); wireCN(tr); upd(); setDirty();
  }

  function addOp(v={}){
    if(!to) return;
    if(to.children.length>=CAP){ alert('Máximo '+CAP+' linhas.'); return; }
    const tr=document.createElement('tr');
    [['ref',{}],['localidade',{}],['eto_lc',{}],['eot_ld',{}],
     ['cn',{'data-field':'cn',list:'cnList',inputmode:'numeric',maxlength:'2'}],
     ['sbc',{}],['faixa_ip',{}]
    ].forEach(([k,a])=>{ const td=document.createElement('td'); const i=mkInput(k,a); i.value=v[k]??''; td.appendChild(i); tr.appendChild(td); });
    const tdC=document.createElement('td'); const ck=mkCheck(); ck.querySelector('input').checked=!!v.concentracao; tdC.appendChild(ck); tr.appendChild(tdC);
    [['endereco_link',{}],['cidade',{'data-field':'cidade'}],['uf',{'data-field':'uf',maxlength:'2'}],['lat',{}],['long',{}]
    ].forEach(([k,a])=>{ const td=document.createElement('td'); const i=mkInput(k,a); i.value=v[k]??''; td.appendChild(i); tr.appendChild(td); });
    if(!ENG){ const td=document.createElement('td'); td.appendChild(mkDel()); tr.appendChild(td); }
    to.appendChild(tr); wireCN(tr); upd(); setDirty();
  }

  (Array.isArray(vi)?vi:[]).forEach(addVivo);
  (Array.isArray(oi)?oi:[]).forEach(addOp);
  document.getElementById('addRowVivo')?.addEventListener('click',()=>addVivo({}));
  document.getElementById('clearRowsVivo')?.addEventListener('click',()=>{ if(tv){ tv.innerHTML=''; upd(); setDirty(); } });
  document.getElementById('addRowOperadora')?.addEventListener('click',()=>addOp({}));
  document.getElementById('clearRowsOperadora')?.addEventListener('click',()=>{ if(to){ to.innerHTML=''; upd(); setDirty(); } });
  upd();
})();

/* ── Engenharia hydrate ── */
(function(){
  if(!ENG) return;
  let s={};
  try{ s=JSON.parse(document.getElementById('formTecnico').getAttribute('data-engenharia-json')||'{}'); }catch(_){}
  document.querySelectorAll('#eng-section .eng-flag[data-key]').forEach(c=>{ const k=c.getAttribute('data-key'); if(k in s) c.checked=!!s[k]; });
  if(typeof s.notes==='string'){ const n=document.getElementById('eng_notes'); if(n) n.value=s.notes; }
  document.getElementById('engCheckAll')?.addEventListener('click',()=>{ document.querySelectorAll('#eng-section .eng-flag').forEach(c=>c.checked=true); document.querySelectorAll('.eng-card').forEach(c=>c.classList.remove('invalid')); setDirty(); });
  document.getElementById('engUncheckAll')?.addEventListener('click',()=>{ document.querySelectorAll('#eng-section .eng-flag').forEach(c=>c.checked=false); setDirty(); });
})();

/* ── Qual? toggle ── */
(function(){
  const at=document.getElementById('atendimento'), qg=document.getElementById('qualGroup');
  const toggle=()=>{ if(qg) qg.classList.toggle('d-none',!(at?.value||'').startsWith('REG')); };
  at?.addEventListener('change',toggle); toggle();
})();

/* ── Escopo chips ── */
(function(){
  if(ENG) return;
  const h=document.getElementById('escopo_flags_json'); if(!h) return;
  let flags=[]; try{ flags=JSON.parse(h.value||'[]'); }catch(_){}
  document.querySelectorAll('.esc-flag').forEach(c=>{
    if(flags.includes(c.getAttribute('data-flag'))) c.checked=true;
    c.addEventListener('change',()=>{
      const cur=[]; document.querySelectorAll('.esc-flag:checked').forEach(x=>cur.push(x.getAttribute('data-flag')));
      h.value=JSON.stringify(cur); setDirty();
    });
    // Visual feedback
    const lbl=c.closest('label');
    const sync=()=>{ if(lbl) lbl.style.cssText='display:flex;align-items:center;gap:.35rem;font-size:.8rem;cursor:pointer;padding:.3rem .6rem;border:1px solid '+(c.checked?'var(--p)':' var(--bdr)')+';border-radius:999px;transition:all .15s;background:'+(c.checked?'var(--p-lt)':'transparent')+';color:'+(c.checked?'var(--p)':'inherit'); };
    c.addEventListener('change',sync); sync();
  });
})();

/* ── Submit ── */
const form=document.getElementById('formTecnico');
function setStatus(v){ const s=document.getElementById('status'); if(s) s.value=v; }
function finalizeSubmit(){
  if(!confirm('Finalizar e enviar para Engenharia?')) return;
  setStatus('enviado');
  document.getElementById('formTecnico').requestSubmit();
}

window.addEventListener('beforeunload',e=>{ if(dirty&&!submitting){ e.preventDefault(); e.returnValue=''; } });

document.getElementById('resetForm')?.addEventListener('click',()=>{
  form.reset();
  const tv=document.querySelector('#tableVivo tbody'); if(tv) tv.innerHTML='';
  const to=document.querySelector('#tableOperadora tbody'); if(to) to.innerHTML='';
  document.getElementById('qualGroup')?.classList.add('d-none');
  upd(); setDirty(); window.scrollTo({top:0,behavior:'smooth'});
});

document.querySelectorAll('#formTecnico input:not([type="hidden"]),#formTecnico select,#formTecnico textarea').forEach(el=>{
  el.addEventListener('input',setDirty); el.addEventListener('change',setDirty);
});

form.addEventListener('submit',e=>{
  // Serializar tabelas
  const vh=document.getElementById('dados_vivo_json');
  const oh=document.getElementById('dados_operadora_json');
  if(ENG&&document.getElementById('tableVivo'))       vh.value=JSON.stringify(serVivo());
  if(!ENG&&document.getElementById('tableOperadora')) oh.value=JSON.stringify(serOp());

  if(ENG){
    const d={};
    document.querySelectorAll('#eng-section .eng-flag[data-key]').forEach(c=>{ d[c.getAttribute('data-key')]=!!c.checked; });
    const n=document.getElementById('eng_notes'); if(n&&n.value.trim()) d.notes=n.value.trim();
    document.getElementById('engenharia_params_json').value=JSON.stringify(d);
    // Validar grupos
    let ok=true;
    document.querySelectorAll('#eng-section .eng-card[data-eng-group]').forEach(card=>{
      card.classList.remove('invalid');
      if(!card.querySelectorAll('.eng-flag:checked').length){ card.classList.add('invalid'); ok=false; }
    });
    if(!ok){
      e.preventDefault(); e.stopPropagation();
      alert('Preencha todos os grupos de parâmetros da Seção 9.');
      document.querySelector('.eng-card.invalid')?.scrollIntoView({behavior:'smooth',block:'center'});
      return;
    }
  }

  if(!form.checkValidity()){
    e.preventDefault(); e.stopPropagation();
    const inv=form.querySelector(':invalid');
    inv?.scrollIntoView({behavior:'smooth',block:'center'}); inv?.focus({preventScroll:true});
    return;
  }
  submitting=true;
  // Spinner
  if(ENG){ const s=document.getElementById('spin'); if(s) s.classList.remove('d-none'); }
  else if(document.getElementById('btnSave')?.matches(':active')){ const s=document.getElementById('spinSave'); if(s) s.classList.remove('d-none'); }
  else { const s=document.getElementById('spinFinish'); if(s) s.classList.remove('d-none'); }
});

/* ── Hotkeys ── */
document.addEventListener('keydown',e=>{
  if(['INPUT','TEXTAREA','SELECT'].includes(e.target?.tagName)){
    if((e.ctrlKey||e.metaKey)&&e.key.toLowerCase()==='s'){
      e.preventDefault();
      if(ENG) document.getElementById('btnSaveEng')?.click();
      else{ setStatus('rascunho'); document.getElementById('btnSave')?.click(); }
    }
    return;
  }
  if(e.key==='g'||e.key==='G') window.scrollTo({top:0,behavior:'smooth'});
});
</script>
{% endblock %}
