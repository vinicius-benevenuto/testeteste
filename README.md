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
