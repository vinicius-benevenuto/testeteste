PS C:\Users\40418843\Desktop\vivo_hub> py run.py
2026-04-17 11:36:59 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: C:/Users/40418843/Desktop/dados-sbcs
 * Serving Flask app 'app'
 * Debug mode: off
2026-04-17 11:36:59 [INFO] werkzeug: WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
2026-04-17 11:36:59 [INFO] werkzeug: Press CTRL+C to quit
2026-04-17 11:37:03 [INFO] siprouter_sp: siprouter_sp: 180 registros inseridos.
2026-04-17 11:37:03 [ERROR] app: Exception on / [GET]
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
2026-04-17 11:37:03 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:03] "GET / HTTP/1.1" 500 -
2026-04-17 11:37:03 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:03] "GET /favicon.ico HTTP/1.1" 404 -
2026-04-17 11:37:07 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:07] "GET /login HTTP/1.1" 200 -
2026-04-17 11:37:08 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:08] "GET /static/favicon.ico HTTP/1.1" 404 -
2026-04-17 11:37:12 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:12] "GET /admin_login HTTP/1.1" 200 -
2026-04-17 11:37:16 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:16] "POST /admin_login HTTP/1.1" 302 -
2026-04-17 11:37:16 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:16] "GET /register HTTP/1.1" 200 -
2026-04-17 11:37:35 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:35] "POST /register HTTP/1.1" 302 -
2026-04-17 11:37:35 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:35] "GET /register HTTP/1.1" 200 -
2026-04-17 11:37:56 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:56] "POST /register HTTP/1.1" 302 -
2026-04-17 11:37:56 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:56] "GET /register HTTP/1.1" 200 -
2026-04-17 11:37:57 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:57] "GET /admin_login HTTP/1.1" 200 -
2026-04-17 11:37:59 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:37:59] "GET /login HTTP/1.1" 200 -
2026-04-17 11:38:08 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:38:08] "POST /login HTTP/1.1" 302 -
2026-04-17 11:38:08 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:38:08] "GET /central_atacado HTTP/1.1" 200 -
2026-04-17 11:38:10 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:38:10] "GET /atacado_formularios/new HTTP/1.1" 200 -
2026-04-17 11:38:10 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:38:10] "GET /api/cns HTTP/1.1" 200 -
2026-04-17 11:39:06 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:06] "POST /atacado_formularios/new HTTP/1.1" 302 -
2026-04-17 11:39:06 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:06] "GET /atacado_formularios HTTP/1.1" 200 -
2026-04-17 11:39:11 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:11] "GET /logout HTTP/1.1" 302 -
2026-04-17 11:39:11 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:11] "GET /login HTTP/1.1" 200 -
2026-04-17 11:39:24 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:24] "POST /login HTTP/1.1" 302 -
2026-04-17 11:39:24 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:24] "GET /central_engenharia HTTP/1.1" 200 -
2026-04-17 11:39:28 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:28] "GET /engenharia_formularios HTTP/1.1" 200 -
2026-04-17 11:39:30 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:30] "GET /engenharia_formularios/1 HTTP/1.1" 200 -
2026-04-17 11:39:30 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:30] "GET /api/cns HTTP/1.1" 200 -
2026-04-17 11:39:30 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:39:30] "GET /api/siprouter/query?cn=01&rn1=RN1%20TESTE&scm=0&av=0 HTTP/1.1" 200 -
2026-04-17 11:40:16 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:16] "POST /engenharia_formularios/1 HTTP/1.1" 302 -
2026-04-17 11:40:16 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:16] "GET /engenharia_formularios HTTP/1.1" 200 -
2026-04-17 11:40:19 [INFO] excel.builder: IPAM DEBUG | vivo_rows=1 op_rows=1
2026-04-17 11:40:19 [INFO] excel.builder: IPAM DEBUG | linha 0 | cn='01' | mask_op='' | mask_vr=''
2026-04-17 11:40:19 [WARNING] excel.builder: IPAM | linha 0 pulada: cn='01' mask=''
2026-04-17 11:40:19 [INFO] excel.builder: Gerando PTI Excel para 'OPERADORA TESTE'...
2026-04-17 11:40:20 [INFO] excel.builder: PTI Excel gerado com sucesso.
2026-04-17 11:40:20 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:20] "GET /formularios/1/excel_index HTTP/1.1" 200 -
2026-04-17 11:40:30 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:30] "GET /engenharia_formularios/1 HTTP/1.1" 200 -
2026-04-17 11:40:30 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:30] "GET /api/cns HTTP/1.1" 200 -
2026-04-17 11:40:31 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:31] "GET /api/siprouter/query?cn=01&rn1=RN1%20TESTE&scm=0&av=0 HTTP/1.1" 200 -
2026-04-17 11:40:40 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:40] "POST /engenharia_formularios/1 HTTP/1.1" 302 -
2026-04-17 11:40:40 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:40] "GET /engenharia_formularios HTTP/1.1" 200 -
2026-04-17 11:40:42 [INFO] excel.builder: IPAM DEBUG | vivo_rows=1 op_rows=1
2026-04-17 11:40:42 [INFO] excel.builder: IPAM DEBUG | linha 0 | cn='01' | mask_op='' | mask_vr=''
2026-04-17 11:40:42 [WARNING] excel.builder: IPAM | linha 0 pulada: cn='01' mask=''
2026-04-17 11:40:42 [INFO] excel.builder: Gerando PTI Excel para 'OPERADORA TESTE'...
2026-04-17 11:40:43 [INFO] excel.builder: PTI Excel gerado com sucesso.
2026-04-17 11:40:43 [INFO] werkzeug: 127.0.0.1 - - [17/Apr/2026 11:40:43] "GET /formularios/1/excel_index HTTP/1.1" 200 -
