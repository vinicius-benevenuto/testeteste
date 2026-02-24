Realize as alterações abaixo no arquivo de planilha seguindo exatamente todas as instruções. Não modifique nada além do solicitado.

====================================================================
1) ABA “Concentração”
====================================================================
- Excluir completamente esta aba do arquivo.

====================================================================
2) ABA “Plan NUM_Oper”
====================================================================
- Manter toda a estrutura existente: colunas, cabeçalhos, fórmulas, larguras e formatação.
- Apenas limpar os valores preenchidos nas células.
- Não excluir colunas, linhas, mesclagens ou fórmulas.

2.1 Tabela de Prefixos (Ref, Município, Área Local etc.)
- A tabela deve começar com apenas UMA linha visível.
- Qualquer linha adicional deve ser OCULTADA.
- Não excluir linhas; apenas ocultar as que estiverem vazias.

2.2 Padronização dos títulos (RN1, EOT Local, EOT LDN/LDI, CSP, Código Especial, CNG, RN2)
- Todas as células de TÍTULO devem ter:
  - Altura fixa e igual entre todas.
  - Largura fixa e igual entre todas.
  - Mesma formatação, mesma fonte e mesmo alinhamento.
  - Nenhuma célula deve ser maior, mais larga ou mais alta que as demais.

2.3 Células de resposta
- As células de RESPOSTA (à direita dos títulos):
  - Devem se adaptar automaticamente ao conteúdo.
  - Podem quebrar linha quando necessário.
  - Podem aumentar de altura conforme o conteúdo exigir.
  - NUNCA devem alterar a largura ou o tamanho das células de título.

- Todo o bloco deve ter alinhamento perfeito entre títulos e respostas.

====================================================================
3) ABA “Dados MTL”
====================================================================
A referência deve ocupar duas linhas com mesclagem vertical.

3.1 Referências
- Para cada referência (ex.: 1):
  - Criar duas linhas consecutivas.
  - Mesclar verticalmente a célula da referência, cobrindo as duas linhas.
  - Repetir esse comportamento para todas as referências (2, 3, 4 etc.).

3.2 CN
- Inserir dois CNs por referência (mesmo que iguais).
- Cada CN deve ocupar uma linha.

3.3 Campo ID
- Deixar em branco.

3.4 Campo MTL
- Linha superior da referência: nome da operadora.
- Linha inferior da referência: “VIVO”.

3.5 Ponta Vivo
- Usar o endereço informado em DADOS VIVO.
- Repetir nas duas linhas da referência.

3.6 Ponta Operadora
- Usar o endereço informado em DADOS OPERADORA.
- Repetir nas duas linhas da referência.

3.7 Campos a deixar em branco:
- Designador do circuito
- Provedor
- BGP Vivo
- BGP Operadora
- Observações

====================================================================
4) ABA “Parâmetros de Programação”
====================================================================
- Todos os “X” vermelhos devem ser substituídos por “✔️” verde.
- Não alterar nenhum outro conteúdo da aba.

====================================================================
5) ABA “Versões”
====================================================================
- Somente as linhas que possuem conteúdo devem permanecer visíveis.
- Todas as linhas vazias devem ser OCULTADAS.
- Não excluir linhas.

5.1 Centralização
- TODO o conteúdo desta aba deve estar CENTRALIZADO horizontalmente em suas células:
  - Versão
  - Data
  - Responsável Eng. de ITX
  - Responsável Gestão de ITX
  - Escopo
  - CN
  - Áreas Locais
  - ATA

====================================================================
6) ABA “Diagrama de Interligação”
====================================================================
6.1 Alinhamento dos títulos (SBCs)
- Cada título (SBC) deve iniciar exatamente na MESMA COLUNA inicial da tabela logo abaixo.
- Ajustar para que fiquem alinhados horizontalmente e não deslocados para a direita ou esquerda.

6.2 Posição da imagem
- Mover a imagem para baixo, garantindo:
  - Não sobrepor tabelas.
  - Não encostar nas tabelas.
  - Manter espaçamento limpo acima e abaixo.

6.3 Separação entre tabelas
- Remover QUALQUER linha, borda, traço, conector ou elemento visual ligando as tabelas.
- Entre as tabelas deve existir somente um espaço completamente em branco.

6.4 Linhas das tabelas
- A quantidade de linhas depende dos tipos de tráfego escolhidos pelo usuário.
- Apenas linhas com conteúdo devem permanecer visíveis.
- Linhas vazias devem ser OCULTADAS (nunca excluídas).

6.5 Coluna NET MASK
- O valor de NET MASK é único por tabela.
- Mesclar verticalmente a célula de NET MASK, cobrindo SOMENTE as linhas visíveis.
- O valor de NET MASK deve aparecer uma única vez por tabela.

====================================================================
INSTRUÇÕES FINAIS
====================================================================
- Não alterar qualquer outra aba, estrutura, formatação, valores ou elementos não mencionados neste prompt.
- Preserve formatos, alinhamentos, bordas e estilos existentes, exceto quando explicitamente solicitado.
- Garantir consistência, organização visual e total conformidade com todas as regras acima.
