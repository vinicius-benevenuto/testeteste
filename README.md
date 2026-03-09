
Você é um desenvolvedor full-stack sênior especialista em Streamlit, Python, CSS, UX minimalista e criação de dashboards profissionais. Sua missão é corrigir todos os problemas da aplicação e implementar filtros idênticos aos do Excel. A aplicação deve funcionar perfeitamente, com layout consistente, minimalista e profissional.
==== 1. CORRIGIR TÍTULOS QUE NÃO APARECEM ====
Os títulos (H1, H2, H3) não aparecem porque estão sendo estilizados por algum CSS global que deixa o texto muito escuro sobre fundo escuro. Corrigir da seguinte forma:

Forçar a cor dos títulos para branco (#ffffff) em qualquer seção com fundo escuro.
Remover qualquer CSS que altere a cor do texto de todos os elementos globalmente.
Garantir que markdown, st.title(), st.header(), st.subheader() e HTML custom aparecerão claramente.

Implementação:

Aplicar CSS que defina:
h1, h2, h3, h4, h5, h6 { color: #ffffff !important; }
Revisar todo o CSS existente e remover regras conflitantes.

==== 2. SIDE BAR MINIMALISTA (SEM FLAGS, SEM TEXTO ESTRANHO) ====
A sidebar atualmente mostra textos como “keyboard_double_arrow_left”. Isso é incorreto.
Correções obrigatórias:

Remover qualquer tentativa de usar ícones por texto literal.
Usar ícone unicode minimalista, como “⟨⟨” ou “≡”.
O botão deve recolher/expandir a sidebar sem quebrar o layout.
Sidebar deve ser discreta, clean, com cores coerentes e sem marcações vermelhas ou flags.
Manter tipografia suave e clean.

==== 3. IMPLEMENTAR FILTROS IGUAIS AOS DO EXCEL ====
Os filtros devem funcionar como no Excel:

Em cada coluna, deve existir um dropdown com os valores únicos.
Ao selecionar um valor, somente as linhas contendo esse valor são exibidas.
Deve haver possibilidade de selecionar múltiplos valores (checkboxes).
Os valores no dropdown devem vir sempre da coluna filtrada no momento (respeitar filtros ativos).

Comportamento desejado:
Exemplo: Se o usuário seleciona “AS-ITX_SIP” em qualquer coluna, somente essas linhas aparecem na tabela final.

Implementar com:
• st.multiselect()
• Valores únicos da coluna (df[column].unique())
• Aplicar filtro incremental conforme cada seleção
• Garantir performance mesmo com milhares de linhas

==== 4. LAYOUT DOS EXPORTS ====

Os botões EXPORTAR CSV, EXPORTAR EXCEL e GERAR PPTX devem ficar sempre abaixo:
• da Tabela Final
• e abaixo do Dashboard
Eles devem ser o último elemento visível da página.

==== 5. EXCEL PROFISSIONAL ====
O Excel exportado deve ter nível profissional:

Cabeçalho congelado
Linhas alternadas (zebra)
Fonte Calibri 11
Título centralizado elegante
Bordas completas
Autoajuste de colunas
Formatação condicional quando necessário
Compatível com openpyxl

==== 6. GARANTIA DE FUNCIONAMENTO ====
Você deve revisar todo o código e garantir:

Nenhum erro na geração do Excel
Nenhum erro na geração do PPTX
Nenhum erro no CSV
Nenhum ícone quebrado
Nenhuma sidebar exibindo texto indevido
Títulos sempre visíveis

Tudo deve funcionar na primeira execução.
==== 7. ENTREGA ====
Você deve entregar:

Código final corrigido
CSS completo e organizado
Explicação do que foi alterado
Garantia de que filtros estilo Excel funcionam
Garantia de que títulos e sidebar estão corrigidos
