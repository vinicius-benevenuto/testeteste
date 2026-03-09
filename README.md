Você é um desenvolvedor full‑stack sênior especialista em Streamlit, Python, Pandas, CSS e geração de arquivos (Excel/CSV/PPTX). Sua missão é corrigir a aplicação e implementar filtros por coluna idênticos aos do Excel, diretamente no cabeçalho da tabela. O resultado deve ser minimalista, profissional e 100% funcional.
========================

CONTEXTO E OBJETIVOS
========================


Corrigir: títulos que não aparecem; sidebar poluída; ícones quebrados; disposição dos botões de exportação.
Implementar: filtros no cabeçalho das colunas, iguais aos do Excel (ícone de filtro ao lado do nome, dropdown com valores únicos, busca e seleção única/múltipla).
Manter: dashboards e PPTX como estão visualmente (apenas prevenir erros).
Excel exportado: nível profissional (consultoria), com formatação premium.

=================================
2) REGRAS GERAIS DE IMPLEMENTAÇÃO

Código limpo, organizado, com comentários essenciais.
Nada de emojis, bandeirinhas, pontos de progresso ou indicadores coloridos na UI.
Minimalismo: tipografia limpa, bom contraste, sem poluição visual.
Compatível com Streamlit Cloud e execução local.
Todos os recursos devem funcionar na primeira execução (sem exceções ou warnings).

==================================
3) TÍTULOS APARENTES E COM CONTRASTE
Problema: títulos (H1, H2, H3) não aparecem por baixa relação de contraste com o fundo escuro.
Correção obrigatória:

Forçar cor branca (#FFFFFF) para h1–h6 quando o fundo for escuro.
Remover/ajustar qualquer CSS global que pinte “body, p, h1–h6” com tom escuro.
Garantir que st.title(), st.header(), st.subheader() e st.markdown com <h*> fiquem legíveis.
CSS mínimo exigido:
h1, h2, h3, h4, h5, h6 { color: #FFFFFF !important; }
Em containers com fundo claro, usar #0F1220 ou #111827 (sem comprometer contraste AA).

======================================
4) SIDEBAR MINIMALISTA (SEM BOTÕES/FLAGS)

Remover completamente botões vermelhos, “flags”, pontinhos de status, checkmarks, indicadores de etapa e quaisquer ícones/textos como “keyboard_double_arrow_left”.
A navegação deve ocorrer apenas clicando no nome da aba (texto clicável). Sem emojis.
Estilo discreto: fundo escuro (#0B1428), texto branco suave (#E6ECFF), sem bordas pesadas.
Hover sutil (leve aumento de brilho). Sem mudanças de cor “de etapa concluída”.
Se houver alternância de páginas, usar session_state e rótulos de abas como links/botões invisíveis (cursor:pointer) — sem ícones.

===================================================
5) FILTROS IGUAIS AO EXCEL – NO CABEÇALHO DA TABELA
Objetivo: filtros por coluna com UX igual ao Excel, integrados à tabela.
Requisitos funcionais:

Um ícone/indicador de filtro ao lado do nome da coluna (no header).
Ao clicar, abrir um dropdown com:
• Lista de valores únicos da coluna (atualizada conforme filtros ativos em outras colunas).
• Campo de busca para filtrar rapidamente a lista.
• Seleção múltipla por checkboxes e possibilidade de seleção única.
• Opções “Selecionar tudo / Limpar” como no Excel.
• Inclusão/Exclusão de vazios (exibir “(Blanks)”/“(Vazios)”).
Filtragem acumulativa entre colunas (AND lógico).
Atualização imediata da tabela ao alterar filtros.
Ordenação e paginação devem respeitar o conjunto filtrado.
Performance aceitável para dezenas/centenas de milhares de linhas (usar tipos otimizados, filtering no backend e virtualização no grid).

Decisão de tecnologia (obrigatória):

Implementar com AG Grid via streamlit-aggrid para obter UX “Excel-like” real no cabeçalho:
• header filter: ‘agSetColumnFilter’ (multi-select) com search box.
• floatingFilter: true.
• suppressMenuHide: false (menu/ícone visível apenas no hover, como no Excel).
• sideBar: false (não usar barra lateral do grid).
• enableCellTextSelection: true; ensureDomOrder: true.
• Manter o visual minimalista; sem emojis.

Comportamento esperado:

Exemplo: ao selecionar “AS-ITX_SIP” em uma coluna, apenas linhas com esse valor permanecem visíveis.
Filtros de diferentes colunas se acumulam corretamente.
O estado de filtros deve ser preservado em st.session_state ao navegar entre abas.

=================================================
6) BOTÕES DE EXPORTAÇÃO – POSICIONAMENTO E ESCOPO

“EXPORTAR CSV”, “EXPORTAR EXCEL” e “GERAR DASHBOARD PPTX” devem aparecer SOMENTE:
• abaixo da Tabela Final, e
• abaixo do Dashboard (último elemento visível da página).
Os dados exportados DEVEM refletir filtros e ordenações aplicadas no grid (exportar o dataset filtrado).

=====================================
7) EXCEL EXPORTADO – NÍVEL PROFISSIONAL
Formato profissional (via openpyxl ou xlsxwriter):

Cabeçalho congelado (freeze panes: primeira linha).
Zebra (linhas alternadas) com cores suaves (cinza claro #F7F8FA).
Fonte Calibri 11; cabeçalhos em semibold.
Bordas finas e consistentes; alinhamento adequado; números com formatação local.
Título da planilha no topo (mesclar células, título centralizado, data/hora da extração na direita).
Autoajuste de colunas (computar largura pela string mais longa).
Formatação condicional para destaques (quando aplicável).
Nome da aba coerente e “limpo”.
Compatível com Excel (sem estilos que quebrem em versões antigas).

==========================
8) PPTX – PREVENÇÃO DE ERROS

Resolver erros como: “Unknown format code 'd' for object of type 'str'”.
• Garantir cast correto antes de formatar (ex.: números vs. strings).
• Padronizar formatos (datas, números, percentuais) antes de inserir no slide.
Geração de PPTX deve completar sem exceções.
Manter visuais atuais (não alterar design aprovado).

=================================
9) ACESSIBILIDADE E CONSISTÊNCIA

Contraste mínimo WCAG AA para textos e títulos.
Tamanhos de fonte coerentes; evitar texto muito pequeno no grid.
Estados de foco/hover discretos e consistentes.
Nada de emojis, bandeiras, “pontinhos” ou indicadores intrusivos.

=============================
10) TESTES DE ACEITAÇÃO (QA)

Títulos visíveis em todas as páginas/containers escuros e claros.
Sidebar:
• Sem botões/flags/pontos/ícones quebrados (nada de textos tipo “keyboard_double_arrow_left”).
• Navegação apenas clicando no nome da aba.
Filtros Excel-like:
• Ícone de filtro no cabeçalho de cada coluna.
• Dropdown com busca e múltipla seleção.
• Selecionar “AS-ITX_SIP” deve mostrar exclusivamente linhas com esse valor.
• Filtros acumulam entre colunas e respeitam valores restantes.
• “(Vazios)” aparecem quando há NaN/None em coluna e podem ser incluídos/excluídos.
Exportações:
• CSV/XLSX exportam exatamente o subconjunto filtrado e ordenado.
• PPTX gera sem erros.
Responsividade e performance:
• Tabela fluida em 1366×768 e acima.
• Interação suave com >50k linhas (usar server-side/modelo eficiente quando necessário).

========================
11) ENTREGÁVEIS FINAIS

Código completo (Python + Streamlit) com a tabela AG Grid e filtros no cabeçalho.
CSS organizado para:
• Títulos com contraste,
• Sidebar minimalista,
• Ajustes visuais do grid (sem emojis).
Funções de exportação (CSV/XLSX/PPTX) atualizadas para usar o dataset filtrado.
Notas de implementação explicando o que foi alterado e por quê.

========================
12) RESTRIÇÕES/PROIBIÇÕES

Não usar emojis, bandeiras, bolinhas/pontos de status ou ícones de texto “quebrados”.
Não mover os botões de exportar para o topo; devem ficar no final.
Não alterar o visual dos dashboards e PPTX (apenas corrigir erros).
Não deixar CSS global escurecer textos de títulos.

==================
13) PADRÕES TÉCNICOS

Pandas: tratar tipos (astype), normalizar datas e números antes de formatar/exportar.
Grid (streamlit‑aggrid):
• defaultColDef: filter: 'agSetColumnFilter', floatingFilter:true, sortable:true, resizable:true.
• pagination:true; paginationAutoPageSize:true; rowSelection:'multiple' (se necessário).
• sideBar:false; suppressExcelExport:false (se for usar export nativo do grid, opcional).
Estado:
• Persistir estado de filtros/ordenação em st.session_state para navegação entre abas.
Segurança:
• Sanitizar nomes de arquivos; evitar sobrescrever; usar timestamps.

=================
14) CRITÉRIO DE ACEITE
Considerar entregue somente se:

Títulos aparecem claramente em todos os temas/containers.
Sidebar minimalista, sem botões/flags/pontos, navegação por clique no nome.
Filtros no cabeçalho idênticos ao Excel (ícone + dropdown com busca e multi‑seleção).
Selecionar “AS-ITX_SIP” exibe apenas linhas com esse valor.
Exportações refletem o conjunto filtrado.
PPTX gera sem exceções.
Código limpo e comentado, pronto para rodar em Streamlit Cloud.

FIM DO PROMPT
Se quiser, eu também gero o código pronto (com streamlit-aggrid) já configurado com:

filtros no cabeçalho (agSetColumnFilter + floatingFilter),
export de CSV/XLSX baseado no subconjunto filtrado,
CSS da sidebar minimalista e títulos com contraste,
correção do erro do PPTX.
