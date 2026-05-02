PS C:\Users\40418843\Desktop\vivo_hub> py run.py
2026-05-02 16:02:36 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: C:/Users/40418843/Desktop/dados-sbcs
 * Serving Flask app 'app'
 * Debug mode: off
2026-05-02 16:02:36 [INFO] werkzeug: WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
2026-05-02 16:02:36 [INFO] werkzeug: Press CTRL+C to quit
2026-05-02 16:02:47 [INFO] siprouter_sp: siprouter_sp: 180 registros inseridos.
2026-05-02 16:02:47 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:02:47] "GET /login HTTP/1.1" 200 -
2026-05-02 16:02:58 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:02:58] "GET /admin_login HTTP/1.1" 200 -
2026-05-02 16:03:06 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:06] "POST /admin_login HTTP/1.1" 302 -
2026-05-02 16:03:06 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:06] "GET /register HTTP/1.1" 200 -
2026-05-02 16:03:20 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:20] "POST /register HTTP/1.1" 302 -
2026-05-02 16:03:20 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:20] "GET /register HTTP/1.1" 200 -
2026-05-02 16:03:34 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:34] "POST /register HTTP/1.1" 302 -
2026-05-02 16:03:35 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:35] "GET /register HTTP/1.1" 200 -
2026-05-02 16:03:36 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:36] "GET /logout HTTP/1.1" 302 -
2026-05-02 16:03:37 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:37] "GET /login HTTP/1.1" 200 -
2026-05-02 16:03:57 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:57] "POST /login HTTP/1.1" 302 -
2026-05-02 16:03:57 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:03:57] "GET /central_atacado HTTP/1.1" 200 -
2026-05-02 16:04:10 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:04:10] "GET /atacado_formularios?status=aprovado HTTP/1.1" 200 -
2026-05-02 16:04:17 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:04:17] "GET /logout HTTP/1.1" 302 -
2026-05-02 16:04:17 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:04:17] "GET /login HTTP/1.1" 200 -
2026-05-02 16:04:33 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:04:33] "POST /login HTTP/1.1" 302 -
2026-05-02 16:04:33 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:04:33] "GET /central_atacado HTTP/1.1" 200 -
2026-05-02 16:04:36 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:04:36] "GET /atacado_formularios/new HTTP/1.1" 200 -
2026-05-02 16:06:16 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:16] "POST /atacado_formularios/new HTTP/1.1" 302 -
2026-05-02 16:06:16 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:16] "GET /atacado_formularios HTTP/1.1" 200 -
2026-05-02 16:06:27 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:27] "GET /atacado_formularios HTTP/1.1" 200 -
2026-05-02 16:06:30 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:30] "GET /atacado_formularios?status=aprovado HTTP/1.1" 200 -
2026-05-02 16:06:34 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:34] "GET /logout HTTP/1.1" 302 -
2026-05-02 16:06:34 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:34] "GET /login HTTP/1.1" 200 -
2026-05-02 16:06:41 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:41] "POST /login HTTP/1.1" 302 -
2026-05-02 16:06:41 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:41] "GET /central_engenharia HTTP/1.1" 200 -
2026-05-02 16:06:45 [ERROR] app: Exception on /engenharia_formularios [GET]
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
  File "C:\Users\40418843\Desktop\vivo_hub\routes\engenharia.py", line 42, in form_list
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
  File "C:\Users\40418843\Desktop\vivo_hub\templates\engenharia_formularios.html", line 38, in template
    {% endif %}
jinja2.exceptions.TemplateSyntaxError: Encountered unknown tag 'endif'. Jinja was looking for the following tags: 'endblock'. The innermost block that needs to be closed is 'block'.
2026-05-02 16:06:45 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 16:06:45] "GET /engenharia_formularios HTTP/1.1" 500 -
