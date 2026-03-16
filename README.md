PS C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas> $env:PYTHONPATH = "."; py -m streamlit run app.py
>> 

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://192.168.0.75:8501

2026-03-16 09:33:17 [INFO] app.db.repository: CNL seeds: 981 statements executados
2026-03-16 09:33:17 [INFO] app.db.repository: cn_to_uf: 67 registros carregados
2026-03-16 09:34:09 [ERROR] __main__: Upload sci: Missing optional dependency 'xlrd'. Install xlrd >= 2.0.1 for xls Excel support Use pip or conda to install xlrd.
Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\compat\_optional.py", line 135, in import_optional_dependency
    module = importlib.import_module(name)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Program Files\Python312\Lib\importlib\__init__.py", line 90, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1324, in _find_and_load_unlocked
ModuleNotFoundError: No module named 'xlrd'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app.py", line 774, in _handle_upload
    df = read_file(_io.BytesIO(raw), filename=uploaded_file.name, sheet=sheet)
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\io\readers.py", line 40, in read_file
    result = pd.read_excel(buf, sheet_name=sheet_name, engine=engine,
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\excel\_base.py", line 495, in read_excel
    io = ExcelFile(
         ^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\excel\_base.py", line 1567, in __init__
    self._reader = self._engines[engine](
                   ^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\excel\_xlrd.py", line 45, in __init__
    import_optional_dependency("xlrd", extra=err_msg)
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\compat\_optional.py", line 138, in import_optional_dependency
    raise ImportError(msg)
ImportError: Missing optional dependency 'xlrd'. Install xlrd >= 2.0.1 for xls Excel support Use pip or conda to install xlrd.
