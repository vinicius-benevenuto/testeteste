Contexto & Persona
Você é um desenvolvedor sênior full‑stack especializado em Python 3.11, pandas, SQLite e desenvolvimento de UIs acessíveis. Seu objetivo é construir um aplicativo pronto para produção que permita ao usuário carregar dois arquivos de planilhas, combinar colunas específicas, visualizar, filtrar, exportar e persistir os dados em SQLite, garantindo que nenhum dado se perca.

1) Objetivo do App
Criar um aplicativo (preferencialmente web em Python) que:
1. Permite upload de dois arquivos (inicialmente .xlsx, mas o design deve suportar .xls, .csv, .parquet no futuro).
2. Cada arquivo contém uma tabela (Science e Portal de Cadastros).
3. O app deve selecionar colunas de cada tabela e gerar uma tabela combinada com as colunas finais:
[REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação]
4. O usuário deve ver a tabela final num grid com filtros por coluna estilo Excel (filtros fáceis de usar).
5. Salvar todos os dados (originais + transformados) em SQLite (com versionamento de importações e trilhas de auditoria).
6. Nenhum dado pode sumir — manter 100% das linhas e colunas importadas, com rastreabilidade completa.
7. Interface limpa, responsiva, em PT-BR e com acessibilidade (navegação por teclado, contraste adequado, ARIA).

2) Especificações das Tabelas de Entrada
Tabela 1 – “Science” (esquema fornecido):
[COD_CCC Origem, ID Rota, Data AtivaÃ§Ã£o, Data DesativaÃ§Ã£o, Sentido, Tipo,
 OP destino, Ãrea Ponta A - Engenharia, Ãrea Ponta A - MediaÃ§Ã£o, Ãrea Ponta B,
 ServiÃ§o, DescriÃ§Ã£o, Operadora Origem, OP Origem, Central Origem, Operadora destino,
 Sequencial, Sigla Tipo DireÃ§Ã£o, Central Interna, COD_CCC Interna, OPC, DPC,
 Num SSI, CNL, Atendimento MÃ³vel/Fixo, Gateway, SinalizaÃ§Ã£o da Rota,
 Tipo da Rota, Tipo de Trafego]

Tabela 2 – “Portal de Cadastros” (esquema fornecido):
[row_number, SOLICITACAO, TIPO, CENTRAL, BILHETADOR, TIPO_ROTA, TIPO_REGISTRO, ROTA_E,
 ROTA_S, LABEL_E, LABEL_S, EMPRESA, EOT, OC, INTERCON, ECR, OPC, DPC, DESIGNACAO,
 CATEGORIA, CATEG_DESCRICAO, SENTIDO, SINALIZACAO, ROTA_EXCLUSIVA, CNL_PPI, PPI,
 DT_ATIVACAO, TRAF_CURSADO, CSP, DT_CADASTRO, RESP_CADASTRO, DT_EXECUCAO,
 RESP_EXECUCAO, Obs]

Observação de codificação: há casos de mojibake (ex.: AtivaÃ§Ã£o, DescriÃ§Ã£o). O app deve corrigir e padronizar acentuação nos rótulos de colunas (labels) sem alterar o conteúdo dos dados. Manter a versão original também.

3) Colunas Finais (saída) e Mapeamento
A saída deve ter exatamente estas colunas:
[REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação]

Implementar um Assistente de Mapeamento que:
- Sugere mapeamentos automáticos a partir de nomes semelhantes e sinônimos.
- Permite o usuário confirmar/editar o mapeamento.
- Suporta regras de precedência (ex.: usar coluna da Science se existir; senão, da Portal).
- Suporta fórmulas simples (concatenação/COALESCE) para preencher um campo com múltiplas colunas de origem.
- Permite definir chaves de junção entre as tabelas (ex.: Central + Tipo de Rota + Sentido), com opção de junção exata ou aproximação (fuzzy) sob confirmação do usuário.
- Permite definir valores padrão e tabelas de referência para campos não presentes nas fontes (ex.: REDE, UF, CLUSTER).

Sugestões de mapeamento (defaults, o usuário pode trocar):
- Tipo de Rota → Science["Tipo da Rota"] ou Portal["TIPO_ROTA"] (priorize Science)
- Central → Science["Central Interna"] ou Portal["CENTRAL"]
- Rótulos de Linha → Portal["LABEL_E"] ou Portal["LABEL_S"] (use concat com separador quando ambas existirem)
- OPERADORA → escolha configurável entre Science["Operadora Origem"], Science["Operadora destino"] e Portal["EMPRESA"]
- Denominação → Science["Descrição"] (corrigindo acentos) ou Portal["DESIGNACAO"]
- REDE / UF / CLUSTER → permitir:
  a) upload de tabela de referência (ex.: por CNL ou Central),
  b) definição manual (valor global),
  c) derivação por regra (ex.: regex, prefixos), com confirmação do usuário.

Exemplo de linha alvo (demonstração):
REDE: VIVO-SMP
UF: SE
CLUSTER: CLUSTER 3
Tipo de Rota: ITX
Central: MBCAJUD
Rótulos de Linha: MBCAJUD_TCE1C9
OPERADORA: EMBRATEL
Denominação: ROTA VIVO ITX EMBRATEL LC EM ESTÂNCIA (CN 79) - FTW

Importante: Sempre preservar 100% das linhas/colunas originais em tabelas “raw” no SQLite, mesmo que não participem da saída final. Nada pode sumir.

4) Armazenamento em SQLite (persistência e auditoria)
- Banco: data/app.db (configurável).
- Tabelas sugeridas:
  - imports (id, fonte, filename, sheet, file_hash, rows, cols, created_at)
  - raw_science e raw_portal com import_id como coluna (ou tabelas particionadas por import)
  - column_renames (import_id, original_name, normalized_name)
  - mappings (version_id, target_col, rule_json)
  - merged_results (version_id, row_id, REDE, UF, CLUSTER, Tipo_de_Rota, Central, Rotulos_de_Linha, OPERADORA, Denominacao, source_keys_json)
  - logs (timestamp, level, message, context_json)
- Transações: toda importação e merge devem ser transacionais.
- Versionamento: cada execução de merge cria um version_id para rastreabilidade.
- Não deletar dados: usar flags (is_active) se necessário; nunca perder histórico.

5) UI e Acessibilidade
- Stack recomendada: Streamlit com st-aggrid para filtros estilo Excel.
  Alternativa: Dash (Plotly) com DataTable, se justificar.
- Recursos mínimos:
  - Upload de arquivos (seleção de planilha/sheet no caso de XLSX).
  - Preview das tabelas importadas.
  - Assistente de Mapeamento (UI passo a passo).
  - Definição de chaves de junção (com validação e contagem de matches).
  - Grid interativo com filtros por coluna (texto, número, lista, data), ordenar, congelar colunas, pesquisa rápida.
  - Botões: Salvar no SQLite, Exportar CSV/XLSX, Baixar log, Baixar dicionário de mapeamento.
- Acessibilidade (WCAG):
  - Navegação completa por teclado.
  - Labels/aria-labels em inputs.
  - Contraste adequado.
  - Mensagens de erro descritivas.
  - Evitar depender apenas de cores para informação.

6) Qualidade de Dados e Segurança
- Normalização de nomes de colunas: trim, remoção de espaços duplos, preservação de acentos corretos; manter aliases originais. Corrigir casos de mojibake em labels (não no conteúdo).
- Tipos: inferir tipos, tratar datas (DD/MM/YYYY), inteiros, strings.
- Validação: relatório com contagem de nulos, duplicados por chaves definidas e estatísticas básicas.
- Erros: mensagens amigáveis, logging (níveis INFO/WARN/ERROR) e exceções tratadas.
- Segurança: sanitizar nomes de arquivos; sem execução de macros/código; nenhum dado enviado para fora (privacidade).
- Desempenho: suportar pelo menos 200 mil linhas por arquivo; usar leitura sob demanda e escrita em chunks no SQLite se necessário.

7) Lógica de Junção e Fusão de Colunas
- Permitir o usuário escolher chaves de junção (ex.: Central, Tipo de Rota, Sentido).
- Tipo de junção padrão: outer (para não “sumir” linhas).
- Para campos de saída com múltiplas fontes, usar COALESCE na ordem de precedência escolhida.
- Para Rótulos de Linha, se LABEL_E e LABEL_S existirem, permitir concatenação com separador " | ".
- Oferecer fuzzy matching opcional (ex.: Central com pequena variação), pedindo confirmação do usuário e produzindo relatório de confiança.

8) Entregáveis
1. Código completo do app.
2. requirements.txt (ex.: streamlit, pandas, sqlalchemy, openpyxl, xlrd, pyarrow, python-dotenv, unidecode, chardet, st-aggrid).
3. Instruções de execução claras no README.md (como instalar, streamlit run app.py, variáveis de ambiente).
4. Testes com pytest (unidade e integração):
   - Leitura de arquivos multi-formato
   - Normalização de colunas
   - Mapeamento configurável
   - Junção e merge sem perda
   - Escrita/leitura no SQLite
5. Exemplos: pastas samples/ com 2 arquivos de exemplo (mock) e mapping.json de exemplo.
6. Dockerfile opcional.

9) Boas Práticas (Obrigatórias)
- Código modular e tipado (type hints).
- Docstrings e comentários claros.
- Separar camadas: io/ (import/export), core/ (transform/join), db/ (persistência), ui/ (páginas/flows), utils/ (helpers).
- SQLAlchemy para a camada de DB (não usar apenas sqlite3).
- Config via .env e defaults seguros.
- UX em PT-BR.
- Nenhuma perda de dados: provar via testes e via contadores no relatório pós-merge.

10) Esqueleto Sugerido (arquivos)
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
  tests/
    test_readers.py
    test_merge.py
    test_db.py
    test_mapping.py
  samples/
    science_exemplo.xlsx
    portal_exemplo.xlsx
  requirements.txt
  README.md

11) Fluxo do Usuário
1) Upload dos dois arquivos → escolher planilha (XLSX) → preview.
2) Assistente de Mapeamento: escolher fontes para cada coluna final (com sugestões).
3) Definir chaves de junção + tipo de junção (padrão: outer).
4) Gerar Tabela Final → visualizar no grid com filtros e busca.
5) Salvar no SQLite (cria imports e version_id) → Exportar CSV/XLSX se desejar.
6) Relatório: contagem de linhas por fonte, linhas no resultado, colunas mapeadas, nulos e duplicatas.

12) Critérios de Aceite
- Consegue importar .xlsx (com seleção de sheet) e está preparado para .xls, .csv, .parquet.
- Nenhum dado some: todas as colunas e linhas ficam preservadas nas tabelas raw_* com import_id.
- Usuário consegue mapear para [REDE, UF, CLUSTER, Tipo de Rota, Central, Rótulos de Linha, OPERADORA, Denominação].
- Grid com filtros por coluna estilo Excel, ordenar, buscar.
- SQLite com versionamento de importações e merges; logs e auditoria.
- Acessibilidade: navegação por teclado e rótulos ARIA.
- Testes passando (leitura, mapeamento, merge, persistência).

13) Observações Técnicas
- Corrigir mojibake apenas nos nomes das colunas (labels), preservando dados brutos em raw.
- Para performance, usar chunksize em CSV e to_sql(if_exists="append").
- Em Streamlit, usar st-aggrid para filtros avançados (menus de filtro com operadores “contém”, “igual”, etc.).
- Persistir também o JSON de regras de mapeamento e chaves de junção usadas em cada versão.
- Exportações devem refletir exatamente o que o usuário vê no grid (após filtros aplicados), com opção de exportar resultado completo.

Agora gere o código completo do aplicativo com base nas especificações acima, incluindo requirements.txt, README.md com passo a passo, e testes com pytest. Escreva o código pronto para rodar localmente com "streamlit run app.py". Se fizer alguma suposição, explique claramente no README e torne-a configurável no UI.
