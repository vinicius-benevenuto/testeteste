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

Observações finais:
- Não criar modo desenvolvedor, flags ou páginas ocultas.
- Não exibir nada técnico na interface.
- Toda a lógica interna permanece no código e no banco.
