2026-04-16 11:11:03 [INFO] excel.builder: Gerando PTI Excel para 'OPERADORA TESTE'...
2026-04-16 11:11:03 [ERROR] app: Exception on /formularios/1/excel_index [GET]
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
  File "C:\Users\40418843\Desktop\vivo_hub\excel\builder.py", line 1518, in build
    self._b_routing()
  File "C:\Users\40418843\Desktop\vivo_hub\excel\builder.py", line 1126, in _b_routing
    if not ip_op:         missing.append("IP DO IPAM")
           ^^^^^
NameError: name 'ip_op' is not defined. Did you mean: 'cn_op'?
