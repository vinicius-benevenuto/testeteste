PS C:\Users\40418843\Desktop\verificador_de_spoofing> py identificador_operadora.py
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 18
    CAMINHO_ARQUIVO_1 = "C:\Users\40418843\Desktop\bases\TESTE_SPOOFING_18FEV 1 (1).xlsx"
                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
SyntaxError: (unicode error) 'unicodeescape' codec can't decode bytes in position 2-3: truncated \UXXXXXXXX escape
PS C:\Users\40418843\Desktop\verificador_de_spoofing> py identificador_operadora.py
2026-03-17 14:33:38 [INFO] ======================================================================
2026-03-17 14:33:38 [INFO]   IDENTIFICADOR DE OPERADORA + PORTABILIDADE — INICIANDO
2026-03-17 14:33:38 [INFO] ======================================================================
2026-03-17 14:33:38 [INFO] ======================================================================
2026-03-17 14:33:38 [INFO] ETAPA 1 — Carregando e unificando bases de prefixos...
2026-03-17 14:33:38 [INFO]   [1/4] FIXO: C:\Users\40418843\Downloads\SME_20260307_GERAL\SME_20260307_GERAL.txt
2026-03-17 14:33:38 [INFO]   Lendo CSV (latin-1): SME_20260307_GERAL.txt
2026-03-17 14:33:38 [INFO]      → 150 registros ativos carregados.
2026-03-17 14:33:38 [INFO]   [2/4] MOVEL: C:\Users\40418843\Downloads\SMP_20260307_GERAL\SMP_20260307_GERAL.txt
2026-03-17 14:33:38 [INFO]   Lendo CSV (latin-1): SMP_20260307_GERAL.txt
2026-03-17 14:33:39 [INFO]      → 553,972 registros ativos carregados.
2026-03-17 14:33:39 [INFO]   [3/4] SME: C:\Users\40418843\Downloads\STFC_20260307_GERAL\STFC_20260307_GERAL.txt
2026-03-17 14:33:39 [INFO]   Lendo CSV (latin-1): STFC_20260307_GERAL.txt
2026-03-17 14:33:40 [INFO]      → 232,078 registros ativos carregados.
2026-03-17 14:33:40 [INFO]   [4/4] OUTRO: C:\Users\40418843\Downloads\STFCFATB_20260307_GERAL\STFCFATB_20260307_GERAL.txt
2026-03-17 14:33:40 [INFO]   Lendo CSV (latin-1): STFCFATB_20260307_GERAL.txt
2026-03-17 14:33:40 [INFO]      → 480 registros ativos carregados.
2026-03-17 14:33:40 [INFO]   Total unificado: 786,680 prefixos ativos.
2026-03-17 14:33:40 [INFO]   Tempo ETAPA 1: 2.1s
2026-03-17 14:33:40 [INFO]   Indexando tabela de prefixos em memória (lookup dict)...
2026-03-17 14:33:57 [INFO]   Índice criado: 111,060 chaves (DDD, prefixo).
2026-03-17 14:33:57 [INFO]   Tempo indexação: 16.8s
2026-03-17 14:33:58 [INFO] ======================================================================
2026-03-17 14:33:58 [INFO] ETAPA 2 — Carregando base de portabilidade...
Traceback (most recent call last):
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 916, in <module>
    main()
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 799, in main
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 799, in main
    idx_portabilidade = carregar_portabilidade(CAMINHO_ARQUIVO_3)
                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    idx_portabilidade = carregar_portabilidade(CAMINHO_ARQUIVO_3)
                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 515, in carregar_portabilidade
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 515, in carregar_portabilidade
    for chunk in _ler_arquivo_em_chunks(
    for chunk in _ler_arquivo_em_chunks(
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 202, in _ler_arquivo_em_chunks
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 202, in _ler_arquivo_em_chunks
    reader = pd.read_csv(
             ^^^^^^^^^^^^
    reader = pd.read_csv(
             ^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1026, in read_csv
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1026, in read_csv
    return _read(filepath_or_buffer, kwds)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    return _read(filepath_or_buffer, kwds)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 620, in _read
    parser = TextFileReader(filepath_or_buffer, **kwds)
    parser = TextFileReader(filepath_or_buffer, **kwds)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1620, in __init__
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1620, in __init__
    self._engine = self._make_engine(f, self.engine)
    self._engine = self._make_engine(f, self.engine)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\parsers\readers.py", line 1880, in _make_engine
    self.handles = get_handle(
                   ^^^^^^^^^^^
  File "C:\Users\40418843\AppData\Roaming\Python\Python312\site-packages\pandas\io\common.py", line 882, in get_handle
    handle = open(handle, ioargs.mode)
             ^^^^^^^^^^^^^^^^^^^^^^^^^
FileNotFoundError: [Errno 2] No such file or directory: 'CAMINHO/DA/PORTABILIDADE.csv'
