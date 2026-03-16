PS C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas> $env:PYTHONPATH = "."; py -m streamlit run app.py
>> 

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://192.168.0.75:8501

2026-03-16 09:44:29 [INFO] app.db.repository: CNL seeds: 981 statements executados
2026-03-16 09:44:29 [INFO] app.db.repository: cn_to_uf: 67 registros carregados
2026-03-16 09:45:09 [ERROR] __main__: Upload sci: Unsupported format, or corrupt file: Expected BOF record; found b'<html>\n<'
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
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\excel\_xlrd.py", line 46, in __init__
    super().__init__(
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\excel\_base.py", line 573, in __init__
    self.book = self.load_workbook(self.handles.handle, engine_kwargs)
                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\excel\_xlrd.py", line 63, in load_workbook
    return open_workbook(file_contents=data, **engine_kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\xlrd\__init__.py", line 172, in open_workbook
    bk = open_workbook_xls(
         ^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\xlrd\book.py", line 79, in open_workbook_xls
    biff_version = bk.getbof(XL_WORKBOOK_GLOBALS)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\xlrd\book.py", line 1284, in getbof
    bof_error('Expected BOF record; found %r' % self.mem[savpos:savpos+8])
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\xlrd\book.py", line 1278, in bof_error
    raise XLRDError('Unsupported format, or corrupt file: ' + msg)
xlrd.biffh.XLRDError: Unsupported format, or corrupt file: Expected BOF record; found b'<html>\n<'
