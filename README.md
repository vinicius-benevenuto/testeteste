"""
================================================================================
  IDENTIFICADOR DE OPERADORA POR PREFIXO + VERIFICAÇÃO DE PORTABILIDADE
  Versão: 3.1  |  Autor: Vinicius (VIVOHUB)
  Otimizado para alto desempenho em arquivos com milhões de linhas.

  CADEIA DE LOOKUP DA PORTABILIDADE:
    A_BUSCA / B_BUSCA
        └─► ntl (portabilidade, sep=|) ──► rn1
                                               └─► RN1 (Anexo 5) ──► Nome Fantasia
================================================================================
"""

# ==============================================================================
# ██████████████████████████████████████████████████████████████████████████████
#  BLOCO DE CONFIGURAÇÃO — EDITE AQUI ANTES DE EXECUTAR
# ██████████████████████████████████████████████████████████████████████████████
# ==============================================================================

# ──────────────────────────────────────────────────────────────────────────────
# ARQUIVO 1 — BILHETAGEM  (contém A_BUSCA e B_BUSCA)
CAMINHO_ARQUIVO_1      = r"C:\Users\40418843\Desktop\bases\TESTE_SPOOFING_18FEV 1 (1).xlsx"
DELIMITADOR_BILHETAGEM = ";"        # delimitador se for CSV (ignorado para XLSX)
COLUNA_A_BUSCA         = "A_BUSCA"  # nome da coluna de origem A
COLUNA_B_BUSCA         = "B_BUSCA"  # nome da coluna de origem B

# ──────────────────────────────────────────────────────────────────────────────
# ARQUIVOS DE PREFIXOS ANATEL (4 arquivos .txt/.csv, delimitador ';')
# Ordem importa: primeiro arquivo tem maior prioridade em empates
CAMINHOS_PREFIXOS = [
    r"C:\Users\40418843\Downloads\SME_20260307_GERAL\SME_20260307_GERAL.txt",
    r"C:\Users\40418843\Downloads\SMP_20260307_GERAL\SMP_20260307_GERAL.txt",
    r"C:\Users\40418843\Downloads\STFC_20260307_GERAL\STFC_20260307_GERAL.txt",
    r"C:\Users\40418843\Downloads\STFCFATB_20260307_GERAL\STFCFATB_20260307_GERAL.txt",
]
ROTULOS_PREFIXOS = ["SME", "MOVEL", "FIXO", "FATB"]  # mesma ordem dos caminhos

# ──────────────────────────────────────────────────────────────────────────────
# ARQUIVO 2 — BASE DE PORTABILIDADE  (ntl → rn1)
# Delimitador '|', colunas: ntl | opc | rn1 | cnl | nue | nuf | tpb | dat | eip
CAMINHO_PORTABILIDADE     = r"C:\Users\40418843\Downloads\r_portab_t_IIE_348_2026_03_10.txt"
DELIMITADOR_PORTABILIDADE = "|"      # delimitador do arquivo de portabilidade
COLUNA_NTL                = "ntl"    # número portado
COLUNA_RN1_PORTAB         = "rn1"    # código RN1 da operadora atual

# ──────────────────────────────────────────────────────────────────────────────
# ARQUIVO 3 — ANEXO 5  (rn1 → Nome da Operadora)
# O cabeçalho real é detectado automaticamente nas primeiras 15 linhas.
CAMINHO_ANEXO5          = r"C:\Users\40418843\Downloads\Anexo5.xlsx"
DELIMITADOR_ANEXO5      = "\t"            # usado somente se for TXT/CSV
COLUNA_RN1_ANEXO5       = "RN1"           # nome da coluna RN1 no Anexo 5
COLUNA_OPERADORA_ANEXO5 = "Nome Fantasia" # nome da coluna de operadora no Anexo 5

# ──────────────────────────────────────────────────────────────────────────────
# SAÍDA
CAMINHO_SAIDA_XLSX = "RESULTADO.xlsx"
CAMINHO_TEMP_CSV   = "_resultado_temp.csv"

# ──────────────────────────────────────────────────────────────────────────────
# PERFORMANCE
CHUNKSIZE_BILHETAGEM = 200_000
CHUNKSIZE_PORTAB     = 500_000

# ==============================================================================
# ██████████████████████████████████████████████████████████████████████████████
#  FIM DO BLOCO DE CONFIGURAÇÃO
# ██████████████████████████████████████████████████████████████████████████████
# ==============================================================================

import os
import gc
import re
import sys
import time
import logging
import unicodedata
import warnings
from pathlib import Path
from typing import Optional

import pandas as pd

warnings.filterwarnings("ignore", category=UserWarning, module="openpyxl")
warnings.filterwarnings("ignore", category=UserWarning, module="xlrd")

# ==============================================================================
# LOGGING
# ==============================================================================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler("identificador_operadora.log", mode="w", encoding="utf-8"),
    ],
)
log = logging.getLogger(__name__)


# ==============================================================================
# HELPERS GERAIS
# ==============================================================================

def _remover_acentos(texto: str) -> str:
    nfkd = unicodedata.normalize("NFD", texto)
    return "".join(c for c in nfkd if not unicodedata.combining(c))


def _normalizar_nome_coluna(nome: str) -> str:
    """Remove '#', acentos, espaços extras e coloca em lowercase."""
    nome = nome.strip().lstrip("#").strip()
    nome = _remover_acentos(nome).lower()
    nome = re.sub(r"\s+", "_", nome)
    return nome


def _ler_arquivo_completo(caminho: str, delimitador: str = ";", dtype=None) -> pd.DataFrame:
    """Lê CSV ou XLSX completo em memória. Tenta utf-8, latin-1, cp1252."""
    p = Path(caminho)
    if not p.exists():
        raise FileNotFoundError(f"Arquivo não encontrado: {caminho}")
    if not os.access(caminho, os.R_OK):
        raise PermissionError(f"Sem permissão de leitura: {caminho}")

    ext = p.suffix.lower()

    if ext == ".xlsx":
        log.info(f"  Lendo XLSX: {p.name}")
        return pd.read_excel(caminho, dtype=dtype, engine="openpyxl")

    if ext in (".csv", ".txt", ".tsv", ""):
        for enc in ("utf-8", "utf-8-sig", "latin-1", "cp1252"):
            try:
                df = pd.read_csv(
                    caminho, sep=delimitador, dtype=dtype,
                    encoding=enc, low_memory=False,
                )
                log.info(f"  Lendo TXT/CSV ({enc}, sep='{delimitador}'): {p.name}")
                return df
            except UnicodeDecodeError:
                continue
        raise ValueError(f"Não foi possível decodificar: {caminho}")

    raise ValueError(f"Extensão não suportada: {ext}")


def _ler_arquivo_em_chunks(caminho: str, chunksize: int, delimitador: str = ";", dtype=None):
    """
    Gerador de chunks. Para XLSX faz fatiamento manual.
    Para CSV/TXT usa pd.read_csv com chunksize.
    """
    p = Path(caminho)
    ext = p.suffix.lower()

    if ext == ".xlsx":
        log.warning("  XLSX: sem suporte a chunk nativo — carregando completo e fatiando.")
        df_full = pd.read_excel(caminho, dtype=dtype, engine="openpyxl")
        for start in range(0, len(df_full), chunksize):
            yield df_full.iloc[start: start + chunksize].copy()
        del df_full; gc.collect()
        return

    if ext in (".csv", ".txt", ".tsv", ""):
        for enc in ("utf-8", "utf-8-sig", "latin-1", "cp1252"):
            try:
                reader = pd.read_csv(
                    caminho, sep=delimitador, dtype=dtype,
                    encoding=enc, chunksize=chunksize, low_memory=False,
                )
                log.info(f"  Chunks ({enc}, sep='{delimitador}'): {p.name}")
                yield from reader
                return
            except UnicodeDecodeError:
                continue
        raise ValueError(f"Não foi possível decodificar: {caminho}")

    raise ValueError(f"Extensão não suportada: {ext}")


# ==============================================================================
# ETAPA 1 — PREFIXOS
# ==============================================================================

def unificar_bases_prefixos(caminhos: list, rotulos: list) -> pd.DataFrame:
    """
    Carrega os 4 arquivos ANATEL de prefixos, normaliza colunas,
    filtra status==1 e une em uma única tabela.
    """
    log.info("=" * 70)
    log.info("ETAPA 1 — Carregando e unificando bases de prefixos...")
    t0 = time.time()

    MAP_COLUNAS = {
        "nome_da_prestadora": "nome_prestadora",
        "nomeda_prestadora":  "nome_prestadora",
        "nome_prestadora":    "nome_prestadora",
        "cnpj_da_prestadora": "cnpj_prestadora",
        "cnpjda_prestadora":  "cnpj_prestadora",
        "cnpj_prestadora":    "cnpj_prestadora",
        "codigo_nacional":    "codigo_nacional",
        "prefixo":            "prefixo",
        "faixa_inicial":      "faixa_inicial",
        "faixa_final":        "faixa_final",
        "status":             "status",
    }
    OBRIGATORIAS = [
        "nome_prestadora", "codigo_nacional", "prefixo",
        "faixa_inicial", "faixa_final", "status",
    ]

    dfs = []
    for idx, (caminho, rotulo) in enumerate(zip(caminhos, rotulos)):
        log.info(f"  [{idx+1}/{len(caminhos)}] {rotulo}: {Path(caminho).name}")
        df = _ler_arquivo_completo(caminho, delimitador=";", dtype=str)
        df.columns = [_normalizar_nome_coluna(c) for c in df.columns]
        df.rename(columns=MAP_COLUNAS, inplace=True)

        faltando = [c for c in OBRIGATORIAS if c not in df.columns]
        if faltando:
            raise ValueError(
                f"Prefixo '{Path(caminho).name}' — colunas ausentes: {faltando}\n"
                f"Colunas encontradas: {list(df.columns)}"
            )

        df = df[OBRIGATORIAS].copy()
        df["status"] = pd.to_numeric(df["status"], errors="coerce").fillna(0).astype(int)
        df = df[df["status"] == 1].copy()

        df["faixa_inicial"] = pd.to_numeric(df["faixa_inicial"], errors="coerce").astype("Int32")
        df["faixa_final"]   = pd.to_numeric(df["faixa_final"],   errors="coerce").astype("Int32")
        df.dropna(subset=["codigo_nacional", "prefixo", "faixa_inicial", "faixa_final"], inplace=True)

        df["codigo_nacional"] = df["codigo_nacional"].str.strip().str.zfill(2)
        df["prefixo"]         = df["prefixo"].str.strip().str.zfill(4)
        df["fonte_prefixo"]   = rotulo
        df["ordem_fonte"]     = idx

        dfs.append(df)
        log.info(f"     → {len(df):,} registros ativos.")

    tabela = pd.concat(dfs, ignore_index=True)
    del dfs; gc.collect()

    tabela["amplitude_faixa"] = (
        tabela["faixa_final"].astype(int) - tabela["faixa_inicial"].astype(int)
    )

    log.info(f"  Total unificado: {len(tabela):,} prefixos ativos.")
    log.info(f"  Tempo ETAPA 1: {time.time() - t0:.1f}s")
    return tabela


def indexar_prefixos(tabela: pd.DataFrame) -> dict:
    """
    Constrói dict de lookup: {(ddd_str, prefixo_str): DataFrame_ordenado}
    Já ordenado por (amplitude_faixa ASC, ordem_fonte ASC) para desempate O(1).
    """
    log.info("  Indexando prefixos em memória...")
    t0 = time.time()

    tabela = tabela.copy()
    tabela["codigo_nacional"] = tabela["codigo_nacional"].astype(str)
    tabela["prefixo"]         = tabela["prefixo"].astype(str)

    tabela.sort_values(
        ["codigo_nacional", "prefixo", "amplitude_faixa", "ordem_fonte"],
        inplace=True,
    )

    idx = {}
    for (ddd, pfx), grupo in tabela.groupby(
        ["codigo_nacional", "prefixo"], sort=False, observed=True
    ):
        idx[(ddd, pfx)] = grupo.reset_index(drop=True)

    log.info(f"  Índice: {len(idx):,} chaves | Tempo: {time.time() - t0:.1f}s")
    return idx


# ==============================================================================
# ETAPA 2a — PORTABILIDADE (ntl → rn1)
# ==============================================================================

def carregar_portabilidade() -> dict:
    """
    Lê a base de portabilidade com delimitador '|' em chunks e constrói:
        { numero_normalizado : rn1_str }
    """
    log.info("=" * 70)
    log.info("ETAPA 2a — Carregando portabilidade (ntl → rn1)...")
    t0 = time.time()

    portab: dict = {}
    total = erros = 0

    col_ntl = _normalizar_nome_coluna(COLUNA_NTL)
    col_rn1 = _normalizar_nome_coluna(COLUNA_RN1_PORTAB)

    for chunk in _ler_arquivo_em_chunks(
        CAMINHO_PORTABILIDADE,
        chunksize=CHUNKSIZE_PORTAB,
        delimitador=DELIMITADOR_PORTABILIDADE,
        dtype=str,
    ):
        chunk.columns = [_normalizar_nome_coluna(c) for c in chunk.columns]

        if col_ntl not in chunk.columns or col_rn1 not in chunk.columns:
            raise ValueError(
                f"Portabilidade: colunas '{col_ntl}' e/ou '{col_rn1}' não encontradas.\n"
                f"Colunas disponíveis: {list(chunk.columns)}"
            )

        for num_bruto, rn1 in zip(
            chunk[col_ntl].fillna("").astype(str),
            chunk[col_rn1].fillna("").astype(str),
        ):
            num_norm = normalizar_numero(num_bruto)
            if num_norm:
                portab[num_norm] = rn1.strip()
                total += 1
            else:
                erros += 1

        del chunk; gc.collect()

    log.info(f"  Portabilidade: {total:,} números indexados.")
    if erros:
        log.warning(f"  {erros:,} números ignorados (inválidos).")
    log.info(f"  Tempo ETAPA 2a: {time.time() - t0:.1f}s")
    return portab


# ==============================================================================
# ETAPA 2b — ANEXO 5 (rn1 → Nome da Operadora)
# ==============================================================================

def carregar_anexo5() -> dict:
    """
    Lê o Anexo 5 da ANATEL e constrói:
        { rn1_str : nome_operadora_str }

    Detecta automaticamente em qual linha está o cabeçalho real,
    varrendo as primeiras 15 linhas do arquivo em busca da coluna RN1.
    Isso resolve arquivos com títulos ou metadados nas linhas iniciais.
    """
    log.info("ETAPA 2b — Carregando Anexo 5 (rn1 → operadora)...")
    t0 = time.time()

    p   = Path(CAMINHO_ANEXO5)
    ext = p.suffix.lower()

    alvo_rn1 = _normalizar_nome_coluna(COLUNA_RN1_ANEXO5)
    alvo_op  = _normalizar_nome_coluna(COLUNA_OPERADORA_ANEXO5)

    # ── Passo 1: preview das primeiras 15 linhas sem cabeçalho ───────────────
    if ext == ".xlsx":
        df_preview = pd.read_excel(
            CAMINHO_ANEXO5, header=None, nrows=15, dtype=str, engine="openpyxl"
        )
    else:
        for enc in ("utf-8", "utf-8-sig", "latin-1", "cp1252"):
            try:
                df_preview = pd.read_csv(
                    CAMINHO_ANEXO5, sep=DELIMITADOR_ANEXO5,
                    header=None, nrows=15, dtype=str,
                    encoding=enc, low_memory=False,
                )
                break
            except UnicodeDecodeError:
                continue

    # ── Passo 2: encontrar a linha onde está o cabeçalho real ────────────────
    header_row = None
    for i, row in df_preview.iterrows():
        colunas_norm = [_normalizar_nome_coluna(str(v)) for v in row.values]
        if alvo_rn1 in colunas_norm:
            header_row = i
            log.info(f"  Cabeçalho real detectado na linha {header_row} do Anexo 5.")
            break

    if header_row is None:
        raise ValueError(
            f"Anexo 5: coluna '{COLUNA_RN1_ANEXO5}' não encontrada nas primeiras 15 linhas.\n"
            f"Verifique COLUNA_RN1_ANEXO5 no bloco de configuração.\n"
            f"Conteúdo das primeiras linhas do arquivo:\n{df_preview.to_string()}"
        )

    # ── Passo 3: leitura real apontando para a linha correta de cabeçalho ────
    if ext == ".xlsx":
        df = pd.read_excel(
            CAMINHO_ANEXO5, header=header_row, dtype=str, engine="openpyxl"
        )
    else:
        for enc in ("utf-8", "utf-8-sig", "latin-1", "cp1252"):
            try:
                df = pd.read_csv(
                    CAMINHO_ANEXO5, sep=DELIMITADOR_ANEXO5,
                    header=header_row, dtype=str,
                    encoding=enc, low_memory=False,
                )
                break
            except UnicodeDecodeError:
                continue

    # Normaliza nomes de colunas
    df.columns = [_normalizar_nome_coluna(str(c)) for c in df.columns]

    if alvo_rn1 not in df.columns:
        raise ValueError(
            f"Anexo 5: coluna RN1 '{alvo_rn1}' não encontrada após leitura.\n"
            f"Colunas disponíveis: {list(df.columns)}"
        )
    if alvo_op not in df.columns:
        raise ValueError(
            f"Anexo 5: coluna de operadora '{alvo_op}' não encontrada após leitura.\n"
            f"Colunas disponíveis: {list(df.columns)}"
        )

    df = df[[alvo_rn1, alvo_op]].dropna(subset=[alvo_rn1]).copy()
    df[alvo_rn1] = df[alvo_rn1].astype(str).str.strip()
    df[alvo_op]  = df[alvo_op].fillna("").astype(str).str.strip()

    idx_anexo5 = dict(zip(df[alvo_rn1], df[alvo_op]))

    log.info(f"  Anexo 5: {len(idx_anexo5):,} operadoras mapeadas.")
    log.info(f"  Tempo ETAPA 2b: {time.time() - t0:.1f}s")
    return idx_anexo5


# ==============================================================================
# NORMALIZAÇÃO DE NÚMERO
# ==============================================================================

def normalizar_numero(numero: str) -> Optional[str]:
    """
    Normaliza número brasileiro:
      1. Mantém apenas dígitos.
      2. Remove DDI '55' inicial (se len > 12).
      3. Retorna None se resultado < 10 dígitos.
    """
    if not isinstance(numero, str) or not numero.strip():
        return None

    digitos = re.sub(r"\D", "", numero)

    if digitos.startswith("55") and len(digitos) > 12:
        digitos = digitos[2:]

    return digitos if len(digitos) >= 10 else None


# ==============================================================================
# BUSCA DE OPERADORA POR PREFIXO
# ==============================================================================

def buscar_operadora_prefixo(numero_norm: str, idx_prefixos: dict) -> dict:
    """
    Identifica DDD, prefixo, faixa e operadora pelo número normalizado.

    Candidato A: numero_local[0:4]  → fixo e móvel sem '9' inicial
    Candidato B: numero_local[1:5]  → móvel com '9' inicial (11 dígitos)
    Desempate: menor amplitude_faixa → menor ordem_fonte.
    """
    VAZIO = {
        "ddd":                    "",
        "prefixo_casado":         "",
        "faixa_utilizada":        "",
        "fonte_prefixo":          "",
        "operadora_pelo_prefixo": "PREFIXO NÃO ENCONTRADO",
    }

    if not numero_norm or len(numero_norm) < 10:
        return VAZIO

    ddd          = numero_norm[:2].zfill(2)
    numero_local = numero_norm[2:]

    try:
        estacao = int(numero_local[-4:])
    except (ValueError, IndexError):
        return VAZIO

    candidatos = []
    if len(numero_local) >= 4:
        candidatos.append(numero_local[:4].zfill(4))
    if numero_local.startswith("9") and len(numero_local) >= 5:
        candidatos.append(numero_local[1:5].zfill(4))

    melhor = None
    for pfx_cand in candidatos:
        chave = (ddd, pfx_cand)
        if chave not in idx_prefixos:
            continue

        grupo   = idx_prefixos[chave]
        mascara = (
            (grupo["faixa_inicial"].astype(int) <= estacao) &
            (grupo["faixa_final"].astype(int)   >= estacao)
        )
        validos = grupo[mascara]
        if validos.empty:
            continue

        linha = validos.iloc[0]  # já ordenado na indexação
        if melhor is None:
            melhor = (pfx_cand, linha)
        else:
            _, ml = melhor
            if (linha["amplitude_faixa"], linha["ordem_fonte"]) < (
                ml["amplitude_faixa"], ml["ordem_fonte"]
            ):
                melhor = (pfx_cand, linha)

    if melhor is None:
        return VAZIO

    pfx_final, l = melhor
    return {
        "ddd":                    ddd,
        "prefixo_casado":         pfx_final,
        "faixa_utilizada":        f"{int(l['faixa_inicial'])}-{int(l['faixa_final'])}",
        "fonte_prefixo":          str(l["fonte_prefixo"]),
        "operadora_pelo_prefixo": str(l["nome_prestadora"]),
    }


# ==============================================================================
# VERIFICAÇÃO DE PORTABILIDADE (ntl → rn1 → Anexo 5)
# ==============================================================================

def verificar_portabilidade(
    numero_norm: str,
    operadora_prefixo: str,
    idx_portab: dict,
    idx_anexo5: dict,
) -> tuple:
    """
    Cadeia de lookup:
      1. numero_norm → idx_portab → rn1
      2. rn1 → idx_anexo5 → nome da operadora atual
      3. Compara com operadora_prefixo → PORTADO / NÃO PORTADO

    Retorna (rn1, operadora_atual, status_portabilidade).
    """
    if numero_norm not in idx_portab:
        return ("", "", "NÃO ENCONTRADO NA PORTABILIDADE")

    rn1 = idx_portab[numero_norm]
    if not rn1:
        return ("", "", "NÃO ENCONTRADO NA PORTABILIDADE")

    operadora_atual = idx_anexo5.get(rn1, f"RN1={rn1} (não mapeado no Anexo 5)")

    if operadora_atual.strip().lower() != operadora_prefixo.strip().lower():
        status = "PORTADO"
    else:
        status = "NÃO PORTADO"

    return (rn1, operadora_atual, status)


# ==============================================================================
# EXTRAÇÃO DE A_BUSCA / B_BUSCA
# ==============================================================================

def extrair_numeros(caminho: str):
    """
    Gerador: lê bilhetagem em chunks e yields DataFrames com:
        numero_bruto | origem_numero ('A_BUSCA' ou 'B_BUSCA')
    """
    log.info("=" * 70)
    log.info("ETAPA 3 — Lendo bilhetagem (A_BUSCA / B_BUSCA)...")

    col_a = _normalizar_nome_coluna(COLUNA_A_BUSCA)
    col_b = _normalizar_nome_coluna(COLUNA_B_BUSCA)

    for chunk in _ler_arquivo_em_chunks(
        caminho, chunksize=CHUNKSIZE_BILHETAGEM,
        delimitador=DELIMITADOR_BILHETAGEM, dtype=str,
    ):
        chunk.columns = [_normalizar_nome_coluna(c) for c in chunk.columns]

        for col, origem in [(col_a, "A_BUSCA"), (col_b, "B_BUSCA")]:
            if col not in chunk.columns:
                log.warning(f"  Coluna '{col}' não encontrada — ignorando.")
                continue
            parcial = pd.DataFrame({
                "numero_bruto":  chunk[col].fillna("").astype(str),
                "origem_numero": origem,
            })
            parcial = parcial[parcial["numero_bruto"].str.strip() != ""]
            yield parcial

        del chunk; gc.collect()


# ==============================================================================
# PROCESSAMENTO DE CHUNK
# ==============================================================================

def processar_chunk(
    df_nums: pd.DataFrame,
    idx_prefixos: dict,
    idx_portab: dict,
    idx_anexo5: dict,
) -> pd.DataFrame:
    """
    Aplica sobre um chunk de números:
      1. Normalização.
      2. Busca de operadora por prefixo.
      3. Verificação de portabilidade (ntl → rn1 → Anexo 5).
    """
    df = df_nums.copy()

    # 1. Normalizar número
    df["numero_normalizado"] = df["numero_bruto"].apply(normalizar_numero)

    # 2. Buscar prefixo
    pfx_results = df["numero_normalizado"].apply(
        lambda n: buscar_operadora_prefixo(n, idx_prefixos)
        if n else {
            "ddd": "", "prefixo_casado": "", "faixa_utilizada": "",
            "fonte_prefixo": "", "operadora_pelo_prefixo": "PREFIXO NÃO ENCONTRADO"
        }
    )
    df_pfx = pd.DataFrame(pfx_results.tolist(), index=df.index)

    # 3. Verificar portabilidade
    portab_results = df[["numero_normalizado"]].join(
        df_pfx[["operadora_pelo_prefixo"]]
    ).apply(
        lambda row: verificar_portabilidade(
            row["numero_normalizado"] or "",
            row["operadora_pelo_prefixo"],
            idx_portab,
            idx_anexo5,
        ),
        axis=1,
        result_type="expand",
    )
    portab_results.columns = ["rn1", "operadora_atual", "status_portabilidade"]

    # 4. Montar saída
    resultado = pd.concat([
        df[["numero_normalizado", "origem_numero"]].reset_index(drop=True),
        df_pfx.reset_index(drop=True),
        portab_results.reset_index(drop=True),
    ], axis=1)

    COLUNAS_SAIDA = [
        "numero_normalizado",
        "ddd",
        "prefixo_casado",
        "faixa_utilizada",
        "fonte_prefixo",
        "operadora_pelo_prefixo",
        "rn1",
        "operadora_atual",
        "status_portabilidade",
        "origem_numero",
    ]
    for col in COLUNAS_SAIDA:
        if col not in resultado.columns:
            resultado[col] = ""

    return resultado[COLUNAS_SAIDA]


# ==============================================================================
# GERAÇÃO DO RESULTADO (CSV TEMP → XLSX)
# ==============================================================================

def gerar_resultado(caminho_csv_temp: str, caminho_xlsx: str) -> None:
    """Converte CSV temporário acumulado em RESULTADO.xlsx (write_only = baixo RAM)."""
    log.info("=" * 70)
    log.info("ETAPA 5 — Convertendo CSV temporário para XLSX...")
    t0 = time.time()

    from openpyxl import Workbook
    from openpyxl.utils.dataframe import dataframe_to_rows
    from openpyxl.styles import Font, PatternFill, Alignment
    from openpyxl.cell import WriteOnlyCell

    wb = Workbook(write_only=True)
    ws = wb.create_sheet("Resultado")

    FONT_CAB  = Font(bold=True, color="FFFFFF", name="Calibri", size=11)
    FILL_CAB  = PatternFill("solid", fgColor="1F4E79")
    ALIGN_CTR = Alignment(horizontal="center", vertical="center")

    cabecalho_escrito = False
    total_linhas = 0

    for chunk in pd.read_csv(
        caminho_csv_temp, sep=";", encoding="utf-8",
        chunksize=100_000, dtype=str,
    ):
        chunk.fillna("", inplace=True)

        if not cabecalho_escrito:
            row_cab = []
            for col_name in chunk.columns:
                cell = WriteOnlyCell(ws, value=col_name)
                cell.font = FONT_CAB; cell.fill = FILL_CAB; cell.alignment = ALIGN_CTR
                row_cab.append(cell)
            ws.append(row_cab)
            cabecalho_escrito = True

        for row in dataframe_to_rows(chunk, index=False, header=False):
            ws.append(row)
            total_linhas += 1

        del chunk; gc.collect()

    wb.save(caminho_xlsx)
    log.info(f"  XLSX gerado: {caminho_xlsx} ({total_linhas:,} linhas).")
    log.info(f"  Tempo ETAPA 5: {time.time() - t0:.1f}s")


# ==============================================================================
# MAIN
# ==============================================================================

def main() -> None:
    log.info("=" * 70)
    log.info("  IDENTIFICADOR DE OPERADORA + PORTABILIDADE — INICIANDO")
    log.info("=" * 70)
    t_total = time.time()

    # ETAPA 1 — Prefixos
    tabela_pfx   = unificar_bases_prefixos(CAMINHOS_PREFIXOS, ROTULOS_PREFIXOS)
    idx_prefixos = indexar_prefixos(tabela_pfx)
    del tabela_pfx; gc.collect()

    # ETAPA 2 — Portabilidade + Anexo 5
    idx_portab = carregar_portabilidade()
    idx_anexo5 = carregar_anexo5()

    # ETAPA 3+4 — Processar bilhetagem em chunks
    log.info("=" * 70)
    log.info("ETAPA 4 — Processando bilhetagem...")
    t4 = time.time()

    if os.path.exists(CAMINHO_TEMP_CSV):
        os.remove(CAMINHO_TEMP_CSV)

    total = portados = nao_portados = nao_encontrados = sem_prefixo = 0
    chunk_num = 0
    erros_log = []

    for df_nums in extrair_numeros(CAMINHO_ARQUIVO_1):
        chunk_num += 1
        t_chunk = time.time()
        try:
            df_res = processar_chunk(df_nums, idx_prefixos, idx_portab, idx_anexo5)

            total           += len(df_res)
            portados        += (df_res["status_portabilidade"] == "PORTADO").sum()
            nao_portados    += (df_res["status_portabilidade"] == "NÃO PORTADO").sum()
            nao_encontrados += (df_res["status_portabilidade"] == "NÃO ENCONTRADO NA PORTABILIDADE").sum()
            sem_prefixo     += (df_res["operadora_pelo_prefixo"] == "PREFIXO NÃO ENCONTRADO").sum()

            modo_w = (chunk_num == 1)
            df_res.to_csv(
                CAMINHO_TEMP_CSV, sep=";", index=False,
                header=modo_w, mode="w" if modo_w else "a", encoding="utf-8",
            )
            log.info(
                f"  Chunk {chunk_num:04d}: {len(df_res):,} linhas | "
                f"{time.time() - t_chunk:.2f}s"
            )
        except Exception as exc:
            log.error(f"  ERRO chunk {chunk_num}: {exc}", exc_info=True)
            erros_log.append(f"Chunk {chunk_num}: {exc}")

        del df_nums; gc.collect()

    log.info(f"  Tempo ETAPA 4: {time.time() - t4:.1f}s")

    # ETAPA 5 — CSV temp → XLSX
    if total == 0:
        log.warning("  Nenhum número processado. Verifique a bilhetagem.")
    else:
        gerar_resultado(CAMINHO_TEMP_CSV, CAMINHO_SAIDA_XLSX)
        try:
            os.remove(CAMINHO_TEMP_CSV)
        except OSError as e:
            log.warning(f"  Não foi possível remover CSV temp: {e}")

    # RESUMO FINAL
    log.info("=" * 70)
    log.info("  RESUMO FINAL")
    log.info(f"  Total processado             : {total:>12,}")
    log.info(f"  Portados                     : {portados:>12,}")
    log.info(f"  Não portados                 : {nao_portados:>12,}")
    log.info(f"  Não encontrados portabilidade: {nao_encontrados:>12,}")
    log.info(f"  Sem prefixo identificado     : {sem_prefixo:>12,}")
    if erros_log:
        log.warning(f"  Erros ({len(erros_log)}):")
        for msg in erros_log[:10]:
            log.warning(f"    {msg}")
    log.info(f"  Tempo total                  : {time.time() - t_total:.1f}s")
    log.info(f"  Arquivo gerado               : {CAMINHO_SAIDA_XLSX}")
    log.info("=" * 70)


if __name__ == "__main__":
    main()
