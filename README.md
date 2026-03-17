PS C:\Users\40418843\Desktop\verificador_de_spoofing> py identificador_operadora.py
2026-03-17 14:38:07 [INFO] ======================================================================
2026-03-17 14:38:07 [INFO]   IDENTIFICADOR DE OPERADORA + PORTABILIDADE — INICIANDO
2026-03-17 14:38:07 [INFO] ======================================================================
2026-03-17 14:38:07 [INFO] ======================================================================
2026-03-17 14:38:07 [INFO] ETAPA 1 — Carregando e unificando bases de prefixos...
2026-03-17 14:38:07 [INFO]   [1/4] FIXO: C:\Users\40418843\Downloads\SME_20260307_GERAL\SME_20260307_GERAL.txt
2026-03-17 14:38:07 [INFO]   Lendo CSV (latin-1): SME_20260307_GERAL.txt
2026-03-17 14:38:07 [INFO]      → 150 registros ativos carregados.
2026-03-17 14:38:07 [INFO]   [2/4] MOVEL: C:\Users\40418843\Downloads\SMP_20260307_GERAL\SMP_20260307_GERAL.txt
2026-03-17 14:38:07 [INFO]   Lendo CSV (latin-1): SMP_20260307_GERAL.txt
2026-03-17 14:38:08 [INFO]      → 553,972 registros ativos carregados.
2026-03-17 14:38:08 [INFO]   [3/4] SME: C:\Users\40418843\Downloads\STFC_20260307_GERAL\STFC_20260307_GERAL.txt
2026-03-17 14:38:08 [INFO]   Lendo CSV (latin-1): STFC_20260307_GERAL.txt
2026-03-17 14:38:09 [INFO]      → 232,078 registros ativos carregados.
2026-03-17 14:38:09 [INFO]   [4/4] OUTRO: C:\Users\40418843\Downloads\STFCFATB_20260307_GERAL\STFCFATB_20260307_GERAL.txt
2026-03-17 14:38:09 [INFO]   Lendo CSV (latin-1): STFCFATB_20260307_GERAL.txt
2026-03-17 14:38:09 [INFO]      → 480 registros ativos carregados.
2026-03-17 14:38:09 [INFO]   Total unificado: 786,680 prefixos ativos.
2026-03-17 14:38:09 [INFO]   Tempo ETAPA 1: 2.1s
2026-03-17 14:38:09 [INFO]   Indexando tabela de prefixos em memória (lookup dict)...
2026-03-17 14:38:24 [INFO]   Índice criado: 111,060 chaves (DDD, prefixo).
2026-03-17 14:38:24 [INFO]   Tempo indexação: 15.2s
2026-03-17 14:38:27 [INFO] ======================================================================
2026-03-17 14:38:27 [INFO] ETAPA 2 — Carregando base de portabilidade...
2026-03-17 14:38:27 [INFO]   Lendo CSV em chunks (utf-8): r_portab_t_IIE_348_2026_03_10.txt
Traceback (most recent call last):
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 916, in <module>
    main()
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 799, in main
    idx_portabilidade = carregar_portabilidade(CAMINHO_ARQUIVO_3)
                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\Users\40418843\Desktop\verificador_de_spoofing\identificador_operadora.py", line 528, in carregar_portabilidade
    raise ValueError(
ValueError: Portabilidade: colunas 'numero' e/ou 'operadora_atual' não encontradas.
Colunas disponíveis: ['ntl|opc|rn1|cnl|nue|nuf|tpb|dat|eip']
PS C:\Users\40418843\Desktop\verificador_de_spoofing> 

O CAMPO NTL SÃO OS VALORES DE A_BUSCA E B_BUSCA. ENCONTRANDO A_BUSCA E B_BUSCAM, DEVEMOS RECORRER A COLUNA RN1, ATRAVÉS DO RN1 QUE SABEMOS QUAL É A OPERADORA QUE ESTÁ PRESENTE EM OUTRO ARQUIVO. ESTE ARQUIVO É O ANEXO 5 ELE GUARDA DADOS COMO OPERADORA E RN1. VOU TE PASSAR AS COLUNAS. [EOT	Nome Fantasia	Razão Social	CSP	Tipo de Serviço	Modalidade / Banda	Área de Prestação	Holding	CNPJ	Inscrição Estadual	Endereço de Emissão Nota Fiscal	Endereço de Correspondência	UF	Região	Concessão	RN1	SPID]
