Objetivo geral:
Quero reorganizar a interface do meu sistema para deixá-la totalmente limpa, objetiva e focada apenas nas ações que o usuário realmente precisa ver. Todas as funcionalidades internas continuam funcionando normalmente, mas não devem aparecer visualmente na interface.

Regra fundamental:
- NÃO remover código.
- NÃO excluir funções.
- Apenas NÃO exibir determinadas partes visuais ao usuário final.

Partes que NÃO devem aparecer na interface:
- Validação
- Seeds / Referência
- Histórico
- Logs
- Diagnóstico
- Relatório Pipeline
- Mapeamentos internos
- Informações técnicas (CNS, CEI, SIUP, métricas internas, contadores, tabelas auxiliares, etc.)

Essas partes devem continuar existindo e funcionando, apenas não exibidas.

Partes que DEVEM aparecer na interface:
Somente o fluxo essencial:
1. Carregar Arquivos
2. Preview dos arquivos carregados
3. Gerar Tabela Final (logo abaixo do preview)
4. Exportar tabela final (se existir)
5. Botão “Reiniciar Sessão” (discreto)

Reestruturação obrigatória:
- A funcionalidade “Gerar Tabela Final” deve ser movida para dentro da página “Carregar Arquivos”.
- Deve aparecer imediatamente abaixo do preview das bases carregadas.
- Nenhuma outra seção deve ficar visível.

Regras de exibição:
- A barra lateral deve mostrar apenas o que for necessário para o usuário final.
- Não exibir nenhuma página técnica no menu.
- Não criar modo desenvolvedor.
- Não criar flags, permissões ou parâmetros especiais.
- Apenas impedir que essas páginas internas sejam renderizadas ou chamadas na interface.

Como aplicar:
- Envolver a renderização das páginas/desenhos internos em condições simples, como:
  if False:
      (bloco oculto)
- Ou simplesmente não chamar essas funções na interface.
- O código dessas partes internas continua existindo no projeto normalmente.

Resultado esperado:
- Interface extremamente limpa e amigável.
- Usuário vê somente o fluxo lógico: Carregar Arquivos → Preview → Gerar Tabela Final → Exportar.
- Nenhuma informação técnica aparece visualmente.
- Todo o código interno permanece preservado e funcional.

Resumo final:
O programa mantém todas as partes internas.  
A interface mostra apenas o essencial.  
Nada técnico deve ser exibido.  
Nenhuma página é removida, apenas oculta.  


----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Objetivo geral:
Quero organizar o meu sistema Streamlit para deixá-lo extremamente limpo, direto e profissional.  
Nenhuma informação técnica ou interna deve aparecer ao usuário.  
A interface deve focar apenas no fluxo essencial.

Regra fundamental:
Não é para apagar nada do programa.
Não é para excluir funções internas.
Apenas não quero que certas telas/partes apareçam visualmente.

Partes que NÃO devem aparecer para o usuário:
- Validação
- Seeds / Referência
- Histórico
- Logs
- Diagnóstico
- Relatório Pipeline
- Mapeamentos internos
- Informações internas como CNS, CEI, SIUP, contadores, tabelas auxiliares
Todas essas partes continuam existindo no código, só não devem ser exibidas na interface.

Partes que DEVEM aparecer:
Somente o fluxo principal:
1. Carregar Arquivos
2. Preview dos arquivos
3. Gerar Tabela Final
4. Dashboard imediato por coluna (automático ao clique)
5. Exportar resultados, se existir
6. Reiniciar Sessão (discreto)

Reorganização obrigatória:
- “Gerar Tabela Final” deve ficar dentro da mesma página “Carregar Arquivos”, logo abaixo do preview.
- Nenhum outro menu ou tela técnica deve aparecer.
- Nada de selectbox para o dashboard.  
  O dashboard deve aparecer automaticamente quando o usuário clicar em uma coluna da tabela final.
- A tabela final deve ser exibida em um componente interativo onde o clique no cabeçalho da coluna gera automaticamente o dashboard dessa coluna.

Comportamento do Dashboard (muito importante):
Após gerar a Tabela Final, o usuário verá a tabela no formato interativo.
Quando o usuário clicar no nome de uma coluna, automaticamente deve surgir o dashboard daquela coluna.
Sem selectbox.
Sem botão.
Instantâneo.

Quais insights devem ser gerados por coluna (bases nas minhas colunas reais):
Minhas colunas são:
REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação.

Para cada coluna, quero que o sistema gere insights relevantes automaticamente:

1. REDE:
   - Frequência, %, dominância, variedade.

2. UF:
   - Distribuição geográfica por estado, ranking e share.

3. CLUSTER:
   - Frequência, % e detecção de campos vazios.

4. Tipo de Rota:
   - Distribuição INTERNA vs ITX-SIP, participação e comparação.

5. Central:
   - Top 10 centrais, concentração, diversidade, possíveis gargalos.

6. Rótulos de Linha:
   - Frequências, rótulos únicos, possíveis inconsistências.

7. OPERADORA:
   - Quais operadoras dominam, relação entre UF e Tipo de Rota.

8. Denominação:
   - Termos importantes, duplicidades, padrões recorrentes.

Gráficos que devem ser usados automaticamente:
- Para colunas categóricas: Gráfico de Barras, Treemap, Pizza (se ≤ 6 categorias).
- Para texto: ranking de termos.
- Sempre gerar também uma tabela de frequências.

Funcionamento desejado:
- Assim que o usuário clica em uma coluna da tabela final, o sistema:
  - Detecta o tipo da coluna.
  - Calcula automaticamente os melhores insights.
  - Gera o dashboard instantâneo.
  - Exibe gráficos, tabelas e percentuais.

Regras de exibição:
- Nada de menu adicional.
- Nada técnico deve aparecer.
- Apenas não renderizar as partes que não devem aparecer.
- Não criar modo dev ou flags.
- Somente esconder visualmente as partes técnicas ou remover suas chamadas na interface.

Resultado esperado:
Quero que você reorganize meu código ou gere um novo código que siga fielmente:
Carregar Arquivos → Preview → Gerar Tabela Final → Mostrar Tabela Final → Clique na Coluna → Dashboard instantâneo → Exportar

Resumo final:
O sistema deve ser extremamente intuitivo:
o usuário gera a tabela final, clica em uma coluna e vê os gráficos imediatamente.
A interface deve ser limpa, sem itens técnicos.
O código interno e partes avançadas permanecem no projeto, só não aparecem na interface.

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Objetivo geral:
Quero um app Streamlit limpo e objetivo. O fluxo visível ao usuário é:
Carregar Arquivos → Preview → Gerar Tabela Final → Mostrar Tabela Final → (clique no cabeçalho de uma coluna) → Dashboard instantâneo → Exportar.
Nenhuma tela técnica deve aparecer. Todo o restante do código continua existindo internamente.

Regras de visibilidade:
- NÃO remover funcionalidades internas (validação, seeds, histórico, logs, diagnóstico, relatório pipeline, mapeamentos).
- Apenas NÃO exibir essas áreas na interface (não renderizar).
- Sidebar/menu deve mostrar somente o essencial do fluxo acima + “Reiniciar Sessão” discreto.

Colunas (schema de negócio):
REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação.

Persistência em banco (obrigatório):
1) Ao iniciar o app, carregar automaticamente a Tabela Final salva no banco e exibir na tela (se existir).
2) Usar SQLite local (app.db) via SQLAlchemy. Nome da tabela: tabela_final.
3) Criar uma coluna técnica de deduplicação chamada hash_rota (chave composta normalizada), com UNIQUE INDEX para evitar duplicatas.
   - A chave deve considerar a combinação:
     (REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação).
   - Normalização para a chave:
     - strip() em brancos
     - upper()
     - remover acentos
     - colapsar múltiplos espaços
     - substituir None por ""
     - concatenar com "||"
     - aplicar SHA1 → hash_rota
   - Criar índice único: CREATE UNIQUE INDEX IF NOT EXISTS idx_tabela_final_hash ON tabela_final(hash_rota).

Carga de novas bases + deduplicação (obrigatório):
1) Quando o usuário carregar novas bases:
   - Unificar/transformar para o schema final.
   - Gerar hash_rota para cada linha (mesma regra de normalização acima).
   - Fazer anti-join contra os hash_rota já existentes no banco para identificar somente as NOVAS rotas.
   - Inserir no banco apenas as rotas novas (UPSERT/INSERT OR IGNORE).
   - Mensagem simples ao usuário:
     “X rotas novas adicionadas. Y rotas já existiam e foram ignoradas.”
   - Não mostrar detalhes técnicos (sem logs, sem queries).

Geração da Tabela Final:
- A Tabela Final resultante deve ser exibida em componente interativo.
- Salvar a Tabela Final no banco e em st.session_state["tabela_final"].
- Ao voltar ao app (nova sessão), carregar do banco e exibir automaticamente.

Dashboard instantâneo por clique de coluna:
- Exibir a Tabela Final e instruir: “Clique no cabeçalho de uma coluna para ver o dashboard”.
- Ao clicar no cabeçalho de uma coluna, gerar automaticamente o dashboard dessa coluna (sem selectbox, sem botão).
- Pode usar st.data_editor (preferido) ou AgGrid para capturar o clique de coluna.
- O dashboard deve aparecer imediatamente abaixo da tabela.

Insights por tipo de coluna (auto):
- Detectar o tipo da coluna selecionada e renderizar:
  A) Categórica (ex.: REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA):
     - Tabela de frequências: Categoria | Quantidade | %
     - Gráfico de Barras (Top N; se cardinalidade alta, usar Top 15 + “Outros”)
     - Pizza/Doughnut somente se nº de categorias ≤ 6 (com rótulos de %)
     - KPIs rápidos: #Registros, %Nulos, #Únicos, %Top1, %Top3 acumulado
  B) Numérica (se houver no futuro):
     - Histograma + Boxplot
     - Estatísticas: média, mediana, desvio-padrão, P5/P25/P50/P75/P95, outliers%
  C) Data/Tempo (se houver no futuro):
     - Série temporal agregada por período (D/W/M), variação %, média móvel

Colunas reais e perguntas chave:
- REDE: distribuição e dominância por rede.
- UF: ranking por estado e share.
- CLUSTER: frequência e % de vazios.
- Tipo de Rota: mix INTERNA vs ITX-SIP.
- Central: Top 10 + concentração (Pareto).
- Rótulos de Linha: rótulos únicos, duplicidade.
- OPERADORA: participação por operadora e relação com UF/Tipo de Rota.
- Denominação (texto): termos/padrões recorrentes (opcionalmente top palavras).

Exportação do Dashboard (PowerPoint):
- Incluir botão “Exportar Dashboard (PPTX)”.
- Gerar um arquivo PPTX com python-pptx contendo:
  - Slide 1: Título (nome da coluna), data/hora, KPIs (#Registros, %Nulos, #Únicos, %Top1, %Top3 acumulado), logo.
  - Slide 2: Gráfico principal (Barras ou Pizza conforme regra).
  - Slide 3: Gráfico secundário (ex.: Pizza se barra foi principal; ou Treemap).
  - Slide 4: Tabela de Frequências (Categoria | Quantidade | %).
- Os gráficos devem ser incluídos de forma compatível:
  - Opção A (preferida): criar gráficos nativos do PPT via python-pptx ChartData (barras/pizza) com as séries geradas do DataFrame.
  - Opção B: exportar as figuras Plotly para PNG (kaleido) e inserir como imagem.
- Nome do arquivo: dashboard_{coluna}_{AAAA-MM-DD_HHMM}.pptx
- Incluir botão para baixar (st.download_button).

Arquitetura técnica esperada:
- Módulos/funções:
  - load_db_table() → lê do SQLite e retorna DataFrame (cacheável)
  - normalize_key_fields(df) → aplica normalização e devolve df com hash_rota
  - upsert_new_routes(df_new) → faz anti-join por hash_rota e insere apenas novas
  - render_upload_and_preview() → UI de upload + preview
  - gerar_tabela_final() → constrói e salva Tabela Final, retorna df_final
  - render_tabela_final(df_final) → exibe st.data_editor interativo para clique no cabeçalho
  - render_dashboard_coluna(df_final, coluna) → gera KPIs, gráficos e tabela
  - export_dashboard_pptx(df_freq, kpis, coluna) → cria e retorna bytes do PPTX
- Banco:
  - SQLite (app.db), tabela: tabela_final
  - Colunas de negócio + hash_rota (UNIQUE)
- Performance:
  - cachear leitura do banco; invalidar cache ao inserir novas rotas
  - limitar cardinalidade visual (Top N + “Outros”) para gráficos performáticos

Qualidade de dados (silenciosa, sem poluir UI):
- Normalizar strings (upper, acentos, trims).
- Tratar nulos como “(Sem valor)” na frequência (mas manter nulo original no banco).
- Remover espaços duplos.
- Garantir consistência de UF (ex.: siglas válidas).
- (Opcional) log silencioso de linhas rejeitadas, sem exibir ao usuário.

UX e UI:
- Interface minimalista, sem telas técnicas.
- Pós “Gerar Tabela Final”: mostrar a tabela + instrução curta de clique no cabeçalho.
- Dashboard aparece instantaneamente ao clique, sem selectbox.
- Botão de exportar PPT visível no painel do dashboard.
- “Reiniciar Sessão” estilizado de forma discreta.

Entregáveis obrigatórios:
- Código completo Streamlit com:
  1) Persistência em SQLite, leitura automática ao iniciar.
  2) Deduplicação por hash_rota (unique index).
  3) Geração de Tabela Final e exibição interativa.
  4) Clique no cabeçalho → dashboard instantâneo.
  5) Exportação do dashboard para PPTX (python-pptx), pronto para download.
- Instruções de onde colar cada função/arquivo.
- Pequeno README com passos de execução.

Critérios de aceite (teste de mesa):
- Inicie o app sem subir arquivo → a Tabela Final existente no banco aparece automaticamente (se houver).
- Suba novas bases contendo rotas repetidas e rotas novas → o sistema adiciona somente as novas; repetidas são ignoradas (sem erro).
- Gere a Tabela Final → a tabela aparece; ao clicar no cabeçalho de “UF”, o dashboard de UF surge sem clique adicional.
- Clique em “Exportar Dashboard (PPTX)” → faz download de um arquivo .pptx com 3–4 slides (título/KPIs, gráficos, tabela).
- Recarregue o app → a Tabela Final volta a aparecer carregada do banco.


--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Contexto e objetivo:
Quero um sistema Streamlit profissional, minimalista e robusto, com:
- Fluxo limpo para o usuário final
- Persistência completa em banco (carregar ao iniciar)
- Deduplicação automática ao carregar novas bases
- Geração da Tabela Final
- Tabela Final interativa
- (Sem selectbox) Dashboard instantâneo por coluna ao clicar no cabeçalho
- Exportação para PowerPoint (da coluna selecionada e do painel geral)
- Nenhuma tela técnica visível (nada de logs, diagnóstico, mapeamentos etc. no UI)
- Corrigir bugs de UI previamente identificados

📌 Regras fundamentais (obrigatórias):
1) NÃO remover nenhuma funcionalidade interna do programa (validação, seeds/referência, histórico, logs, diagnóstico, relatório pipeline, mapeamentos).
2) NÃO exibir essas áreas na interface do usuário. Apenas não renderizar / não chamar.
3) A interface deve mostrar SOMENTE o fluxo principal:
   Carregar Arquivos → Preview → Gerar Tabela Final → Mostrar Tabela Final → (Clique no cabeçalho de uma coluna) → Dashboard instantâneo → Exportar (coluna e geral).
4) Persistência real em banco: ao abrir o app, a Tabela Final salva no banco deve ser carregada e exibida automaticamente, mesmo sem o usuário subir arquivo.
5) Deduplicação ao importar novas bases: não inserir rotas existentes; inserir apenas as novas.
6) Dashboard aparece instantaneamente após clique no cabeçalho de uma coluna (sem selectbox, sem botão).
7) Exportar o dashboard:
   - da coluna selecionada (PPTX)
   - do painel geral (PPTX)
8) Não criar “modo dev”, flags, permissões ou parâmetros ocultos. Apenas não renderizar o que é técnico.

🧱 Schema de negócio (colunas da Tabela Final):
REDE
UF
CLUSTER
Tipo de Rota
Central
Rótulos de Linha
OPERADORA
Denominação

💾 Persistência (SQLite + SQLAlchemy):
- Banco local: app.db
- Tabela: tabela_final
- Ao iniciar o app:
  - Se tabela_final existir, carregar e exibir automaticamente
- Ao salvar/atualizar Tabela Final:
  - Persistir no banco
  - Guardar em st.session_state["tabela_final"]
- Criar coluna técnica: hash_rota (UNIQUE) para deduplicação
  - hash_rota = SHA1(normalização de: REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação)
  - Normalização:
    * str(value) or "" (None -> "")
    * strip()
    * upper()
    * remover acentos
    * colapsar múltiplos espaços
    * concatenar com "||"
    * aplicar SHA1
  - Índice único:
    CREATE UNIQUE INDEX IF NOT EXISTS idx_tabela_final_hash ON tabela_final(hash_rota)

🔁 Carga de novas bases + deduplicação:
1) Ao importar novas bases:
   - Transformar/unificar para o schema final
   - Gerar hash_rota para cada linha (mesma regra)
   - Anti-join contra o banco por hash_rota para detectar apenas NOVAS rotas
   - Inserir somente novas (UPSERT: INSERT OR IGNORE / SQLAlchemy equivalente)
   - Mensagem ao usuário (simples e clara):
     "X rotas novas adicionadas. Y rotas já existiam e foram ignoradas."
   - Sem logs técnicos visíveis

🧮 Geração da Tabela Final:
- Construir o DataFrame final com as colunas de negócio acima
- Persistir no banco (com hash_rota)
- Colocar em st.session_state["tabela_final"]
- Exibir imediatamente a Tabela Final na interface

🖱️ Interação (sem selectbox) → Dashboard por clique no cabeçalho:
- Exibir a Tabela Final em componente interativo e instruir:
  "Clique no cabeçalho de uma coluna para ver o dashboard."
- Requisito: o dashboard deve surgir instantaneamente ao clique do cabeçalho (sem botões/menus)
- Implementação sugerida:
  - Preferência: st-aggrid (GridOptionsBuilder + eventos) para capturar header click e retornar o nome da coluna
  - Alternativa (fallback nativo): st.data_editor + captura de interação (cell selection) e inferir a coluna clicada
  - O resultado do clique deve disponibilizar o nome da coluna selecionada → renderizar dashboard imediatamente

🤖 Insights automáticos por coluna (todas categóricas neste momento):
- Ao identificar a coluna clicada, gerar:
  KPIs:
    - #Registros (n)
    - %Nulos
    - #Únicos
    - %Top1
    - %Top3 acumulado
  Frequências:
    - Tabela: Categoria | Quantidade | %
    - Se cardinalidade > N (ex.: 15), mostrar Top N + “Outros”
  Gráficos:
    - Barras (padrão; horizontal quando rótulos longos)
    - Pizza/Doughnut somente se nº de categorias ≤ 6, com rótulos de %
    - Treemap (opcional) para participação
- Colunas e perguntas chave (telecom):
  1) REDE → distribuição e dominância
  2) UF → ranking por estado e share
  3) CLUSTER → frequência e % vazios
  4) Tipo de Rota → mix INTERNA vs ITX-SIP
  5) Central → Top 10 + Pareto (concentração)
  6) Rótulos de Linha → rótulos únicos, duplicidade
  7) OPERADORA → participação por operadora; cruzamentos com UF/Tipo
  8) Denominação (texto) → padrões/termos recorrentes, duplicidades

📤 Exportação para PowerPoint (obrigatório):
1) Exportar dashboard da COLUNA selecionada (aparece quando há coluna clicada):
   - Botão: "Exportar dashboard desta coluna (PPTX)"
   - Conteúdo (4 slides):
     * Slide 1: Título (nome da coluna), data/hora, KPIs (#Registros, %Nulos, #Únicos, %Top1, %Top3)
     * Slide 2: Gráfico principal (Barras ou Pizza, conforme regra de cardinalidade)
     * Slide 3: Gráfico secundário (Treemap ou complementar)
     * Slide 4: Tabela de frequências (Categoria | Quantidade | %)
   - Nome do arquivo: dashboard_<COLUNA>_<YYYY-MM-DD_HHMM>.pptx

2) Exportar DASHBOARD GERAL (sempre disponível quando houver Tabela Final):
   - Botão: "Exportar dashboard geral (PPTX)"
   - Conteúdo (6–10+ slides):
     * Capa (projeto, data, total de rotas)
     * Resumo geral: KPIs globais (total, % nulos médios, # colunas, etc.)
     * UF (ranking + share)
     * OPERADORA (mix e/ou por UF)
     * Tipo de Rota (INTERNA vs ITX-SIP)
     * Central (Top10 + Pareto)
     * CLUSTER
     * Rótulos de Linha (Top 10)
     * Denominação (termos recorrentes)
     * Conclusões / próximos passos
   - Nome do arquivo: dashboard_geral_<YYYY-MM-DD_HHMM>.pptx

3) Implementação técnica do PPT (python-pptx):
   - Opção A (preferencial): gráficos nativos (ChartData) com as séries do DataFrame
   - Opção B: exportar gráficos plotly para PNG (kaleido) e inserir como imagem
   - Inserir logo/título padrão e estilos consistentes (tema escuro/claro conforme o app)

🎨 UX e UI (obrigatório):
- Interface minimalista com foco no fluxo principal
- "Reiniciar Sessão" discreto
- Sem telas técnicas (diagnóstico, logs, mapeamentos, seeds, relatório pipeline etc.)
- Barra lateral: somente entradas do fluxo principal
- Após “Gerar Tabela Final”, exibir a tabela + instrução clara de clique no cabeçalho
- Dashboard renderiza instantaneamente ao clique
- Em altas cardinalidades, use Top N + "Outros"
- Rótulos legíveis e percentuais nas pizzas (quando aplicável)

⚙️ Performance e qualidade:
- Pré-calcular e cachear frequências por coluna após gerar a Tabela Final (para o dashboard ficar instantâneo)
- Cachear leitura do banco; invalidar ao inserir novas rotas
- Normalizar/limpar strings: upper, acentos, trims, espaços duplos
- Tratar nulos na frequência como "(Sem valor)" (sem alterar o valor persistido original)
- Garantir UF válidas (se aplicável)
- Registrar rejeições de insert ou inconsistências em log silencioso (não exibido na UI)

🛠️ Correções de bugs de UI (aplicar):
- Remover texto/label incorreto “keyboard_double” ao passar o mouse na área recolhível
- Sidebar recolhível deve recolher/expandir normalmente
- Botão “Reiniciar Sessão” deve ser discreto (sem destaque excessivo)
- Corrigir prévia que mostra `_arrow_right` literal duplicado/sobreposto — usar ícone correto e CSS seguro
- Garantir que nenhuma área técnica reapareça na UI por CSS/estilos globais
  
✅ Critérios de aceite (teste de mesa):
1) Abrir o app sem subir arquivo: a Tabela Final existente aparece automaticamente, carregada do SQLite.
2) Carregar novas bases com rotas repetidas e novas:
   - Inserir apenas as novas
   - Mostrar: "X rotas novas adicionadas. Y já existiam."
3) Gerar Tabela Final → exibir imediatamente a Tabela Final interativa.
4) Clicar no cabeçalho de “UF” → dashboard instantâneo de UF aparece (barras/pizza/tabela + KPIs).
5) Clicar no cabeçalho de “OPERADORA” → dashboard muda instantaneamente.
6) Exportar “dashboard desta coluna (PPTX)” → baixa arquivo com 4 slides.
7) Exportar “dashboard geral (PPTX)” → baixa arquivo consolidado com múltiplos slides.
8) Nenhuma tela técnica aparece em lugar algum.
9) Botão “Reiniciar Sessão” permanece discreto.
10) Sidebar recolhível funciona corretamente e sem labels estranhos.
11) A prévia não exibe `_arrow_right` duplicado ou texto literal de ícones.

Observações finais:
- Não criar modo dev, flags ou parâmetros ocultos; apenas não renderizar áreas técnicas.
- Siga boas práticas de código, nomes claros de funções e comentários sucintos.
- Manter consistência visual e mensagens curtas e executivas ao usuário.
Observações finais:
- Não criar modo desenvolvedor, flags ou páginas ocultas.
- Não exibir nada técnico na interface.
- Toda a lógica interna permanece no código e no banco.
