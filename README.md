
vini_benevenuto@app-vini:~$ docker ps -a

CONTAINER ID   IMAGE                                    COMMAND                  CREATED         STATUS                      PORTS                                                 NAMES

7219c27863b2   agent_studio_frontend:beta1.0            "sh -c 'cd app/ && n…"   4 months ago    Up 4 months                 0.0.0.0:9999->5173/tcp, :::9999->5173/tcp             agent-studio-frontend-test

3973e8e45598   agent_studio:beta1.0                     "sh -c 'cd /app && p…"   4 months ago    Up 4 months                 0.0.0.0:1010->5000/tcp, :::1010->5000/tcp             agent-studio-test

e6c52eff3eb7   9ae202b62138                             "cd /app"                5 months ago    Created                                                                           temp_container

24a4508d0f9e   vivo-crc:1.0                             "python -m flask --a…"   6 months ago    Up 6 months                 8080/tcp, 0.0.0.0:8080->5000/tcp, :::8080->5000/tcp   vivo-crc

71671168c1e3   3a59610af07a                             "docker-entrypoint.s…"   7 months ago    Exited (143) 6 months ago                                                         agent-studio-frontend

38485a26caab   14dd0674f8aa                             "python3 -m flask --…"   7 months ago    Exited (137) 6 months ago                                                         agent-studio

d02cba73a0c2   ia_av:stable_2                           "python -m flask --a…"   8 months ago    Up 6 months                 5500/tcp, 0.0.0.0:5555->5000/tcp, :::5555->5000/tcp   ia_av

5341e9f84d98   bf2c10c6fd6b                             "python3 -m gunicorn…"   12 months ago   Up 2 months                 0.0.0.0:443->5000/tcp, :::443->5000/tcp               portal-itx-v4.6

0676acbf249d   portal-itx-redirect-to-main-portal:1.0   "python -m flask --a…"   22 months ago   Up 6 months                 0.0.0.0:5500->5000/tcp, :::5500->5000/tcp             portal-itx-redirection

vini_benevenuto@app-vini:~$ docker run pti-automatizado:latest

2026-05-06 13:21:31 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: /app/dados-sbcs

[2026-05-06 13:21:31 +0000] [1] [INFO] Starting gunicorn 25.3.0

[2026-05-06 13:21:31 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000 (1)

[2026-05-06 13:21:31 +0000] [1] [INFO] Using worker: sync

[2026-05-06 13:21:31 +0000] [13] [INFO] Booting worker with pid: 13

[2026-05-06 13:21:31 +0000] [14] [INFO] Booting worker with pid: 14

[2026-05-06 13:21:31 +0000] [1] [ERROR] Control server error: [Errno 13] Permission denied: '/home/pti'

2026-05-06 13:22:01 [INFO] siprouter_sp: siprouter_sp: 180 registros inseridos.

127.0.0.1 - - [06/May/2026:13:22:01 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:22:31 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:23:01 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:23:31 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:24:01 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:24:31 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:25:02 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:25:32 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:26:02 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"

127.0.0.1 - - [06/May/2026:13:26:32 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
 
