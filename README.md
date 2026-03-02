❌ Erro: build_merged_df() got an unexpected keyword argument 'cn_to_uf_map'

2026-03-02 11:46:53 [INFO] app.io.readers: PORTAL_ROTAS_FIXA 01.xlsx lido: 9550 × 34 (sheet=0)
2026-03-02 11:47:04 [INFO] app.io.readers: Arquivo 03.xlsx lido: 25835 × 32 (sheet=0)
2026-03-02 11:47:04 [INFO] app.core.normalize: Colunas normalizadas: 1 renomeadas
2026-03-02 11:47:13 [ERROR] app.ui.pages: Merge error: build_merged_df() got an unexpected keyword argument 'cn_to_uf_map'
Traceback (most recent call last):
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\ui\pages.py", line 213, in render_merge_page
    merged, report = build_merged_df(
                     ^^^^^^^^^^^^^^^^
TypeError: build_merged_df() got an unexpected keyword argument 'cn_to_uf_map'
