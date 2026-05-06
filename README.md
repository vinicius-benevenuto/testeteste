Está acessível o container de dentro do servidor agora que eu utilizeo o Docker composse, porém, na hora de exportar o excel, ele fica carregando infinitamente, ao analisar os logs do contêiner eu percebi que temos erros:
 
vini_benevenuto@app-vini:~$ docker logs pti-automatizado
2026-05-06 13:47:36 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: /app/dados-sbcs
[2026-05-06 13:47:36 +0000] [1] [INFO] Starting gunicorn 25.3.0
[2026-05-06 13:47:36 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000 (1)
[2026-05-06 13:47:36 +0000] [1] [INFO] Using worker: sync
[2026-05-06 13:47:36 +0000] [14] [INFO] Booting worker with pid: 14
[2026-05-06 13:47:36 +0000] [15] [INFO] Booting worker with pid: 15
[2026-05-06 13:47:36 +0000] [1] [ERROR] Control server error: [Errno 13] Permission denied: '/home/pti'
2026-05-06 13:47:46 [INFO] siprouter_sp: siprouter_sp: 180 registros inseridos.
10.60.255.161 - - [06/May/2026:13:47:46 +0000] "GET /login HTTP/1.1" 200 11924 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:47:46 +0000] "GET /static/favicon.ico HTTP/1.1" 404 207 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.188 - - [06/May/2026:13:47:48 +0000] "GET /login HTTP/1.1" 200 11924 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36 Edg/147.0.0.0"
10.60.255.188 - - [06/May/2026:13:47:48 +0000] "GET /static/favicon.ico HTTP/1.1" 404 207 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36 Edg/147.0.0.0"
10.60.255.161 - - [06/May/2026:13:47:51 +0000] "GET /admin_login HTTP/1.1" 200 11209 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:47:59 +0000] "POST /admin_login HTTP/1.1" 302 205 "http://10.230.247.3:5000/admin_login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:47:59 +0000] "GET /register HTTP/1.1" 200 12927 "http://10.230.247.3:5000/admin_login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:48:05 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:48:32 +0000] "POST /register HTTP/1.1" 302 205 "http://10.230.247.3:5000/register" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:48:32 +0000] "GET /register HTTP/1.1" 200 12923 "http://10.230.247.3:5000/register" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:48:35 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:49:05 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:49:07 +0000] "POST /register HTTP/1.1" 302 205 "http://10.230.247.3:5000/register" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:49:07 +0000] "GET /register HTTP/1.1" 200 12923 "http://10.230.247.3:5000/register" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:49:08 +0000] "GET /logout HTTP/1.1" 302 199 "http://10.230.247.3:5000/register" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:49:08 +0000] "GET /login HTTP/1.1" 200 12110 "http://10.230.247.3:5000/register" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:49:25 +0000] "POST /login HTTP/1.1" 302 219 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:49:25 +0000] "GET /central_atacado HTTP/1.1" 200 11727 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:49:26 +0000] "GET /atacado_formularios/new HTTP/1.1" 200 51633 "http://10.230.247.3:5000/central_atacado" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:49:35 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:50:06 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:50:36 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:50:57 +0000] "POST /atacado_formularios/new HTTP/1.1" 302 227 "http://10.230.247.3:5000/atacado_formularios/new" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:50:57 +0000] "GET /atacado_formularios HTTP/1.1" 200 15498 "http://10.230.247.3:5000/atacado_formularios/new" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:50:59 +0000] "GET /logout HTTP/1.1" 302 199 "http://10.230.247.3:5000/atacado_formularios" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:50:59 +0000] "GET /login HTTP/1.1" 200 12110 "http://10.230.247.3:5000/atacado_formularios" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:51:06 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:51:17 +0000] "POST /login HTTP/1.1" 302 225 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:51:17 +0000] "GET /central_engenharia HTTP/1.1" 200 11530 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:51:18 +0000] "GET /engenharia_formularios HTTP/1.1" 200 15245 "http://10.230.247.3:5000/central_engenharia" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:51:20 +0000] "GET /engenharia_formularios/1 HTTP/1.1" 200 56369 "http://10.230.247.3:5000/engenharia_formularios" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:51:21 +0000] "GET /api/siprouter/query?cn=11&rn1=55131&scm=0&av=0 HTTP/1.1" 200 249 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:51:36 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:51:46 +0000] "GET /api/siprouter/query?cn=11&rn1=55131&scm=0&av=0 HTTP/1.1" 200 249 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:51:47 +0000] "GET /api/siprouter/query?cn=01&rn1=55131&scm=0&av=0 HTTP/1.1" 200 185 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:52:05 +0000] "POST /engenharia_formularios/1 HTTP/1.1" 302 233 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:52:05 +0000] "GET /engenharia_formularios HTTP/1.1" 200 15448 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:52:06 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:52:08 +0000] "POST /engenharia_formularios/1/validar HTTP/1.1" 302 233 "http://10.230.247.3:5000/engenharia_formularios" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:52:08 +0000] "GET /engenharia_formularios HTTP/1.1" 200 14552 "http://10.230.247.3:5000/engenharia_formularios" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
2026-05-06 13:52:13 [INFO] excel.builder: IPAM | Iniciando reserva: 1 bloco(s) a processar
2026-05-06 13:52:13 [INFO] excel.builder: Reserva IPAM | CN: 01 | /29 | Pool: POOL 4 | Desc: 55131 - OI
2026-05-06 13:52:13 [INFO] ipam: Autenticando na API phpIPAM...
10.60.255.161 - - [06/May/2026:13:52:57 +0000] "GET /engenharia_formularios/1 HTTP/1.1" 200 60094 "http://10.230.247.3:5000/engenharia_formularios" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:52:57 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:52:58 +0000] "GET /api/siprouter/query?cn=11&rn1=55131&scm=0&av=0 HTTP/1.1" 200 249 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:52:58 +0000] "GET /api/siprouter/query?cn=01&rn1=55131&scm=0&av=0 HTTP/1.1" 200 185 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:53:25 +0000] "POST /engenharia_formularios/1 HTTP/1.1" 302 233 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:53:25 +0000] "POST /engenharia_formularios/1 HTTP/1.1" 302 233 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:53:25 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:53:25 +0000] "GET /engenharia_formularios HTTP/1.1" 200 14561 "http://10.230.247.3:5000/engenharia_formularios/1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
2026-05-06 13:53:31 [INFO] excel.builder: IPAM | Iniciando reserva: 1 bloco(s) a processar
2026-05-06 13:53:31 [INFO] excel.builder: Reserva IPAM | CN: 01 | /29 | Pool: POOL 4 | Desc: 55131 - OI
2026-05-06 13:53:31 [INFO] ipam: Autenticando na API phpIPAM...
[2026-05-06 13:54:13 +0000] [1] [CRITICAL] WORKER TIMEOUT (pid:14)
[2026-05-06 13:54:13 +0000] [14] [ERROR] Error handling request GET /formularios/1/excel_index
Traceback (most recent call last):
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 142, in handle
    self.handle_request(listener, req, client, addr)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 185, in handle_request
    respiter = self.wsgi(environ, resp.start_response)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1536, in __call__
    return self.wsgi_app(environ, start_response)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1511, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 917, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 902, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/auth.py", line 26, in wrapped
    return view(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^
  File "/app/routes/engenharia.py", line 234, in exportar_excel
    builder = PTIWorkbookBuilder(form, all_versions=_get_all_versions(db, form))
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/excel/builder.py", line 85, in __init__
    self.reserva_redes_ipam(self.vivo_rows)
  File "/app/excel/builder.py", line 135, in reserva_redes_ipam
    cliente.autenticar()
  File "/app/ipam.py", line 45, in autenticar
    resposta = self.session.post(
               ^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 637, in post
    return self.request("POST", url, data=data, json=json, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 589, in request
    resp = self.send(prep, **send_kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 703, in send
    r = adapter.send(request, **kwargs)
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/adapters.py", line 644, in send
    resp = conn.urlopen(
10.60.255.161 - - [06/May/2026:13:54:13 +0000] "GET /formularios/1/excel_index HTTP/1.1" 500 0 "-" "-"
           ^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 787, in urlopen
    response = self._make_request(
               ^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 493, in _make_request
    conn.request(
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 500, in request
    self.endheaders()
  File "/usr/local/lib/python3.12/http/client.py", line 1353, in endheaders
    self._send_output(message_body, encode_chunked=encode_chunked)
  File "/usr/local/lib/python3.12/http/client.py", line 1113, in _send_output
    self.send(msg)
  File "/usr/local/lib/python3.12/http/client.py", line 1057, in send
    self.connect()
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 331, in connect
    self.sock = self._new_conn()
                ^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 204, in _new_conn
    sock = connection.create_connection(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/util/connection.py", line 73, in create_connection
    sock.connect(sa)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/base.py", line 198, in handle_abort
    sys.exit(1)
SystemExit: 1
[2026-05-06 13:54:13 +0000] [14] [INFO] Worker exiting (pid: 14)
[2026-05-06 13:54:13 +0000] [89] [INFO] Booting worker with pid: 89
[2026-05-06 13:54:13 +0000] [1] [ERROR] Control server error: [Errno 13] Permission denied: '/home/pti'
127.0.0.1 - - [06/May/2026:13:55:02 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:55:02 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:55:03 +0000] "GET /login HTTP/1.1" 200 11924 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
127.0.0.1 - - [06/May/2026:13:55:15 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:13:55:22 +0000] "POST /login HTTP/1.1" 302 225 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:55:22 +0000] "GET /central_engenharia HTTP/1.1" 200 11530 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:13:55:23 +0000] "GET /engenharia_formularios HTTP/1.1" 200 14358 "http://10.230.247.3:5000/central_engenharia" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
[2026-05-06 13:55:31 +0000] [1] [CRITICAL] WORKER TIMEOUT (pid:15)
[2026-05-06 13:55:31 +0000] [15] [ERROR] Error handling request GET /formularios/1/excel_index
Traceback (most recent call last):
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 142, in handle
    self.handle_request(listener, req, client, addr)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 185, in handle_request
    respiter = self.wsgi(environ, resp.start_response)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1536, in __call__
    return self.wsgi_app(environ, start_response)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1511, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 917, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 902, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/auth.py", line 26, in wrapped
    return view(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^
  File "/app/routes/engenharia.py", line 234, in exportar_excel
    builder = PTIWorkbookBuilder(form, all_versions=_get_all_versions(db, form))
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/excel/builder.py", line 85, in __init__
    self.reserva_redes_ipam(self.vivo_rows)
  File "/app/excel/builder.py", line 135, in reserva_redes_ipam
    cliente.autenticar()
  File "/app/ipam.py", line 45, in autenticar
    resposta = self.session.post(
               ^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 637, in post
    return self.request("POST", url, data=data, json=json, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 589, in request
    resp = self.send(prep, **send_kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 703, in send
    r = adapter.send(request, **kwargs)
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/adapters.py", line 644, in send
    resp = conn.urlopen(
           ^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 787, in urlopen
    response = self._make_request(
               ^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 493, in _make_request
    conn.request(
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 500, in request
    self.endheaders()
  File "/usr/local/lib/python3.12/http/client.py", line 1353, in endheaders
    self._send_output(message_body, encode_chunked=encode_chunked)
  File "/usr/local/lib/python3.12/http/client.py", line 1113, in _send_output
    self.send(msg)
  File "/usr/local/lib/python3.12/http/client.py", line 1057, in send
    self.connect()
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 331, in connect
    self.sock = self._new_conn()
                ^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 204, in _new_conn
    sock = connection.create_connection(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/util/connection.py", line 73, in create_connection
    sock.connect(sa)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/base.py", line 198, in handle_abort
    sys.exit(1)
SystemExit: 1
10.60.255.161 - - [06/May/2026:13:55:31 +0000] "GET /formularios/1/excel_index HTTP/1.1" 500 0 "-" "-"
[2026-05-06 13:55:31 +0000] [15] [INFO] Worker exiting (pid: 15)
[2026-05-06 13:55:31 +0000] [103] [INFO] Booting worker with pid: 103
[2026-05-06 13:55:31 +0000] [1] [ERROR] Control server error: [Errno 13] Permission denied: '/home/pti'
10.60.255.161 - - [06/May/2026:13:55:31 +0000] "GET /favicon.ico HTTP/1.1" 404 207 "http://10.230.247.3:5000/formularios/1/excel_index" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
2026-05-06 13:55:31 [INFO] excel.builder: IPAM | Iniciando reserva: 1 bloco(s) a processar
2026-05-06 13:55:31 [INFO] excel.builder: Reserva IPAM | CN: 01 | /29 | Pool: POOL 4 | Desc: 55131 - OI
2026-05-06 13:55:31 [INFO] ipam: Autenticando na API phpIPAM...
2026-05-06 13:56:01 [INFO] excel.builder: IPAM | Iniciando reserva: 1 bloco(s) a processar
2026-05-06 13:56:01 [INFO] excel.builder: Reserva IPAM | CN: 01 | /29 | Pool: POOL 4 | Desc: 55131 - OI
2026-05-06 13:56:01 [INFO] ipam: Autenticando na API phpIPAM...
[2026-05-06 13:57:30 +0000] [1] [CRITICAL] WORKER TIMEOUT (pid:89)
[2026-05-06 13:57:30 +0000] [89] [ERROR] Error handling request GET /formularios/1/excel_index
Traceback (most recent call last):
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 142, in handle
    self.handle_request(listener, req, client, addr)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 185, in handle_request
    respiter = self.wsgi(environ, resp.start_response)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1536, in __call__
    return self.wsgi_app(environ, start_response)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1511, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 917, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 902, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/auth.py", line 26, in wrapped
    return view(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^
  File "/app/routes/engenharia.py", line 234, in exportar_excel
    builder = PTIWorkbookBuilder(form, all_versions=_get_all_versions(db, form))
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/excel/builder.py", line 85, in __init__
    self.reserva_redes_ipam(self.vivo_rows)
  File "/app/excel/builder.py", line 135, in reserva_redes_ipam
    cliente.autenticar()
  File "/app/ipam.py", line 45, in autenticar
    resposta = self.session.post(
               ^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 637, in post
    return self.request("POST", url, data=data, json=json, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 589, in request
    resp = self.send(prep, **send_kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 703, in send
    r = adapter.send(request, **kwargs)
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/adapters.py", line 644, in send
    resp = conn.urlopen(
           ^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 787, in urlopen
    response = self._make_request(
               ^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 493, in _make_request
    conn.request(
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 500, in request
    self.endheaders()
  File "/usr/local/lib/python3.12/http/client.py", line 1353, in endheaders
    self._send_output(message_body, encode_chunked=encode_chunked)
  File "/usr/local/lib/python3.12/http/client.py", line 1113, in _send_output
    self.send(msg)
  File "/usr/local/lib/python3.12/http/client.py", line 1057, in send
    self.connect()
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 331, in connect
    self.sock = self._new_conn()
                ^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 204, in _new_conn
    sock = connection.create_connection(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/util/connection.py", line 73, in create_connection
    sock.connect(sa)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/base.py", line 198, in handle_abort
    sys.exit(1)
SystemExit: 1
10.60.255.161 - - [06/May/2026:13:57:30 +0000] "GET /formularios/1/excel_index HTTP/1.1" 500 0 "-" "-"
[2026-05-06 13:57:30 +0000] [89] [INFO] Worker exiting (pid: 89)
[2026-05-06 13:57:30 +0000] [123] [INFO] Booting worker with pid: 123
[2026-05-06 13:57:30 +0000] [1] [ERROR] Control server error: [Errno 13] Permission denied: '/home/pti'
127.0.0.1 - - [06/May/2026:13:57:30 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:57:30 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:57:30 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
[2026-05-06 13:57:41 +0000] [1] [CRITICAL] WORKER TIMEOUT (pid:103)
[2026-05-06 13:57:41 +0000] [103] [ERROR] Error handling request GET /formularios/1/excel_index
Traceback (most recent call last):
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 142, in handle
    self.handle_request(listener, req, client, addr)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/sync.py", line 185, in handle_request
    respiter = self.wsgi(environ, resp.start_response)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1536, in __call__
    return self.wsgi_app(environ, start_response)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 1511, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 917, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/flask/app.py", line 902, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/auth.py", line 26, in wrapped
    return view(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^
  File "/app/routes/engenharia.py", line 234, in exportar_excel
    builder = PTIWorkbookBuilder(form, all_versions=_get_all_versions(db, form))
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/app/excel/builder.py", line 85, in __init__
    self.reserva_redes_ipam(self.vivo_rows)
  File "/app/excel/builder.py", line 135, in reserva_redes_ipam
    cliente.autenticar()
  File "/app/ipam.py", line 45, in autenticar
    resposta = self.session.post(
               ^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 637, in post
    return self.request("POST", url, data=data, json=json, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 589, in request
    resp = self.send(prep, **send_kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/sessions.py", line 703, in send
    r = adapter.send(request, **kwargs)
        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/requests/adapters.py", line 644, in send
    resp = conn.urlopen(
           ^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 787, in urlopen
    response = self._make_request(
               ^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connectionpool.py", line 493, in _make_request
    conn.request(
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 500, in request
    self.endheaders()
  File "/usr/local/lib/python3.12/http/client.py", line 1353, in endheaders
    self._send_output(message_body, encode_chunked=encode_chunked)
  File "/usr/local/lib/python3.12/http/client.py", line 1113, in _send_output
    self.send(msg)
  File "/usr/local/lib/python3.12/http/client.py", line 1057, in send
    self.connect()
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 331, in connect
    self.sock = self._new_conn()
                ^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/connection.py", line 204, in _new_conn
    sock = connection.create_connection(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/local/lib/python3.12/site-packages/urllib3/util/connection.py", line 73, in create_connection
    sock.connect(sa)
  File "/usr/local/lib/python3.12/site-packages/gunicorn/workers/base.py", line 198, in handle_abort
    sys.exit(1)
SystemExit: 1
10.60.255.161 - - [06/May/2026:13:57:41 +0000] "GET /formularios/1/excel_index HTTP/1.1" 500 0 "-" "-"
[2026-05-06 13:57:41 +0000] [103] [INFO] Worker exiting (pid: 103)
[2026-05-06 13:57:41 +0000] [125] [INFO] Booting worker with pid: 125
[2026-05-06 13:57:41 +0000] [1] [ERROR] Control server error: [Errno 13] Permission denied: '/home/pti'
127.0.0.1 - - [06/May/2026:13:57:45 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:58:15 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:58:45 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:59:15 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:13:59:45 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
127.0.0.1 - - [06/May/2026:14:00:15 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
10.60.255.161 - - [06/May/2026:14:00:27 +0000] "GET /login HTTP/1.1" 200 11924 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
10.60.255.161 - - [06/May/2026:14:00:28 +0000] "GET /static/favicon.ico HTTP/1.1" 404 207 "http://10.230.247.3:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36"
vini_benevenuto@app-vini:~$
