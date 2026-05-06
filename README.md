Quando digito "docker compose up na minha maquina, antes de enviar uma imagem pro servidor, ela está assim:
❯ docker compose up
WARN[0000] /home/gustavo/vivo_hub/docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
[+] up 0/1
⠧ Image pti-automatizado:latest Pulling                                                                                                                                                                     0.8s
[+] Building 38.8s (15/15) FINISHED
=> [internal] load local bake definitions                                                                                                                                                                  0.0s
=> => reading from stdin 497B                                                                                                                                                                              0.0s
=> [internal] load build definition from Dockerfile                                                                                                                                                        0.0s
=> => transferring dockerfile: 1.25kB                                                                                                                                                                      0.0s
=> [internal] load metadata for docker.io/library/python:3.12-slim                                                                                                                                         1.0s
=> [internal] load .dockerignore                                                                                                                                                                           0.0s
=> => transferring context: 2B                                                                                                                                                                             0.0s
=> [internal] load build context                                                                                                                                                                           2.4s
=> => transferring context: 380.36MB                                                                                                                                                                       2.4s
=> [1/8] FROM docker.io/library/python:3.12-slim@sha256:46cb7cc2877e60fbd5e21a9ae6115c30ace7a077b9f8772da879e4590c18c2e3                                                                                   0.0s
=> => resolve docker.io/library/python:3.12-slim@sha256:46cb7cc2877e60fbd5e21a9ae6115c30ace7a077b9f8772da879e4590c18c2e3                                                                                   0.0s
=> CACHED [2/8] RUN apt-get update && apt-get install -y --no-install-recommends         libfreetype6         libjpeg62-turbo         zlib1g         curl         fonts-dejavu-core     && rm -rf /var/li  0.0s
=> CACHED [3/8] WORKDIR /app                                                                                                                                                                               0.0s
=> CACHED [4/8] COPY requirements.txt .                                                                                                                                                                    0.0s
=> CACHED [5/8] RUN pip install --no-cache-dir --upgrade pip  && pip install --no-cache-dir -r requirements.txt                                                                                            0.0s
=> [6/8] COPY . .                                                                                                                                                                                          4.2s
=> [7/8] RUN mkdir -p /app/data /app/dados-sbcs /app/static                                                                                                                                                0.3s
=> [8/8] RUN useradd -r -m -s /bin/false -u 1001 pti  && chown -R pti:pti /app                                                                                                                             1.1s
=> exporting to image                                                                                                                                                                                     28.5s
=> => exporting layers                                                                                                                                                                                    14.9s
=> => exporting manifest sha256:c753a01045e307a990f0190ead9db33aefc1ffa2fb60bc732eaef2271420c034                                                                                                           0.0s
=> => exporting config sha256:3bb6ffaa88d207c006c43f9c249ce2b379c251a92ad63addfe5b77e0c3ba9fff                                                                                                             0.0s
=> => exporting attestation manifest sha256:7ac956f0401fb28c2a7df2ec634b33a1a7080dc0375d4c90414235d9f65602d0                                                                                               0.0s
=> => exporting manifest list sha256:7a01e0a8ad1f828b29f65591946820dcf5d277776a7b766fcb33be9eb648d4b1                                                                                                      0.0s
=> => naming to docker.io/library/pti-automatizado:latest                                                                                                                                                  0.0s
[+] up 2/2acking to docker.io/library/pti-automatizado:latest                                                                                                                                              13.2s
✔ Image pti-automatizado:latest Built                                                                                                                                                                      39.8s
✔ Container pti-automatizado    Created                                                                                                                                                                     0.4s
Attaching to pti-automatizado
pti-automatizado  | 2026-05-06 13:38:39 [INFO] vivohub: VIVOHUB iniciado. Diretório SBC: /app/dados-sbcs
pti-automatizado  | [2026-05-06 13:38:39 +0000] [1] [INFO] Starting gunicorn 21.2.0
pti-automatizado  | [2026-05-06 13:38:39 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000 (1)
pti-automatizado  | [2026-05-06 13:38:39 +0000] [1] [INFO] Using worker: sync
pti-automatizado  | [2026-05-06 13:38:39 +0000] [20] [INFO] Booting worker with pid: 20
pti-automatizado  | [2026-05-06 13:38:39 +0000] [21] [INFO] Booting worker with pid: 21
pti-automatizado  | 2026-05-06 13:38:44 [INFO] siprouter_sp: siprouter_sp: 180 registros inseridos.
pti-automatizado  | 127.0.0.1 - - [06/May/2026:13:38:44 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
pti-automatizado  | 172.18.0.1 - - [06/May/2026:13:39:00 +0000] "GET /login HTTP/1.1" 200 11924 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36 Edg/147.0.0.0"
pti-automatizado  | 172.18.0.1 - - [06/May/2026:13:39:00 +0000] "GET /static/favicon.ico HTTP/1.1" 404 207 "http://127.0.0.1:5000/login" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36 Edg/147.0.0.0"
pti-automatizado  | 127.0.0.1 - - [06/May/2026:13:39:14 +0000] "GET /login HTTP/1.1" 200 11924 "-" "curl/8.14.1"
 
 
depois disso eu simplesmente digito:
 
docker save -o vivo-hub.tar pti-automatizado:latest
 
e envio por sftp para o servidor, está faltando algo no processo ou dessa vez eu posso enviar sem medo que ele vai fazer o bind de ip no servidor corretamente com o novo dockerfile 4?
