QUERO QUE VOCÊ AJA COMO UM DESENVOLVEDOR SÊNIOR ESPECIALIZADO EM PYTHON, TRATAMENTO DE DADOS EM GRANDE ESCALA E OTIMIZAÇÃO DE PERFORMANCE.

GERAR UM SCRIPT COMPLETO, MODULARIZADO, ROBUSTO E ALTAMENTE COMENTADO, PARA IDENTIFICAR OPERADORA POR PREFIXO, VALIDAR PORTABILIDADE (INCLUINDO TABELA DE RN1) E PRODUZIR RESULTADO.xlsx.

====================================================================
1) BLOCO INICIAL DE CONFIGURAÇÃO (EDITÁVEL)
====================================================================

O script deve começar com variáveis configuráveis:

CAMINHO_ARQUIVO_1 = "CAMINHO/DO/ARQUIVO_1.csv"   # bilhetagem (A_BUSCA, B_BUSCA ou NTL)
CAMINHOS_PREFIXOS = [
    "CAMINHO/DO/PREFIXO_FIXO.csv",
    "CAMINHO/DO/PREFIXO_MOVEL.csv",
    "CAMINHO/DO/PREFIXO_SME.csv",
    "CAMINHO/DO/PREFIXO_OUTRO.csv"
]

# PORTABILIDADE
CAMINHO_ARQUIVO_3 = "CAMINHO/DA/PORTABILIDADE.csv"
COLUNA_NUMERO_ATUAL = "numero"
COLUNA_OPERADORA_ATUAL = "operadora_atual"

# TABELA DE RN1 (ANEXO 5)
CAMINHO_ANEXO5_RN1 = "CAMINHO/DO/ANEXO5.csv"
COLUNA_RN1 = "RN1"
COLUNA_OPERADORA_RN1 = "Nome Fantasia"

# CHUNKS
CHUNKSIZE = 200000   # configurável

====================================================================
2) FONTES DE DADOS — REGRAS
====================================================================

------------------------
2.1 ARQUIVO 1: BILHETAGEM
------------------------
- Pode ser CSV (delimitador ;) ou XLSX.
- Deve conter A_BUSCA e B_BUSCA.
- OU pode conter NTL (equivalente às duas).
- SE EXISTIR NTL → usar somente NTL e ignorar A_BUSCA/B_BUSCA.
- Tratar tudo como string.
- Leitura obrigatória em chunks.

origem_numero deve registrar:
"A_BUSCA", "B_BUSCA", ou "NTL"

---------------------------------------------
2.2 BASES DE PREFIXOS (4 ARQUIVOS CSV ; )
---------------------------------------------
Colunas originais:
"# Nome da Prestadora;CNPJ da Prestadora;Código Nacional;Prefixo;Faixa Inicial;Faixa Final;Status"

Regras obrigatórias:
1) delimiter=';', encoding UTF-8 (fallback Latin-1)
2) remover caractere '#' do início das colunas
3) padronizar nomes das colunas para:
   nome_prestadora | cnpj_prestadora | codigo_nacional | prefixo | faixa_inicial | faixa_final | status
4) codigo_nacional e prefixo → STRING
5) faixa_inicial e faixa_final → INT
6) considerar somente status == 1
7) criar coluna fonte_prefixo = FIXO, MOVEL, SME, OUTRO
8) unificar as 4 tabelas em memória
9) criar índice por (codigo_nacional, prefixo)

---------------------------------------------
2.3 PORTABILIDADE (ARQUIVO 3)
---------------------------------------------
- pode ser CSV (;) ou XLSX
- colunas configuráveis
- tudo como string
- criar índice pelo número normalizado

---------------------------------------------
2.4 TABELA RN1 (ANEXO 5)
---------------------------------------------
O arquivo contém colunas como:
EOT, Nome Fantasia, Razão Social, CSP, Tipo de Serviço, ..., RN1, SPID

Regras:
- Usar RN1 como chave.
- Criar dicionário RN1 → Nome Fantasia (operadora).
- Usar essa tabela para identificar operadora na portabilidade.

====================================================================
3) NORMALIZAÇÃO DE NÚMEROS
====================================================================

Regras obrigatórias:
1) manter apenas dígitos
2) se começar com 55 → remover
3) garantir no mínimo 10 dígitos
4) extrair:
   ddd = primeiros 2 dígitos
   numero_local = restante
5) estacao = 4 últimos dígitos (int)
6) prefixo candidatos:
   - candidato A: numero_local[0:4]
   - candidato B: se começar com 9 → numero_local[1:5]

====================================================================
4) MATCH DE PREFIXO
====================================================================

Match válido quando:
- codigo_nacional == ddd
- prefixo == prefixo_cand
- faixa_inicial <= estacao <= faixa_final

Se houver múltiplos matches:
1) escolher o intervalo mais específico (faixa menor)
2) empatar → prioridade FIXO > MOVEL > SME > OUTRO
3) logar ambiguidades

Resultado: operadora_pelo_prefixo = nome_prestadora

====================================================================
5) VERIFICAÇÃO DE PORTABILIDADE (COM RN1)
====================================================================

Para cada número:
- buscar na tabela de portabilidade
- se encontrar RN1 → consultar operadora no Anexo 5
- comparar:
    se operadora_atual != operadora_pelo_prefixo → PORTADO
    senão → NÃO PORTADO
- se número não existir → "NÃO ENCONTRADO NA PORTABILIDADE"

====================================================================
6) SAÍDA FINAL — RESULTADO.xlsx
====================================================================

Colunas obrigatórias:
- numero_normalizado
- ddd
- prefixo_casado
- faixa_utilizada (faixa_inicial-faixa_final)
- fonte_prefixo
- operadora_pelo_prefixo
- operadora_atual
- status_portabilidade
- origem_numero (A_BUSCA | B_BUSCA | NTL)

====================================================================
7) ARQUITETURA DO CÓDIGO
====================================================================

O script deve conter, no mínimo, as funções:

carregar_arquivos()
normalizar_numero()
extrair_numeros()                 # leitura em chunks
unificar_bases_prefixos()
buscar_operadora_prefixo()
verificar_portabilidade()
gerar_resultado()
main()

====================================================================
8) PERFORMANCE — OBRIGATÓRIO
====================================================================

- leitura do arquivo 1 em chunks (CHUNKSIZE configurável)
- prefixos ficam 100% em memória
- prefixos pré-indexados
- portabilidade indexada
- liberar memória entre etapas
- escrever resultados parciais em CSV temporário (append)
- converter para RESULTADO.xlsx no final

====================================================================
9) LOGS E TRATAMENTO DE ERROS
====================================================================

- erros de arquivo inexistente
- erros de permissão
- colunas faltando → erro amigável
- logar:
  * tempo por etapa
  * volume processado
  * prefixos não encontrados
  * casos ambíguos
  * números não encontrados na portabilidade

====================================================================
ENTREGA FINAL
====================================================================

GERAR:
- código Python completo e altamente comentado
- modularizado
- capaz de rodar em grandes volumes sem esgotar memória
- resultado final em RESULTADO.xlsx
