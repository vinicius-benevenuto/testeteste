VOCÊ É UM ESPECIALISTA EM ENGENHARIA E SANEAMENTO DE DADOS.
Temos três bases já carregadas:
Science
Portal de Cadastros
Arquivo 3
Neste momento, execute processamento APENAS na base SCIENCE.
OBJETIVO: Realizar limpeza, validação e filtragem para manter exclusivamente rotas externas válidas.
ETAPA 1 – FILTRO POR CENTRAL
Analisar a coluna "Central".
Manter apenas registros cuja Central comece com a letra "M".
Excluir registros:
Que não comecem com "M"
Que estejam vazios
Que sejam nulos
Esse filtro deve ser aplicado primeiro.
ETAPA 2 – VALIDAÇÃO DA COLUNA "data desativação"
Após aplicar o filtro de Central:
Manter registros quando:
O campo estiver vazio (null ou em branco)
O campo for igual a 1899
O campo for igual a 1900
Excluir registros quando:
A data for válida
E for inferior à data atual (hoje)
Regras técnicas:
Considerar a data atual do sistema no momento da execução.
Padronizar formato da data antes da comparação.
Não alterar outras colunas.
Após essa etapa, a base deve conter apenas rotas ativas.
ETAPA 3 – CONSIDERAR SOMENTE ROTAS EXTERNAS
Para cada registro restante:
Construção da chave de validação:
Pegar o valor da coluna "Central".
Extrair os 7 primeiros caracteres.
Concatenar com "_".
Concatenar com o valor da coluna "Rota".
Formato final da chave: <7 primeiros caracteres da Central>_
Validação:
Procurar essa chave na coluna "Rótulos de Linha".
Se a chave existir exatamente igual → MANTER o registro.
Se não existir → EXCLUIR o registro.
Regras obrigatórias:
Remover espaços antes e depois dos campos.
Padronizar texto (tudo maiúsculo ou tudo minúsculo) antes da comparação.
A comparação deve ser exata.
Não modificar outras colunas.
REGRAS GERAIS DE EXECUÇÃO
Executar as etapas na ordem descrita.
Cada etapa deve usar o resultado da etapa anterior.
Não processar Portal de Cadastros nem Arquivo 3 neste momento.
Não criar colunas permanentes adicionais, apenas variáveis temporárias se necessário.
SAÍDA ESPERADA
Retornar obrigatoriamente:
Total de registros iniciais
Total após filtro de Central
Total após filtro de data desativação
Total após validação de rotas externas
Total geral removido
Percentual de redução da base
Se houver inconsistências de formato, reportar antes de excluir registros.