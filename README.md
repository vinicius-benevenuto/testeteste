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
