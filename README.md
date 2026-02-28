PS C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas> $env:PYTHONPATH = "."; py -m streamlit run app.py
>> 

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://192.168.0.75:8501

2026-02-28 13:12:44 [INFO] app.io.readers: CSV lido: 123972 × 29 (enc=latin-1)
2026-02-28 13:12:44 [INFO] app.core.normalize: Colunas normalizadas: 10 renomeadas
2026-02-28 13:12:49.503 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
2026-02-28 13:13:16 [INFO] app.io.readers: CSV lido: 123972 × 29 (enc=latin-1)
2026-02-28 13:13:16 [INFO] app.core.normalize: Colunas normalizadas: 10 renomeadas
C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\openpyxl\styles\stylesheet.py:237: UserWarning: Workbook contains no default style, apply openpyxl's default
  warn("Workbook contains no default style, apply openpyxl's default")
2026-02-28 13:13:27 [ERROR] app.ui.pages: Upload error: 'dict' object has no attribute 'fillna'
Traceback (most recent call last):
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\ui\pages.py", line 65, in _handle_upload
    df = read_file(io.BytesIO(raw), filename=uploaded_file.name, sheet=sheet)
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\io\readers.py", line 40, in read_file
    df = df.fillna("")
         ^^^^^^^^^
AttributeError: 'dict' object has no attribute 'fillna'
2026-02-28 13:13:27.234 Please replace `use_container_width` with `width`.

`use_container_width` will be removed after 2025-12-31.

For `use_container_width=True`, use `width='stretch'`. For `use_container_width=False`, use `width='content'`.
