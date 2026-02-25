Realize as alterações abaixo no arquivo de planilha seguindo exatamente todas as instruções. 
NÃO modifique nada além do solicitado.

====================================================================
1) ABA “Concentração”
====================================================================
- Excluir completamente esta aba do arquivo.

====================================================================
2) ABA “Plan NUM_Oper”
====================================================================
- Manter toda a estrutura existente: colunas, cabeçalhos, fórmulas, 
  larguras, bordas, mesclagens e formatações originais.
- Apenas limpar os valores preenchidos nas células.
- Não excluir colunas, linhas ou fórmulas.

--------------------------------------------------------------------
2.1 Tabela de Prefixos (Ref, Município, Área Local etc.)
--------------------------------------------------------------------
- A tabela deve começar com APENAS 1 linha visível.
- Qualquer linha adicional abaixo da primeira deve ser OCULTADA.
- Não excluir linhas — apenas ocultar as que estiverem vazias.

--------------------------------------------------------------------
2.2 Títulos da seção RN1 / EOT Local / EOT LDN/LDI / CSP / Código Especial / CNG / RN2
--------------------------------------------------------------------
Os títulos RN1, EOT Local, EOT LDN/LDI, CSP, Código Especial, CNG e RN2 devem:

- Ter altura fixa e igual entre todos.
- Ter largura fixa e igual entre todos.
- Manter mesma fonte, tamanho, bordas e alinhamento.
- NÃO podem crescer, reduzir ou se deformar para se ajustar ao conteúdo.
- NÃO podem ser redimensionados de forma automática.

Ou seja: **o tamanho dos TÍTULOS deve permanecer FIXO.**

--------------------------------------------------------------------
2.3 Células de Resposta (à direita dos títulos)
--------------------------------------------------------------------
As células de resposta devem:

- Se adaptar automaticamente ao conteúdo inserido.
- Podem quebrar linha quando necessário.
- Podem aumentar de ALTURA conforme o texto exigir.
- NÃO devem alterar a largura ou altura das células de título.
- Podem ser mescladas verticalmente se necessário para manter estética.
- As linhas internas não devem aparecer se não houver conteúdo.

--------------------------------------------------------------------
2.4 Preenchimento condicional dos campos:
--------------------------------------------------------------------

CSP:
- Se houver valor no formulário → preencher esse valor.
- Se NÃO houver → inserir “X” vermelho.

Código Especial:
- Se houver valor → preencher.
- Se não houver → inserir “X” vermelho.

CNG:
- Se houver valor → preencher.
- Se não houver → inserir “X” vermelho.

RN2:
- Se o valor for positivo ou válido → inserir ✔️ verde.
- Se não houver valor → inserir X vermelho.

RN1 e EOT Local / EOT LDN/LDI:
- Preencher conforme os valores do formulário.
- Se ausente, usar X vermelho.

====================================================================
3) ABA “Dados MTL”
====================================================================
A referência deve ocupar DUAS LINHAS com mesclagem vertical.

3.1 Estrutura por referência
- Para cada referência (ex.: 1), criar duas linhas consecutivas.
- Mesclar verticalmente a célula da referência para cobrir ambas.
- Repetir para todas as referências seguintes.

3.2 Coluna CN
- Inserir dois CNs por referência (mesmo se forem iguais).
- Cada CN deve ocupar uma das duas linhas.

3.3 Campo ID
- Deixar em branco.

3.4 Campo MTL
- Linha superior: nome da operadora.
- Linha inferior: “VIVO”.

3.5 Ponta Vivo
- Usar o endereço informado em DADOS VIVO.
- Repetir nas duas linhas da referência.

3.6 Ponta Operadora
- Usar o endereço informado em DADOS OPERADORA.
- Repetir nas duas linhas da referência.

3.7 Deixar em branco:
- Designador do circuito
- Provedor
- BGP Vivo
- BGP Operadora
- Observações

====================================================================
4) ABA “Parâmetros de Programação”
====================================================================
- Todos os “X” vermelhos devem ser substituídos por “✔️” verdes.
- Não alterar nenhum outro elemento.

====================================================================
5) ABA “Versões”
====================================================================
- Somente as linhas que possuem conteúdo devem permanecer visíveis.
- TODAS as linhas vazias devem ser OBLIGATORIAMENTE OCULTADAS.

--------------------------------------------------------------------
5.1 Critério para linha vazia
--------------------------------------------------------------------
Uma linha deve ser considerada vazia quando:
- TODAS as células estiverem sem texto,
- sem números,
- sem espaços,
- sem fórmulas com resultado vazio,
- sem qualquer caractere.

Se a linha for vazia → **OCULTAR**.  
Se tiver qualquer conteúdo → **manter visível**.

--------------------------------------------------------------------
5.2 Centralização
--------------------------------------------------------------------
- Todo o conteúdo das células visíveis deve estar CENTRALIZADO horizontalmente.
- Aplicar em: Versão, Data, Responsável Eng. ITX, Responsável Gestão ITX,
  Escopo, CN, Áreas Locais e ATA.

====================================================================
6) ABA “Diagrama de Interligação”
====================================================================

--------------------------------------------------------------------
6.1 Alinhamento dos títulos (SBCs)
--------------------------------------------------------------------
- Cada título (SBC) deve iniciar EXATAMENTE na mesma coluna inicial da tabela abaixo.
- Ajustar posicionamento para eliminar qualquer deslocamento para a direita ou esquerda.

--------------------------------------------------------------------
6.2 Posição da imagem
--------------------------------------------------------------------
- Mover a imagem para baixo.
- Garantir que não encoste, nem sobreponha as tabelas.

--------------------------------------------------------------------
6.3 Separação das tabelas
--------------------------------------------------------------------
- Remover qualquer linha, borda, traço ou elemento que conecte visualmente as tabelas.
- Deve existir apenas um ESPAÇO EM BRANCO limpo entre elas.

--------------------------------------------------------------------
6.4 Linhas das tabelas (dinâmicas)
--------------------------------------------------------------------
- O número de linhas depende dos tipos de tráfego escolhidos pelo usuário.
- Apenas linhas com conteúdo devem permanecer visíveis.
- Linhas sem conteúdo devem ser OCULTADAS (nunca excluídas).

--------------------------------------------------------------------
6.5 Campo NET MASK
--------------------------------------------------------------------
- NET MASK é valor único por tabela.
- Mesclar verticalmente a célula para aparecer apenas uma vez.
- A mesclagem deve cobrir APENAS as linhas realmente visíveis.
- Não repetir NET MASK linha a linha.

====================================================================
INSTRUÇÕES FINAIS (OBRIGATÓRIAS)
====================================================================
- Não alterar nenhuma aba além das mencionadas.
- Não modificar larguras, bordas, estilos ou formatações que não tenham sido solicitadas.
- Não excluir colunas ou fórmulas.
- Garantir alinhamento perfeito, consistência visual e conformidade total com todas as regras.
- Executar todas as etapas com precisão absoluta.
