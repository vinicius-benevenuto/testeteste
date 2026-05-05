PS C:\Users\40418843\Desktop\vivo_hub> py run.py
2026-05-05 14:18:56 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: C:/Users/40418843/Desktop/dados-sbcs
 * Serving Flask app 'app'
 * Debug mode: off
2026-05-05 14:18:56 [INFO] werkzeug: WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://10.128.69.137:5000
2026-05-05 14:18:56 [INFO] werkzeug: Press CTRL+C to quit
2026-05-05 14:19:07 [ERROR] app: Exception on / [GET]
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
2026-05-05 14:19:07 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:07] "GET / HTTP/1.1" 500 -
2026-05-05 14:19:07 [ERROR] app: Exception on / [GET]
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
2026-05-05 14:19:07 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:07] "GET / HTTP/1.1" 500 -
2026-05-05 14:19:07 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:07] "GET /favicon.ico HTTP/1.1" 404 -
2026-05-05 14:19:07 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:07] "GET /favicon.ico HTTP/1.1" 404 -
2026-05-05 14:19:16 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:16] "GET /login HTTP/1.1" 200 -
2026-05-05 14:19:16 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:16] "GET /static/favicon.ico HTTP/1.1" 404 -
2026-05-05 14:19:27 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:27] "POST /login HTTP/1.1" 302 -
2026-05-05 14:19:27 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:27] "GET /central_engenharia HTTP/1.1" 200 -
2026-05-05 14:19:30 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:30] "GET /engenharia_formularios HTTP/1.1" 200 -
2026-05-05 14:19:32 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:32] "GET /central_engenharia HTTP/1.1" 200 -
2026-05-05 14:19:34 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:34] "GET /logout HTTP/1.1" 302 -
2026-05-05 14:19:34 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:19:34] "GET /login HTTP/1.1" 200 -
2026-05-05 14:20:11 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:20:11] "GET /login HTTP/1.1" 200 -
2026-05-05 14:20:18 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:20:18] "POST /login HTTP/1.1" 302 -
2026-05-05 14:20:18 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:20:18] "GET /central_atacado HTTP/1.1" 200 -
2026-05-05 14:20:19 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:20:19] "GET /atacado_formularios/new HTTP/1.1" 200 -
2026-05-05 14:20:31 [ERROR] app: Exception on /atacado_formularios/new [POST]
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
  File "C:\Users\40418843\Desktop\vivo_hub\auth.py", line 39, in wrapped
    return view(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\vivo_hub\routes\atacado.py", line 129, in form_create
    db.execute(
sqlite3.OperationalError: table atacado_forms has no column named version
2026-05-05 14:20:31 [INFO] werkzeug: 127.0.0.1 - - [05/May/2026 14:20:31] "POST /atacado_formularios/new HTTP/1.1" 500 -
