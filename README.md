  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1511, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 919, in full_dispatch_request
    rv = self.handle_user_exception(e)
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 917, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 902, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\vivo_hub\auth.py", line 25, in wrapped
    return redirect(url_for("login"))
                    ^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\helpers.py", line 232, in url_for
    return current_app.url_for(
           ^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1121, in url_for
    return self.handle_url_build_error(error, endpoint, values)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1110, in url_for
    rv = url_adapter.build(  # type: ignore[union-attr]
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\werkzeug\routing\map.py", line 924, in build
    raise BuildError(endpoint, values, method, self)
werkzeug.routing.exceptions.BuildError: Could not build url for endpoint 'login'. Did you mean 'auth.login' instead?
2026-04-16 16:36:46 [INFO] werkzeug: 127.0.0.1 - - [16/Apr/2026 16:36:46] "GET / HTTP/1.1" 500 -
2026-04-16 16:36:47 [INFO] werkzeug: 127.0.0.1 - - [16/Apr/2026 16:36:47] "GET /favicon.ico HTTP/1.1" 404 -
2026-04-16 16:36:51 [INFO] werkzeug: 127.0.0.1 - - [16/Apr/2026 16:36:51] "GET /login HTTP/1.1" 200 -
2026-04-16 16:36:51 [INFO] werkzeug: 127.0.0.1 - - [16/Apr/2026 16:36:51] "GET /static/favicon.ico HTTP/1.1" 404 -
2026-04-16 16:37:01 [INFO] werkzeug: 127.0.0.1 - - [16/Apr/2026 16:37:01] "POST /login HTTP/1.1" 302 -
2026-04-16 16:37:01 [INFO] werkzeug: 127.0.0.1 - - [16/Apr/2026 16:37:01] "GET /central_engenharia HTTP/1.1" 200 -
2026-04-16 16:37:02 [INFO] werkzeug: 127.0.0.1 - - [16/Apr/2026 16:37:02] "GET /engenharia_formularios HTTP/1.1" 200 -
2026-04-16 16:37:04 [INFO] excel.builder: Reserva IPAM | CN: 01 | /28 | Pool: POOL 3 | Desc: RN1 TESTE - OPERADORA TESTE
2026-04-16 16:37:04 [INFO] ipam: Autenticando na API phpIPAM...
2026-04-16 16:37:04 [INFO] ipam: Autenticação bem-sucedida.
2026-04-16 16:37:04 [INFO] ipam: Buscando CN 'CN 01'...
2026-04-16 16:37:06 [INFO] ipam: CN 'CN 01' → ID 969910
2026-04-16 16:37:06 [INFO] ipam: Buscando pool 'POOL 3' no CN ID 969910...
2026-04-16 16:37:06 [INFO] ipam: Pool 'POOL 3' → ID 969912
2026-04-16 16:37:06 [INFO] ipam: Verificando rede pai /28 no pool ID 969912...
2026-04-16 16:37:06 [WARNING] ipam: Rede pai /28 não localizada no pool ID 969912. Prosseguindo.
2026-04-16 16:37:06 [INFO] ipam: Reservando rede /28 no pool ID 969912...
2026-04-16 16:37:06 [ERROR] excel.builder: Falha IPAM CN '01': Falha ao reservar subnet (HTTP 404): {"code":404,"success":false,"message":"No subnets found","time":0.009}
2026-04-16 16:37:06 [INFO] excel.builder: Gerando PTI Excel para 'OPERADORA TESTE'...
2026-04-16 16:37:06 [ERROR] app: Exception on /formularios/1/excel_index [GET]
Traceback (most recent call last):
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1511, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 919, in full_dispatch_request
    rv = self.handle_user_exception(e)
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 917, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 902, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\vivo_hub\auth.py", line 26, in wrapped
    return view(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\vivo_hub\routes\engenharia.py", line 114, in exportar_excel
    wb  = PTIWorkbookBuilder(form).build()
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\vivo_hub\excel\builder.py", line 1560, in build
    self._b_diagram()
  File "C:\Users\40418843\Desktop\vivo_hub\excel\builder.py", line 1017, in _b_diagram
    self._border_all_tables(ws)
  File "C:\Users\40418843\Desktop\vivo_hub\excel\builder.py", line 378, in _border_all_tables
    cell.border.left.style or cell.border.right.style
    ^^^^^^^^^^^^^^^^^^^^^^
AttributeError: 'NoneType' object has no attribute 'style'
