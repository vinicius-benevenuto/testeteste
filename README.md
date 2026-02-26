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

from app.sqlite_db import get_db

log = logging.getLogger(__name__)

bp = Blueprint('vivo_crc', __name__, url_prefix="/vivo_crc")

DEFAULT_UPLOAD_FOLDER = 'uploads'
ALLOWED_EXTENSIONS = {"csv", "xlsx"}

EXCEL_MAX_ROWS   = 1_048_576
MAX_XLSX_SHEETS  = 10
BULK_INSERT_SIZE = 5_000
CNL_IN_CHUNK     = 900

UNIFIED_COLS = [
    "ID", "Portal", "Tipo_de_Trafego", "EOT", "Operadora", "CN",
    "ID_Rota", "LABEL_E", "LABEL_S", "Origem_Rota", "Tipo_de_Rota",
    "Sinalizacao", "Trafego", "Descricao", "Central", "Bilhetador",
    "OPC", "DPC", "Data_Desativacao",
]

ORIGENS_PERMITIDAS = {"science", "portal"}
_ESSENCIAIS        = ["ID", "EOT", "Operadora", "Descricao"]
_VAZIOS            = {"", "n/a", "-----", "nan", "none"}

_GROUPABLE     = {"Operadora", "EOT", "CN", "Tipo_de_Trafego"}
_VALID_FILTERS = set(UNIFIED_COLS)


# =========================================================
# Utilitários gerais
# =========================================================

def _upload_folder() -> str:
    return current_app.config.get("UPLOAD_FOLDER", DEFAULT_UPLOAD_FOLDER)


def allowed_file(filename: str) -> bool:
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS


def detect_encoding(filepath: str, sample_bytes: int = 65_536) -> str:
    with open(filepath, "rb") as fh:
        sample = fh.read(sample_bytes)
    return chardet.detect(sample).get("encoding") or "utf-8"


def fix_mojibake(text: str) -> str:
    try:
        return text.encode("latin-1").decode("utf-8")
    except (UnicodeEncodeError, UnicodeDecodeError):
        return text


def normalize_column_name(col: str) -> str:
    col = fix_mojibake(str(col))
    col = re.sub(r"[\x00-\x1F\x7F]", "", col).strip()
    col = unicodedata.normalize("NFKD", col)
    col = "".join(c for c in col if not unicodedata.combining(c))
    col = col.upper()
    col = re.sub(r"[\s\-/]+", "_", col)
    col = re.sub(r"_+", "_", col).strip("_")
    return col


def sanitizar_valor(valor: Any) -> Any:
    if isinstance(valor, str):
        valor = re.sub(r"[\x00-\x1F\x7F]", "", valor)
        if valor[:1] in ("=", "+", "-", "@"):
            return "'" + valor
    return valor


def _sanitize_df(df: pd.DataFrame) -> pd.DataFrame:
    str_cols = df.select_dtypes(include="object").columns
    for col in str_cols:
        df[col] = df[col].apply(sanitizar_valor)
    return df


def iso_date_or_none(s: Any) -> Optional[str]:
    if s is None or (isinstance(s, float) and pd.isna(s)):
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


def validate_csrf_header():
    if request.method in ("POST", "PUT", "PATCH", "DELETE"):
        token = request.headers.get("X-CSRF-Token")
        if not token or token != session.get("csrf_token"):
            return jsonify({"error": "CSRF token inválido ou ausente."}), 403
    return None


# =========================================================
# Leitura de arquivos — CSV e XLSX
# =========================================================

def read_file(filepath: str, chunksize: int = 50_000) -> Iterable[pd.DataFrame]:
    ext = filepath.rsplit(".", 1)[-1].lower()

    if ext == "xlsx":
        df_full = pd.read_excel(
            filepath,
            dtype=str,
            engine="openpyxl",
            keep_default_na=False,
        )
        for start in range(0, max(len(df_full), 1), chunksize):
            yield df_full.iloc[start: start + chunksize].copy()

    elif ext == "csv":
        encoding = detect_encoding(filepath)
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
        yield from reader

    else:
        raise ValueError(f"Extensão não suportada: '{ext}'. Use CSV ou XLSX.")


# =========================================================
# Normalização de colunas por origem
# =========================================================

_MAP_SCIENCE_NORM = {
    "SEQUENCIAL":        "ID",
    "OP_DESTINO":        "EOT",
    "OPERADORA_DESTINO": "Operadora",
    "AREA_PONTA_B":      "CN",
    "ID_ROTA":           "ID_Rota",
    "TIPO":              "Origem_Rota",
    "TIPO_DA_ROTA":      "Tipo_de_Rota",
    "SINALIZACAO_DA_ROTA": "Sinalizacao",
    "TIPO_DE_TRAFEGO":   "Trafego",
    "DESCRICAO":         "Descricao",
    "CENTRAL_ORIGEM":    "Central",
    "OPC":               "OPC",
    "DPC":               "DPC",
    "DATA_DESATIVACAO":  "Data_Desativacao",
}

_MAP_PORTAL_NORM = {
    "SOLICITACAO":    "ID",
    "EOT":            "EOT",
    "EMPRESA":        "Operadora",
    "CNL_PPI":        "CNL_PPI",
    "CNL":            "CNL",
    "COD_CNL":        "COD_CNL",
    "TIPO":           "Origem_Rota",
    "CATEG_DESCRICAO":"Tipo_de_Rota",
    "SINALIZACAO":    "Sinalizacao",
    "TRAFEGO_CURSADO":"Trafego",
    "DESIGNACAO":     "Descricao",
    "CENTRAL":        "Central",
    "BILHETADOR":     "Bilhetador",
    "LABEL_E":        "LABEL_E",
    "LABEL_S":        "LABEL_S",
    "OPC":            "OPC",
    "DPC":            "DPC",
}


def _normalize_headers(df: pd.DataFrame) -> pd.DataFrame:
    df.columns = [normalize_column_name(c) for c in df.columns]
    return df


def _map_columns_science(df: pd.DataFrame) -> pd.DataFrame:
    df = _normalize_headers(df)
    df = df.rename(columns={k: v for k, v in _MAP_SCIENCE_NORM.items() if k in df.columns})
    df["Portal"] = "Science"
    df["Tipo_de_Trafego"] = "SMP"
    if "CN" not in df.columns:
        df["CN"] = "N/A"
    return df


def _map_columns_portal(df: pd.DataFrame) -> pd.DataFrame:
    df = _normalize_headers(df)
    df = df.rename(columns={k: v for k, v in _MAP_PORTAL_NORM.items() if k in df.columns})
    df["Portal"] = "CSO"
    df["Tipo_de_Trafego"] = "STFC"
    for col, default in [("ID_Rota", "N/A"), ("OPC", "-----"), ("DPC", "-----")]:
        if col not in df.columns:
            df[col] = default
    return df


def _apply_cnl_mapping_portal(df: pd.DataFrame, conn) -> pd.DataFrame:
    if df.empty:
        return df

    cand = next((c for c in df.columns if c.upper() in {"CNL_PPI", "CNL", "COD_CNL"}), None)
    if not cand:
        if "CN" not in df.columns:
            df["CN"] = "N/A"
        return df

    valores = df[cand].dropna().astype(str).unique().tolist()
    if not valores:
        if "CN" not in df.columns:
            df["CN"] = "N/A"
        return df

    cursor = conn.cursor()
    mapa: Dict[str, str] = {}
    for i in range(0, len(valores), CNL_IN_CHUNK):
        pedaco = valores[i: i + CNL_IN_CHUNK]
        placeholders = ",".join(["?"] * len(pedaco))
        cursor.execute(
            f"SELECT COD_CNL, CN FROM cnl WHERE COD_CNL IN ({placeholders})", pedaco
        )
        for cod, cn in cursor.fetchall():
            mapa[str(cod)] = cn
    cursor.close()

    df["CN"] = df[cand].astype(str).map(mapa).fillna(df.get("CN", "N/A"))
    df["CN"] = df["CN"].astype(str).str.replace(".0", "", regex=False)
    return df


def _coerce_dates(df: pd.DataFrame) -> pd.DataFrame:
    if "Data_Desativacao" in df.columns:
        df["Data_Desativacao"] = df["Data_Desativacao"].apply(iso_date_or_none)
    return df


def _discard_empty_rows(df: pd.DataFrame) -> pd.DataFrame:
    for col in _ESSENCIAIS:
        if col not in df.columns:
            df[col] = "N/A"
    mask = df[_ESSENCIAIS].apply(
        lambda r: all(str(v).strip().lower() in _VAZIOS for v in r), axis=1
    )
    return df[~mask]


def _ensure_unified_cols(df: pd.DataFrame) -> pd.DataFrame:
    for col in UNIFIED_COLS:
        if col not in df.columns:
            df[col] = "N/A"
    return df


# =========================================================
# Construção centralizada do WHERE SQL
# =========================================================

def _build_where(
    cols_validas: set,
    filtros: Dict[str, Any],
    trafego: Optional[str],
    termo_geral: str,
    data_inicio: Optional[str],
    data_fim: Optional[str],
    aplicar_data_sci: bool,
    strict_validation: bool = True,
) -> Tuple[str, List[Any]]:
    where_cols: List[str] = ["1=1"]
    params_cols: List[Any] = []

    for coluna, valor in (filtros or {}).items():
        base = coluna.replace("__inicio", "").replace("__fim", "")
        if base not in cols_validas:
            if strict_validation:
                raise ValueError(f"Coluna inválida: {coluna}")
            continue
        if coluna.endswith("__inicio"):
            where_cols.append(f"date({base}) >= date(?)")
            params_cols.append(valor)
        elif coluna.endswith("__fim"):
            where_cols.append(f"date({base}) <= date(?)")
            params_cols.append(valor)
        else:
            where_cols.append(f"{base} LIKE ?")
            params_cols.append(f"%{valor}%")

    if trafego in ("STFC", "SMP"):
        where_cols.append("Tipo_de_Trafego = ?")
        params_cols.append(trafego)

    if termo_geral:
        where_cols.append(
            "(ID LIKE ? OR EOT LIKE ? OR Operadora LIKE ? OR Descricao LIKE ?)"
        )
        params_cols.extend([f"%{termo_geral}%"] * 4)

    where_cols_sql = " AND ".join(where_cols)

    date_clause = ""
    date_params: List[Any] = []

    if data_inicio or data_fim:
        if aplicar_data_sci:
            sc_parts = ["Portal = 'Science'"]
            if data_inicio:
                sc_parts.append("date(Data_Desativacao) >= date(?)")
                date_params.append(data_inicio)
            if data_fim:
                sc_parts.append("date(Data_Desativacao) <= date(?)")
                date_params.append(data_fim)
            date_clause = (
                "(" + " AND ".join(sc_parts) + ") OR (Portal <> 'Science')"
            )
            date_clause = f"({date_clause})"
        else:
            if data_inicio:
                where_cols_sql += " AND date(Data_Desativacao) >= date(?)"
                params_cols.append(data_inicio)
            if data_fim:
                where_cols_sql += " AND date(Data_Desativacao) <= date(?)"
                params_cols.append(data_fim)

    if date_clause:
        final_where = f"({where_cols_sql}) AND {date_clause}"
        final_params = params_cols + date_params
    else:
        final_where = where_cols_sql
        final_params = params_cols

    return final_where, final_params


# =========================================================
# Agente NLQ
# =========================================================

def _parse_question_pt(question: str) -> Dict[str, Any]:
    q = (question or "").strip().lower()
    intent = "count"
    group_by = None
    limit = None

    m = re.search(r"top\s+(\d+)\s+(operadoras?|eots?|cns?|tr[aá]fegos?)", q)
    if m:
        intent = "top"
        limit = max(1, min(int(m.group(1)), 50))
        alvo = m.group(2)
        group_by = (
            "Operadora" if "operador" in alvo
            else "EOT" if "eot" in alvo
            else "CN" if "cn" in alvo
            else "Tipo_de_Trafego"
        )

    m2 = re.search(
        r"(distribui[cç][aã]o|quebra|por)\s+por\s+(operadora|eot|cn|tr[aá]fego|tipo de tr[aá]fego)", q
    )
    if m2:
        intent = "groupby"
        chave = m2.group(2)
        group_by = (
            "Operadora" if "operadora" in chave
            else "EOT" if "eot" in chave
            else "CN" if "cn" in chave
            else "Tipo_de_Trafego"
        )

    if intent == "count" and re.search(r"\bpor\s+(operadora|eot|cn|tr[aá]fego)\b", q):
        intent = "groupby"
        group_by = (
            "Operadora" if "operadora" in q
            else "EOT" if "eot" in q
            else "CN" if "cn" in q
            else "Tipo_de_Trafego"
        )

    nl_filters: Dict[str, Any] = {}
    if re.search(r"\bstfc\b", q):
        nl_filters["Tipo_de_Trafego"] = "STFC"
    if re.search(r"\bsmp\b", q):
        nl_filters["Tipo_de_Trafego"] = "SMP"

    m3 = re.search(r"operadora\s+([a-z0-9\-\._ ]{2,})", q)
    if m3:
        nl_filters["Operadora"] = m3.group(1).strip().upper()

    m4 = re.search(r"\beot\s+([0-9\-]+)", q)
    if m4:
        nl_filters["EOT"] = m4.group(1).strip()

    m5 = re.search(r"\bcn\s+([0-9\-]+)", q)
    if m5:
        nl_filters["CN"] = m5.group(1).strip()

    m6 = re.search(r"entre\s+(\d{4}-\d{2}-\d{2})\s+e\s+(\d{4}-\d{2}-\d{2})", q)
    if m6:
        nl_filters["Data_Desativacao__inicio"] = m6.group(1)
        nl_filters["Data_Desativacao__fim"] = m6.group(2)

    return {
        "intent":   intent,
        "group_by": group_by if group_by in _GROUPABLE else None,
        "limit":    limit,
        "nl_filters": nl_filters,
    }


def _merge_filters(base: Dict[str, Any], extra: Dict[str, Any]) -> Dict[str, Any]:
    base = dict(base or {})
    f = dict(base.get("filtros_coluna") or {})
    for k, v in (extra or {}).items():
        if k.endswith("__inicio") or k.endswith("__fim") or k in _VALID_FILTERS:
            f[k] = v
    base["filtros_coluna"] = f
    if "Tipo_de_Trafego" in extra:
        base["trafego"] = extra["Tipo_de_Trafego"]
    return base


def _extract_where_from_payload(cur, payload: Dict[str, Any]) -> Tuple[str, List[Any]]:
    cur.execute("PRAGMA table_info(tabela_unificada)")
    cols_validas = {row[1] for row in cur.fetchall()}

    return _build_where(
        cols_validas=cols_validas,
        filtros=payload.get("filtros_coluna") or {},
        trafego=payload.get("trafego"),
        termo_geral=(payload.get("termo_geral") or "").strip(),
        data_inicio=(payload.get("data_inicio") or "").strip() or None,
        data_fim=(payload.get("data_fim") or "").strip() or None,
        aplicar_data_sci=bool(payload.get("aplicar_data_somente_science")),
        strict_validation=False,
    )


# =========================================================
# Views
# =========================================================

@bp.route("/")
def index():
    csrf_token = ensure_csrf_token()
    return render_template("vivoCRC/index.html", csrf_token=csrf_token)


# ─── Upload (CSV + XLSX) ──────────────────────────────────

@bp.route("/api/upload-arquivo", methods=["POST"])
def upload_csv():
    try:
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        file = request.files.get("arquivo")
        origem = (request.form.get("origem") or "").strip().lower()

        if not file or not file.filename or not allowed_file(file.filename):
            return jsonify({"error": "Apenas arquivos .csv ou .xlsx são aceitos."}), 400
        if origem not in ORIGENS_PERMITIDAS:
            return jsonify({"error": "Origem inválida. Use 'science' ou 'portal'."}), 400

        os.makedirs(_upload_folder(), exist_ok=True)
        filename = secure_filename(file.filename)
        filepath = os.path.join(_upload_folder(), f"{uuid4().hex}_{filename}")
        file.save(filepath)

        conn = get_db()
        cur = conn.cursor()
        cur.execute("PRAGMA table_info(tabela_unificada)")
        colunas_tabela = {row[1] for row in cur.fetchall()}
        cur.close()

        cols_insert = [c for c in UNIFIED_COLS if c in colunas_tabela]
        placeholders = ",".join(["?"] * len(cols_insert))
        sql_insert = (
            f"INSERT OR IGNORE INTO tabela_unificada "
            f"({','.join(cols_insert)}) VALUES ({placeholders})"
        )

        total_inseridos = 0

        try:
            for chunk in read_file(filepath):
                if origem == "science":
                    chunk = _map_columns_science(chunk)
                else:
                    chunk = _map_columns_portal(chunk)
                    chunk = _apply_cnl_mapping_portal(chunk, conn)

                chunk = _coerce_dates(chunk)
                chunk = _sanitize_df(chunk)
                chunk = _ensure_unified_cols(chunk)
                chunk = _discard_empty_rows(chunk)

                if chunk.empty:
                    continue

                values = (
                    chunk[cols_insert]
                    .fillna("N/A")
                    .astype(str)
                    .values.tolist()
                )

                cur = conn.cursor()
                for i in range(0, len(values), BULK_INSERT_SIZE):
                    cur.executemany(sql_insert, values[i: i + BULK_INSERT_SIZE])
                    conn.commit()
                    if cur.rowcount and cur.rowcount > 0:
                        total_inseridos += cur.rowcount
                cur.close()

        finally:
            try:
                os.remove(filepath)
            except OSError:
                pass

        return jsonify({"message": f"Processamento concluído. Registros inseridos: {total_inseridos}."})

    except Exception as e:
        traceback.print_exc()
        return jsonify({"error": f"Erro ao processar arquivo: {str(e)}"}), 500


# ─── Consulta paginada ────────────────────────────────────

@bp.route("/api/dados-v2", methods=["POST"])
def api_dados_v2():
    try:
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        payload = request.get_json() or {}
        page      = max(int(payload.get("page", 1)), 1)
        page_size = min(max(int(payload.get("page_size", 100)), 1), 1000)

        conn = get_db()
        cur  = conn.cursor()

        cur.execute("PRAGMA table_info(tabela_unificada)")
        cols_validas = {row[1] for row in cur.fetchall()}

        try:
            where_sql, params = _build_where(
                cols_validas=cols_validas,
                filtros=payload.get("filtros_coluna") or {},
                trafego=payload.get("trafego"),
                termo_geral=(payload.get("termo_geral") or "").strip(),
                data_inicio=(payload.get("data_inicio") or "").strip() or None,
                data_fim=(payload.get("data_fim") or "").strip() or None,
                aplicar_data_sci=bool(payload.get("aplicar_data_somente_science")),
                strict_validation=True,
            )
        except ValueError as ve:
            return jsonify({"error": str(ve)}), 400

        cur.execute(f"SELECT COUNT(*) FROM tabela_unificada WHERE {where_sql}", params)
        total = int(cur.fetchone()[0])

        offset      = (page - 1) * page_size
        select_list = ", ".join([c for c in UNIFIED_COLS if c in cols_validas]) or "*"

        cur.execute(
            f"SELECT {select_list} FROM tabela_unificada "
            f"WHERE {where_sql} ORDER BY rowid DESC LIMIT ? OFFSET ?",
            params + [page_size, offset],
        )
        colunas = [d[0] for d in cur.description]
        dados   = [dict(zip(colunas, r)) for r in cur.fetchall()]

        cur.execute(
            f"SELECT Tipo_de_Trafego, COUNT(*) FROM tabela_unificada "
            f"WHERE {where_sql} GROUP BY Tipo_de_Trafego",
            params,
        )
        resumo = {(k or "N/A"): int(v) for k, v in cur.fetchall()}
        cur.close()

        for item in dados:
            if item.get("Data_Desativacao"):
                item["Data_Desativacao"] = (
                    iso_date_or_none(item["Data_Desativacao"]) or item["Data_Desativacao"]
                )

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


# ─── Exportação adaptativa ────────────────────────────────

@bp.route("/api/exportar", methods=["POST"])
def exportar():
    try:
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        payload    = request.get_json() or {}
        ignorar    = bool(payload.get("ignorar_filtros"))

        conn = get_db()
        cur  = conn.cursor()

        cur.execute("PRAGMA table_info(tabela_unificada)")
        cols_validas_list = [row[1] for row in cur.fetchall()]
        cols_validas      = set(cols_validas_list)

        if not cols_validas_list:
            return jsonify({"error": "Tabela vazia ou inexistente."}), 400

        if ignorar:
            where_sql, params = "1=1", []
        else:
            try:
                where_sql, params = _build_where(
                    cols_validas=cols_validas,
                    filtros=payload.get("filtros_coluna") or {},
                    trafego=payload.get("trafego"),
                    termo_geral=(payload.get("termo_geral") or "").strip(),
                    data_inicio=(payload.get("data_inicio") or "").strip() or None,
                    data_fim=(payload.get("data_fim") or "").strip() or None,
                    aplicar_data_sci=bool(payload.get("aplicar_data_somente_science")),
                    strict_validation=True,
                )
            except ValueError as ve:
                return jsonify({"error": str(ve)}), 400

        cur.execute(f"SELECT COUNT(*) FROM tabela_unificada WHERE {where_sql}", params)
        total = int(cur.fetchone()[0])

        if total == 0:
            def header_only():
                yield "\ufeff".encode("utf-8")
                yield "sep=,\r\n".encode("utf-8")
                yield (",".join(cols_validas_list) + "\r\n").encode("utf-8")

            fn = f"dados_filtrados_{uuid4().hex}.csv"
            return Response(
                header_only(),
                mimetype="text/csv; charset=utf-8",
                headers={"Content-Disposition": f"attachment; filename={fn}"},
            )

        def _write_rows_to_sheet(ws, cursor, cols, max_col_width):
            while True:
                rows = cursor.fetchmany(100_000)
                if not rows:
                    break
                for r in rows:
                    out = []
                    for v, col in zip(r, cols):
                        if col == "Data_Desativacao":
                            v = iso_date_or_none(v) or v
                        out.append(sanitizar_valor(v))
                    ws.append(out)
                    for ci, v in enumerate(out, start=1):
                        l = len(str(v)) if v is not None else 0
                        if l > max_col_width.get(ci, 0):
                            max_col_width[ci] = min(l, 60)

        if total <= EXCEL_MAX_ROWS:
            wb  = Workbook()
            ws  = wb.active
            ws.title = "Dados"
            max_w: Dict[int, int] = {}

            cur2 = conn.cursor()
            cur2.execute(f"SELECT * FROM tabela_unificada WHERE {where_sql}", params)
            cols = [d[0] for d in cur2.description]
            ws.append(cols)
            for ci, c in enumerate(cols, start=1):
                max_w[ci] = len(str(c))

            _write_rows_to_sheet(ws, cur2, cols, max_w)
            cur2.close()

            for ci, w in max_w.items():
                ws.column_dimensions[get_column_letter(ci)].width = w + 2

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

        if total <= EXCEL_MAX_ROWS * MAX_XLSX_SHEETS:
            wb             = Workbook()
            ws             = wb.active
            ws.title       = "Dados 1"
            max_rows_sheet = EXCEL_MAX_ROWS - 1
            sheet_rows     = 0
            sheet_index    = 1
            max_w: Dict[int, int] = {}

            cur2 = conn.cursor()
            cur2.execute(f"SELECT * FROM tabela_unificada WHERE {where_sql}", params)
            cols = [d[0] for d in cur2.description]

            def add_header(_ws):
                _ws.append(cols)
                for ci, c in enumerate(cols, start=1):
                    max_w[ci] = max(max_w.get(ci, 0), len(str(c)))

            add_header(ws)

            while True:
                rows = cur2.fetchmany(100_000)
                if not rows:
                    break
                for r in rows:
                    if sheet_rows >= max_rows_sheet:
                        sheet_index += 1
                        ws = wb.create_sheet(title=f"Dados {sheet_index}")
                        add_header(ws)
                        sheet_rows = 0

                    out = []
                    for v, col in zip(r, cols):
                        if col == "Data_Desativacao":
                            v = iso_date_or_none(v) or v
                        out.append(sanitizar_valor(v))
                    ws.append(out)
                    sheet_rows += 1

                    for ci, v in enumerate(out, start=1):
                        l = len(str(v)) if v is not None else 0
                        if l > max_w.get(ci, 0):
                            max_w[ci] = min(l, 60)

            cur2.close()

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

        def stream_csv():
            CHUNK = 100_000
            cur2  = conn.cursor()
            cur2.execute(f"SELECT * FROM tabela_unificada WHERE {where_sql}", params)
            cols  = [d[0] for d in cur2.description] if cur2.description else cols_validas_list

            yield "\ufeff".encode("utf-8")
            yield "sep=,\r\n".encode("utf-8")
            yield (",".join(cols) + "\r\n").encode("utf-8")

            while True:
                rows = cur2.fetchmany(CHUNK)
                if not rows:
                    break
                lines = []
                for r in rows:
                    vals = []
                    for v in r:
                        if v is None:
                            vals.append("")
                        else:
                            s = str(v).replace("\r", " ").replace("\n", " ")
                            if "," in s or '"' in s:
                                s = '"' + s.replace('"', '""') + '"'
                            vals.append(s)
                    lines.append(",".join(vals))
                yield ("\r\n".join(lines) + "\r\n").encode("utf-8")

            cur2.close()

        fn = f"dados_filtrados_{uuid4().hex}.csv"
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
        csrf_err = validate_csrf_header()
        if csrf_err:
            return csrf_err

        body     = request.get_json(force=True) or {}
        question = body.get("question", "")
        contexto = body.get("contexto") or {}

        parsed   = _parse_question_pt(question)
        contexto = _merge_filters(contexto, parsed.get("nl_filters", {}))

        conn = get_db()
        cur  = conn.cursor()

        where_sql, params = _extract_where_from_payload(cur, contexto)
        intent   = parsed["intent"]
        group_by = parsed.get("group_by")
        limit    = parsed.get("limit") or 10

        if intent == "count" or not group_by:
            sql = f"SELECT COUNT(*) FROM tabela_unificada WHERE {where_sql}"
            cur.execute(sql, params)
            total = int(cur.fetchone()[0])

            cur.execute(
                f"SELECT Tipo_de_Trafego, COUNT(*) FROM tabela_unificada "
                f"WHERE {where_sql} GROUP BY Tipo_de_Trafego",
                params,
            )
            dist   = cur.fetchall()
            labels = [r[0] or "N/A" for r in dist]
            data   = [int(r[1]) for r in dist]
            cur.close()

            resp: Dict[str, Any] = {
                "answer":   f"Encontrei **{total:,}** registros conforme os filtros atuais.".replace(",", "."),
                "sql_used": sql,
            }
            if dist:
                resp["chart"] = {
                    "type": "bar", "labels": labels, "data": data,
                    "title": "Distribuição por Tráfego",
                }
            return jsonify(resp)

        if intent in ("groupby", "top") and group_by in _GROUPABLE:
            lim = f" LIMIT {int(limit)}" if intent == "top" else ""
            sql = (
                f"SELECT {group_by} AS grupo, COUNT(*) AS cnt "
                f"FROM tabela_unificada WHERE {where_sql} "
                f"GROUP BY {group_by} ORDER BY cnt DESC{lim}"
            )
            cur.execute(sql, params)
            rows = cur.fetchall()
            cur.close()

            labels = [(r[0] or "N/A") for r in rows]
            data   = [int(r[1]) for r in rows]
            total  = sum(data)

            answer = (
                f"Top {len(rows)} por **{group_by}** (desc): "
                + ", ".join(f"{labels[i]}: {data[i]:,}" for i in range(len(labels))).replace(",", ".")
                if intent == "top"
                else f"Distribuição por **{group_by}** ({total:,} no total):".replace(",", ".")
            )

            return jsonify({
                "answer":  answer,
                "columns": ["Grupo", "Quantidade"],
                "rows":    [{"Grupo": labels[i], "Quantidade": data[i]} for i in range(len(labels))],
                "chart":   {"type": "bar", "labels": labels, "data": data, "title": f"Por {group_by}"},
                "sql_used": sql,
            })

        cur.close()
        return jsonify({
            "answer": (
                "Consegui aplicar os filtros, mas não entendi o tipo de resultado desejado. "
                "Você pode pedir 'total', 'distribuição por operadora' ou 'top 10 operadoras', por exemplo."
            ),
            "sql_used": "",
        })

    except Exception as e:
        traceback.print_exc()
        return jsonify({"error": f"Erro no agente: {str(e)}"}), 500
