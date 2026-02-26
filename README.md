
Recebi o seguint erro: [werkzeug.routing.exceptions.BuildError: Could not build url for endpoint 'vivo_crc.upload_csv'. Did you mean 'vivo_crc.upload_arquivo' instead?

Traceback (most recent call last)
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1536, in __call__
return self.wsgi_app(environ, start_response)
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1514, in wsgi_app
response = self.handle_exception(e)
           ^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask_cors\extension.py", line 176, in wrapped_function
return cors_after_request(app.make_response(f(*args, **kwargs)))
                                            ^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1511, in wsgi_app
response = self.full_dispatch_request()
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 919, in full_dispatch_request
rv = self.handle_user_exception(e)
     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask_cors\extension.py", line 176, in wrapped_function
return cors_after_request(app.make_response(f(*args, **kwargs)))
                                            ^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 917, in full_dispatch_request
rv = self.dispatch_request()
     ^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 902, in dispatch_request
return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\Downloads\PortalITX 1\Vivo CRC\app\vivo_crc.py", line 476, in index
return render_template("vivoCRC/index.html", csrf_token=csrf_token)
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 150, in render_template
return _render(app, template, context)
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\templating.py", line 131, in _render
rv = template.render(context)
     ^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 1295, in render
self.environment.handle_exception()
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\jinja2\environment.py", line 942, in handle_exception
raise rewrite_traceback_stack(source=source)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\Downloads\PortalITX 1\Vivo CRC\app\templates\vivoCRC\index.html", line 223, in top-level template code
<form id="form-upload" action="{{ url_for('vivo_crc.upload_csv') }}" method="POST" enctype="multipart/form-data">
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1121, in url_for
return self.handle_url_build_error(error, endpoint, values)
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\flask\app.py", line 1110, in url_for
rv = url_adapter.build(  # type: ignore[union-attr]
     
File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\werkzeug\routing\map.py", line 924, in build
raise BuildError(endpoint, values, method, self)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
werkzeug.routing.exceptions.BuildError: Could not build url for endpoint 'vivo_crc.upload_csv'. Did you mean 'vivo_crc.upload_arquivo' instead?
The debugger caught an exception in your WSGI application. You can now look at the traceback which led to the error.
To switch between the interactive traceback and the plaintext one, you can click on the "Traceback" headline. From the text traceback you can also create a paste of it. For code execution mouse-over the frame you want to debug and click on the console icon on the right side.

You can execute arbitrary Python code in the stack frames and there are some extra helpers available for introspection:

dump() shows all variables in the frame
dump(obj) dumps all that's known about the object
Brought]
