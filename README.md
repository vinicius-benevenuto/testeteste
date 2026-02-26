Realize as alterações abaixo no arquivo de planilha seguindo exatamente todas as instruções.
NÃO modifique nada além do solicitado. Priorize precisão, consistência visual e validações obrigatórias.

====================================================================
ABA “Concentração”
====================================================================
- Excluir completamente esta aba do arquivo.

====================================================================
ABA “Plan NUM_Oper”
====================================================================
- Manter toda a estrutura existente: colunas, cabeçalhos, fórmulas, larguras, bordas, mesclagens e formatações originais.
- Apenas limpar os valores preenchidos nas células.
- Não excluir colunas, linhas, mesclagens ou fórmulas.

--------------------------------------------------------------------
Tabela de Prefixos (Ref, Município, Área Local etc.)
--------------------------------------------------------------------
- A tabela deve começar com APENAS 1 (uma) linha de entrada visível.
- Qualquer linha adicional abaixo da primeira deve ser OCULTADA (não excluir).

--------------------------------------------------------------------
Títulos RN1 / EOT Local / EOT LDN/LDI / CSP / Código Especial (ou “Código Postal” no seu modelo) / CNG / RN2
--------------------------------------------------------------------
- Todas as células de TÍTULO devem ter TAMANHO PADRÃO FIXO e uniforme:
  - Mesma ALTURA entre todos os títulos.
  - Mesma LARGURA entre todos os títulos.
  - Mesma formatação (fonte, tamanho, bordas, cores e alinhamento).
  - NÃO podem aumentar, diminuir ou se deformar em função do conteúdo das respostas.
  - Permanecem estáticas (fixas) independentemente do texto das respostas.

--------------------------------------------------------------------
Células de RESPOSTA (à direita dos títulos)
--------------------------------------------------------------------
- As células de RESPOSTA devem se adaptar ao conteúdo (e NUNCA os títulos):
  - Quebrar linha automaticamente quando necessário.
  - Podem aumentar apenas em ALTURA, conforme o conteúdo exigir.
  - NÃO alterar a LARGURA nem a ALTURA das células de TÍTULO.
  - Se houver múltiplas linhas aparentes, mesclar verticalmente quando apropriado para manter estética limpa.
  - Garantir alinhamento visual e espaçamento uniforme entre título e resposta.

--------------------------------------------------------------------
Indicadores (substitui regras anteriores)
--------------------------------------------------------------------
- CSP, CNG e RN2 devem ser marcadores binários:
  - Se a condição/valor for válido/positivo → inserir ✔️ VERDE.
  - Se ausente/negativo/inválido → inserir X VERMELHO.
- RN1, EOT Local e EOT LDN/LDI:
  - Preencher conforme os valores do formulário; caso ausente, aplicar X VERMELHO.
- Não escrever textos literais nos campos definidos como indicadores, a menos que explicitamente solicitado.
  
====================================================================
ABA “Versões”
====================================================================
- Somente as linhas com conteúdo devem permanecer visíveis.
- TODAS as linhas vazias devem ser OBRIGATORIAMENTE OCULTADAS (não excluir).

  Critério de linha vazia
- Considerar uma linha vazia quando TODAS as células da linha estiverem sem:
  texto, números, espaços, caracteres ocultos e fórmulas com resultado visível.
- Linha vazia → OCULTAR. Linha com conteúdo → MANTER visível.

====================================================================
FORMULÁRIO — “Parâmetros Técnicos – Engenharia”
====================================================================
  Layout (seis títulos em GRID 3×2)
- A seção possui 6 (seis) títulos (ex.: “Dados do tratamento das chamadas” + outros).
- Reorganizar em dois andares:
  - Linha superior: 3 títulos lado a lado (colunas de largura igual).
  - Linha inferior: 3 títulos lado a lado, alinhados exatamente sob os de cima.
- Manter:
  - Mesmas alturas de linhas para os dois andares.
  - Mesmas larguras de coluna entre os seis blocos.
  - Margens e espaçamentos consistentes (espaço uniforme horizontal e vertical).

    Caixinhas para selecionar (checkboxes) — padronização forte
- Usar CAIXAS DE SELEÇÃO reais (controles) — não caracteres simulados.
- Tamanho único e consistente (ex.: ~14×14 px), ancoradas à célula, “mover e redimensionar com células”.
- Estilo VISUAL padronizado para TODO o formulário:
  - Estado MARCADO: mesmo padrão do “Formulário Atacado” (usar o mesmo tipo de controle e o mesmo visual; se o tema do arquivo usar preenchimento azul ao marcar, aplicar o mesmo; se for ✔, aplicar ✔ consistente).
  - Estado DESMARCADO: visual limpo e uniforme, sem variações.
- Alinhar os checkboxes ao centro de cada opção; manter o texto da opção alinhado e sem quebras incorretas.
- Não permitir caixas parcialmente preenchidas/indeterminadas.

    Regras de obrigatoriedade (antes de salvar)
- É OBRIGATÓRIO responder TODAS as questões do formulário.
- Não aceitar: “N/A”, “NA”, “n/a”, “-”, espaços vazios ou qualquer variação que indique ausência de resposta.
- Ao tentar salvar:
  - Validar se cada um dos 6 títulos tem pelo menos uma seleção válida (conforme a regra de negócio de cada um).
  - Se faltar resposta em qualquer título, BLOQUEAR o salvamento e exibir mensagem: 
    “Preencha todas as respostas da seção Parâmetros Técnicos – Engenharia antes de salvar.”

    Princípio estrutural (títulos fixos, respostas adaptáveis)
- Os BLOCOS DE TÍTULO (cabeçalhos dos 6 itens) TÊM TAMANHO FIXO e uniforme (altura e largura).
- As ÁREAS DE RESPOSTA (onde ficam as caixinhas/opções) se adaptam ao conteúdo:
  - Podem quebrar linha e crescer em ALTURA quando necessário.
  - NUNCA devem alterar a LARGURA ou ALTURA dos títulos.
  - Se houver múltiplas linhas, mesclar verticalmente as células de resposta apenas para manter estética e eliminar linhas divisórias indesejadas.

====================================================================
INSTRUÇÕES FINAIS (OBRIGATÓRIAS)
====================================================================
- Não alterar qualquer outra aba, estrutura, formatação, valores ou elementos não mencionados neste prompt.
- Não excluir colunas, linhas ou fórmulas.
- Preservar o tema/cores do arquivo e os alinhamentos já definidos, exceto quando explicitamente solicitado acima.
- Garantir: alinhamento perfeito, espaçamento limpo, consistência visual e conformidade total com todas as regras.
- Executar todas as etapas com precisão absoluta.
