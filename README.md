PS C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas> $env:PYTHONPATH = "."; py -m streamlit run app.py
>> 

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://10.128.67.129:8501

2026-03-18 13:04:35 [INFO] app.db.repository: CNL seeds: 981 statements executados
2026-03-18 13:04:35 [INFO] app.db.repository: cn_to_uf: 67 registros carregados
2026-03-18 13:05:17 [INFO] app.io.readers: CSV lido: 124561 × 29 (enc=UTF-8-SIG)
2026-03-18 13:05:17 [INFO] app.core.normalize: Colunas normalizadas: 10 renomeadas
2026-03-18 13:05:20.027 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
2026-03-18 13:05:34 [INFO] app.io.readers: CSV lido: 124561 × 29 (enc=UTF-8-SIG)
2026-03-18 13:05:34 [INFO] app.core.normalize: Colunas normalizadas: 10 renomeadas
C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\openpyxl\styles\stylesheet.py:237: UserWarning: Workbook contains no default style, apply openpyxl's default
  warn("Workbook contains no default style, apply openpyxl's default")
2026-03-18 13:05:39 [INFO] app.io.readers: PORTAL_ROTAS_FIXA_NOVA.xlsx lido: 9743 × 34 (sheet=0)
2026-03-18 13:05:39.481 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
2026-03-18 13:05:39.481 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
2026-03-18 13:05:49 [INFO] app.io.readers: CSV lido: 124561 × 29 (enc=UTF-8-SIG)
2026-03-18 13:05:49 [INFO] app.core.normalize: Colunas normalizadas: 10 renomeadas
C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\openpyxl\styles\stylesheet.py:237: UserWarning: Workbook contains no default style, apply openpyxl's default
  warn("Workbook contains no default style, apply openpyxl's default")
2026-03-18 13:05:53 [INFO] app.io.readers: PORTAL_ROTAS_FIXA_NOVA.xlsx lido: 9743 × 34 (sheet=0)
2026-03-18 13:05:56 [INFO] app.io.readers: ARQUIVO_COMPLEMENTAR.xlsx lido: 25835 × 32 (sheet=0)
2026-03-18 13:05:56 [INFO] app.core.normalize: Colunas normalizadas: 1 renomeadas
2026-03-18 13:05:56.978 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
2026-03-18 13:05:56.995 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
2026-03-18 13:05:57.001 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
2026-03-18 13:06:01 [INFO] app.core.pipeline: [Science] Início: 124561 linhas
2026-03-18 13:06:01 [INFO] app.core.pipeline: [Science] Colunas: central='Central Interna' data_desativ='Data Desativação' id_rota='ID Rota'
2026-03-18 13:06:03 [INFO] app.core.pipeline: [Science] Após etapa 1 (Central M): 124561 → 39873
2026-03-18 13:06:06 [INFO] app.core.pipeline: [Science] Após etapa 2 (data desativação): 30153
2026-03-18 13:06:06 [INFO] app.core.pipeline: [Science] Arq3 carregado: 24328 rótulos únicos (col='Rótulos de Linha')
2026-03-18 13:06:07 [INFO] app.core.pipeline: [Science] Após etapa 3 (validação Arq3): 6665
2026-03-18 13:06:07 [INFO] app.core.pipeline: [Science] Finalizado: 6665 linhas de saída, 6665 enriquecidas, 23488 inconsistências
2026-03-18 13:06:07 [INFO] app.core.pipeline: [Portal] Início: 9743 linhas
2026-03-18 13:06:07 [INFO] app.core.pipeline: [Portal] Colunas: central='CENTRAL' label_e='LABEL_E' label_s='LABEL_S' cnl_ppi='CNL_PPI'
2026-03-18 13:06:08 [INFO] app.core.pipeline: [Portal] Arq3: 24328 rótulos (col='Rótulos de Linha')
2026-03-18 13:06:08 [INFO] app.core.pipeline: [Portal] Finalizado: 9743 linhas, 812 novas rotas, 0 inconsistências
2026-03-18 13:06:09 [INFO] app.core.pipeline: [Pipeline] Etapa 10: 12564 rotas do Arq3 sem match adicionadas
2026-03-18 13:06:10 [INFO] app.core.pipeline: [Pipeline] Concluído: Science=124561→6665 Portal=9743→9743 Arq3SemMatch=12564 Total=28972 | REDE corrigida=5315 | TipoRota preenchido=0 | UF via CNL_PPI: preenchido=9743 inválido=0 divergente=1256 | Inconsistências=119152
2026-03-18 13:12:30.480 Uncaught app execution
Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\exec_code.py", line 129, in exec_func_with_error_handling
    result = func()
             ^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\script_runner.py", line 689, in code_to_exec
    exec(code, module.__dict__)  # noqa: S102
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app.py", line 546, in <module>
    render_interactive_table(tf, selected_col=sel_col, height=520, component_key="tbl")
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\ui\interactive_table.py", line 339, in render_interactive_table
    st.markdown(
    ^^
NameError: name 'st' is not defined

