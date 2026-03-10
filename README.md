PS C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas> $env:PYTHONPATH = "."; py -m streamlit run app.py
>> 

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://10.128.66.222:8501

Task exception was never retrieved
future: <Task finished name='Task-151' coro=<WebSocketProtocol13.write_message.<locals>.wrapper() done, defined at C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\tornado\websocket.py:1111> exception=WebSocketClosedError()>
Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\tornado\websocket.py", line 1113, in wrapper
    await fut
tornado.iostream.StreamClosedError: Stream is closed

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\tornado\websocket.py", line 1115, in wrapper
    raise WebSocketClosedError()
tornado.websocket.WebSocketClosedError
2026-03-10 09:12:02.624 Uncaught app execution
Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\exec_code.py", line 129, in exec_func_with_error_handling
    result = func()
             ^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\streamlit\runtime\scriptrunner\script_runner.py", line 689, in code_to_exec
    exec(code, module.__dict__)  # noqa: S102
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app.py", line 1106, in <module>
    render_interactive_table(
  File "C:\Users\40418843\Desktop\novo-modelo-controle-de-rotas-cadastradas\app\ui\interactive_table.py", line 589, in render_interactive_table
    st.markdown(
    ^^
NameError: name 'st' is not defined
