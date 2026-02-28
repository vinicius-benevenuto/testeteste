Contexto & Persona
Você é um desenvolvedor sênior full‑stack especializado em Python 3.11, pandas, SQLite, SQLAlchemy e UIs acessíveis (Streamlit + AG Grid). Seu objetivo é construir um aplicativo pronto para produção que permita ao usuário:
- carregar dois arquivos de planilhas (Science e Portal de Cadastros) e um terceiro arquivo de referência (Arquivo 3),
- combinar colunas específicas a partir de regras de mapeamento de negócio,
- visualizar com filtros estilo Excel (caixa de filtro logo abaixo do cabeçalho),
- exportar e persistir tudo em SQLite com auditoria e versionamento,
- garantindo que nenhum dado se perca (100% preservado).

1) Objetivo do App
Criar um aplicativo web em Python (Streamlit + st-aggrid) que:
1. Permite upload de 3 arquivos:
   - Planilha 1: “Science” (xlsx inicialmente),
   - Planilha 2: “Portal de Cadastros” (xlsx inicialmente),
   - Planilha 3: “Arquivo 3” (xlsx inicialmente) — referência para CLUSTER, Rótulos de Linha e Denominação.
   (O design deve estar preparado para .xls, .csv, .parquet no futuro.)
2. Gera uma tabela final com colunas:
   [REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação]
3. Exibe a tabela final em um grid com filtros por coluna estilo Excel, com caixa de filtro logo abaixo do cabeçalho (floating filter).
4. Salva 100% dos dados (originais + transformados) em SQLite com versionamento, sem exclusão de histórico.
5. O usuário consegue ver a tabela consolidada mesmo sem carregar nada novo (dados persistem entre sessões).
6. Interface limpa, responsiva, PT-BR, acessível (teclado, contraste, ARIA), mensagens claras.

2) Esquemas de Entrada (labels podem vir com mojibake; corrigir rótulos, preservar dados brutos)
Science (exemplos de colunas relevantes):
[COD_CCC Origem, ID Rota, Data AtivaÃ§Ã£o, Data DesativaÃ§Ã£o, Sentido, Tipo,
 OP destino, Ãrea Ponta A - Engenharia, Ãrea Ponta A - MediaÃ§Ã£o, Ãrea Ponta B,
 ServiÃ§o, DescriÃ§Ã£o, Operadora Origem, OP Origem, Central Origem, Operadora destino,
 Sequencial, Sigla Tipo DireÃ§Ã£o, Central Interna, COD_CCC Interna, OPC, DPC, Num SSI,
 CNL, Atendimento MÃ³vel/Fixo, Gateway, SinalizaÃ§Ã£o da Rota, Tipo da Rota, Tipo de Trafego]

Portal de Cadastros (exemplos de colunas relevantes):
[row_number, SOLICITACAO, TIPO, CENTRAL, BILHETADOR, TIPO_ROTA, TIPO_REGISTRO, ROTA_E,
 ROTA_S, LABEL_E, LABEL_S, EMPRESA, EOT, OC, INTERCON, ECR, OPC, DPC, DESIGNACAO,
 CATEGORIA, CATEG_DESCRICAO, SENTIDO, SINALIZACAO, ROTA_EXCLUSIVA, CNL_PPI, PPI,
 DT_ATIVACAO, TRAF_CURSADO, CSP, DT_CADASTRO, RESP_CADASTRO, DT_EXECUCAO, RESP_EXECUCAO, Obs]

Arquivo 3 (referência) — estrutura:
[REDE, UF, CLUSTER, Tipo de Rota, x, Central, Rótulos de Linha, OPERADORA, Denominação, ... (demais métricas)]

Observação:
- Corrigir mojibake apenas nos nomes/labels (ex.: “AtivaÃ§Ã£o” → “Ativação”), sem alterar os conteúdos das células.
- Manter a versão dos nomes originais também (tabela de renomes).

3) Regras de Mapeamento (negócio)
A saída deve ter exatamente:
[REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação]

3.1) REDE (regra específica solicitada)
- Quando a linha advém do arquivo Science (Planilha 1), classifique como “SMP”.
- Quando a linha advém do arquivo Portal de Cadastros (Planilha 2), classifique como “STFC”.
- Valor final de REDE: “VIVO-” + {SMP|STFC}.
- Regras de consolidação:
  a) Se uma linha resulta de junção de registros Science com Portal, adotar “SMP” se houver contribuição de Science; caso contrário “STFC”.
  b) Persistir também uma coluna técnica “source_tag” (SCIENCE, PORTAL, BOTH) para auditoria.

3.2) UF (derivação via CNL/CN → UF)
Objetivo: preencher com as siglas oficiais dos estados (ex.: SP, RJ, AM, TO).
Estratégia robusta em camadas:
- Criar tabelas de referência no SQLite:
  a) “cnl” (COD_CNL TEXT PRIMARY KEY, CN TEXT) → mapeia COD_CNL → CN.
     • Popular esta tabela com os INSERTs fornecidos pelo usuário (vide anexo/seed).
  b) “cn_to_uf” (CN TEXT PRIMARY KEY, UF TEXT NOT NULL) → mapeia CN → UF (sigla).
     • Popular com um dicionário canônico (ex.: 11=SP, 21=RJ, 31=MG, 41=PR, 51=MT, 61=DF, 63=TO, 79=SE, etc.). Forneça arquivo seed (csv ou .sql) e permita edição via UI.
- Durante a importação:
  1) Extrair o possível COD_CNL:
     • Science: usar coluna “CNL” (ou equivalentes).
     • Portal: tentar “CNL_PPI”; se ausente, tentar “PPI”; se não existir, deixar nulo.
     • Normalizar para dígitos (remover espaços, zeros à esquerda conforme necessário).
  2) LEFT JOIN com “cnl” para obter CN.
  3) LEFT JOIN com “cn_to_uf” para obter UF (sigla).
  4) Fallbacks:
     • Se não encontrado via CNL → CN → UF, tentar usar “UF” do Arquivo 3 (se houver match pelo par [Central + Rótulos de Linha] ou [Central + Tipo de Rota]).
     • Se ainda não der match, deixar UF vazio e sinalizar no relatório de qualidade (linhas pendentes de UF).
- Tabelas seed:
  • Criar pasta seeds/ com:
    - seeds/cnl.sql  → incluir TODOS os INSERTs de COD_CNL→CN fornecidos pelo usuário.
    - seeds/cn_to_uf.csv → mapeamento CN→UF (manutenível).

3.3) Tipo de Rota
- Coalescer a partir de:
  • Portal["TIPO_ROTA"]
  • Science["Sinalização da Rota"]
- Normalizar: strip, upper, remover espaços duplicados; registrar de qual origem veio o valor no log (para auditoria).

3.4) Central
- Coalescer a partir de:
  • Portal["CENTRAL"]
  • Science["Central Origem"]
- Normalizar (upper, trim). Registrar origem do valor.

3.5) CLUSTER, Rótulos de Linha, Denominação (via Arquivo 3)
- Preparar ingestão do Arquivo 3 (tabela “ref_rotas”) com as colunas: [REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação, ...]
- Encontrar o registro correspondente para cada rota proveniente de Science/Portal com chaves configuráveis (UI):
  • Chave sugerida (editável pelo usuário): [Central + Rótulos de Linha] e/ou [Central + Tipo de Rota].
  • Se “Rótulos de Linha” vier do Portal (LABEL_E/LABEL_S), considerar:
    - Usar LABEL_E como principal; se ambos existirem, permitir concat “LABEL_E | LABEL_S” para melhorar chance de match (opção habilitável).
- Preencher:
  • CLUSTER ⟵ Arquivo 3
  • Rótulos de Linha ⟵ Arquivo 3 (prioritário para padronizar)
  • Denominação ⟵ Arquivo 3
  • OPERADORA ⟵ Arquivo 3 (prioritário quando existir)
- Caso não haja match no Arquivo 3:
  • Rótulos de Linha: fallback em Portal[“LABEL_E” ou “LABEL_S”] (com concat opcional).
  • Denominação: fallback em Science[“Descrição”] ou Portal[“DESIGNACAO”].
  • CLUSTER: deixar vazio e sinalizar no relatório (pendente de referência).

3.6) OPERADORA (regra final)
- Precedência:
  1) Arquivo 3["OPERADORA"] (se houve match)
  2) Portal["EMPRESA"]
  3) Science[“Operadora Origem”] ou Science[“Operadora destino”] (configurável)
- Normalizar (upper/trim).

4) Persistência e Auditoria em SQLite
- Banco: data/app.db (config por .env).
- Tabelas:
  - imports (id, fonte [SCIENCE|PORTAL|ARQ3], filename, sheet, file_hash, rows, cols, created_at)
  - raw_science (import_id, todas as colunas originais, created_at)
  - raw_portal  (import_id, todas as colunas originais, created_at)
  - ref_rotas   (import_id, REDE, UF, CLUSTER, Tipo_de_Rota, Central, Rotulos_de_Linha, OPERADORA, Denominacao, ... , created_at)
  - cnl (COD_CNL TEXT PK, CN TEXT)                         ← popular via seeds/cnl.sql
  - cn_to_uf (CN TEXT PK, UF TEXT NOT NULL)                ← popular via seeds/cn_to_uf.csv
  - column_renames (import_id, original_name, normalized_name)
  - mappings (version_id, target_col, rule_json)
  - merged_results (version_id, row_id, REDE, UF, CLUSTER, Tipo_de_Rota, Central, Rotulos_de_Linha, OPERADORA, Denominacao, source_keys_json, created_at)
  - logs (timestamp, level, message, context_json)
- Políticas:
  • Sempre usar transações na importação/merge.
  • Nunca deletar dados; usar versionamento (version_id) e flags se necessário.
  • Manter “views” para facilitar leitura:
    - v_merged_latest (última versão consolidada)
    - v_quality_pending (linhas sem UF/CLUSTER/Denominação)
- O usuário deve conseguir abrir o app e visualizar v_merged_latest antes de qualquer novo upload.

5) UI (Streamlit + AG Grid) e Acessibilidade
- Uploads (3 arquivos), escolha de sheet (XLSX), preview tabular.
- “Assistente de Mapeamento” para:
  • Confirmar chaves de junção (ex.: Central + Tipo de Rota + Sentido).
  • Definir precedências/COALESCE e concat (ex.: LABEL_E | LABEL_S).
  • Apontar qual coluna contém COD_CNL/CNL/CNL_PPI/PPI.
- Grid principal (st-aggrid):
  • enableSorting: true
  • enableFilter: true
  • floatingFilter: true  ← caixas de filtro logo abaixo do cabeçalho (estilo Excel)
  • defaultColDef.filter = 'agTextColumnFilter' (ajustar por tipo)
  • sideBar: { filters, columns }
  • rowSelection: 'multiple', pagination
  • clipboard: excelMode = true
  • tooltips, column pinning, quickFilter global
- Acessibilidade:
  • Navegação por teclado
  • aria-labels e labels descritivos
  • contraste adequado
  • mensagens de erro claras e focáveis

6) Qualidade de Dados e Segurança
- Normalizar labels (corrigir mojibake), registrar renomeações em column_renames.
- Inferir tipos (datas DD/MM/YYYY), inteiros, strings; não falhar em parsing (usar safe conversions).
- Relatório de qualidade:
  • contagem de nulos por coluna de saída
  • duplicados com base nas chaves definidas
  • linhas sem UF (mostrar contagem e amostras)
  • linhas sem match no Arquivo 3
- Segurança:
  • Sanitizar nomes de arquivos
  • Não executar macros/código incorporado
  • Nenhum dado enviado para fora
- Desempenho:
  • Suportar 200k+ linhas por arquivo
  • Escrever em chunks no SQLite
  • Evitar manter dataframes gigantes em memória desnecessariamente

7) Lógica de Junção e Preenchimento
- Chaves de junção configuráveis pelo usuário (UI). Padrão: outer join (para não sumir linhas).
- Campos com múltiplas fontes: aplicar COALESCE conforme precedência definida no item 3.
- “Rótulos de Linha”:
  • Se houver LABEL_E/LABEL_S no Portal, permitir concat com separador " | ".
  • Priorizar padronização do Arquivo 3 quando houver match.
- REDE:
  • Derivar “VIVO-SMP” se a contribuição incluir Science; caso contrário “VIVO-STFC”.
- UF:
  • Derivar via CNL→CN→UF (JOIN com cnl e cn_to_uf), com fallbacks descritos.
- OPERADORA:
  • Precedência: Arquivo 3 → Portal → Science.
- Gerar version_id a cada merge, com persistência integral de inputs/outputs.

8) Filtros “como no Excel”
- Implementar filtros em caixas logo abaixo dos títulos (AG Grid Floating Filters).
- Exemplo de configuração (conceitual, não precisa colar literal):
  grid_options = {
    "defaultColDef": {
      "filter": True,
      "floatingFilter": True,
      "sortable": True,
    },
    "sideBar": {"toolPanels": ["filters", "columns"]},
    "rowSelection": "multiple",
    "enableRangeSelection": True,
  }
- Incluir também quickFilter (barra de busca global) e salvar/recuperar estado de filtros por sessão.

9) Entregáveis
1. Código completo do app (Python + Streamlit + SQLAlchemy).
2. requirements.txt (streamlit, pandas, sqlalchemy, openpyxl, xlrd, pyarrow, python-dotenv, unidecode, chardet, st-aggrid).
3. README.md com:
   • como rodar (streamlit run app.py),
   • onde colocar seeds/ (cnl.sql e cn_to_uf.csv),
   • como configurar chaves de junção e mapeamentos,
   • detalhes de acessibilidade e limites de performance.
4. Testes com pytest (unidade e integração):
   • leitura multi-formato
   • normalização de labels
   • derivação de REDE (SMP/STFC) por origem
   • derivação de UF via CNL→CN→UF (com seeds)
   • mapeamento de Arquivo 3 para CLUSTER/Rotulos/Denominação
   • merge outer (sem perda)
   • persistência em SQLite e versionamento
5. Pasta samples/ com 3 exemplos (mock): science.xlsx, portal.xlsx, arquivo3.xlsx.
6. seeds/:
   • cnl.sql (TODOS os INSERTs de COD_CNL→CN fornecidos)
   • cn_to_uf.csv (dicionário CN→UF, editável)
7. Dockerfile opcional.

10) Boas Práticas (Obrigatórias)
- Código modular e tipado (type hints).
- Camadas:
  • io/ (readers/writers)
  • core/ (normalize, map_rules, merge, validate)
  • db/ (models, session, repository)
  • ui/ (pages, mapping_wizard, grid)
  • utils/ (logging, encoding, ids)
- SQLAlchemy para DB (evitar operar só com sqlite3 cru).
- Config via .env (caminho do DB, tamanhos de chunk, flags de desempenho).
- Internacionalização de mensagens possíveis (PT-BR padrão).
- Provar que nenhum dado se perde (contadores antes/depois e testes).

11) Esqueleto de Pastas (sugerido)
app/
  app.py
  ui/
    pages.py
    mapping_wizard.py
    grid.py
  core/
    normalize.py
    map_rules.py
    merge.py
    validate.py
  io/
    readers.py
    writers.py
  db/
    models.py
    session.py
    repository.py
  utils/
    logging.py
    encoding.py
    ids.py
  seeds/
    cnl.sql              ← COLE AQUI os INSERTs de COD_CNL→CN fornecidos pelo usuário
    cn_to_uf.csv         ← mapeamento CN,UF (ex.: “11,SP”; “63,TO”; “79,SE”...)
  samples/
    science_exemplo.xlsx
    portal_exemplo.xlsx
    arquivo3_exemplo.xlsx
  tests/
    test_readers.py
    test_merge.py
    test_db.py
    test_mapping.py
    test_uf_derivation.py
    test_rede_rule.py
  requirements.txt
  README.md

12) Critérios de Aceite
- Importa .xlsx e está preparado para .xls/.csv/.parquet.
- REDE preenchida exatamente como “VIVO-SMP” (Science) ou “VIVO-STFC” (Portal), com regra de consolidação quando ambos contribuem.
- UF derivada via CNL→CN→UF (JOIN em cnl e cn_to_uf); pendências relatadas.
- CLUSTER, Rótulos de Linha e Denominação vindos do Arquivo 3 quando houver match; fallbacks claros quando não houver.
- Central coalescida (Portal.CENTRAL, Science.Central Origem).
- Tipo de Rota coalescido (Portal.TIPO_ROTA, Science.Sinalização da Rota).
- Grid com filtros por coluna estilo Excel (floating filters) e quickFilter.
- Dados persistem entre sessões; usuário vê a última versão (v_merged_latest) sem precisar recarregar.
- Nenhum dado some (raw_* preserva 100%; merge é outer).
- Testes passando (inclusive REDE, UF, Arquivo 3).

13) Observações Técnicas
- Corrigir mojibake só em labels (guardar mapeamento em column_renames).
- Desduplicar linhas conforme necessidade com chaves claras (sem excluir histórico).
- Logar decisões de precedência (ex.: de onde veio cada campo coalescido).
- Expor no UI: baixar CSV/XLSX do que está filtrado na tela e do resultado completo.
- Permitir baixar log e JSON das regras de mapeamento e chaves de junção usadas em cada versão.
- Se houver dúvidas na derivação de UF, destacar “pendente de referência CNL/CN” no relatório.

AGORA GERE O CÓDIGO COMPLETO do aplicativo conforme as especificações acima, incluindo:
- o setup de seeds (executar seeds/cnl.sql e carregar seeds/cn_to_uf.csv na primeira execução),
- o uso de AG Grid com floatingFilter habilitado,
- testes automatizados para as regras novas (REDE e UF),
- README com instruções de como atualizar cn_to_uf e como trocar as chaves de junção do Arquivo 3.
