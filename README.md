PS C:\Users\40418843\Desktop\vivo_hub> py run.py
2026-05-02 15:54:22 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: C:/Users/40418843/Desktop/dados-sbcs
 * Serving Flask app 'app'
 * Debug mode: off
2026-05-02 15:54:22 [INFO] werkzeug: WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
2026-05-02 15:54:22 [INFO] werkzeug: Press CTRL+C to quit
2026-05-02 15:54:56 [INFO] siprouter_sp: siprouter_sp: 180 registros inseridos.
2026-05-02 15:54:56 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:54:56] "GET /login HTTP/1.1" 200 -
2026-05-02 15:55:05 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:05] "GET /admin_login HTTP/1.1" 200 -
2026-05-02 15:55:12 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:12] "POST /admin_login HTTP/1.1" 302 -
2026-05-02 15:55:12 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:12] "GET /register HTTP/1.1" 200 -
2026-05-02 15:55:35 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:35] "POST /register HTTP/1.1" 302 -
2026-05-02 15:55:35 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:35] "GET /register HTTP/1.1" 200 -
2026-05-02 15:55:52 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:52] "POST /register HTTP/1.1" 302 -
2026-05-02 15:55:52 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:52] "GET /register HTTP/1.1" 200 -
2026-05-02 15:55:54 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:54] "GET /admin_login HTTP/1.1" 200 -
2026-05-02 15:55:56 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:55:56] "GET /login HTTP/1.1" 200 -
2026-05-02 15:56:04 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:56:04] "POST /login HTTP/1.1" 302 -
2026-05-02 15:56:04 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:56:04] "GET /central_atacado HTTP/1.1" 200 -
2026-05-02 15:56:11 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:56:11] "GET /atacado_formularios/new HTTP/1.1" 200 -
2026-05-02 15:58:24 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:58:24] "POST /atacado_formularios/new HTTP/1.1" 302 -
2026-05-02 15:58:24 [ERROR] app: Exception on /atacado_formularios [GET]
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
  File "C:\Users\40418843\Desktop\vivo_hub\routes\atacado.py", line 56, in form_list
    return render_template(
           ^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 149, in render_template
    template = app.jinja_env.get_or_select_template(template_name_or_list)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 1087, in get_or_select_template
    return self.get_template(template_name_or_list, parent, globals)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 1016, in get_template
    return self._load_template(name, globals)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 975, in _load_template
    template = self.loader.load(self, name, self.make_globals(globals))
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\loaders.py", line 138, in load
    code = environment.compile(source, name, filename)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 771, in compile
    self.handle_exception(source=source_hint)
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 942, in handle_exception
    raise rewrite_traceback_stack(source=source)
  File "C:\Users\40418843\Desktop\vivo_hub\templates\atacado_formularios.html", line 30, in template
    {% endif %}
jinja2.exceptions.TemplateSyntaxError: Encountered unknown tag 'endif'. Jinja was looking for the following tags: 'endblock'. The innermost block that needs to be closed is 'block'.
2026-05-02 15:58:24 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:58:24] "GET /atacado_formularios HTTP/1.1" 500 -
2026-05-02 15:58:24 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 15:58:24] "GET /favicon.ico HTTP/1.1" 404 -
