PS C:\Users\40418843\Desktop\verificador_de_spoofing> py identificador_operadora.py
2026-03-17 14:57:24 [INFO] ======================================================================
2026-03-17 14:57:24 [INFO]   IDENTIFICADOR DE OPERADORA + PORTABILIDADE — INICIANDO
2026-03-17 14:57:24 [INFO] ======================================================================
2026-03-17 14:57:24 [INFO] ======================================================================
2026-03-17 14:57:24 [INFO] ETAPA 1 — Carregando e unificando bases de prefixos...
2026-03-17 14:57:24 [INFO]   [1/4] SME: SME_20260307_GERAL.txt
2026-03-17 14:57:24 [INFO]   Lendo TXT/CSV (latin-1, sep=';'): SME_20260307_GERAL.txt
2026-03-17 14:57:24 [INFO]      → 150 registros ativos.
2026-03-17 14:57:24 [INFO]   [2/4] MOVEL: SMP_20260307_GERAL.txt
2026-03-17 14:57:24 [INFO]   Lendo TXT/CSV (latin-1, sep=';'): SMP_20260307_GERAL.txt
2026-03-17 14:57:25 [INFO]      → 553,972 registros ativos.
2026-03-17 14:57:25 [INFO]   [3/4] FIXO: STFC_20260307_GERAL.txt
2026-03-17 14:57:25 [INFO]   Lendo TXT/CSV (latin-1, sep=';'): STFC_20260307_GERAL.txt
2026-03-17 14:57:26 [INFO]      → 232,078 registros ativos.
2026-03-17 14:57:26 [INFO]   [4/4] FATB: STFCFATB_20260307_GERAL.txt
2026-03-17 14:57:26 [INFO]   Lendo TXT/CSV (latin-1, sep=';'): STFCFATB_20260307_GERAL.txt
2026-03-17 14:57:26 [INFO]      → 480 registros ativos.
2026-03-17 14:57:26 [INFO]   Total unificado: 786,680 prefixos ativos.
2026-03-17 14:57:26 [INFO]   Tempo ETAPA 1: 1.9s
2026-03-17 14:57:26 [INFO]   Indexando prefixos em memória...
2026-03-17 14:57:40 [INFO]   Índice: 111,060 chaves | Tempo: 14.7s
2026-03-17 14:57:41 [INFO] ======================================================================
2026-03-17 14:57:41 [INFO] ETAPA 2a — Carregando portabilidade (ntl → rn1)...
2026-03-17 14:57:41 [INFO]   Chunks (utf-8, sep='|'): r_portab_t_IIE_348_2026_03_10.txt
2026-03-17 15:04:00 [INFO]   Portabilidade: 58,550,634 números indexados.
2026-03-17 15:04:00 [INFO]   Tempo ETAPA 2a: 378.2s
2026-03-17 15:04:00 [INFO] ETAPA 2b — Carregando Anexo 5 (rn1 → operadora)...
2026-03-17 15:04:00 [INFO]   Lendo XLSX: Anexo5.xlsx
Traceback (most recent call last):
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 760, in <module>
    main()
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 689, in main
    idx_anexo5 = carregar_anexo5()
                 ^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 361, in carregar_anexo5
    raise ValueError(
ValueError: Anexo 5: coluna RN1 'rn1' não encontrada.
Colunas disponíveis: ['unnamed:_0', 'unnamed:_1', '', 'unnamed:_3', 'unnamed:_4', 'unnamed:_5', 'unnamed:_6', 'unnamed:_7', 'unnamed:_8', 'unnamed:_9', 'unnamed:_10', 'unnamed:_11', 'unnamed:_12', 'unnamed:_13', 'unnamed:_14', 'unnamed:_15', 'unnamed:_16', 'unnamed:_17']
PS C:\Users\40418843\Desktop\verificador_de_spoofing> 
