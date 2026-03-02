from __future__ import annotations

import logging
import os
import re
import traceback
import unicodedata
from io import BytesIO
from typing import Any, Dict, Iterable, List, Optional, Tuple
from uuid import uuid4

import chardet
import pandas as pd
from flask import (
    Blueprint, Response, current_app, jsonify,
    render_template, request, send_file, session,
)
from openpyxl import Workbook
from openpyxl.utils import get_column_letter
from werkzeug.utils import secure_filename

from .sqlite_db import get_db, get_table_columns, table_exists

log = logging.getLogger(__name__)

bp = Blueprint("vivo_crc", __name__, url_prefix="/vivo_crc")

# =========================================================
# Constantes
# =========================================================

DEFAULT_UPLOAD_FOLDER = "uploads"
ALLOWED_EXTENSIONS    = {"csv", "xlsx"}

EXCEL_MAX_ROWS   = 1_048_576
MAX_XLSX_SHEETS  = 10
BULK_INSERT_SIZE = 5_000
CNL_IN_CHUNK     = 900

TABELA = "tabela_unificada"

UNIFIED_COLS: List[str] = [
    "ID", "Portal", "Tipo_de_Trafego", "EOT", "Operadora", "CN",
    "ID_Rota", "LABEL_E", "LABEL_S", "Origem_Rota", "Tipo_de_Rota",
    "Sinalizacao", "Trafego", "Descricao", "Central", "Bilhetador",
    "OPC", "DPC", "Data_Desativacao",
]

ORIGENS_PERMITIDAS = {"science", "portal"}

_ESSENCIAIS = ["ID", "EOT", "Operadora", "Descricao"]
_VAZIOS     = {"", "n/a", "-----", "nan", "none", "null"}

_GROUPABLE     = {"Operadora", "EOT", "CN", "Tipo_de_Trafego"}
_VALID_FILTERS = set(UNIFIED_COLS)

# =========================================================
# Mapeamentos de colunas por origem
# Chaves: nome normalizado (uppercase, sem acento, espacos→_)
# Valores: nome final no UNIFIED_COLS / DataFrame
# =========================================================

_MAP_SCIENCE: Dict[str, str] = {
    "SEQUENCIAL":           "ID",
    "COD_CCC_ORIGEM":       "ID",
    "OP_DESTINO":           "EOT",
    "OPERADORA_DESTINO":    "Operadora",
    "AREA_PONTA_B":         "CN",
    "ID_ROTA":              "ID_Rota",
    "TIPO":                 "Origem_Rota",
    "TIPO_DA_ROTA":         "Tipo_de_Rota",
    "SINALIZACAO_DA_ROTA":  "Sinalizacao",
    "TIPO_DE_TRAFEGO":      "Trafego",
    "DESCRICAO":            "Descricao",
    "CENTRAL_ORIGEM":       "Central",
    "OPC":                  "OPC",
    "DPC":                  "DPC",
    "DATA_DESATIVACAO":     "Data_Desativacao",
}

_MAP_PORTAL: Dict[str, str] = {
    "SOLICITACAO":      "ID",
    "EOT":              "EOT",
    "EMPRESA":          "Operadora",
    "CNL_PPI":          "CNL_PPI",
    "CNL":              "CNL",
    "COD_CNL":          "COD_CNL",
    "TIPO":             "Origem_Rota",
    "CATEG_DESCRICAO":  "Tipo_de_Rota",
    "SINALIZACAO":      "Sinalizacao",
    "TRAFEGO_CURSADO":  "Trafego",
    "DESIGNACAO":       "Descricao",
    "CENTRAL":          "Central",
    "BILHETADOR":       "Bilhetador",
    "LABEL_E":          "LABEL_E",
    "LABEL_S":          "LABEL_S",
    "OPC":              "OPC",
    "DPC":              "DPC",
}


# =========================================================
# Utilitarios gerais
# =========================================================

def _upload_folder() -> str:
    return current_app.config.get("UPLOAD_FOLDER", DEFAULT_UPLOAD_FOLDER)


def allowed_file(filename: str) -> bool:
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS


def detect_encoding(filepath: str, sample_bytes: int = 65_536) -> str:
    with open(filepath, "rb") as fh:
        sample = fh.read(sample_bytes)
    result = chardet.detect(sample)
    enc = result.get("encoding") or "utf-8"
    log.debug("Encoding detectado: %s (confianca %.0f%%)", enc, (result.get("confidence") or 0) * 100)
    return enc


def fix_mojibake(text: str) -> str:
    """Corrige UTF-8 bytes lidos como latin-1 (mojibake classico)."""
    try:
        return text.encode("latin-1").decode("utf-8")
    except (UnicodeEncodeError, UnicodeDecodeError):
        return text


def normalize_col(col: str) -> str:
    """
    Normaliza nome de coluna para chave de mapeamento:
    1. Corrige mojibake
    2. Remove bytes de controle
    3. Remove diacriticos (NFKD)
    4. Uppercase
    5. Espacos / hifens / barras → _
    6. Colapsa underscores multiplos
    """
    col = fix_mojibake(str(col)).strip()
    col = re.sub(r"[\x00-\x1F\x7F]", "", col)
    col = unicodedata.normalize("NFKD", col)
    col = "".join(c for c in col if not unicodedata.combining(c))
    col = col.upper()
    col = re.sub(r"[\s\-/\\]+", "_", col)
    col = re.sub(r"_+", "_", col).strip("_")
    return col


def sanitize_value(v: Any) -> Any:
    """Remove bytes de controle e protege contra formula injection no Excel."""
    if isinstance(v, str):
        v = re.sub(r"[\x00-\x1F\x7F]", "", v)
        if v[:1] in ("=", "+", "-", "@"):
            v = "'" + v
    return v


def sanitize_df(df: pd.DataFrame) -> pd.DataFrame:
    for col in df.select_dtypes(include="object").columns:
        df[col] = df[col].map(lambda v: sanitize_value(v) if pd.notna(v) else v)
    return df


def iso_date(s: Any) -> Optional[str]:
    if s is None or s == "" or (isinstance(s, float) and pd.isna(s)):
        return None
    try:
        dt = pd.to_datetime(s, errors="coerce", dayfirst=True)
        return None if pd.isna(dt) else dt.strftime("%Y-%m-%d")
    except Exception:
        return None


def ensure_csrf_token() -> str:
    token = session.get("csrf_token")
    if not token:
        token = uuid4().hex
        session["csrf_token"] = token
    return token


def validate_csrf() -> Optional[tuple]:
    if request.method in ("POST", "PUT", "PATCH", "DELETE"):
        token = request.headers.get("X-CSRF-Token")
        if not token or token != session.get("csrf_token"):
            return jsonify({"error": "CSRF token invalido ou ausente."}), 403
    return None


# =========================================================
# Leitura de arquivos — CSV e XLSX
# =========================================================

def read_file(filepath: str, chunksize: int = 50_000) -> Iterable[pd.DataFrame]:
    """
    Detecta o formato pelo nome do arquivo e devolve chunks de DataFrame.
    Todos os valores sao lidos como string (dtype=str).
    """
    ext = filepath.rsplit(".", 1)[-1].lower()

    if ext == "xlsx":
        log.info("Lendo XLSX: %s", os.path.basename(filepath))
        df = pd.read_excel(
            filepath,
            dtype=str,
            engine="openpyxl",
            keep_default_na=False,
        )
        df = df.fillna("")
        for start in range(0, max(len(df), 1), chunksize):
            yield df.iloc[start: start + chunksize].reset_index(drop=True).copy()

    elif ext == "csv":
        log.info("Lendo CSV: %s", os.path.basename(filepath))
        encoding = detect_encoding(filepath)
        try:
            reader = pd.read_csv(
                filepath,
                sep=None,
                engine="python",
                encoding=encoding,
                dtype=str,
                chunksize=chunksize,
                on_bad_lines="skip",
                keep_default_na=False,
            )
            for chunk in reader:
                yield chunk.fillna("")
        except UnicodeDecodeError:
            log.warning("Encoding '%s' falhou, tentando latin-1.", encoding)
            reader = pd.read_csv(
                filepath,
                sep=None,
                engine="python",
                encoding="latin-1",
                dtype=str,
                chunksize=chunksize,
                on_bad_lines="skip",
                keep_default_na=False,
            )
            for chunk in reader:
                yield chunk.fillna("")

    else:
        raise ValueError(f"Extensao nao suportada: '{ext}'. Use CSV ou XLSX.")


# =========================================================
# Normalizacao de colunas por origem
# =========================================================

def _normalize_headers(df: pd.DataFrame) -> pd.DataFrame:
    original = list(df.columns)
    df.columns = [normalize_col(c) for c in df.columns]
    log.debug("Headers normalizados: %s → %s", original, list(df.columns))
    return df


def map_science(df: pd.DataFrame) -> pd.DataFrame:
    df = _normalize_headers(df)
    df = df.rename(columns={k: v for k, v in _MAP_SCIENCE.items() if k in df.columns})
    df["Portal"]          = "Science"
    df["Tipo_de_Trafego"] = "SMP"
    if "CN" not in df.columns:
        df["CN"] = "N/A"
    log.debug("Science — colunas apos mapeamento: %s", sorted(df.columns.tolist()))
    return df


def map_portal(df: pd.DataFrame) -> pd.DataFrame:
    df = _normalize_headers(df)
    df = df.rename(columns={k: v for k, v in _MAP_PORTAL.items() if k in df.columns})
    df["Portal"]          = "CSO"
    df["Tipo_de_Trafego"] = "STFC"
    for col, default in [("ID_Rota", "N/A"), ("OPC", "-----"), ("DPC", "-----")]:
        if col not in df.columns:
            df[col] = default
    log.debug("Portal — colunas apos mapeamento: %s", sorted(df.columns.tolist()))
    return df


def apply_cnl_mapping(df: pd.DataFrame, conn) -> pd.DataFrame:
    """
    Resolve CN para o Portal CSO consultando a tabela auxiliar 'cnl'.
    Procura coluna CNL_PPI, CNL ou COD_CNL (nessa ordem de prioridade).
    """
    if df.empty:
        if "CN" not in df.columns:
            df["CN"] = "N/A"
        return df

    if not table_exists(conn, "cnl"):
        log.warning("Tabela 'cnl' nao encontrada — CN sera 'N/A'.")
        if "CN" not in df.columns:
            df["CN"] = "N/A"
        return df

    cand = next((c for c in ["CNL_PPI", "CNL", "COD_CNL"] if c in df.columns), None)
    if not cand:
        log.warning("Nenhuma coluna CNL no DataFrame do Portal. CN = N/A.")
        if "CN" not in df.columns:
            df["CN"] = "N/A"
        return df

    valores = (
        df[cand].dropna().astype(str).str.strip()
        .pipe(lambda s: s[s != ""])
        .unique().tolist()
    )

    if not valores:
        if "CN" not in df.columns:
            df["CN"] = "N/A"
        return df

    mapa: Dict[str, str] = {}
    cur = conn.cursor()
    cur.row_factory = None
    try:
        for i in range(0, len(valores), CNL_IN_CHUNK):
            pedaco = valores[i: i + CNL_IN_CHUNK]
            ph = ",".join(["?"] * len(pedaco))
            cur.execute(f"SELECT COD_CNL, CN FROM cnl WHERE COD_CNL IN ({ph})", pedaco)
            for cod, cn in cur.fetchall():
                mapa[str(cod).strip()] = str(cn).strip()
    finally:
        cur.close()

    log.debug("CNL: %d valores → %d mapeamentos encontrados", len(valores), len(mapa))

    serie = df[cand].astype(str).str.strip().str.replace(r"\.0$", "", regex=True)
    df["CN"] = serie.map(mapa).fillna("N/A")
    return df


def coerce_dates(df: pd.DataFrame) -> pd.DataFrame:
    if "Data_Desativacao" in df.columns:
        df["Data_Desativacao"] = df["Data_Desativacao"].apply(iso_date)
    return df


def discard_empty_rows(df: pd.DataFrame) -> pd.DataFrame:
    """Descarta linhas onde TODOS os campos essenciais estao vazios/N/A."""
    for col in _ESSENCIAIS:
        if col not in df.columns:
            df[col] = "N/A"

    mask = df[_ESSENCIAIS].apply(
        lambda row: all(str(v).strip().lower() in _VAZIOS for v in row),
        axis=1,
    )
    n = mask.sum()
    if n:
        log.debug("Descartando %d linhas com campos essenciais vazios.", n)
    return df[~mask].reset_index(drop=True)


def fill_missing_unified_cols(df: pd.DataFrame) -> pd.DataFrame:
    for col in UNIFIED_COLS:
        if col not in df.columns:
            df[col] = "N/A"
    return df


# =========================================================
# Pipeline de ETL por chunk
# =========================================================

def etl_chunk(chunk: pd.DataFrame, origem: str, conn) -> pd.DataFrame:
    if origem == "science":
        chunk = map_science(chunk)
    else:
        chunk = map_portal(chunk)
        chunk = apply_cnl_mapping(chunk, conn)

    chunk = coerce_dates(chunk)
    chunk = sanitize_df(chunk)
    chunk = fill_missing_unified_cols(chunk)
    chunk = discard_empty_rows(chunk)
    return chunk


# =========================================================
# Construcao centralizada do WHERE SQL
# =========================================================

def build_where(
    cols_validas: set,
    filtros: Dict[str, Any],
    trafego: Optional[str],
    termo_geral: str,
    data_inicio: Optional[str],
    data_fim: Optional[str],
    aplicar_data_sci: bool,
    strict: bool = True,
) -> Tuple[str, List[Any]]:
    parts:  List[str] = ["1=1"]
    params: List[Any] = []

    for coluna, valor in (filtros or {}).items():
        base = coluna.replace("__inicio", "").replace("__fim", "")
        if base not in cols_validas:
            if strict:
                raise ValueError(f"Coluna invalida nos filtros: '{coluna}'")
            continue
        if coluna.endswith("__inicio"):
            parts.append(f"date({base}) >= date(?)")
            params.append(valor)
        elif coluna.endswith("__fim"):
            parts.append(f"date({base}) <= date(?)")
            params.append(valor)
        else:
            parts.append(f"{base} LIKE ?")
            params.append(f"%{valor}%")

    if trafego in ("STFC", "SMP"):
        parts.append("Tipo_de_Trafego = ?")
        params.append(trafego)

    if termo_geral:
        parts.append("(ID LIKE ? OR EOT LIKE ? OR Operadora LIKE ? OR Descricao LIKE ?)")
        params.extend([f"%{termo_geral}%"] * 4)

    base_sql = " AND ".join(parts)

    if not (data_inicio or data_fim):
        return base_sql, params

    date_params: List[Any] = []
    if aplicar_data_sci:
        sc = ["Portal = 'Science'"]
        if data_inicio:
            sc.append("date(Data_Desativacao) >= date(?)")
            date_params.append(data_inicio)
        if data_fim:
            sc.append("date(Data_Desativacao) <= date(?)")
            date_params.append(data_fim)
        date_clause = f"(({' AND '.join(sc)}) OR (Portal <> 'Science'))"
        return f"({base_sql}) AND {date_clause}", params + date_params
    else:
        if data_inicio:
            base_sql += " AND date(Data_Desativacao) >= date(?)"
            params.append(data_inicio)
        if data_fim:
            base_sql += " AND date(Data_Desativacao) <= date(?)"
            params.append(data_fim)
        return base_sql, params


def _where_from_payload(conn, payload: Dict[str, Any]) -> Tuple[str, List[Any]]:
    cols = set(get_table_columns(conn, TABELA))
    return build_where(
        cols_validas=cols,
        filtros=payload.get("filtros_coluna") or {},
        trafego=payload.get("trafego"),
        termo_geral=(payload.get("termo_geral") or "").strip(),
        data_inicio=(payload.get("data_inicio") or "").strip() or None,
        data_fim=(payload.get("data_fim") or "").strip() or None,
        aplicar_data_sci=bool(payload.get("aplicar_data_somente_science")),
        strict=False,
    )


# =========================================================
# Agente NLQ
# =========================================================

def parse_question(question: str) -> Dict[str, Any]:
    q = (question or "").strip().lower()
    intent   = "count"
    group_by = None
    limit    = None

    m = re.search(r"top\s+(\d+)\s+(operadoras?|eots?|cns?|tr[aá]fegos?)", q)
    if m:
        intent   = "top"
        limit    = max(1, min(int(m.group(1)), 50))
        group_by = (
            "Operadora"       if "operador" in m.group(2)
            else "EOT"        if "eot"      in m.group(2)
            else "CN"         if "cn"       in m.group(2)
            else "Tipo_de_Trafego"
        )

    m2 = re.search(r"(distribui[cç][aã]o|quebra)\s+por\s+(operadora|eot|cn|tr[aá]fego)", q)
    if not m2:
        m2 = re.search(r"\bpor\s+(operadora|eot|cn|tr[aá]fego)\b", q)
    if m2:
        intent   = "groupby"
        chave    = m2.group(m2.lastindex)
        group_by = (
            "Operadora"       if "operadora" in chave
            else "EOT"        if "eot"       in chave
            else "CN"         if "cn"        in chave
            else "Tipo_de_Trafego"
        )

    nl: Dict[str, Any] = {}
    if re.search(r"\bstfc\b", q): nl["Tipo_de_Trafego"] = "STFC"
    if re.search(r"\bsmp\b",  q): nl["Tipo_de_Trafego"] = "SMP"

    m3 = re.search(r"operadora\s+([a-z0-9\-\._ ]{2,})", q)
    if m3: nl["Operadora"] = m3.group(1).strip().upper()

    m4 = re.search(r"\beot\s+([0-9\-]+)", q)
    if m4: nl["EOT"] = m4.group(1).strip()

    m5 = re.search(r"\bcn\s+([0-9\-]+)", q)
    if m5: nl["CN"] = m5.group(1).strip()

    m6 = re.search(r"entre\s+(\d{4}-\d{2}-\d{2})\s+e\s+(\d{4}-\d{2}-\d{2})", q)
    if m6:
        nl["Data_Desativacao__inicio"] = m6.group(1)
        nl["Data_Desativacao__fim"]    = m6.group(2)

    return {
        "intent":     intent,
        "group_by":   group_by if group_by in _GROUPABLE else None,
        "limit":      limit,
        "nl_filters": nl,
    }


def merge_filters(base: Dict[str, Any], extra: Dict[str, Any]) -> Dict[str, Any]:
    base = dict(base or {})
    f = dict(base.get("filtros_coluna") or {})
    for k, v in (extra or {}).items():
        if k.endswith("__inicio") or k.endswith("__fim") or k in _VALID_FILTERS:
            f[k] = v
    base["filtros_coluna"] = f
    if "Tipo_de_Trafego" in extra:
        base["trafego"] = extra["Tipo_de_Trafego"]
    return base


# =========================================================
# Helpers de exportacao Excel / CSV
# =========================================================

def _row_to_list(r: Any, cols: List[str]) -> List[Any]:
    row = tuple(r)
    out = []
    for v, col in zip(row, cols):
        if col == "Data_Desativacao":
            v = iso_date(v) or v
        out.append(sanitize_value(v))
    return out


def _write_xlsx_sheet(ws, cursor, cols: List[str], max_w: Dict[int, int]):
    ws.append(cols)
    for ci, c in enumerate(cols, 1):
        max_w[ci] = max(max_w.get(ci, 0), len(c))

    while True:
        rows = cursor.fetchmany(100_000)
        if not rows:
            break
        for r in rows:
            out = _row_to_list(r, cols)
            ws.append(out)
            for ci, v in enumerate(out, 1):
                l = len(str(v)) if v is not None else 0
                if l > max_w.get(ci, 0):
                    max_w[ci] = min(l, 60)


def _xlsx_response(wb: Workbook, max_w: Dict[int, int]) -> Response:
    for sh in wb.worksheets:
        for ci, w in max_w.items():
            sh.column_dimensions[get_column_letter(ci)].width = w + 2
    bio = BytesIO()
    wb.save(bio)
    bio.seek(0)
    fn = f"dados_filtrados_{uuid4().hex}.xlsx"
    return send_file(
        bio,
        as_attachment=True,
        download_name=fn,
        mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    )


# =========================================================
# Views
# =========================================================

@bp.route("/")
def index():
    return render_template("vivoCRC/index.html", csrf_token=ensure_csrf_token())


# ─── Diagnostico ──────────────────────────────────────────

@bp.route("/api/status", methods=["GET"])
def api_status():
    try:
        conn = get_db()

        if not table_exists(conn, TABELA):
            return jsonify({
                "status":   "erro",
                "mensagem": f"Tabela '{TABELA}' nao existe. Execute 'flask init-db'.",
                "colunas":  [],
                "totais":   {},
            })

        colunas = get_table_columns(conn, TABELA)

        cur = conn.cursor()
        cur.row_factory = None
        cur.execute(f"SELECT Portal, COUNT(*) FROM {TABELA} GROUP BY Portal")
        totais = {(str(r[0]) if r[0] else "N/A"): int(r[1]) for r in cur.fetchall()}
        cur.close()

        return jsonify({
            "status":            "ok",
            "colunas":           colunas,
            "totais_por_portal": totais,
        })
    except Exception as e:
        traceback.print_exc()
        return jsonify({"status": "erro", "mensagem": str(e)}), 500


# ─── Upload CSV / XLSX ────────────────────────────────────
# Duas rotas mapeiam para a mesma funcao:
#   /api/upload-csv     → retrocompatibilidade com templates/JS existentes
#   /api/upload-arquivo → nome semantico novo

@bp.route("/api/upload-csv",     methods=["POST"])
@bp.route("/api/upload-arquivo", methods=["POST"])
def upload_csv():
    try:
        csrf_err = validate_csrf()
        if csrf_err:
            return csrf_err

        file   = request.files.get("arquivo")
        origem = (request.form.get("origem") or "").strip().lower()

        if not file or not file.filename:
            return jsonify({"error": "Nenhum arquivo enviado."}), 400
        if not allowed_file(file.filename):
            return jsonify({"error": "Apenas arquivos .csv ou .xlsx sao aceitos."}), 400
        if origem not in ORIGENS_PERMITIDAS:
            return jsonify({"error": f"Origem invalida '{origem}'. Use 'science' ou 'portal'."}), 400

        os.makedirs(_upload_folder(), exist_ok=True)
        filename = secure_filename(file.filename)
        filepath = os.path.join(_upload_folder(), f"{uuid4().hex}_{filename}")
        file.save(filepath)
        log.info("Arquivo salvo: %s (origem=%s)", filepath, origem)

        conn = get_db()

        if not table_exists(conn, TABELA):
            return jsonify({
                "error": f"Tabela '{TABELA}' nao existe. Execute 'flask init-db' antes de carregar dados."
            }), 500

        tabela_cols = get_table_columns(conn, TABELA)
        log.info("Colunas da tabela '%s': %s", TABELA, tabela_cols)

        cols_insert = [c for c in UNIFIED_COLS if c in tabela_cols]
        if not cols_insert:
            return jsonify({
                "error": (
                    f"Nenhuma coluna de UNIFIED_COLS existe na tabela '{TABELA}'. "
                    f"Colunas da tabela: {tabela_cols}. Verifique o schema.sql."
                )
            }), 500

        log.info("Colunas para INSERT (%d): %s", len(cols_insert), cols_insert)

        placeholders = ",".join(["?"] * len(cols_insert))
        sql_insert   = (
            f"INSERT OR IGNORE INTO {TABELA} "
            f"({', '.join(cols_insert)}) VALUES ({placeholders})"
        )

        total_lidas   = 0
        changes_antes = conn.total_changes

        try:
            cur = conn.cursor()
            cur.row_factory = None

            for chunk in read_file(filepath):
                total_lidas += len(chunk)
                log.debug("Chunk lido: %d linhas, colunas=%s", len(chunk), list(chunk.columns))

                chunk = etl_chunk(chunk, origem, conn)

                if chunk.empty:
                    log.debug("Chunk vazio apos ETL, pulando.")
                    continue

                log.debug("Chunk apos ETL: %d linhas", len(chunk))

                values = (
                    chunk[cols_insert]
                    .fillna("N/A")
                    .astype(str)
                    .values.tolist()
                )

                for i in range(0, len(values), BULK_INSERT_SIZE):
                    cur.executemany(sql_insert, values[i: i + BULK_INSERT_SIZE])
                    conn.commit()

            cur.close()

        finally:
            try:
                os.remove(filepath)
                log.debug("Arquivo temporario removido: %s", filepath)
            except OSError:
                pass

        total_inseridos = conn.total_changes - changes_antes
        log.info("Upload concluido: %d lidas, %d inseridas.", total_lidas, total_inseridos)

        return jsonify({
            "message": (
                f"Upload concluido com sucesso. "
                f"Linhas lidas: {total_lidas} | "
                f"Registros novos inseridos: {total_inseridos}."
            )
        })

    except Exception as e:
        traceback.print_exc()
        return jsonify({"error": f"Erro ao processar arquivo: {str(e)}"}), 500


# ─── Consulta paginada ────────────────────────────────────

@bp.route("/api/dados-v2", methods=["POST"])
def api_dados_v2():
    try:
        csrf_err = validate_csrf()
        if csrf_err:
            return csrf_err

        payload   = request.get_json() or {}
        page      = max(int(payload.get("page", 1)), 1)
        page_size = min(max(int(payload.get("page_size", 100)), 1), 1000)

        conn = get_db()
        cols = set(get_table_columns(conn, TABELA))

        if not cols:
            return jsonify({"error": f"Tabela '{TABELA}' nao encontrada ou vazia."}), 500

        try:
            where_sql, params = build_where(
                cols_validas=cols,
                filtros=payload.get("filtros_coluna") or {},
                trafego=payload.get("trafego"),
                termo_geral=(payload.get("termo_geral") or "").strip(),
                data_inicio=(payload.get("data_inicio") or "").strip() or None,
                data_fim=(payload.get("data_fim") or "").strip() or None,
                aplicar_data_sci=bool(payload.get("aplicar_data_somente_science")),
                strict=True,
            )
        except ValueError as ve:
            return jsonify({"error": str(ve)}), 400

        cur = conn.cursor()
        cur.row_factory = None

        cur.execute(f"SELECT COUNT(*) FROM {TABELA} WHERE {where_sql}", params)
        total = int(cur.fetchone()[0])

        select_cols = [c for c in UNIFIED_COLS if c in cols]
        select_sql  = ", ".join(select_cols) if select_cols else "*"
        offset      = (page - 1) * page_size

        cur.execute(
            f"SELECT {select_sql} FROM {TABELA} "
            f"WHERE {where_sql} ORDER BY rowid DESC LIMIT ? OFFSET ?",
            params + [page_size, offset],
        )
        colunas = [d[0] for d in cur.description]
        dados   = [dict(zip(colunas, tuple(r))) for r in cur.fetchall()]

        cur.execute(
            f"SELECT Tipo_de_Trafego, COUNT(*) FROM {TABELA} "
            f"WHERE {where_sql} GROUP BY Tipo_de_Trafego",
            params,
        )
        resumo = {(str(r[0]) if r[0] else "N/A"): int(r[1]) for r in cur.fetchall()}
        cur.close()

        for item in dados:
            if item.get("Data_Desativacao"):
                item["Data_Desativacao"] = iso_date(item["Data_Desativacao"]) or item["Data_Desativacao"]

        return jsonify({
            "total":     total,
            "page":      page,
            "page_size": page_size,
            "colunas":   colunas,
            "dados":     dados,
            "resumo":    resumo,
        })

    except Exception as e:
        traceback.print_exc()
        return jsonify({"error": f"Erro ao buscar dados: {str(e)}"}), 500


# ─── Exportacao adaptativa ────────────────────────────────

@bp.route("/api/exportar", methods=["POST"])
def exportar():
    try:
        csrf_err = validate_csrf()
        if csrf_err:
            return csrf_err

        payload      = request.get_json() or {}
        ignorar      = bool(payload.get("ignorar_filtros"))
        conn         = get_db()
        tabela_cols  = get_table_columns(conn, TABELA)
        cols_validas = set(tabela_cols)

        if not tabela_cols:
            return jsonify({"error": f"Tabela '{TABELA}' vazia ou inexistente."}), 400

        if ignorar:
            where_sql, params = "1=1", []
        else:
            try:
                where_sql, params = build_where(
                    cols_validas=cols_validas,
                    filtros=payload.get("filtros_coluna") or {},
                    trafego=payload.get("trafego"),
                    termo_geral=(payload.get("termo_geral") or "").strip(),
                    data_inicio=(payload.get("data_inicio") or "").strip() or None,
                    data_fim=(payload.get("data_fim") or "").strip() or None,
                    aplicar_data_sci=bool(payload.get("aplicar_data_somente_science")),
                    strict=True,
                )
            except ValueError as ve:
                return jsonify({"error": str(ve)}), 400

        cur = conn.cursor()
        cur.row_factory = None
        cur.execute(f"SELECT COUNT(*) FROM {TABELA} WHERE {where_sql}", params)
        total = int(cur.fetchone()[0])
        cur.close()

        if total == 0:
            fn = f"dados_filtrados_{uuid4().hex}.csv"

            def header_only():
                yield "\ufeff".encode("utf-8")
                yield "sep=,\r\n".encode("utf-8")
                yield (",".join(tabela_cols) + "\r\n").encode("utf-8")

            return Response(
                header_only(),
                mimetype="text/csv; charset=utf-8",
                headers={"Content-Disposition": f"attachment; filename={fn}"},
            )

        # Colunas de exportação: apenas colunas de negócio (sem rowid)
        export_cols   = [c for c in UNIFIED_COLS if c in cols_validas]
        export_select = ", ".join(export_cols) if export_cols else "*"

        # ── XLSX 1 aba ───────────────────────────────────
        if total <= EXCEL_MAX_ROWS:
            wb    = Workbook()
            ws    = wb.active
            ws.title = "Dados"
            max_w: Dict[int, int] = {}

            cur2 = conn.cursor()
            cur2.row_factory = None
            cur2.execute(f"SELECT {export_select} FROM {TABELA} WHERE {where_sql}", params)
            cols = [d[0] for d in cur2.description]

            _write_xlsx_sheet(ws, cur2, cols, max_w)
            cur2.close()

            return _xlsx_response(wb, max_w)

        # ── XLSX multi-aba ───────────────────────────────
        if total <= EXCEL_MAX_ROWS * MAX_XLSX_SHEETS:
            wb          = Workbook()
            ws          = wb.active
            ws.title    = "Dados 1"
            max_rows_sh = EXCEL_MAX_ROWS - 1
            sh_rows     = 0
            sh_index    = 1
            max_w: Dict[int, int] = {}

            cur2 = conn.cursor()
            cur2.row_factory = None
            cur2.execute(f"SELECT {export_select} FROM {TABELA} WHERE {where_sql}", params)
            cols = [d[0] for d in cur2.description]

            ws.append(cols)
            for ci, c in enumerate(cols, 1):
                max_w[ci] = len(c)

            while True:
                rows = cur2.fetchmany(100_000)
                if not rows:
                    break
                for r in rows:
                    if sh_rows >= max_rows_sh:
                        sh_index += 1
                        ws = wb.create_sheet(title=f"Dados {sh_index}")
                        ws.append(cols)
                        sh_rows = 0

                    out = _row_to_list(r, cols)
                    ws.append(out)
                    sh_rows += 1
                    for ci, v in enumerate(out, 1):
                        l = len(str(v)) if v is not None else 0
                        if l > max_w.get(ci, 0):
                            max_w[ci] = min(l, 60)

            cur2.close()
            return _xlsx_response(wb, max_w)

        # ── CSV streaming ────────────────────────────────
        fn = f"dados_filtrados_{uuid4().hex}.csv"

        def stream_csv():
            c2 = conn.cursor()
            c2.row_factory = None
            c2.execute(f"SELECT {export_select} FROM {TABELA} WHERE {where_sql}", params)
            col_names = [d[0] for d in c2.description]

            yield "\ufeff".encode("utf-8")
            yield "sep=,\r\n".encode("utf-8")
            yield (",".join(col_names) + "\r\n").encode("utf-8")

            while True:
                rows = c2.fetchmany(100_000)
                if not rows:
                    break
                lines: List[str] = []
                for r in rows:
                    vals: List[str] = []
                    for v in tuple(r):
                        if v is None:
                            vals.append("")
                        else:
                            s = str(v).replace("\r", " ").replace("\n", " ")
                            if "," in s or '"' in s:
                                s = '"' + s.replace('"', '""') + '"'
                            vals.append(s)
                    lines.append(",".join(vals))
                yield ("\r\n".join(lines) + "\r\n").encode("utf-8")

            c2.close()

        return Response(
            stream_csv(),
            mimetype="text/csv; charset=utf-8",
            headers={"Content-Disposition": f"attachment; filename={fn}"},
        )

    except Exception as e:
        traceback.print_exc()
        return jsonify({"error": f"Erro ao exportar: {str(e)}"}), 500


# ─── Agente NLQ ───────────────────────────────────────────

@bp.route("/api/ask", methods=["POST"])
def api_ask():
    try:
        csrf_err = validate_csrf()
        if csrf_err:
            return csrf_err

        body     = request.get_json(force=True) or {}
        question = body.get("question", "")
        contexto = body.get("contexto") or {}

        parsed   = parse_question(question)
        contexto = merge_filters(contexto, parsed.get("nl_filters", {}))

        conn      = get_db()
        where_sql, params = _where_from_payload(conn, contexto)

        intent   = parsed["intent"]
        group_by = parsed.get("group_by")
        limit    = parsed.get("limit") or 10

        cur = conn.cursor()
        cur.row_factory = None

        if intent == "count" or not group_by:
            sql = f"SELECT COUNT(*) FROM {TABELA} WHERE {where_sql}"
            cur.execute(sql, params)
            total = int(cur.fetchone()[0])

            cur.execute(
                f"SELECT Tipo_de_Trafego, COUNT(*) FROM {TABELA} "
                f"WHERE {where_sql} GROUP BY Tipo_de_Trafego",
                params,
            )
            dist   = cur.fetchall()
            labels = [str(r[0]) if r[0] else "N/A" for r in dist]
            data   = [int(r[1]) for r in dist]
            cur.close()

            resp: Dict[str, Any] = {
                "answer":   f"Encontrei **{total:,}** registros conforme os filtros atuais.".replace(",", "."),
                "sql_used": sql,
            }
            if dist:
                resp["chart"] = {
                    "type":   "bar",
                    "labels": labels,
                    "data":   data,
                    "title":  "Distribuicao por Trafego",
                }
            return jsonify(resp)

        if intent in ("groupby", "top") and group_by in _GROUPABLE:
            lim = f" LIMIT {int(limit)}" if intent == "top" else ""
            sql = (
                f"SELECT {group_by} AS grupo, COUNT(*) AS cnt "
                f"FROM {TABELA} WHERE {where_sql} "
                f"GROUP BY {group_by} ORDER BY cnt DESC{lim}"
            )
            cur.execute(sql, params)
            rows = cur.fetchall()
            cur.close()

            labels = [str(r[0]) if r[0] else "N/A" for r in rows]
            data   = [int(r[1]) for r in rows]
            total  = sum(data)

            if intent == "top":
                partes = ", ".join(f"{labels[i]}: {data[i]:,}" for i in range(len(labels)))
                answer = f"Top {len(rows)} por **{group_by}**: {partes}".replace(",", ".")
            else:
                answer = f"Distribuicao por **{group_by}** ({total:,} no total):".replace(",", ".")

            return jsonify({
                "answer":   answer,
                "columns":  ["Grupo", "Quantidade"],
                "rows":     [{"Grupo": labels[i], "Quantidade": data[i]} for i in range(len(labels))],
                "chart":    {"type": "bar", "labels": labels, "data": data, "title": f"Por {group_by}"},
                "sql_used": sql,
            })

        cur.close()
        return jsonify({
            "answer": (
                "Consegui aplicar os filtros, mas nao entendi o tipo de resultado desejado. "
                "Tente: 'total', 'distribuicao por operadora' ou 'top 10 operadoras'."
            ),
            "sql_used": "",
        })

    except Exception as e:
        traceback.print_exc()
        return jsonify({"error": f"Erro no agente: {str(e)}"}), 500
