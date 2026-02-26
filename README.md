Você é um engenheiro de software sênior especializado em:

Processamento de planilhas CSV UTF-8 e XLSX
ETL, engenharia de dados e automações
Limpeza, validação e padronização de bases
Junção de planilhas com diferentes estruturas
Otimização e reescrita de código legado
Arquitetura limpa, modular e de alta performance

Eu vou enviar em seguida um programa completo que trabalha com duas planilhas: Portal Science e Portal de Cadastros. O código atual lê apenas arquivos CSV UTF-8, e quero que ele também aceite XLSX, detectando automaticamente o formato pelo nome do arquivo.
Estrutura da Planilha 1 – Portal Science (com problemas de encoding que precisam ser corrigidos):
COD_CCC Origem
ID Rota
Data Ativação
Data Desativação
Sentido
Tipo
OP destino
Área Ponta A - Engenharia
Área Ponta A - Mediação
Área Ponta B
Serviço
Descrição
Operadora Origem
OP Origem
Central Origem
Operadora destino
Sequencial
Sigla Tipo Direção
Central Interna
COD_CCC Interna
OPC
DPC
Num SSI
CNL
Atendimento Móvel/Fixo
Gateway
Sinalização da Rota
Tipo da Rota
Tipo de Trafego
Estrutura da Planilha 2 – Portal de Cadastros:
row_number
SOLICITACAO
TIPO
CENTRAL
BILHETADOR
TIPO_ROTA
TIPO_REGISTRO
ROTA_E
ROTA_S
LABEL_E
LABEL_S
EMPRESA
EOT
OC
INTERCON
ECR
OPC
DPC
DESIGNACAO
CATEGORIA
CATEG_DESCRICAO
SENTIDO
SINALIZACAO
ROTA_EXCLUSIVA
CNL_PPI
PPI
DT_ATIVACAO
TRAF_CURSADO
CSP
DT_CADASTRO
RESP_CADASTRO
DT_EXECUCAO
RESP_EXECUCAO
Obs
O programa que enviarei em breve:

Carrega essas duas planilhas
Seleciona colunas específicas de cada uma
Faz junções, cruzamentos e transformações
Consulta outras bases internas
Gera uma nova planilha consolidada
Permite exportar para Excel

O que você deve fazer quando receber o programa:

Ler o código completamente e entender cada detalhe.
Identificar erros, inconsistências, redundâncias e riscos.
Validar se as colunas usadas realmente existem nas planilhas.
Corrigir problemas de encoding dos cabeçalhos (ex.: “AtivaÃ§Ã£o” → “Ativação”).
Reescrever o programa deixando-o limpo, modular e profissional.
Atualizar a parte de leitura de arquivos para aceitar automaticamente CSV e XLSX.
Criar funções separadas para:

leitura de arquivos
validação de colunas
normalização e limpeza
junções e regras de negócio
exportação final


Garantir robustez com tratamento de exceções, logs e validações.
Melhorar performance sempre que possível.
Explicar detalhadamente todas as melhorias que aplicar.

Instruções importantes:

Não execute nada.
Apenas analise, explique e reescreva o código da melhor forma possível.
Aguarde até eu enviar o programa.
Quando eu enviar, você deve automaticamente iniciar a análise completa e a reescrita otimizada.

# app/routes/vivo_crc.py
from __future__ import annotations

from flask import (
    request, jsonify, render_template, send_file,
    Blueprint, current_app, session, Response
)
from werkzeug.utils import secure_filename
import pandas as pd
import os
from uuid import uuid4
from openpyxl import Workbook
from openpyxl.utils import get_column_letter
import traceback
from io import BytesIO
import re
import chardet
from typing import Iterable, List, Dict, Any, Optional, Tuple

from app.sqlite_db import get_db

bp = Blueprint('vivo_crc', __name__, url_prefix="/vivo_crc")

# =========================================================
# Config / Constantes
# =========================================================
DEFAULT_UPLOAD_FOLDER = 'uploads'
ALLOWED_EXTENSIONS = {"csv"}

EXCEL_MAX_ROWS = 1_048_576            # limite do Excel por aba
MAX_XLSX_SHEETS = 10                  # até 10 abas antes de cair para CSV
BULK_INSERT_SIZE = 5_000
CNL_IN_CHUNK = 900                    # evitar >999 params no SQLite

# Esquema padronizado — alinhar com schema.sql (sem acentos)
UNIFIED_COLS = [
    "ID", "Portal", "Tipo_de_Trafego", "EOT", "Operadora", "CN",
    "ID_Rota", "LABEL_E", "LABEL_S", "Origem_Rota", "Tipo_de_Rota",
    "Sinalizacao", "Trafego", "Descricao", "Central", "Bilhetador",
    "OPC", "DPC", "Data_Desativacao"
]

ORIGENS_PERMITIDAS = {"science", "portal"}

# =========================================================
# Helpers básicos
# =========================================================
def _upload_folder() -> str:
    return current_app.config.get('UPLOAD_FOLDER', DEFAULT_UPLOAD_FOLDER)

def allowed_file(filename: str) -> bool:
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def sample_detect_encoding(filepath: str) -> str:
    """Detecta encoding lendo apenas ~64KB do começo do arquivo."""
    with open(filepath, 'rb') as f:
        amostra = f.read(65536)
    return chardet.detect(amostra).get('encoding') or 'utf-8'

def padronizar_coluna(col: str) -> str:
    """Remove controles, normaliza separadores e diacríticos, e colapsa underscores."""
    col = re.sub(r'[\x00-\x1F\x7F]', '', str(col)).strip()
    col = col.replace(' ', '_').replace('-', '_').replace('/', '_')
    try:
        import unicodedata
        col = ''.join(c for c in unicodedata.normalize('NFKD', col) if not unicodedata.combining(c))
    except Exception:
        pass
    col = re.sub(r'__+', '_', col)
    return col

def sanitizar_valor(valor: Any) -> Any:
    """Remove controles e protege contra formula injection no Excel."""
    if isinstance(valor, str):
        valor = re.sub(r'[\x00-\x1F\x7F]', '', valor)
        if valor[:1] in ('=', '+', '-', '@'):
            return "'" + valor
    return valor

def iso_date_or_none(s: Any) -> Optional[str]:
    """Converte para 'YYYY-MM-DD' quando possível; senão None."""
    if s is None or (isinstance(s, float) and pd.isna(s)):
        return None
    try:
        dt = pd.to_datetime(s, errors='coerce', dayfirst=True)
        if pd.isna(dt):
            return None
        return dt.strftime('%Y-%m-%d')
    except Exception:
        return None

def ensure_csrf_token() -> str:
    token = session.get('csrf_token')
    if not token:
        token = uuid4().hex
        session['csrf_token'] = token
    return token

def validate_csrf_header():
    token_header = request.headers.get('X-CSRF-Token')
    if request.method in ('POST', 'PUT', 'PATCH', 'DELETE'):
        if not token_header or token_header != session.get('csrf_token'):
            return jsonify({'error': 'CSRF token inválido ou ausente.'}), 403
    return None

# =========================================================
# Normalização por origem
# =========================================================
def _map_columns_science(df: pd.DataFrame) -> pd.DataFrame:
    """
    Science:
      - CN = 'Área Ponta B' (ou variações/mojibake) — NÃO consultar tabela cnl;
      - Tipo_de_Trafego = 'SMP';
      - Data_Desativacao existe nessa base (quando houver).
    """
    mapeamento = {
        'Sequencial': 'ID',
        'OP destino': 'EOT',
        'Operadora destino': 'Operadora',

        # “Área Ponta B” e mojibake comuns:
        'Área Ponta B': 'CN',
        'Ãrea Ponta B': 'CN',
        'AREA PONTA B': 'CN',

        'ID Rota': 'ID_Rota',
        'Tipo': 'Origem_Rota',
        'Tipo da Rota': 'Tipo_de_Rota',

        'Sinalização da Rota': 'Sinalizacao',
        'SinalizaÃ§Ã£o da Rota': 'Sinalizacao',

        'Tipo de Trafego': 'Trafego',
        'Descrição': 'Descricao',
        'DescriÃ§Ã£o': 'Descricao',
        'Central Origem': 'Central',

        'OPC': 'OPC',
        'DPC': 'DPC',

        'Data Desativação': 'Data_Desativacao',
        'Data DesativaÃ§Ã£o': 'Data_Desativacao',
    }
    ren = {padronizar_coluna(k): v for k, v in mapeamento.items()}
    df.columns = [padronizar_coluna(c) for c in df.columns]
    df = df.rename(columns={k: v for k, v in ren.items() if k in df.columns})

    df['Portal'] = 'Science'
    df['Tipo_de_Trafego'] = df.get('Tipo_de_Trafego', 'SMP')

    # CN já veio mapeado por 'Área Ponta B'. Se não existir, tenta colunas padronizadas
    if 'CN' not in df.columns:
        for cand in ['AREA_PONTA_B', 'AREA_PONTA_B_', 'AREA_PONTA_B__', 'AREA_PONTA_B___']:
            if cand in df.columns:
                df['CN'] = df[cand]
                break
    return df

def _map_columns_portal(df: pd.DataFrame) -> pd.DataFrame:
    """
    Portal (CSO):
      - CN será resolvido via tabela 'cnl' a partir de possíveis colunas CNL;
      - Tipo_de_Trafego = 'STFC'.
    """
    mapeamento = {
        'SOLICITACAO': 'ID',
        'EOT': 'EOT',
        'EMPRESA': 'Operadora',
        'CNL_PPI': 'CNL_PPI',          # manter original para mapear CN abaixo
        'AREA_PONTA_B': 'AREA_PONTA_B',# pode existir, mas CN do Portal vem de CNL via tabela aux
        'CNL': 'CNL',
        'COD_CNL': 'COD_CNL',

        'TIPO': 'Origem_Rota',
        'CATEG_DESCRICAO': 'Tipo_de_Rota',
        'SINALIZACAO': 'Sinalizacao',
        'TRAFEGO CURSADO': 'Trafego',
        'DESIGNACAO': 'Descricao',
        'CENTRAL': 'Central',
        'BILHETADOR': 'Bilhetador',
        'LABEL_E': 'LABEL_E',
        'LABEL_S': 'LABEL_S',
        'OPC': 'OPC',
        'DPC': 'DPC',
    }
    ren = {padronizar_coluna(k): v for k, v in mapeamento.items()}
    df.columns = [padronizar_coluna(c) for c in df.columns]
    df = df.rename(columns={k: v for k, v in ren.items() if k in df.columns})

    df['Portal'] = 'CSO'
    df['Tipo_de_Trafego'] = df.get('Tipo_de_Trafego', 'STFC')
    df['ID_Rota'] = df.get('ID_Rota', 'N/A')
    df['OPC'] = df.get('OPC', '-----')
    df['DPC'] = df.get('DPC', '-----')
    return df

def _apply_cnl_mapping_portal(df: pd.DataFrame) -> pd.DataFrame:
    """
    Apenas para Portal:
      - tenta encontrar coluna com códigos CNL (CNL_PPI, AREA_PONTA_B, CNL, COD_CNL)
      - mapeia para CN através da tabela auxiliar 'cnl' (COD_CNL -> CN)
    """
    if df.empty:
        return df

    possiveis = ['CNL_PPI', 'AREA_PONTA_B', 'CNL', 'COD_CNL']
    cand = None
    for c in df.columns:
        if c.upper() in possiveis:
            cand = c
            break
    if not cand:
        if 'CN' not in df.columns:
            df['CN'] = df.get('CN', 'N/A')
        return df

    valores = df[cand].dropna().astype(str).unique().tolist()
    if not valores:
        if 'CN' not in df.columns:
            df['CN'] = df.get('CN', 'N/A')
        return df

    conn = get_db()
    cursor = conn.cursor()

    def chunks(lst: List[str], n: int) -> Iterable[List[str]]:
        for i in range(0, len(lst), n):
            yield lst[i:i+n]

    mapa: Dict[str, str] = {}
    for pedaco in chunks(valores, CNL_IN_CHUNK):
        q = f"SELECT COD_CNL, CN FROM cnl WHERE COD_CNL IN ({','.join(['?']*len(pedaco))})"
        cursor.execute(q, pedaco)
        for cod, cn in cursor.fetchall():
            mapa[str(cod)] = cn
    cursor.close()

    df['CN'] = df[cand].astype(str).map(mapa).fillna(df.get('CN', 'N/A'))
    df['CN'] = df['CN'].astype(str).str.replace('.0', '', regex=False)
    return df

def _coerce_dates(df: pd.DataFrame) -> pd.DataFrame:
    if 'Data_Desativacao' in df.columns:
        df['Data_Desativacao'] = df['Data_Desativacao'].apply(iso_date_or_none)
    return df

# =========================================================
# Agente de IA (NLQ simplificado => SQL seguro)
# =========================================================
# O agente permite perguntas livres sobre os dados carregados, respeitando os mesmos filtros da UI.
# Ele NÃO executa SQL livre; gera consultas parametrizadas com base em heurísticas.

_GROUPABLE = {'Operadora', 'EOT', 'CN', 'Tipo_de_Trafego'}
_VALID_FILTERS = set(UNIFIED_COLS)

def _build_where_from_payload(cur, payload: Dict[str, Any]) -> Tuple[str, List[Any]]:
    """Reutiliza a semântica de filtros de /api/dados-v2 para perguntas do agente, favorecendo índice de data (ISO)."""
    cur.execute("PRAGMA table_info(tabela_unificada)")
    cols_validas = {row[1] for row in cur.fetchall()}

    termo_geral = (payload.get('termo_geral') or '').strip()
    filtros = payload.get('filtros_coluna', {}) or {}
    trafego = payload.get('trafego')
    data_inicio = (payload.get('data_inicio') or '').strip() or None
    data_fim = (payload.get('data_fim') or '').strip() or None
    aplicar_data_sci = bool(payload.get('aplicar_data_somente_science'))

    where_parts: List[str] = []
    params: List[Any] = []

    # 1) Filtros por coluna
    where_cols: List[str] = ["1=1"]
    params_cols: List[Any] = []

    for coluna, valor in filtros.items():
        base = coluna.replace('__inicio', '').replace('__fim', '')
        if base not in cols_validas:
            continue  # ignora silenciosamente no agente
        if coluna.endswith('__inicio'):
            # comparação direta (assumindo ISO) para possível uso de índice
            where_cols.append(f"{base} >= ?")
            params_cols.append(valor)
        elif coluna.endswith('__fim'):
            where_cols.append(f"{base} <= ?")
            params_cols.append(valor)
        else:
            where_cols.append(f"{base} LIKE ?")
            params_cols.append(f"%{valor}%")

    if trafego in ('STFC', 'SMP'):
        where_cols.append("Tipo_de_Trafego = ?")
        params_cols.append(trafego)

    if termo_geral:
        where_cols.append("(" + " OR ".join([
            "ID LIKE ?", "EOT LIKE ?", "Operadora LIKE ?", "Descricao LIKE ?"
        ]) + ")")
        params_cols.extend([f"%{termo_geral}%"] * 4)

    where_cols_sql = " AND ".join(where_cols)

    # 2) Intervalo global de datas (sem date() para favorecer índice se ISO)
    date_clause = ""
    date_params: List[Any] = []

    if data_inicio or data_fim:
        if aplicar_data_sci:
            subconds: List[str] = []
            subparams: List[Any] = []

            sc_where: List[str] = ["Portal = 'Science'"]
            if data_inicio:
                sc_where.append("Data_Desativacao >= ?")
                subparams.append(data_inicio)
            if data_fim:
                sc_where.append("Data_Desativacao <= ?")
                subparams.append(data_fim)
            subconds.append("(" + " AND ".join(sc_where) + ")")

            # Portal != Science (entra sempre)
            subconds.append("(Portal <> 'Science')")
            date_clause = "(" + " OR ".join(subconds) + ")"
            date_params.extend(subparams)
        else:
            if data_inicio:
                where_cols_sql += " AND Data_Desativacao >= ?"
                params_cols.append(data_inicio)
            if data_fim:
                where_cols_sql += " AND Data_Desativacao <= ?"
                params_cols.append(data_fim)

    if date_clause:
        where_sql = "(" + where_cols_sql + ") AND " + date_clause
        params = params_cols + date_params
    else:
        where_sql = where_cols_sql
        params = params_cols

    return where_sql, params

def _parse_question_pt(question: str) -> Dict[str, Any]:
    """
    Heurística simples para intents comuns em PT-BR:
    - 'quantos', 'total' => COUNT
    - 'distribuicao por X' / 'por operadora' => GROUP BY X
    - 'top N operadoras/eots/cns' => TOP-N por coluna
    - filtros inline: stfc/smp, 'operadora XYZ', 'eot 1234', 'cn 11', datas 'entre YYYY-MM-DD e YYYY-MM-DD'
    """
    q = (question or '').strip().lower()
    intent = 'count'
    group_by = None
    limit = None

    import re
    # intent: top N
    m = re.search(r'top\s+(\d+)\s+(operadoras?|eots?|cns?|tr[aá]fegos?)', q)
    if m:
        intent = 'top'
        limit = max(1, min(int(m.group(1)), 50))
        alvo = m.group(2)
        if 'operador' in alvo: group_by = 'Operadora'
        elif 'eot' in alvo: group_by = 'EOT'
        elif 'cn' in alvo: group_by = 'CN'
        else: group_by = 'Tipo_de_Trafego'
    # intent: distribuição por X
    m2 = re.search(r'(distribui[cç][aã]o|quebra|por)\s+por\s+(operadora|eot|cn|tr[aá]fego|tipo de tr[aá]fego)', q)
    if m2:
        intent = 'groupby'
        chave = m2.group(2)
        if 'operadora' in chave: group_by = 'Operadora'
        elif 'eot' in chave: group_by = 'EOT'
        elif 'cn' in chave: group_by = 'CN'
        else: group_by = 'Tipo_de_Trafego'
    # intent: "por operadora" isolado
    if intent == 'count' and re.search(r'\bpor\s+(operadora|eot|cn|tr[aá]fego)\b', q):
        intent = 'groupby'
        if 'operadora' in q: group_by = 'Operadora'
        elif 'eot' in q: group_by = 'EOT'
        elif 'cn' in q: group_by = 'CN'
        else: group_by = 'Tipo_de_Trafego'

    # filtros inline
    nl_filters: Dict[str, Any] = {}
    # tráfego
    if re.search(r'\bstfc\b', q): nl_filters['Tipo_de_Trafego'] = 'STFC'
    if re.search(r'\bsmp\b', q):  nl_filters['Tipo_de_Trafego'] = 'SMP'
    # operadora "XYZ"
    m3 = re.search(r'operadora\s+([a-z0-9\-\._ ]{2,})', q)
    if m3: nl_filters['Operadora'] = m3.group(1).strip().upper()
    # eot 1234
    m4 = re.search(r'\beot\s+([0-9\-]+)', q)
    if m4: nl_filters['EOT'] = m4.group(1).strip()
    # cn 11
    m5 = re.search(r'\bcn\s+([0-9\-]+)', q)
    if m5: nl_filters['CN'] = m5.group(1).strip()
    # datas: entre YYYY-MM-DD e YYYY-MM-DD
    m6 = re.search(r'entre\s+(\d{4}-\d{2}-\d{2})\s+e\s+(\d{4}-\d{2}-\d{2})', q)
    if m6:
        nl_filters['Data_Desativacao__inicio'] = m6.group(1)
        nl_filters['Data_Desativacao__fim'] = m6.group(2)

    return {
        'intent': intent,
        'group_by': group_by if group_by in _GROUPABLE else None,
        'limit': limit,
        'nl_filters': nl_filters
    }

def _merge_filters(base: Dict[str, Any], extra: Dict[str, Any]) -> Dict[str, Any]:
    """Mescla filtros do front (contexto) com filtros extraídos da pergunta."""
    base = dict(base or {})
    f = dict(base.get('filtros_coluna') or {})
    for k, v in (extra or {}).items():
        if k.endswith('__inicio') or k.endswith('__fim') or k in _VALID_FILTERS:
            f[k] = v
    base['filtros_coluna'] = f
    # tráfego explícito na pergunta sobrepõe
    if 'Tipo_de_Trafego' in extra:
        base['trafego'] = extra['Tipo_de_Trafego']
    return base

# =========================================================
# Views
# =========================================================
@bp.route('/')
def index():
    csrf_token = ensure_csrf_token()
    return render_template('vivoCRC/index.html', csrf_token=csrf_token)

# ----------------- Upload CSV -----------------
@bp.route('/api/upload-csv', methods=['POST'])
def upload_csv():
    try:
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        file = request.files.get('arquivo')
        origem = (request.form.get('origem') or '').strip().lower()

        if not file or not allowed_file(file.filename):
            return jsonify({'error': 'Apenas arquivos .csv são aceitos.'}), 400
        if origem not in ORIGENS_PERMITIDAS:
            return jsonify({'error': 'Origem inválida.'}), 400

        os.makedirs(_upload_folder(), exist_ok=True)
        filename = secure_filename(file.filename)
        filepath = os.path.join(_upload_folder(), f"{uuid4().hex}_{filename}")
        file.save(filepath)

        encoding_detectado = sample_detect_encoding(filepath)
        leitor = pd.read_csv(
            filepath,
            sep=None,
            engine='python',
            encoding=encoding_detectado,
            dtype=str,
            chunksize=50_000,
            on_bad_lines='skip'
        )

        conn = get_db()
        cur = conn.cursor()
        cur.execute("PRAGMA table_info(tabela_unificada)")
        colunas_tabela = [row[1] for row in cur.fetchall()]
        cur.close()

        total_inseridos = 0

        for chunk in leitor:
            # Mapeia colunas por origem
            if origem == 'science':
                chunk = _map_columns_science(chunk)
                # CN já mapeado por Área Ponta B; NÃO usar cnl
            else:
                chunk = _map_columns_portal(chunk)
                chunk = _apply_cnl_mapping_portal(chunk)

            chunk = _coerce_dates(chunk)

            # sanitiza
            for c in chunk.columns:
                chunk[c] = chunk[c].map(sanitizar_valor)

            # garantir colunas padronizadas
            for col in UNIFIED_COLS:
                if col not in chunk.columns:
                    chunk[col] = 'N/A'

            # remover linhas “vazias” nas essenciais
            essenciais = ['ID', 'EOT', 'Operadora', 'Descricao']
            for col in essenciais:
                if col not in chunk.columns:
                    chunk[col] = 'N/A'
            chunk = chunk[~chunk[essenciais].apply(
                lambda r: all(str(v).strip() in ('', 'N/A', '-----') for v in r), axis=1
            )]

            if chunk.empty:
                continue

            cols_insert = [c for c in UNIFIED_COLS if c in colunas_tabela]
            values = chunk[cols_insert].fillna('N/A').astype(str).values.tolist()

            placeholders = ','.join(['?'] * len(cols_insert))
            sql = f"INSERT OR IGNORE INTO tabela_unificada ({','.join(cols_insert)}) VALUES ({placeholders})"

            cur = conn.cursor()
            for i in range(0, len(values), BULK_INSERT_SIZE):
                cur.executemany(sql, values[i:i+BULK_INSERT_SIZE])
                conn.commit()
                total_inseridos += cur.rowcount
            cur.close()

        try:
            os.remove(filepath)
        except Exception:
            pass

        return jsonify({'message': f'{total_inseridos} novos registros inseridos.'})

    except Exception as e:
        traceback.print_exc()
        return jsonify({'error': f'Erro ao processar CSV: {str(e)}'}), 500

# ----------------- Consulta paginada + resumo -----------------
@bp.route('/api/dados-v2', methods=['POST'])
def api_dados_v2():
    """
    Payload (JSON):
      - page, page_size
      - termo_geral
      - filtros_coluna: { <col>: "valor", "Data_Desativacao__inicio": "YYYY-MM-DD", "Data_Desativacao__fim": "YYYY-MM-DD", ... }
      - trafego: 'STFC' | 'SMP' | None
      - data_inicio, data_fim: intervalo global de datas
      - aplicar_data_somente_science: bool -> se True, aplica o intervalo global APENAS para registros Portal='Science'
    """
    try:
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        payload = request.get_json() or {}
        page = max(int(payload.get('page', 1)), 1)
        page_size = min(max(int(payload.get('page_size', 100)), 1), 1000)
        termo_geral = (payload.get('termo_geral') or '').strip()
        filtros = payload.get('filtros_coluna', {}) or {}
        trafego = payload.get('trafego')  # 'STFC' | 'SMP' | None

        data_inicio = (payload.get('data_inicio') or '').strip() or None
        data_fim = (payload.get('data_fim') or '').strip() or None
        aplicar_data_sci = bool(payload.get('aplicar_data_somente_science'))

        conn = get_db()
        cur = conn.cursor()
        cur.execute("PRAGMA table_info(tabela_unificada)")
        cols_validas = {row[1] for row in cur.fetchall()}

        where_parts: List[str] = []
        params: List[Any] = []

        # 1) Filtros por coluna (whitelist)
        where_cols: List[str] = ["1=1"]
        params_cols: List[Any] = []

        for coluna, valor in filtros.items():
            base = coluna.replace('__inicio', '').replace('__fim', '')
            if base not in cols_validas:
                return jsonify({'error': f'Coluna inválida: {coluna}'}), 400
            if coluna.endswith('__inicio'):
                where_cols.append(f"date({base}) >= date(?)")
                params_cols.append(valor)
            elif coluna.endswith('__fim'):
                where_cols.append(f"date({base}) <= date(?)")
                params_cols.append(valor)
            else:
                where_cols.append(f"{base} LIKE ?")
                params_cols.append(f"%{valor}%")

        if trafego in ('STFC', 'SMP'):
            where_cols.append("Tipo_de_Trafego = ?")
            params_cols.append(trafego)

        if termo_geral:
            where_cols.append("(" + " OR ".join([
                "ID LIKE ?", "EOT LIKE ?", "Operadora LIKE ?", "Descricao LIKE ?"
            ]) + ")")
            params_cols.extend([f"%{termo_geral}%"] * 4)

        where_cols_sql = " AND ".join(where_cols)

        # 2) Intervalo global de datas
        #    - se aplicar_data_sci=True => (Portal='Science' AND range) OR (Portal!='Science')
        #    - senão => aplica para todos (quando datas informadas)
        date_clause = ""
        date_params: List[Any] = []

        if data_inicio or data_fim:
            if aplicar_data_sci:
                # Apenas Science recebe o range. Os demais entram sem filtro de Data_Desativacao.
                subconds: List[str] = []
                subparams: List[Any] = []

                sc_where: List[str] = ["Portal = 'Science'"]
                if data_inicio:
                    sc_where.append("date(Data_Desativacao) >= date(?)")
                    subparams.append(data_inicio)
                if data_fim:
                    sc_where.append("date(Data_Desativacao) <= date(?)")
                    subparams.append(data_fim)
                subconds.append("(" + " AND ".join(sc_where) + ")")

                # Portal != Science (entra sempre)
                subconds.append("(Portal <> 'Science')")

                date_clause = "(" + " OR ".join(subconds) + ")"
                date_params.extend(subparams)
            else:
                # Aplica para todos
                if data_inicio:
                    where_cols_sql += " AND date(Data_Desativacao) >= date(?)"
                    params_cols.append(data_inicio)
                if data_fim:
                    where_cols_sql += " AND date(Data_Desativacao) <= date(?)"
                    params_cols.append(data_fim)

        # Consolida WHERE final
        if date_clause:
            where_parts.append("(" + where_cols_sql + ")")
            where_parts.append(date_clause)
            params = params_cols + date_params
        else:
            where_parts.append(where_cols_sql)
            params = params_cols

        where_sql = " AND ".join(where_parts) if where_parts else "1=1"

        # total
        cur.execute(f"SELECT COUNT(*) FROM tabela_unificada WHERE {where_sql}", params)
        total = int(cur.fetchone()[0])

        # pagina
        offset = (page - 1) * page_size
        # selecionar na ordem do UNIFIED_COLS quando existirem na tabela
        cur.execute("PRAGMA table_info(tabela_unificada)")
        tabela_cols = [row[1] for row in cur.fetchall()]
        select_list = ", ".join([c for c in UNIFIED_COLS if c in tabela_cols]) or "*"

        cur.execute(
            f"SELECT {select_list} FROM tabela_unificada "
            f"WHERE {where_sql} ORDER BY rowid DESC LIMIT ? OFFSET ?",
            params + [page_size, offset]
        )
        colunas = [d[0] for d in cur.description]
        dados = [dict(zip(colunas, r)) for r in cur.fetchall()]

        # resumo (contagem por Tipo_de_Trafego) com mesmos filtros
        cur.execute(
            f"SELECT Tipo_de_Trafego, COUNT(*) FROM tabela_unificada WHERE {where_sql} GROUP BY Tipo_de_Trafego",
            params
        )
        resumo = {k if k else 'N/A': int(v) for k, v in cur.fetchall()}
        cur.close()

        # normaliza data (ISO)
        for item in dados:
            if item.get('Data_Desativacao'):
                item['Data_Desativacao'] = iso_date_or_none(item['Data_Desativacao']) or item['Data_Desativacao']

        return jsonify({
            'total': total,
            'page': page,
            'page_size': page_size,
            'colunas': colunas,
            'dados': dados,
            'resumo': resumo
        })

    except Exception as e:
        traceback.print_exc()
        return jsonify({'error': f'Erro ao buscar dados: {str(e)}'}), 500

# ----------------- Export adaptativo -----------------
@bp.route('/api/exportar', methods=['POST'])
def exportar():
    """
    Exporta dados:
      - XLSX 1 aba quando couber;
      - XLSX multi-aba quando total > EXCEL_MAX_ROWS e <= EXCEL_MAX_ROWS*MAX_XLSX_SHEETS;
      - CSV streaming (BOM UTF-8 + 'sep=,' + CRLF) para bases muito grandes.

    Payload (JSON):
      - ignorar_filtros: bool -> exporta tudo quando True
      - termo_geral, filtros_coluna, trafego
      - data_inicio, data_fim
      - aplicar_data_somente_science: bool
    """
    try:
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        payload = request.get_json() or {}
        ignorar = bool(payload.get('ignorar_filtros'))

        termo_geral = '' if ignorar else (payload.get('termo_geral') or '').strip()
        filtros = {} if ignorar else (payload.get('filtros_coluna', {}) or {})
        trafego = None if ignorar else payload.get('trafego')

        data_inicio = None if ignorar else ((payload.get('data_inicio') or '').strip() or None)
        data_fim    = None if ignorar else ((payload.get('data_fim') or '').strip() or None)
        aplicar_data_sci = False if ignorar else bool(payload.get('aplicar_data_somente_science'))

        conn = get_db()
        cur = conn.cursor()
        cur.execute("PRAGMA table_info(tabela_unificada)")
        cols_validas_list = [row[1] for row in cur.fetchall()]
        cols_validas = set(cols_validas_list)
        if not cols_validas_list:
            return jsonify({'error': 'Tabela vazia ou inexistente.'}), 400

        # WHERE base
        where_cols: List[str] = ["1=1"]
        params_cols: List[Any] = []

        if not ignorar:
            for coluna, valor in (filtros or {}).items():
                base = coluna.replace('__inicio', '').replace('__fim', '')
                if base not in cols_validas:
                    return jsonify({'error': f'Coluna inválida: {coluna}'}), 400
                if coluna.endswith('__inicio'):
                    where_cols.append(f"date({base}) >= date(?)")
                    params_cols.append(valor)
                elif coluna.endswith('__fim'):
                    where_cols.append(f"date({base}) <= date(?)")
                    params_cols.append(valor)
                else:
                    where_cols.append(f"{base} LIKE ?")
                    params_cols.append(f"%{valor}%")

            if trafego in ('STFC', 'SMP'):
                where_cols.append("Tipo_de_Trafego = ?")
                params_cols.append(trafego)

            if termo_geral:
                where_cols.append("(" + " OR ".join([
                    "ID LIKE ?", "EOT LIKE ?", "Operadora LIKE ?", "Descricao LIKE ?"
                ]) + ")")
                params_cols.extend([f"%{termo_geral}%"] * 4)

        where_cols_sql = " AND ".join(where_cols)

        # Intervalo global de datas (igual à consulta)
        date_clause = ""
        date_params: List[Any] = []
        if not ignorar and (data_inicio or data_fim):
            if aplicar_data_sci:
                subconds: List[str] = []
                subparams: List[Any] = []

                sc_where: List[str] = ["Portal = 'Science'"]
                if data_inicio:
                    sc_where.append("date(Data_Desativacao) >= date(?)")
                    subparams.append(data_inicio)
                if data_fim:
                    sc_where.append("date(Data_Desativacao) <= date(?)")
                    subparams.append(data_fim)
                subconds.append("(" + " AND ".join(sc_where) + ")")

                subconds.append("(Portal <> 'Science')")

                date_clause = "(" + " OR ".join(subconds) + ")"
                date_params.extend(subparams)
            else:
                if data_inicio:
                    where_cols_sql += " AND date(Data_Desativacao) >= date(?)"
                    params_cols.append(data_inicio)
                if data_fim:
                    where_cols_sql += " AND date(Data_Desativacao) <= date(?)"
                    params_cols.append(data_fim)

        # WHERE final
        where_parts: List[str] = []
        if date_clause:
            where_parts.append("(" + where_cols_sql + ")")
            where_parts.append(date_clause)
            params = params_cols + date_params
        else:
            where_parts.append(where_cols_sql)
            params = params_cols

        where_sql = " AND ".join(where_parts) if where_parts else "1=1"

        # Contagem para decidir formato
        cur.execute(f"SELECT COUNT(*) FROM tabela_unificada WHERE {where_sql}", params)
        total = int(cur.fetchone()[0])

        if total == 0:
            # CSV apenas com cabeçalho (Excel-friendly)
            def header_only():
                yield "\ufeff".encode("utf-8")           # BOM
                yield "sep=,\r\n".encode("utf-8")        # dica de separador para Excel
                yield (",".join(cols_validas_list) + "\r\n").encode("utf-8")
            fn = f"dados_filtrados_{uuid4().hex}.csv"
            return Response(header_only(), mimetype='text/csv; charset=utf-8',
                            headers={'Content-Disposition': f'attachment; filename={fn}'})

        # ======================================
        # 1) XLSX 1 aba
        # ======================================
        if total <= EXCEL_MAX_ROWS:
            wb = Workbook()
            ws = wb.active
            ws.title = "Dados"
            max_col_width: Dict[int, int] = {}

            cur2 = conn.cursor()
            cur2.execute(f"SELECT * FROM tabela_unificada WHERE {where_sql}", params)
            cols = [d[0] for d in cur2.description]
            ws.append(list(cols))
            for ci, c in enumerate(cols, start=1):
                max_col_width[ci] = max(max_col_width.get(ci, 0), len(str(c)))

            while True:
                rows = cur2.fetchmany(100_000)
                if not rows:
                    break
                for r in rows:
                    out = []
                    for v, col in zip(r, cols):
                        if col == 'Data_Desativacao':
                            v = iso_date_or_none(v) or v
                        out.append(sanitizar_valor(v))
                    ws.append(out)
                    for ci, v in enumerate(out, start=1):
                        l = len(str(v)) if v is not None else 0
                        if l > max_col_width.get(ci, 0):
                            max_col_width[ci] = min(l, 60)
            cur2.close()

            for ci, w in max_col_width.items():
                ws.column_dimensions[get_column_letter(ci)].width = w + 2

            bio = BytesIO()
            wb.save(bio)
            bio.seek(0)
            filename = f"dados_filtrados_{uuid4().hex}.xlsx"
            return send_file(
                bio,
                as_attachment=True,
                download_name=filename,
                mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
            )

        # ======================================
        # 2) XLSX multi-aba (até MAX_XLSX_SHEETS)
        # ======================================
        if total <= EXCEL_MAX_ROWS * MAX_XLSX_SHEETS:
            wb = Workbook()
            ws = wb.active
            ws.title = "Dados 1"
            linhas_por_aba = EXCEL_MAX_ROWS - 1  # reserva 1 p/ cabeçalho
            current_sheet_rows = 0
            sheet_index = 1
            max_col_width: Dict[int, int] = {}

            cur2 = conn.cursor()
            cur2.execute(f"SELECT * FROM tabela_unificada WHERE {where_sql}", params)
            cols = [d[0] for d in cur2.description]

            def add_header(_ws):
                _ws.append(list(cols))
                for ci, c in enumerate(cols, start=1):
                    max_col_width[ci] = max(max_col_width.get(ci, 0), len(str(c)))

            add_header(ws)

            while True:
                rows = cur2.fetchmany(100_000)
                if not rows:
                    break
                for r in rows:
                    if current_sheet_rows >= linhas_por_aba:
                        sheet_index += 1
                        ws = wb.create_sheet(title=f"Dados {sheet_index}")
                        add_header(ws)
                        current_sheet_rows = 0

                    out = []
                    for v, col in zip(r, cols):
                        if col == 'Data_Desativacao':
                            v = iso_date_or_none(v) or v
                        out.append(sanitizar_valor(v))
                    ws.append(out)
                    current_sheet_rows += 1

                    for ci, v in enumerate(out, start=1):
                        l = len(str(v)) if v is not None else 0
                        if l > max_col_width.get(ci, 0):
                            max_col_width[ci] = min(l, 60)
            cur2.close()

            for sh in wb.worksheets:
                for ci, w in max_col_width.items():
                    sh.column_dimensions[get_column_letter(ci)].width = w + 2

            bio = BytesIO()
            wb.save(bio)
            bio.seek(0)
            filename = f"dados_filtrados_{uuid4().hex}.xlsx"
            return send_file(
                bio,
                as_attachment=True,
                download_name=filename,
                mimetype='application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
            )

        # ======================================
        # 3) CSV streaming (Excel-friendly)
        # ======================================
        def stream_csv(select_sql: str, select_params: List[Any], filename: str):
            CHUNK = 100_000
            cur2 = conn.cursor()
            cur2.execute(select_sql, select_params)
            cols = [d[0] for d in cur2.description] if cur2.description else cols_validas_list

            def rows_as_csv_bytes():
                # BOM + dica de separador + cabeçalho
                yield "\ufeff".encode("utf-8")
                yield "sep=,\r\n".encode("utf-8")
                yield (",".join(cols) + "\r\n").encode("utf-8")

                while True:
                    rows = cur2.fetchmany(CHUNK)
                    if not rows:
                        break
                    out_lines = []
                    for r in rows:
                        vals = []
                        for v in r:
                            if v is None:
                                vals.append("")
                            else:
                                s = str(v).replace("\r", " ").replace("\n", " ")
                                if (',' in s) or ('"' in s):
                                    s = '"' + s.replace('"', '""') + '"'
                                vals.append(s)
                        out_lines.append(",".join(vals))
                    yield ("\r\n".join(out_lines) + "\r\n").encode("utf-8")
                cur2.close()

            return Response(
                rows_as_csv_bytes(),
                mimetype='text/csv; charset=utf-8',
                headers={'Content-Disposition': f'attachment; filename={filename}'}
            )

        select_sql = f"SELECT * FROM tabela_unificada WHERE {where_sql}"
        return stream_csv(select_sql, params, f"dados_filtrados_{uuid4().hex}.csv")

    except Exception as e:
        traceback.print_exc()
        return jsonify({'error': f'Erro ao exportar: {str(e)}'}), 500

# ----------------- Agente de IA -----------------
@bp.route('/api/ask', methods=['POST'])
def api_ask():
    """
    Perguntas livres sobre os dados carregados.
    Body JSON:
      - question: string
      - contexto (opcional): payload igual ao de /api/dados-v2 (page/page_size ignorados)
    Retorno:
      - { answer, columns?, rows?, chart?{type,labels,data}, sql_used }
    """
    try:
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        body = request.get_json(force=True) or {}
        question = body.get('question', '')
        contexto = body.get('contexto', {}) or {}

        parsed = _parse_question_pt(question)
        contexto = _merge_filters(contexto, parsed.get('nl_filters', {}))

        conn = get_db()
        cur = conn.cursor()

        where_sql, params = _build_where_from_payload(cur, contexto)
        intent = parsed['intent']
        group_by = parsed.get('group_by')
        limit = parsed.get('limit') or 10

        # Execução
        if intent == 'count' or not group_by:
            sql = f"SELECT COUNT(*) FROM tabela_unificada WHERE {where_sql}"
            cur.execute(sql, params)
            total = int(cur.fetchone()[0])
            cur.close()
            resp = {
                'answer': f"Encontrei **{total:,}** registros conforme os filtros atuais.".replace(',', '.'),
                'sql_used': sql
            }
            # Pequena distribuição por tráfego para contexto
            cur = conn.cursor()
            cur.execute(f"SELECT Tipo_de_Trafego, COUNT(*) FROM tabela_unificada WHERE {where_sql} GROUP BY Tipo_de_Trafego", params)
            dist = cur.fetchall()
            if dist:
                labels = [r[0] or 'N/A' for r in dist]
                data = [int(r[1]) for r in dist]
                resp['chart'] = {'type': 'bar', 'labels': labels, 'data': data, 'title': 'Distribuição por Tráfego'}
            cur.close()
            return jsonify(resp)

        if intent in ('groupby', 'top') and group_by in _GROUPABLE:
            order = "ORDER BY cnt DESC"
            lim = f" LIMIT {int(limit)}" if intent == 'top' else ""
            sql = f"""
                SELECT {group_by} as grupo, COUNT(*) as cnt
                FROM tabela_unificada
                WHERE {where_sql}
                GROUP BY {group_by}
                {order}
                {lim}
            """
            cur.execute(sql, params)
            rows = cur.fetchall()
            cur.close()

            total = sum(int(r['cnt']) for r in rows) if rows else 0
            labels = [(r['grupo'] or 'N/A') for r in rows]
            data = [int(r['cnt']) for r in rows]

            if intent == 'top':
                answer = f"Top {len(rows)} por **{group_by}** (desc): " + ", ".join(
                    [f"{labels[i]}: {data[i]:,}".replace(',', '.') for i in range(len(labels))]
                )
            else:
                answer = f"Distribuição por **{group_by}** ({total:,} no total):".replace(',', '.')

            return jsonify({
                'answer': answer,
                'columns': ['Grupo', 'Quantidade'],
                'rows': [{'Grupo': labels[i], 'Quantidade': data[i]} for i in range(len(labels))],
                'chart': {'type': 'bar', 'labels': labels, 'data': data, 'title': f'Por {group_by}'},
                'sql_used': sql
            })

        # fallback
        return jsonify({'answer': 'Consegui aplicar os filtros, mas não entendi o tipo de resultado desejado. Você pode pedir "total", "distribuição por operadora" ou "top 10 operadoras", por exemplo.', 'sql_used': ''})

    except Exception as e:
        traceback.print_exc()
        return jsonify({'error': f'Erro no agente: {str(e)}'}), 500
