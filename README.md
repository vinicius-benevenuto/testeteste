## Vamos às modificações

### 1. Bloco IP

O bloco IP deve aparecer somente dentro do campo “CDSIP\_SPO\_PL / CDSIP\_SPO\_JG” em Dados VIVO.  
Não deve existir nenhuma informação extra sobre o bloco IP abaixo das linhas, comentários ou observações.  
O campo deve continuar sendo preenchido como já ocorre hoje, porém qualquer informação adicional exibida fora do campo deve ser removida, pois pode anular o formulário.

***

### 2. Descrição da rede no IPAM

O escopo não deve mais ser utilizado como descrição da rede reservada.  
A descrição correta deve ser composta pelo valor do RN1, seguido de um traço (-) e o nome da operadora.

Exemplo de descrição:  
RN1-XXXX - Nome da Operadora

Não utilizar escopo, textos automáticos ou descrições genéricas.

***

## No arquivo Excel

### 1. Aba “Diagrama de Interligação” – lógica de endereços IP

Os endereços IP exibidos nas tabelas que contêm informações de tráfego, endereço IP e netmask devem seguir a lógica abaixo.  
O valor base deve ser retirado do campo “CDSIP\_SPO\_PL / CDSIP\_SPO\_JG”, por exemplo:

PL: 10.11.130.16/28  
JG: 10.11.131.16/28

***

### 3.1. Tabela da esquerda (SipRouter)

Para cada bloco IP:

Considerar o endereço antes da barra (/).  
Somar +4 ao último octeto.

Exemplo:  
10.11.130.16/28 torna-se 10.11.130.20/28  
10.11.131.16/28 torna-se 10.11.131.20/28

A partir desse novo IP base, os tipos de tráfego devem ser distribuídos de forma incremental, somando +1 para cada novo tipo de tráfego, mantendo a mesma máscara.

Exemplo:  
LC = IP base  
LD15 + CNG = IP base +1  
Transporte = IP base +2  
Concentração = IP base +3  
E assim sucessivamente.

***

### 3.2. Dois SipRouters

Existem dois SipRouters distintos: CDSIP\_SPO\_PL e CDSIP\_SPO\_JG.

Devem existir duas tabelas à esquerda, uma abaixo da outra:  
A primeira representando o CDSIP\_SPO\_PL  
A segunda representando o CDSIP\_SPO\_JG

Cada tabela deve aplicar a lógica de cálculo de IP de forma independente, pois os blocos são diferentes.

***

### 3.3. Tabela da direita (lado remoto)

A lógica da tabela da direita é diferente.  
Não deve ser aplicado o incremento de +4.  
Deve ser aplicado apenas o incremento de +2.

O cálculo não deve ser feito sobre o bloco IP do formulário.  
O cálculo deve ser feito sobre o IP efetivamente reservado no IPAM.

Ou seja, o IP base vem do IPAM e sobre ele é somado +2.

***

## Ajustes no diagrama

Devem ser adicionados dois textos na imagem com os nomes dos SipRouters:  
CDSIP\_SPO\_PL  
CDSIP\_SPO\_JG

O SBC não deve mais aparecer no diagrama, nem como objeto visual nem como texto.

***

## Aba “Encaminhamento”

Os campos:  
DE A > B (Formato de entrega)  
DE B > A  
CODEC

não estão sendo exibidos para os tipos de tráfego:  
LC  
LD15 + CNG  
Transporte  
Concentração

Esses campos aparecem apenas para LDS/CSP + CNG e VC1.  
É necessário identificar a regra condicional incorreta e corrigir para que todos os tipos de tráfego exibam corretamente esses campos.

***

## Verificação de exports

A funcionalidade de verificação de exports não será mais utilizada.  
Toda lógica, validação ou exibição relacionada a exports deve ser removida.

***

## Organização dos formulários por operadora

Os formulários devem ser organizados por operadora.  
Deve ser possível visualizar todos os formulários preenchidos para uma mesma operadora.

Ao editar um formulário, o usuário deve ter a opção de criar uma nova versão.  
O histórico de versões deve ser mantido, exibindo versão 1, versão 2, versão 3 e assim por diante, cada uma com seus respectivos dados.

***

## Validação e preenchimento dos campos

Os campos do formulário não podem conter:  
n/a  
N/A  
não se aplica  
Não se aplica  
valores genéricos  
ou permanecer em branco.

Devem ser aplicadas validações rigorosas, incluindo campos obrigatórios, listas controladas, valores padrão técnicos e demais regras necessárias para garantir o preenchimento correto.

O objetivo é evitar preenchimentos incorretos por usuários do atacado e garantir a integridade dos dados técnicos.

O resultado esperado é um sistema consistente, padronizado, com cálculos de IP corretos, formulários organizados por operadora, versionamento funcional, diagramas limpos e dados confiáveis.
