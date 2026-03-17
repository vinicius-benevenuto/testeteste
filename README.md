"""
================================================================================
  IDENTIFICADOR DE OPERADORA POR PREFIXO + VERIFICAÇÃO DE PORTABILIDADE
  Versão: 2.0  |  Autor: Vinicius (VIVOHUB)
  Otimizado para alto desempenho em arquivos com milhões de linhas.
================================================================================
"""

# ==============================================================================
# ██████████████████████████████████████████████████████████████████████████████
#  BLOCO DE CONFIGURAÇÃO — EDITE AQUI ANTES DE EXECUTAR
# ██████████████████████████████████████████████████████████████████████████████
# ==============================================================================

# ──────────────────────────────────────────────────────────────────────────────
# ARQUIVO 1 — BILHETAGEM  (contém A_BUSCA e B_BUSCA)
# Suporta .csv (delimitador ';') ou .xlsx
CAMINHO_ARQUIVO_1 = "CAMINHO/DO/ARQUIVO_1.csv"

# ──────────────────────────────────────────────────────────────────────────────
# ARQUIVOS DE PREFIXOS — 4 CSV delimitados por ';'
# Ordem importa: FIXO tem prioridade sobre MOVEL, SME e OUTRO em caso de empate
CAMINHOS_PREFIXOS = [
    "CAMINHO/DO/PREFIXO_FIXO.csv",
    "CAMINHO/DO/PREFIXO_MOVEL.csv",
    "CAMINHO/DO/PREFIXO_SME.csv",
    "CAMINHO/DO/PREFIXO_OUTRO.csv",
]

# Rótulos correspondentes (mesma ordem dos caminhos acima)
ROTULOS_PREFIXOS = ["FIXO", "MOVEL", "SME", "OUTRO"]

# ──────────────────────────────────────────────────────────────────────────────
# ARQUIVO 3 — BASE DE PORTABILIDADE  (.csv ';' ou .xlsx)
CAMINHO_ARQUIVO_3 = "CAMINHO/DA/PORTABILIDADE.csv"

# Nomes das colunas dentro do arquivo de portabilidade
COLUNA_NUMERO_PORTABILIDADE = "numero"          # coluna com o número portado
COLUNA_OPERADORA_PORTABILIDADE = "operadora_atual"  # coluna com a operadora atual

# ──────────────────────────────────────────────────────────────────────────────
# SAÍDA
CAMINHO_SAIDA_XLSX = "RESULTADO.xlsx"           # arquivo final gerado
CAMINHO_TEMP_CSV  = "_resultado_temp.csv"       # CSV intermediário (apagado ao final)

# ──────────────────────────────────────────────────────────────────────────────
# PERFORMANCE
CHUNKSIZE_BILHETAGEM = 200_000   # linhas por chunk da bilhetagem
CHUNKSIZE_PORTAB     = 500_000   # linhas por chunk da portabilidade (para indexação)

# ──────────────────────────────────────────────────────────────────────────────
# COLUNAS DA BILHETAGEM QUE CONTÊM OS NÚMEROS A PESQUISAR
COLUNA_A_BUSCA = "A_BUSCA"
COLUNA_B_BUSCA = "B_BUSCA"

# ==============================================================================
# ██████████████████████████████████████████████████████████████████████████████
#  FIM DO BLOCO DE CONFIGURAÇÃO
# ██████████████████████████████████████████████████████████████████████████████
# ==============================================================================


# ──────────────────────────────────────────────────────────────────────────────
# IMPORTS
# ──────────────────────────────────────────────────────────────────────────────
import os
import gc
import re
import sys
import time
import logging
import tempfile
import unicodedata
import warnings
from pathlib import Path
from typing import Optional

import pandas as pd
import numpy as np

# Suprime avisos desnecessários do openpyxl/xlrd
warnings.filterwarnings("ignore", category=UserWarning, module="openpyxl")
warnings.filterwarnings("ignore", category=UserWarning, module="xlrd")

# ==============================================================================
# CONFIGURAÇÃO DE LOGGING
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
# FUNÇÕES AUXILIARES
# ==============================================================================

def _remover_acentos(texto: str) -> str:
    """Remove acentos de uma string usando normalização Unicode NFD."""
    nfkd = unicodedata.normalize("NFD", texto)
    return "".join(c for c in nfkd if not unicodedata.combining(c))


def _normalizar_nome_coluna(nome: str) -> str:
    """
    Padroniza nome de coluna:
      - Remove '#' inicial
      - Strip de espaços
      - Lowercase
      - Remove acentos
      - Substitui espaços internos por '_'
    """
    nome = nome.strip().lstrip("#").strip()
    nome = _remover_acentos(nome).lower()
    nome = re.sub(r"\s+", "_", nome)
    return nome


def _ler_arquivo(caminho: str, dtype: Optional[dict] = None) -> pd.DataFrame:
    """
    Lê um único arquivo .csv (delimitador ';') ou .xlsx completo em memória.
    Tenta UTF-8; em caso de falha, tenta latin-1.

    Parâmetros
    ----------
    caminho : str
        Caminho completo para o arquivo.
    dtype : dict, opcional
        Dicionário de tipos para forçar durante a leitura.

    Retorna
    -------
    pd.DataFrame com o conteúdo do arquivo.
    """
    p = Path(caminho)
    if not p.exists():
        raise FileNotFoundError(f"Arquivo não encontrado: {caminho}")
    if not os.access(caminho, os.R_OK):
        raise PermissionError(f"Sem permissão de leitura: {caminho}")

    ext = p.suffix.lower()

    if ext == ".xlsx":
        log.info(f"  Lendo XLSX: {p.name}")
        df = pd.read_excel(caminho, dtype=dtype, engine="openpyxl")
    elif ext in (".csv", ".txt"):
        for enc in ("utf-8", "latin-1", "cp1252"):
            try:
                df = pd.read_csv(
                    caminho,
                    sep=";",
                    dtype=dtype,
                    encoding=enc,
                    low_memory=False,
                )
                log.info(f"  Lendo CSV ({enc}): {p.name}")
                break
            except UnicodeDecodeError:
                continue
        else:
            raise ValueError(f"Não foi possível decodificar o arquivo: {caminho}")
    else:
        raise ValueError(f"Extensão não suportada: {ext}  (use .csv ou .xlsx)")

    return df


def _ler_arquivo_em_chunks(caminho: str, chunksize: int, dtype: Optional[dict] = None):
    """
    Gerador que lê um arquivo .csv (';') ou .xlsx em chunks de `chunksize` linhas.
    Para .xlsx carrega tudo na memória e fatia manualmente (limitação do pandas).

    Yields
    ------
    pd.DataFrame : chunk do arquivo.
    """
    p = Path(caminho)
    ext = p.suffix.lower()

    if ext == ".xlsx":
        # xlsx não suporta chunk nativo — carrega completo e itera manualmente
        log.warning(
            "  XLSX não suporta leitura em chunks nativos. "
            "Carregando completo na memória e fatiando..."
        )
        df_full = pd.read_excel(caminho, dtype=dtype, engine="openpyxl")
        total = len(df_full)
        for start in range(0, total, chunksize):
            yield df_full.iloc[start : start + chunksize].copy()
        del df_full
        gc.collect()

    elif ext in (".csv", ".txt"):
        for enc in ("utf-8", "latin-1", "cp1252"):
            try:
                reader = pd.read_csv(
                    caminho,
                    sep=";",
                    dtype=dtype,
                    encoding=enc,
                    chunksize=chunksize,
                    low_memory=False,
                )
                log.info(f"  Lendo CSV em chunks ({enc}): {p.name}")
                yield from reader
                return
            except UnicodeDecodeError:
                continue
        raise ValueError(f"Não foi possível decodificar: {caminho}")
    else:
        raise ValueError(f"Extensão não suportada: {ext}")


# ==============================================================================
# 1. UNIFICAR BASES DE PREFIXOS
# ==============================================================================

def unificar_bases_prefixos(
    caminhos: list[str],
    rotulos: list[str],
) -> pd.DataFrame:
    """
    Lê os 4 arquivos CSV de prefixos ANATEL, normaliza colunas, filtra status==1
    e concatena em uma única tabela.

    Retorna DataFrame com colunas:
        nome_prestadora | cnpj_prestadora | codigo_nacional | prefixo |
        faixa_inicial | faixa_final | status | fonte_prefixo

    Também adiciona coluna `ordem_fonte` (int) para desempate por prioridade.
    """
    log.info("=" * 70)
    log.info("ETAPA 1 — Carregando e unificando bases de prefixos...")
    t0 = time.time()

    # Mapeamento esperado após normalização → nome canônico
    MAP_COLUNAS = {
        "nome_da_prestadora":   "nome_prestadora",
        "nomeda_prestadora":    "nome_prestadora",
        "nome_prestadora":      "nome_prestadora",
        "cnpj_da_prestadora":   "cnpj_prestadora",
        "cnpjda_prestadora":    "cnpj_prestadora",
        "cnpj_prestadora":      "cnpj_prestadora",
        "codigo_nacional":      "codigo_nacional",
        "prefixo":              "prefixo",
        "faixa_inicial":        "faixa_inicial",
        "faixa_final":          "faixa_final",
        "status":               "status",
    }
    COLUNAS_OBRIGATORIAS = [
        "nome_prestadora", "codigo_nacional", "prefixo",
        "faixa_inicial", "faixa_final", "status",
    ]

    dfs = []
    for idx, (caminho, rotulo) in enumerate(zip(caminhos, rotulos)):
        log.info(f"  [{idx+1}/{len(caminhos)}] {rotulo}: {caminho}")
        df = _ler_arquivo(caminho, dtype=str)  # tudo como str para preservar zeros

        # ── Normalizar nomes de colunas ──────────────────────────────────────
        df.columns = [_normalizar_nome_coluna(c) for c in df.columns]
        df.rename(columns=MAP_COLUNAS, inplace=True)

        # ── Validar colunas obrigatórias ─────────────────────────────────────
        faltando = [c for c in COLUNAS_OBRIGATORIAS if c not in df.columns]
        if faltando:
            raise ValueError(
                f"Arquivo de prefixo '{caminho}' está faltando colunas: {faltando}\n"
                f"Colunas encontradas: {list(df.columns)}"
            )

        # ── Selecionar apenas colunas necessárias ────────────────────────────
        df = df[COLUNAS_OBRIGATORIAS].copy()

        # ── Converter tipos ──────────────────────────────────────────────────
        df["status"] = pd.to_numeric(df["status"], errors="coerce").fillna(0).astype(int)
        df = df[df["status"] == 1].copy()  # somente registros ativos

        df["faixa_inicial"] = pd.to_numeric(df["faixa_inicial"], errors="coerce").astype("Int32")
        df["faixa_final"]   = pd.to_numeric(df["faixa_final"],   errors="coerce").astype("Int32")

        # ── Remover linhas com NaN em campos críticos ────────────────────────
        df.dropna(subset=["codigo_nacional", "prefixo", "faixa_inicial", "faixa_final"], inplace=True)

        # ── Garantir zeros à esquerda: codigo_nacional (2 dígitos), prefixo (4) ──
        df["codigo_nacional"] = df["codigo_nacional"].str.strip().str.zfill(2)
        df["prefixo"]         = df["prefixo"].str.strip().str.zfill(4)

        # ── Metadados de origem ──────────────────────────────────────────────
        df["fonte_prefixo"] = rotulo
        df["ordem_fonte"]   = idx   # menor = maior prioridade

        dfs.append(df)
        log.info(f"     → {len(df):,} registros ativos carregados.")

    tabela = pd.concat(dfs, ignore_index=True)
    del dfs
    gc.collect()

    # ── Otimizar memória usando categorias ────────────────────────────────────
    for col in ["nome_prestadora", "fonte_prefixo", "codigo_nacional", "prefixo"]:
        tabela[col] = tabela[col].astype("category")

    # ── Criar amplitude de faixa para critério de especificidade ─────────────
    tabela["amplitude_faixa"] = (
        tabela["faixa_final"].astype(int) - tabela["faixa_inicial"].astype(int)
    )

    log.info(f"  Total unificado: {len(tabela):,} prefixos ativos.")
    log.info(f"  Tempo ETAPA 1: {time.time() - t0:.1f}s")
    return tabela


# ==============================================================================
# 2. INDEXAR TABELA DE PREFIXOS  (pré-processamento para lookup rápido)
# ==============================================================================

def indexar_prefixos(tabela: pd.DataFrame) -> dict:
    """
    Constrói um dicionário de lookup no formato:
        {(codigo_nacional_str, prefixo_str): DataFrame_filtrado}

    Cada valor é um DataFrame já ordenado por critérios de desempate:
        1º amplitude_faixa (menor = mais específico)
        2º ordem_fonte (menor = maior prioridade)

    Isso evita re-filtragem a cada linha processada, reduzindo O(n) de buscas.
    """
    log.info("  Indexando tabela de prefixos em memória (lookup dict)...")
    t0 = time.time()

    # Converte colunas categóricas para str para usar como chave de dict
    tabela = tabela.copy()
    tabela["codigo_nacional"] = tabela["codigo_nacional"].astype(str)
    tabela["prefixo"]         = tabela["prefixo"].astype(str)

    # Ordena globalmente para que .head(1) após filtro de faixa já seja o melhor
    tabela.sort_values(
        ["codigo_nacional", "prefixo", "amplitude_faixa", "ordem_fonte"],
        inplace=True,
    )

    idx: dict = {}
    for (ddd, pfx), grupo in tabela.groupby(
        ["codigo_nacional", "prefixo"], sort=False, observed=True
    ):
        idx[(ddd, pfx)] = grupo.reset_index(drop=True)

    log.info(f"  Índice criado: {len(idx):,} chaves (DDD, prefixo).")
    log.info(f"  Tempo indexação: {time.time() - t0:.1f}s")
    return idx


# ==============================================================================
# 3. NORMALIZAR NÚMERO
# ==============================================================================

def normalizar_numero(numero: str) -> Optional[str]:
    """
    Normaliza um número de telefone brasileiro:
      1. Remove tudo que não for dígito.
      2. Remove DDI '55' inicial se presente.
      3. Retorna None se o resultado tiver menos de 10 dígitos.

    Parâmetros
    ----------
    numero : str
        Número bruto da bilhetagem.

    Retorna
    -------
    str com apenas dígitos (mínimo 10) ou None se inválido.
    """
    if not isinstance(numero, str) or not numero.strip():
        return None

    digitos = re.sub(r"\D", "", numero)   # mantém apenas 0-9

    # Remove DDI 55 se houver (ex.: 5511987654321 → 11987654321)
    if digitos.startswith("55") and len(digitos) > 12:
        digitos = digitos[2:]

    if len(digitos) < 10:
        return None

    return digitos


# ==============================================================================
# 4. BUSCAR OPERADORA PELO PREFIXO
# ==============================================================================

def buscar_operadora_prefixo(
    numero_norm: str,
    idx_prefixos: dict,
) -> dict:
    """
    Dado um número normalizado (somente dígitos, sem DDI), identifica:
        - DDD
        - prefixo casado
        - faixa utilizada
        - fonte_prefixo
        - nome_prestadora

    Estratégia:
        DDD = 2 primeiros dígitos
        numero_local = restante
        estacao = últimos 4 dígitos de numero_local (int)
        Candidato A: prefixo = numero_local[0:4]   (números fixos ou sem '9' inicial)
        Candidato B: prefixo = numero_local[1:5]   (móvel com '9' inicial)
        Para cada candidato, verifica se (DDD, prefixo) está no índice
        e se faixa_inicial <= estacao <= faixa_final.
        Prioridade: menor amplitude_faixa → menor ordem_fonte.

    Retorna dict com os campos identificados, ou com status 'PREFIXO NÃO ENCONTRADO'.
    """
    VAZIO = {
        "ddd":                  "",
        "prefixo_casado":       "",
        "faixa_utilizada":      "",
        "fonte_prefixo":        "",
        "operadora_pelo_prefixo": "PREFIXO NÃO ENCONTRADO",
    }

    if not numero_norm or len(numero_norm) < 10:
        return VAZIO

    ddd          = numero_norm[:2].zfill(2)
    numero_local = numero_norm[2:]
    estacao      = None

    # Estação = últimos 4 dígitos do número local
    try:
        estacao = int(numero_local[-4:])
    except (ValueError, IndexError):
        return VAZIO

    # Gera candidatos de prefixo
    candidatos = []
    if len(numero_local) >= 4:
        candidatos.append(numero_local[:4].zfill(4))           # Candidato A
    if numero_local.startswith("9") and len(numero_local) >= 5:
        candidatos.append(numero_local[1:5].zfill(4))          # Candidato B (móvel 9xxxxxxxx)

    melhor = None
    for pfx_cand in candidatos:
        chave = (ddd, pfx_cand)
        if chave not in idx_prefixos:
            continue

        grupo = idx_prefixos[chave]

        # Filtra pela estação dentro da faixa
        mascara = (
            (grupo["faixa_inicial"].astype(int) <= estacao) &
            (grupo["faixa_final"].astype(int)   >= estacao)
        )
        candidatos_validos = grupo[mascara]

        if candidatos_validos.empty:
            continue

        # Já ordenado por amplitude_faixa, ordem_fonte → pega o 1º
        linha = candidatos_validos.iloc[0]

        if melhor is None:
            melhor = (pfx_cand, linha)
        else:
            # Desempate explícito entre candidatos A e B
            _, melhor_linha = melhor
            if (linha["amplitude_faixa"], linha["ordem_fonte"]) < (
                melhor_linha["amplitude_faixa"], melhor_linha["ordem_fonte"]
            ):
                melhor = (pfx_cand, linha)

    if melhor is None:
        return VAZIO

    pfx_final, linha_final = melhor
    return {
        "ddd":          ddd,
        "prefixo_casado": pfx_final,
        "faixa_utilizada": f"{int(linha_final['faixa_inicial'])}-{int(linha_final['faixa_final'])}",
        "fonte_prefixo": str(linha_final["fonte_prefixo"]),
        "operadora_pelo_prefixo": str(linha_final["nome_prestadora"]),
    }


# ==============================================================================
# 5. CARREGAR BASE DE PORTABILIDADE EM ÍNDICE
# ==============================================================================

def carregar_portabilidade(caminho: str) -> dict:
    """
    Carrega a base de portabilidade em um dicionário de lookup:
        { numero_normalizado_str : operadora_atual_str }

    Lida com arquivos muito grandes lendo em chunks e populando o dict
    de forma incremental para não estourar a memória de uma vez.
    """
    log.info("=" * 70)
    log.info("ETAPA 2 — Carregando base de portabilidade...")
    t0 = time.time()

    portab: dict = {}
    total_linhas = 0
    erros = 0

    for chunk in _ler_arquivo_em_chunks(
        caminho,
        chunksize=CHUNKSIZE_PORTAB,
        dtype=str,
    ):
        # Normalizar nomes de colunas
        chunk.columns = [_normalizar_nome_coluna(c) for c in chunk.columns]

        # Validar colunas
        col_num = _normalizar_nome_coluna(COLUNA_NUMERO_PORTABILIDADE)
        col_op  = _normalizar_nome_coluna(COLUNA_OPERADORA_PORTABILIDADE)

        if col_num not in chunk.columns or col_op not in chunk.columns:
            raise ValueError(
                f"Portabilidade: colunas '{col_num}' e/ou '{col_op}' não encontradas.\n"
                f"Colunas disponíveis: {list(chunk.columns)}"
            )

        # Normaliza e indexa
        nums  = chunk[col_num].fillna("").astype(str)
        ops   = chunk[col_op].fillna("").astype(str)

        for num_bruto, op in zip(nums, ops):
            num_norm = normalizar_numero(num_bruto)
            if num_norm:
                portab[num_norm] = op
                total_linhas += 1
            else:
                erros += 1

        del chunk
        gc.collect()

    log.info(f"  Portabilidade indexada: {total_linhas:,} números.")
    if erros > 0:
        log.warning(f"  {erros:,} números ignorados (inválidos) na portabilidade.")
    log.info(f"  Tempo ETAPA 2: {time.time() - t0:.1f}s")
    return portab


# ==============================================================================
# 6. VERIFICAR PORTABILIDADE
# ==============================================================================

def verificar_portabilidade(
    numero_norm: str,
    operadora_prefixo: str,
    idx_portabilidade: dict,
) -> tuple[str, str]:
    """
    Verifica se um número foi portado, comparando a operadora do prefixo
    com a operadora atual na base de portabilidade.

    Retorna
    -------
    (operadora_atual: str, status_portabilidade: str)
        status pode ser: 'PORTADO', 'NÃO PORTADO', 'NÃO ENCONTRADO NA PORTABILIDADE'
    """
    if numero_norm not in idx_portabilidade:
        return ("", "NÃO ENCONTRADO NA PORTABILIDADE")

    op_atual = idx_portabilidade[numero_norm]

    if not op_atual:
        return ("", "NÃO ENCONTRADO NA PORTABILIDADE")

    # Comparação case-insensitive e sem espaços extras
    if op_atual.strip().lower() != operadora_prefixo.strip().lower():
        return (op_atual, "PORTADO")
    else:
        return (op_atual, "NÃO PORTADO")


# ==============================================================================
# 7. EXTRAIR NÚMEROS (A_BUSCA e B_BUSCA) EM CHUNKS
# ==============================================================================

def extrair_numeros(caminho: str) -> pd.DataFrame:
    """
    Gerador: lê a bilhetagem em chunks e yields DataFrames com:
        numero_bruto | origem_numero

    onde origem_numero é 'A_BUSCA' ou 'B_BUSCA'.
    Os dois blocos de cada chunk são empilhados (pd.concat vertical).
    """
    log.info("=" * 70)
    log.info("ETAPA 3 — Lendo bilhetagem e extraindo A_BUSCA / B_BUSCA...")

    for chunk in _ler_arquivo_em_chunks(
        caminho,
        chunksize=CHUNKSIZE_BILHETAGEM,
        dtype=str,
    ):
        # Normaliza nomes de colunas do chunk
        chunk.columns = [_normalizar_nome_coluna(c) for c in chunk.columns]

        col_a = _normalizar_nome_coluna(COLUNA_A_BUSCA)
        col_b = _normalizar_nome_coluna(COLUNA_B_BUSCA)

        for col, origem in [(col_a, "A_BUSCA"), (col_b, "B_BUSCA")]:
            if col not in chunk.columns:
                log.warning(f"  Coluna '{col}' não encontrada no chunk. Ignorando.")
                continue

            parcial = pd.DataFrame({
                "numero_bruto":  chunk[col].fillna("").astype(str),
                "origem_numero": origem,
            })
            # Remove linhas completamente vazias
            parcial = parcial[parcial["numero_bruto"].str.strip() != ""]
            yield parcial

        del chunk
        gc.collect()


# ==============================================================================
# 8. PROCESSAR CHUNK COMPLETO (PREFIXO + PORTABILIDADE)
# ==============================================================================

def processar_chunk(
    df_nums: pd.DataFrame,
    idx_prefixos: dict,
    idx_portabilidade: dict,
) -> pd.DataFrame:
    """
    Dado um DataFrame com colunas [numero_bruto, origem_numero], aplica:
      1. Normalização do número.
      2. Busca de operadora por prefixo.
      3. Verificação de portabilidade.

    Retorna DataFrame com todas as colunas de saída.
    """
    # ── 1. Normalizar números ─────────────────────────────────────────────────
    df_nums = df_nums.copy()
    df_nums["numero_normalizado"] = df_nums["numero_bruto"].apply(normalizar_numero)

    # ── 2. Buscar operadora por prefixo (vetorizado via apply) ────────────────
    resultados_prefixo = df_nums["numero_normalizado"].apply(
        lambda n: buscar_operadora_prefixo(n, idx_prefixos) if n else {
            "ddd": "", "prefixo_casado": "", "faixa_utilizada": "",
            "fonte_prefixo": "", "operadora_pelo_prefixo": "PREFIXO NÃO ENCONTRADO"
        }
    )

    # Expande o dict retornado em colunas separadas
    df_pfx = pd.DataFrame(resultados_prefixo.tolist(), index=df_nums.index)

    # ── 3. Verificar portabilidade ────────────────────────────────────────────
    portab_results = df_nums[["numero_normalizado"]].join(
        df_pfx[["operadora_pelo_prefixo"]]
    ).apply(
        lambda row: verificar_portabilidade(
            row["numero_normalizado"] or "",
            row["operadora_pelo_prefixo"],
            idx_portabilidade,
        ),
        axis=1,
        result_type="expand",
    )
    portab_results.columns = ["operadora_atual", "status_portabilidade"]

    # ── 4. Montar resultado final do chunk ────────────────────────────────────
    resultado = pd.concat(
        [
            df_nums[["numero_normalizado", "origem_numero"]].reset_index(drop=True),
            df_pfx.reset_index(drop=True),
            portab_results.reset_index(drop=True),
        ],
        axis=1,
    )

    # Reordenar colunas conforme especificação
    COLUNAS_SAIDA = [
        "numero_normalizado",
        "ddd",
        "prefixo_casado",
        "faixa_utilizada",
        "fonte_prefixo",
        "operadora_pelo_prefixo",
        "operadora_atual",
        "status_portabilidade",
        "origem_numero",
    ]
    # Garante que todas as colunas existam
    for col in COLUNAS_SAIDA:
        if col not in resultado.columns:
            resultado[col] = ""

    return resultado[COLUNAS_SAIDA]


# ==============================================================================
# 9. GERAR RESULTADO (CSV TEMP → XLSX)
# ==============================================================================

def gerar_resultado(caminho_csv_temp: str, caminho_xlsx: str) -> None:
    """
    Converte o CSV temporário acumulado em RESULTADO.xlsx usando openpyxl.
    Lida com o arquivo em chunks para não estourar memória.
    """
    log.info("=" * 70)
    log.info("ETAPA 5 — Convertendo CSV temporário para XLSX final...")
    t0 = time.time()

    from openpyxl import Workbook
    from openpyxl.utils.dataframe import dataframe_to_rows
    from openpyxl.styles import Font, PatternFill, Alignment

    wb = Workbook(write_only=True)   # write_only = muito menor uso de RAM
    ws = wb.create_sheet("Resultado")

    cabecalho_escrito = False
    total_linhas = 0

    # Estilo do cabeçalho (será gravado na 1ª iteração)
    COR_CABECALHO = "1F4E79"  # azul escuro
    FONT_CABECALHO = Font(bold=True, color="FFFFFF", name="Calibri", size=11)
    FILL_CABECALHO = PatternFill("solid", fgColor=COR_CABECALHO)
    ALIGN_CENTER   = Alignment(horizontal="center", vertical="center")

    for chunk in pd.read_csv(
        caminho_csv_temp,
        sep=";",
        encoding="utf-8",
        chunksize=100_000,
        dtype=str,
    ):
        chunk.fillna("", inplace=True)

        if not cabecalho_escrito:
            # Grava cabeçalho com estilo
            from openpyxl.cell import WriteOnlyCell
            row_cab = []
            for col_name in chunk.columns:
                cell = WriteOnlyCell(ws, value=col_name)
                cell.font  = FONT_CABECALHO
                cell.fill  = FILL_CABECALHO
                cell.alignment = ALIGN_CENTER
                row_cab.append(cell)
            ws.append(row_cab)
            cabecalho_escrito = True

        for row in dataframe_to_rows(chunk, index=False, header=False):
            ws.append(row)
            total_linhas += 1

        del chunk
        gc.collect()

    wb.save(caminho_xlsx)
    log.info(f"  XLSX gerado: {caminho_xlsx}  ({total_linhas:,} linhas de dados).")
    log.info(f"  Tempo ETAPA 5: {time.time() - t0:.1f}s")


# ==============================================================================
# 10. MAIN — ORQUESTRA TUDO
# ==============================================================================

def main() -> None:
    """
    Função principal. Orquestra todas as etapas do pipeline:
      1. Unifica e indexa bases de prefixos.
      2. Carrega e indexa portabilidade.
      3. Processa bilhetagem em chunks (A_BUSCA + B_BUSCA).
      4. Grava resultados incrementalmente em CSV temp.
      5. Converte CSV temp → RESULTADO.xlsx.
    """
    log.info("=" * 70)
    log.info("  IDENTIFICADOR DE OPERADORA + PORTABILIDADE — INICIANDO")
    log.info("=" * 70)
    t_total = time.time()

    # ──────────────────────────────────────────────────────────────────────────
    # ETAPA 1 — Prefixos
    # ──────────────────────────────────────────────────────────────────────────
    tabela_prefixos = unificar_bases_prefixos(CAMINHOS_PREFIXOS, ROTULOS_PREFIXOS)
    idx_prefixos    = indexar_prefixos(tabela_prefixos)
    del tabela_prefixos
    gc.collect()

    # ──────────────────────────────────────────────────────────────────────────
    # ETAPA 2 — Portabilidade
    # ──────────────────────────────────────────────────────────────────────────
    idx_portabilidade = carregar_portabilidade(CAMINHO_ARQUIVO_3)

    # ──────────────────────────────────────────────────────────────────────────
    # ETAPA 3+4 — Processar bilhetagem em chunks e gravar CSV temp
    # ──────────────────────────────────────────────────────────────────────────
    log.info("=" * 70)
    log.info("ETAPA 4 — Processando bilhetagem e gravando resultado parcial...")
    t_etapa4 = time.time()

    # Garante que o CSV temp começa vazio
    if os.path.exists(CAMINHO_TEMP_CSV):
        os.remove(CAMINHO_TEMP_CSV)

    total_processado  = 0
    total_portados    = 0
    total_nao_portados = 0
    total_nao_encontrados = 0
    total_sem_prefixo = 0
    chunk_num         = 0
    excecoes_log      = []

    for df_nums in extrair_numeros(CAMINHO_ARQUIVO_1):
        chunk_num += 1
        t_chunk = time.time()

        try:
            df_resultado = processar_chunk(df_nums, idx_prefixos, idx_portabilidade)

            # Estatísticas rápidas
            total_processado    += len(df_resultado)
            total_portados      += (df_resultado["status_portabilidade"] == "PORTADO").sum()
            total_nao_portados  += (df_resultado["status_portabilidade"] == "NÃO PORTADO").sum()
            total_nao_encontrados += (
                df_resultado["status_portabilidade"] == "NÃO ENCONTRADO NA PORTABILIDADE"
            ).sum()
            total_sem_prefixo   += (
                df_resultado["operadora_pelo_prefixo"] == "PREFIXO NÃO ENCONTRADO"
            ).sum()

            # Registrar casos ambíguos (mesmo número encontrado em mais de 1 prefixo ativo)
            # (Neste design já são resolvidos pelo sort; logamos aqui para auditoria)
            dupes = df_resultado[
                df_resultado["numero_normalizado"].duplicated(keep=False) &
                (df_resultado["numero_normalizado"] != "")
            ]
            if not dupes.empty:
                excecoes_log.append(
                    f"Chunk {chunk_num}: {len(dupes):,} linhas com número duplicado "
                    f"(múltiplas origens A/B_BUSCA)."
                )

            # Gravar CSV temporário (append; sem índice; cabeçalho somente no 1º chunk)
            modo_header = (chunk_num == 1)
            df_resultado.to_csv(
                CAMINHO_TEMP_CSV,
                sep=";",
                index=False,
                header=modo_header,
                mode="w" if modo_header else "a",
                encoding="utf-8",
            )

            log.info(
                f"  Chunk {chunk_num:04d} processado: "
                f"{len(df_resultado):,} linhas | "
                f"{time.time() - t_chunk:.2f}s"
            )

        except Exception as exc:
            log.error(f"  ERRO no chunk {chunk_num}: {exc}", exc_info=True)
            excecoes_log.append(f"Chunk {chunk_num}: ERRO — {exc}")

        del df_nums
        gc.collect()

    log.info(f"  Tempo ETAPA 4: {time.time() - t_etapa4:.1f}s")

    # ──────────────────────────────────────────────────────────────────────────
    # ETAPA 5 — Converter CSV temp → XLSX
    # ──────────────────────────────────────────────────────────────────────────
    if total_processado == 0:
        log.warning("  Nenhum número processado. Verifique o arquivo de bilhetagem.")
    else:
        gerar_resultado(CAMINHO_TEMP_CSV, CAMINHO_SAIDA_XLSX)

        # Remove CSV temporário
        try:
            os.remove(CAMINHO_TEMP_CSV)
            log.info(f"  CSV temporário removido: {CAMINHO_TEMP_CSV}")
        except OSError as e:
            log.warning(f"  Não foi possível remover CSV temp: {e}")

    # ──────────────────────────────────────────────────────────────────────────
    # RESUMO FINAL
    # ──────────────────────────────────────────────────────────────────────────
    log.info("=" * 70)
    log.info("  RESUMO FINAL")
    log.info(f"  Total de números processados : {total_processado:>12,}")
    log.info(f"  Portados                     : {total_portados:>12,}")
    log.info(f"  Não portados                 : {total_nao_portados:>12,}")
    log.info(f"  Não encontrados portabilidade: {total_nao_encontrados:>12,}")
    log.info(f"  Sem prefixo identificado     : {total_sem_prefixo:>12,}")
    if excecoes_log:
        log.warning(f"  Casos ambíguos/erros ({len(excecoes_log)}):")
        for msg in excecoes_log[:20]:   # limita exibição no terminal
            log.warning(f"    {msg}")
        if len(excecoes_log) > 20:
            log.warning(f"    ... e mais {len(excecoes_log) - 20} ocorrências (ver log).")
    log.info(f"  Tempo total                  : {time.time() - t_total:.1f}s")
    log.info(f"  Arquivo gerado               : {CAMINHO_SAIDA_XLSX}")
    log.info("=" * 70)


# ==============================================================================
# ENTRY POINT
# ==============================================================================
if __name__ == "__main__":
    main()

