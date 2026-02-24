==================================================
MELHORIA 1 — ABA “DIAGRAMA DE INTERLIGAÇÃO”
UNIFICAÇÃO DAS NET MASK
==================================================
1) Nas duas tabelas da aba "Diagrama de Interligação" que contêm as colunas:
   - Tráfego
   - Endereço IP
   - NET MASK

   A coluna “NET MASK” deve funcionar da mesma forma que fizemos anteriormente com:
   - CN da Vivo
   - CN da Operadora
   - POI/PPI
   - Ponta A / Ponta B
   - Área Local

2) REGRAS:
   a) A NET MASK deve aparecer apenas uma vez por tabela.
   b) A célula que contém a NET MASK deve ficar centralizada verticalmente na coluna.
   c) Todas as demais células da coluna NET MASK naquela tabela devem existir, porém devem estar invisíveis.
   d) Não repetir a NET MASK linha a linha.

3) Se houver duas tabelas lado a lado (como no modelo), aplicar esta regra separadamente em cada tabela.

==================================================
MELHORIA 2 — LINHAS COLOREADAS ENTRE AS TABELAS
==================================================
Entre as duas tabelas (lado esquerdo e lado direito) existem linhas coloridas (visualmente destacadas).  
Nestas linhas **não pode existir absolutamente nenhum conteúdo**.

REGRAS:
1) As linhas coloridas devem permanecer completamente vazias.
2) Nenhum texto, nenhum dado, nenhuma NET MASK, nenhum Tráfego, nenhum Endereço IP pode aparecer nelas.
3) Se houver qualquer lógica de preenchimento automático, ignorar completamente essas linhas.
4) As linhas coloridas servem apenas como "separadores visuais" e não fazem parte das tabelas.

==================================================
MELHORIA 3 — POSICIONAMENTO DA IMAGEM
==================================================
No final da aba “Diagrama de Interligação”, existe uma IMAGEM (diagrama).

Atualmente, a imagem está surgindo por cima das últimas informações (endereços, NET MASK, etc.).

REGRAS DE POSICIONAMENTO:
1) A imagem deve sempre ser posicionada ABAIXO da última linha de informações.
2) Garantir que haja espaço vertical suficiente antes de inserir a imagem.
3) Nunca permitir que a imagem sobreponha texto, tabelas, cabeçalhos ou linhas de dados.
4) Inserir a imagem apenas quando todas as tabelas estiverem completamente renderizadas e finalizadas.
5) Se necessário, criar um espaço inferior adicional antes de posicionar a imagem.

==================================================
MELHORIA 4 — ABA “CONCENTRAÇÃO”
OCULTAR LINHAS VAZIAS
==================================================
Na aba “Concentração”, linhas vazias não devem aparecer.

REGRAS:
1) Qualquer linha cuja todas as colunas estejam vazias deve ser removida ou ocultada.
2) Linhas parcialmente vazias permanecem — somente as TOTALMENTE vazias são removidas.
3) A tabela final da aba “Concentração” deve conter APENAS linhas com conteúdo.
4) Nunca exibir múltiplas linhas vazias consecutivas como separação.
5) A remoção de linhas vazias não deve quebrar a formatação nem a ordem lógica da aba.

==================================================
STATUS
==================================================
Após aplicar essas melhorias, retornar:
“Melhorias aplicadas: Diagrama de Interligação (NET MASK unificada, linhas coloridas vazias, imagem reposicionada) e Concentração (linhas vazias removidas). Aguardando próximos ajustes.”
