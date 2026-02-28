PS C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas> $env:PYTHONPATH = "."; py -m streamlit run app.py
>> 

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://192.168.0.75:8501

2026-02-28 13:02:25 [INFO] app.db.repository: CNL seeds: 30 statements executados
2026-02-28 13:02:25.394 Uncaught app execution
Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\exec_code.py", line 129, in exec_func_with_error_handling
    result = func()
             ^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\script_runner.py", line 689, in code_to_exec
    exec(code, module.__dict__)  # noqa: S102
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app.py", line 47, in <module>
    _auto_load_seeds()
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app.py", line 45, in _auto_load_seeds
    repo.load_cn_to_uf_csv(str(uf_csv))
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\db\repository.py", line 62, in load_cn_to_uf_csv
    df = pd.read_csv(csv_path, dtype=str).fillna("")
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1026, in read_csv
    return _read(filepath_or_buffer, kwds)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 620, in _read
    parser = TextFileReader(filepath_or_buffer, **kwds)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1620, in __init__
    self._engine = self._make_engine(f, self.engine)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1898, in _make_engine
    return mapping[engine](f, **self.options)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\c_parser_wrapper.py", line 93, in __init__
    self._reader = parsers.TextReader(src, **kwds)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "pandas/_libs/parsers.pyx", line 581, in pandas._libs.parsers.TextReader.__cinit__
pandas.errors.EmptyDataError: No columns to parse from file
