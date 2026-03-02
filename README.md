PS C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas> $env:PYTHONPATH = "."; py -m streamlit run app.py
>> 

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://192.168.0.75:8501

2026-03-02 08:02:09 [INFO] app.db.repository: CNL seeds: 30 statements executados
2026-03-02 08:02:10.657 Uncaught app execution
Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\exec_code.py", line 129, in exec_func_with_error_handling
    result = func()
             ^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\script_runner.py", line 689, in code_to_exec
    exec(code, module.__dict__)  # noqa: S102
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app.py", line 72, in <module>
    from app.ui.pages import (
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\ui\__init__.py", line 5, in <module>
    from app.ui.pages import (
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\ui\pages.py", line 16, in <module>
    from app.db.session import session_scopeA
ImportError: cannot import name 'session_scopeA' from 'app.db.session' (C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\db\session.py)
