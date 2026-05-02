PS C:\Users\40418843\Desktop\vivo_hub> py run.py
2026-05-02 19:15:31 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: C:/Users/40418843/Desktop/dados-sbcs
 * Serving Flask app 'app'
 * Debug mode: off
2026-05-02 19:15:31 [INFO] werkzeug: WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
2026-05-02 19:15:31 [INFO] werkzeug: Press CTRL+C to quit
2026-05-02 19:15:51 [INFO] siprouter_sp: siprouter_sp: 180 registros inseridos.
2026-05-02 19:15:51 [ERROR] app: Exception on /login [GET]
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
  File "C:\Users\40418843\Desktop\vivo_hub\routes\auth.py", line 20, in login
    return render_template("login.html")
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 150, in render_template
    return _render(app, template, context)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 131, in _render
    rv = template.render(context)
         ^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 1295, in render
    self.environment.handle_exception()
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 942, in handle_exception
    raise rewrite_traceback_stack(source=source)
  File "C:\Users\40418843\Desktop\vivo_hub\templates\login.html", line 2, in top-level template code
    {% set hide_nav = true %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  [Previous line repeated 974 more times]
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\utils.py", line 477, in get
    return self[key]
           ~~~~^^^^^
RecursionError: maximum recursion depth exceeded
2026-05-02 19:15:51 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 19:15:51] "GET /login HTTP/1.1" 500 -
2026-05-02 19:15:51 [ERROR] app: Exception on /login [GET]
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
  File "C:\Users\40418843\Desktop\vivo_hub\routes\auth.py", line 20, in login
    return render_template("login.html")
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 150, in render_template
    return _render(app, template, context)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 131, in _render
    rv = template.render(context)
         ^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 1295, in render
    self.environment.handle_exception()
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 942, in handle_exception
    raise rewrite_traceback_stack(source=source)
  File "C:\Users\40418843\Desktop\vivo_hub\templates\login.html", line 2, in top-level template code
    {% set hide_nav = true %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  [Previous line repeated 974 more times]
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\utils.py", line 477, in get
    return self[key]
           ~~~~^^^^^
RecursionError: maximum recursion depth exceeded
2026-05-02 19:15:52 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 19:15:52] "GET /login HTTP/1.1" 500 -
2026-05-02 19:16:45 [ERROR] app: Exception on / [GET]
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
2026-05-02 19:16:45 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 19:16:45] "GET / HTTP/1.1" 500 -
2026-05-02 19:16:45 [INFO] werkzeug: 127.0.0.1 - - [02/May/2026 19:16:45] "GET /favicon.ico HTTP/1.1" 404 -
2026-05-02 19:16:50 [ERROR] app: Exception on /login [GET]
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
  File "C:\Users\40418843\Desktop\vivo_hub\routes\auth.py", line 20, in login
    return render_template("login.html")
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 150, in render_template
    return _render(app, template, context)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 131, in _render
    rv = template.render(context)
         ^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 1295, in render
    self.environment.handle_exception()
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 942, in handle_exception
    raise rewrite_traceback_stack(source=source)
  File "C:\Users\40418843\Desktop\vivo_hub\templates\login.html", line 2, in top-level template code
    {% set hide_nav = true %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  File "C:\Users\40418843\Desktop\vivo_hub\templates\base.html", line 1, in top-level template code
    {% extends "base.html" %}
  [Previous line repeated 974 more times]
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\utils.py", line 477, in get
    return self[key]
           ~~~~^^^^^
RecursionError: maximum recursion depth exceeded
