<!DOCTYPE html>
<html lang="pt-BR" data-bs-theme="light">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Controle de Rotas Cadastradas</title>

  <!-- CSRF injetado pelo Flask -->
  <meta name="csrf-token" content="{{ csrf_token }}"/>

  <!-- Bootstrap CSS -->
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" />

  <!-- Estilos (tema Vivo + tabelas sticky + ajustes gerais) -->
  <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}" />

  <!-- Chart.js -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body class="bg-body-tertiary">

  <!-- Navbar -->
  <nav class="navbar navbar-expand-lg sticky-top">
    <div class="container-fluid">
      <a class="navbar-brand fw-semibold" href="#">Controle de Rotas Cadastradas</a>
      <div class="d-flex gap-2">
        <button id="alternar-tema" class="btn btn-outline-secondary" type="button" aria-pressed="false" title="Alternar tema">🌙</button>
      </div>
    </div>
  </nav>

  <div class="container-fluid py-3">
    <div class="row g-3">
      <!-- Sidebar / Dashboard -->
      <aside class="col-12 col-lg-3 col-xxl-2">
        <div class="card shadow-sm">
          <div class="card-body">
            <h6 class="card-title mb-3">Resumo</h6>
            <div class="d-flex justify-content-between mb-1">
              <span class="text-secondary">STFC:</span>
              <span id="contagem-stfc" class="fw-semibold">0</span>
            </div>
            <div class="d-flex justify-content-between">
              <span class="text-secondary">SMP:</span>
              <span id="contagem-smp" class="fw-semibold">0</span>
            </div>
            <div class="chart-wrap mt-3" style="height:220px;">
              <canvas id="grafico-rede" aria-label="Distribuição de Tráfego" role="img"></canvas>
            </div>
          </div>
        </div>
      </aside>

      <!-- Main -->
      <main class="col-12 col-lg-9 col-xxl-10">
        <!-- Toolbar de filtros e ações -->
        <div class="card shadow-sm mb-3">
          <div class="card-body">
            <div class="row g-3 align-items-end">
              <!-- Busca geral -->
              <div class="col-12 col-xl-4">
                <label for="campo-pesquisa" class="form-label mb-1">Busca geral</label>
                <input id="campo-pesquisa" class="form-control" placeholder="Operadora, EOT, CN, descrição..." autocomplete="off" />
              </div>

              <!-- Tráfego -->
              <div class="col-12 col-sm-4 col-xl-2">
                <span class="form-label mb-1 d-block">Tipo de tráfego</span>
                <div class="d-flex align-items-center gap-3">
                  <div class="form-check">
                    <input class="form-check-input" type="checkbox" id="filtro-stfc" checked>
                    <label class="form-check-label" for="filtro-stfc">STFC</label>
                  </div>
                  <div class="form-check">
                    <input class="form-check-input" type="checkbox" id="filtro-smp" checked>
                    <label class="form-check-label" for="filtro-smp">SMP</label>
                  </div>
                </div>
              </div>

              <!-- Período (datas globais) + Base onde aplicar -->
              <div class="col-12 col-sm-8 col-xl-6">
                <span class="form-label mb-1 d-block">Período (Data_Desativacao) &nbsp;•&nbsp; aplicar datas em:</span>
                <div class="row g-2">
                  <div class="col-12 col-md-4">
                    <input id="data-inicio" type="date" class="form-control" placeholder="Início (YYYY-MM-DD)">
                  </div>
                  <div class="col-12 col-md-4">
                    <input id="data-fim" type="date" class="form-control" placeholder="Fim (YYYY-MM-DD)">
                  </div>
                  <div class="col-12 col-md-4">
                    <select id="base-data-aplicacao" class="form-select">
                      <option value="">Todas as bases</option>
                      <option value="science">Somente Science</option>
                    </select>
                  </div>
                </div>
                <small class="text-secondary d-block mt-1">
                  Ex.: selecione um intervalo e "Somente Science" para que o filtro atinja apenas os dados de Science; os dados do Portal permanecem como estão.
                </small>
              </div>

              <!-- Ações -->
              <div class="col-12">
                <div class="d-flex flex-wrap gap-2">
                  <button id="botao-aplicar-filtros" class="btn btn-primary" title="Aplicar filtros e atualizar a tabela">
                    Aplicar filtros
                  </button>
                  <button id="botao-limpar-filtros" class="btn btn-outline-primary" title="Limpar todos os filtros">
                    Limpar
                  </button>

                  <button id="abrir-modal-upload" class="btn btn-primary ms-auto" title="Carregar novas bases CSV">
                    Carregar bases
                  </button>

                  <!-- Dropdown Exportar -->
                  <div class="dropdown">
                    <button class="btn btn-primary dropdown-toggle" type="button"
                            data-bs-toggle="dropdown" data-bs-display="static" aria-expanded="false">
                      Exportar
                    </button>
                    <ul class="dropdown-menu dropdown-menu-end shadow">
                      <li>
                        <button class="dropdown-item" id="abrir-modal-exportar" type="button">
                          Exportar filtrado (XLSX/CSV)
                        </button>
                      </li>
                      <li><hr class="dropdown-divider"></li>
                      <li>
                        <button class="dropdown-item" id="botao-baixar-tudo" type="button">
                          Exportar tudo (CSV)
                        </button>
                      </li>
                    </ul>
                  </div>
                </div>
                <small class="text-secondary d-block mt-1">
                  Dica: além do período global, você pode usar os filtros por coluna na tabela para refinar ainda mais.
                </small>
              </div>
            </div>
          </div>
        </div>

        <!-- Mensagem inicial -->
        <div id="mensagem-inicial" class="alert alert-secondary">
          Informe os critérios e clique em <strong>Aplicar filtros</strong>.
        </div>

        <!-- Tabela -->
        <div class="card shadow-sm" id="card-tabela" hidden>
          <div class="table-responsive">
            <table id="tabela" class="table table-hover table-sm align-middle">
              <thead class="table-light">
                <tr id="cabecalho"></tr>
                <tr id="filtros"></tr>
              </thead>
              <tbody id="corpo-tabela"></tbody>
            </table>
          </div>

          <!-- Rodapé com total + paginação embaixo -->
          <div class="card-footer d-flex flex-wrap gap-2 justify-content-between align-items-center">
            <div>
              <strong>Resultados:</strong> <span id="total-resultados">0</span>
            </div>
            <nav id="paginacao" aria-label="Paginação"></nav>
          </div>
        </div>

        <!-- Agente de IA em largura total (abaixo da tabela) -->
        <div class="card shadow-sm mt-3">
          <div class="card-body d-flex flex-column">
            <h6 class="card-title mb-3">Agente de IA</h6>

            <div id="ia-mensagens" class="flex-grow-1 border rounded p-2 overflow-auto bg-body-tertiary"
                 style="min-height: 260px; max-height: 52vh;">
              <div class="ia-msg ia-bot">
                Olá! Posso responder perguntas como:
                <br><em>"quantos registros temos?"</em>,
                <em>"distribuição por operadora"</em>,
                <em>"top 10 EOTs"</em>,
                <em>"entre 2024-01-01 e 2024-03-31 só STFC"</em>.
              </div>
            </div>

            <div class="input-group mt-2">
              <input id="ia-pergunta" class="form-control" placeholder="Pergunte sobre os dados..." autocomplete="off">
              <button id="ia-enviar" class="btn btn-primary" type="button" title="Perguntar">Perguntar</button>
            </div>

            <div class="d-flex gap-2 flex-wrap mt-2">
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="quantos registros?">quantos registros?</button>
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="distribuição por operadora">por operadora</button>
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="top 10 operadoras">top 10 operadoras</button>
              <button class="btn btn-outline-primary btn-sm ia-sugestao" data-q="top 10 EOTs">top 10 EOTs</button>
            </div>
          </div>
        </div>
        <!-- /Agente de IA -->
      </main>
    </div>
  </div>

  <!-- Modal Upload -->
  <div class="modal fade" id="modal-upload" tabindex="-1" aria-labelledby="titulo-upload" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" id="titulo-upload">Carregar Base</h1>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Fechar"></button>
        </div>
        <div class="modal-body">
          <div class="d-grid gap-2 mb-2">
            <label class="btn btn-primary">
              CARREGAR BASE SCIENCE <input type="file" id="input-csv-science" accept=".csv" hidden>
            </label>
            <label class="btn btn-outline-primary">
              CARREGAR BASE PORTAL ITX <input type="file" id="input-csv-portal" accept=".csv" hidden>
            </label>
          </div>
          <form id="form-upload" action="{{ url_for('vivo_crc.upload_csv') }}" method="POST" enctype="multipart/form-data">
            <input type="hidden" name="origem" id="origem-upload" />
            <button id="enviar-upload" type="submit" class="visually-hidden">Enviar</button>
          </form>
          <small class="text-secondary">Apenas arquivos <code>.csv</code> são aceitos.</small>
        </div>
      </div>
    </div>
  </div>

  <!-- Modal Export (filtrado) -->
  <div class="modal fade" id="modal-envio" tabindex="-1" aria-labelledby="titulo-exportar" aria-hidden="true">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h1 class="modal-title fs-5" id="titulo-exportar">Exportar filtrado</h1>
          <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Fechar"></button>
        </div>
        <div class="modal-body">
          <p>Baixe o arquivo considerando os filtros atuais (o sistema escolherá o melhor formato — XLSX em múltiplas abas ou CSV).</p>
          <div class="d-grid">
            <button id="botao-baixar-excel" class="btn btn-primary">Baixar agora</button>
          </div>
          <small class="text-secondary d-block mt-2">
            Dica: Para baixar todos os registros sem filtros, use <em>Exportar → Exportar tudo (CSV)</em>.
          </small>
        </div>
      </div>
    </div>
  </div>

  <!-- Loading -->
  <div id="carregando" class="position-fixed top-0 start-0 w-100 h-100 d-none" style="background:rgba(0,0,0,.35); z-index: 1080;">
    <div class="d-flex h-100 justify-content-center align-items-center">
      <div class="spinner-border text-light" role="status" aria-label="Carregando"></div>
    </div>
  </div>

  <!-- Scripts: Bootstrap primeiro; seu script por último -->
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
  <script src="{{ url_for('static', filename='script.js') }}"></script>
</body>
</html>
