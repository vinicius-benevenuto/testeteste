# app.py
# -*- coding: utf-8 -*-

import os
import json
import sqlite3
from pathlib import Path
from functools import wraps
from datetime import datetime
from io import BytesIO

from flask import (
    Flask, render_template, request, redirect, url_for,
    session, flash, g, abort, send_file, jsonify
)
from werkzeug.security import generate_password_hash, check_password_hash

# =============================================================================
# Excel (openpyxl)
# =============================================================================
try:
    from openpyxl import Workbook
    from openpyxl.styles import Font, Alignment, PatternFill, Border, Side
    from openpyxl.utils import get_column_letter
    from openpyxl.drawing.image import Image as XLImage
except Exception:  # pragma: no cover
    Workbook = None
    XLImage = None

# =============================================================================
# Pillow (PIL) — para escrever rótulos SOBRE a imagem do diagrama
# =============================================================================
try:
    from PIL import Image as PILImage, ImageDraw, ImageFont
    _PIL_AVAILABLE = True
except Exception:  # pragma: no cover
    _PIL_AVAILABLE = False


# =============================================================================
# App & Config
# =============================================================================
app = Flask(__name__, instance_relative_config=True)

# Em produção, configure via variáveis de ambiente
app.config["SECRET_KEY"] = os.environ.get("SECRET_KEY", "dev-secret-change-me")
app.config["DATABASE"] = os.path.join(app.instance_path, "vivohub.db")
ADMIN_CODE = os.environ.get("ADMIN_CODE", "v28B112004")

# Pastas essenciais
Path(app.instance_path).mkdir(parents=True, exist_ok=True)
EXPORT_DIR = os.path.join(app.instance_path, "exports")
Path(EXPORT_DIR).mkdir(parents=True, exist_ok=True)


# =============================================================================
# DB Helpers (SQLite)
# =============================================================================
def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(app.config["DATABASE"])
        g.db.row_factory = sqlite3.Row
        g.db.execute("PRAGMA foreign_keys = ON;")
    return g.db


@app.teardown_appcontext
def close_db(exception=None):  # pragma: no cover
    db = g.pop("db", None)
    if db is not None:
        db.close()


def ensure_column(db, table, column, coldef):
    """Adiciona coluna se não existir (migração leve e segura)."""
    cur = db.execute(f"PRAGMA table_info({table});")
    cols = [r["name"] for r in cur.fetchall()]
    if column not in cols:
        db.execute(f"ALTER TABLE {table} ADD COLUMN {column} {coldef};")


# =========================
# CNs — seed interno (códigos)
# =========================
_CN_SEED_RAW = """
68 82 97 92 96 77 75 74 73 71 88 85 61 27 28 64 62 61 98 99
34 37 31 35 32 38 33 67 66 65 91 94 93 83 81 87 86 89
43 44 45 46 41 24 22 21 84 69 95 51 53 54 55 47 48 49 79
18 14 15 16 13 19 17 11 12 63
""".strip()


def _parse_cn_seed(raw: str):
    seen = set()
    out = []
    for tok in raw.split():
        tok = tok.strip()
        if not tok:
            continue
        if tok.isdigit():
            code = tok.zfill(2)
            if code not in seen:
                seen.add(code)
                out.append(code)
    return out


def _seed_cns_if_needed(db):
    db.execute(
        """
        CREATE TABLE IF NOT EXISTS cns (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            codigo TEXT UNIQUE NOT NULL,
            nome TEXT,
            uf TEXT,
            ativo INTEGER NOT NULL DEFAULT 1
        )
        """
    )
    cur = db.execute("SELECT COUNT(*) AS c FROM cns")
    if (cur.fetchone()["c"] or 0) == 0:
        codes = _parse_cn_seed(_CN_SEED_RAW)
        db.executemany("INSERT OR IGNORE INTO cns (codigo, ativo) VALUES (?, 1)", [(c,) for c in codes])
        db.commit()


# =========================
# CNs — metadados oficiais (cidade principal)
# =========================
_CN_METADATA = {
    # SP
    "11": ("São Paulo", "SP"),
    "12": ("São José dos Campos", "SP"),
    "13": ("Santos", "SP"),
    "14": ("Bauru", "SP"),
    "15": ("Sorocaba", "SP"),
    "16": ("Ribeirão Preto", "SP"),
    "17": ("São José do Rio Preto", "SP"),
    "18": ("Presidente Prudente", "SP"),
    "19": ("Campinas", "SP"),

    # RJ / ES
    "21": ("Rio de Janeiro", "RJ"),
    "22": ("Campos dos Goytacazes", "RJ"),
    "24": ("Volta Redonda", "RJ"),
    "27": ("Vitória", "ES"),
    "28": ("Cachoeiro de Itapemirim", "ES"),

    # MG
    "31": ("Belo Horizonte", "MG"),
    "32": ("Juiz de Fora", "MG"),
    "33": ("Governador Valadares", "MG"),
    "34": ("Uberlândia", "MG"),
    "35": ("Poços de Caldas", "MG"),
    "37": ("Divinópolis", "MG"),
    "38": ("Montes Claros", "MG"),

    # PR / SC / RS
    "41": ("Curitiba", "PR"),
    "42": ("Ponta Grossa", "PR"),
    "43": ("Londrina", "PR"),
    "44": ("Maringá", "PR"),
    "45": ("Foz do Iguaçu", "PR"),
    "46": ("Francisco Beltrão", "PR"),

    "47": ("Joinville", "SC"),
    "48": ("Florianópolis", "SC"),
    "49": ("Chapecó", "SC"),

    "51": ("Porto Alegre", "RS"),
    "53": ("Pelotas", "RS"),
    "54": ("Caxias do Sul", "RS"),
    "55": ("Santa Maria", "RS"),

    # DF / GO / MS / MT / TO
    "61": ("Brasília", "DF"),
    "62": ("Goiânia", "GO"),
    "64": ("Rio Verde", "GO"),
    "63": ("Palmas", "TO"),
    "65": ("Cuiabá", "MT"),
    "66": ("Rondonópolis", "MT"),
    "67": ("Campo Grande", "MS"),

    # BA / SE / AL / PE / PB / RN / CE / PI / MA
    "71": ("Salvador", "BA"),
    "73": ("Ilhéus", "BA"),
    "74": ("Juazeiro", "BA"),
    "75": ("Feira de Santana", "BA"),
    "77": ("Vitória da Conquista", "BA"),

    "79": ("Aracaju", "SE"),

    "82": ("Maceió", "AL"),

    "81": ("Recife", "PE"),
    "87": ("Petrolina", "PE"),

    "83": ("João Pessoa", "PB"),

    "84": ("Natal", "RN"),

    "85": ("Fortaleza", "CE"),
    "88": ("Juazeiro do Norte", "CE"),

    "86": ("Teresina", "PI"),
    "89": ("Picos", "PI"),

    "98": ("São Luís", "MA"),
    "99": ("Imperatriz", "MA"),

    # PA / AP / AM / RR / AC / RO
    "91": ("Belém", "PA"),
    "93": ("Santarém", "PA"),
    "94": ("Marabá", "PA"),

    "96": ("Macapá", "AP"),

    "92": ("Manaus", "AM"),
    "97": ("Coari", "AM"),

    "95": ("Boa Vista", "RR"),

    "68": ("Rio Branco", "AC"),

    "69": ("Porto Velho", "RO"),

    # Não utilizado
    "72": ("—", ""),
}


def _apply_cn_metadata(db):
    """
    Atualiza/insere nomes/UF para todos os CNs conhecidos.
    - Se o CN já existir, atualiza nome/uf mantendo ativo=1.
    - Se não existir, insere com ativo=1.
    """
    db.execute(
        """
        CREATE TABLE IF NOT EXISTS cns (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            codigo TEXT UNIQUE NOT NULL,
            nome TEXT,
            uf TEXT,
            ativo INTEGER NOT NULL DEFAULT 1
        )
        """
    )
    for code, (nome, uf) in _CN_METADATA.items():
        db.execute(
            """
            INSERT INTO cns (codigo, nome, uf, ativo)
            VALUES (?, ?, ?, 1)
            ON CONFLICT(codigo) DO UPDATE SET
                nome=excluded.nome,
                uf=excluded.uf,
                ativo=1
            """,
            (code, nome, uf)
        )
    db.commit()


def init_db():
    """Cria/atualiza o schema de forma idempotente."""
    db = get_db()
    db.executescript(
        """
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            role TEXT NOT NULL CHECK (role IN ('engenharia','atacado')),
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS atacado_forms (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            owner_id INTEGER NOT NULL,

            status TEXT DEFAULT 'rascunho',

            -- 1) Identificação
            nome_operadora TEXT,
            rn1 TEXT,
            csp INTEGER DEFAULT 0,
            servicos_especiais INTEGER DEFAULT 0,
            cng INTEGER DEFAULT 0,
            atendimento TEXT,
            redes TEXT,
            qual TEXT,
            tmr TEXT,

            -- 2) Contatos/Responsáveis
            responsavel_operadora TEXT,
            responsavel_vivo TEXT,
            sbc_ativo INTEGER DEFAULT 0,
            ip_reservado INTEGER DEFAULT 0,
            vivo_reserva INTEGER DEFAULT 0,
            asn TEXT,

            -- 3) Escopo (Atacado) / Dados VIVO (Engenharia)
            escopo_text TEXT,
            escopo_flags_json TEXT,
            dados_vivo_json TEXT,

            -- 4) Dados Operadora
            dados_operadora_json TEXT,

            -- 5/6/7/8
            operadora_ciente INTEGER DEFAULT 0,
            responsavel_infra TEXT,
            lcr_nacional INTEGER DEFAULT 0,
            white_list INTEGER DEFAULT 0,
            prefixos_liberados_abr INTEGER DEFAULT 0,
            premissas_ok INTEGER DEFAULT 0,
            aprovado_por TEXT,

            -- 9) Engenharia
            engenharia_params_json TEXT,

            -- Responsáveis (novos)
            responsavel_atacado TEXT,
            responsavel_engenharia TEXT,

            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

            FOREIGN KEY(owner_id) REFERENCES users(id)
        );

        CREATE TABLE IF NOT EXISTS exports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            form_id INTEGER NOT NULL,
            filename TEXT NOT NULL,
            filepath TEXT NOT NULL,
            size_bytes INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(form_id) REFERENCES atacado_forms(id)
        );
        """
    )

    # Migrações leves — garante colunas importantes caso schema antigo exista
    ensure_column(db, "atacado_forms", "owner_id", "INTEGER NOT NULL DEFAULT 0")
    ensure_column(db, "atacado_forms", "engenharia_params_json", "TEXT")
    ensure_column(db, "atacado_forms", "dados_vivo_json", "TEXT")
    ensure_column(db, "atacado_forms", "dados_operadora_json", "TEXT")
    ensure_column(db, "atacado_forms", "escopo_text", "TEXT")
    ensure_column(db, "atacado_forms", "escopo_flags_json", "TEXT")
    ensure_column(db, "atacado_forms", "responsavel_atacado", "TEXT")
    ensure_column(db, "atacado_forms", "responsavel_engenharia", "TEXT")
    db.commit()

    _seed_cns_if_needed(db)
    _apply_cn_metadata(db)


@app.before_request
def _ensure_db():
    init_db()


@app.after_request
def _security_headers(resp):  # pragma: no cover
    # Cabeçalhos básicos de segurança
    resp.headers.setdefault("X-Content-Type-Options", "nosniff")
    resp.headers.setdefault("X-Frame-Options", "SAMEORIGIN")
    resp.headers.setdefault("X-XSS-Protection", "0")
    return resp


# =============================================================================
# Decorators (auth)
# =============================================================================
def login_required(view):
    @wraps(view)
    def wrapped(*args, **kwargs):
        if "user_id" not in session:
            return redirect(url_for("login"))
        return view(*args, **kwargs)
    return wrapped


def role_required(required_role):
    def decorator(view):
        @wraps(view)
        def wrapped(*args, **kwargs):
            if "user_id" not in session:
                return redirect(url_for("login"))
            if session.get("role") != required_role:
                flash("Acesso negado para esta área.", "danger")
                return redirect(url_for(f"central_{session.get('role')}"))
            return view(*args, **kwargs)
        return wrapped
    return decorator


def admin_required(view):
    @wraps(view)
    def wrapped(*args, **kwargs):
        if not session.get("is_admin"):
            flash("Área restrita ao administrador.", "danger")
            return redirect(url_for("admin_login"))
        return view(*args, **kwargs)
    return wrapped


# =============================================================================
# Helpers de formulário e utilidades
# =============================================================================

# Conjunto permitido para os checkboxes de "3) Escopo"
SCOPE_FLAGS_ALLOWED = {
    "LC", "LD15 + CNG", "LDS/CSP + CNG", "Transporte", "VC1", "Concentração"
}

# Campos booleanos/toggle
BOOL_FIELDS = [
    "csp", "servicos_especiais", "cng",
    "sbc_ativo", "ip_reservado", "vivo_reserva",
    "operadora_ciente", "lcr_nacional", "white_list",
    "prefixos_liberados_abr", "premissas_ok"
]

# Campos texto simples
TEXT_FIELDS = [
    "nome_operadora", "rn1", "atendimento", "redes", "qual", "tmr",
    "responsavel_operadora", "responsavel_vivo", "asn",
    "responsavel_infra", "aprovado_por",
    "status",
    "escopo_text",
    "responsavel_atacado", "responsavel_engenharia",
]

# Campos em JSON (guardados como TEXT)
JSON_FIELDS = [
    "escopo_flags_json", "dados_vivo_json", "dados_operadora_json", "engenharia_params_json",
]

# Limite das tabelas
MAX_TABELA_ROWS = 10


def _truncate_rows_list(raw_text, default="[]"):
    """
    Recebe string JSON de uma lista; garante lista e trunca para MAX_TABELA_ROWS.
    Retorna string JSON válida.
    """
    if not raw_text:
        return default
    try:
        obj = json.loads(raw_text) if isinstance(raw_text, str) else raw_text
        if not isinstance(obj, list):
            obj = []
    except Exception:
        obj = []
    if len(obj) > MAX_TABELA_ROWS:
        obj = obj[:MAX_TABELA_ROWS]
    return json.dumps(obj, ensure_ascii=False)


def _clean_scope_flags_from_request():
    """
    Lê os checkboxes 'escopo_flags' do request (getlist) e higieniza
    contra valores fora do conjunto permitido. Retorna JSON string.
    """
    items = request.form.getlist("escopo_flags") or []
    clean = [x for x in items if x in SCOPE_FLAGS_ALLOWED]
    # remove duplicatas preservando ordem
    seen = set()
    unique = []
    for x in clean:
        if x not in seen:
            seen.add(x)
            unique.append(x)
    return json.dumps(unique, ensure_ascii=False)


def form_payload_from_request():
    """Extrai e normaliza dados vindos do HTML (Atacado OU Engenharia)."""
    payload = {}

    # Textos
    for k in TEXT_FIELDS:
        payload[k] = (request.form.get(k) or "").strip()

    # Bools
    for k in BOOL_FIELDS:
        payload[k] = 1 if request.form.get(k) in ("on", "1", "true", "True") else 0

    # escopo_flags via checkboxes
    payload["escopo_flags_json"] = _clean_scope_flags_from_request()

    # JSONs
    for k in JSON_FIELDS:
        if k == "escopo_flags_json":
            continue
        raw = request.form.get(k, "")
        if k == "engenharia_params_json":
            if not raw:
                payload[k] = "{}"
            else:
                try:
                    obj = raw if isinstance(raw, (list, dict)) else json.loads(raw)
                    if isinstance(obj, dict):
                        payload[k] = json.dumps(obj, ensure_ascii=False)
                    else:
                        payload[k] = "{}"
                except Exception:
                    payload[k] = "{}"
        else:
            payload[k] = _truncate_rows_list(raw, "[]")

    return payload


def _safe_name(s: str) -> str:
    """Nome seguro para arquivo."""
    s = (s or "").strip().replace(" ", "_")
    for bad in ("..", "/", "\\", ":"):
        s = s.replace(bad, "_")
    return s or "Operadora"


def _parse_db_datetime(s: str):
    """
    Converte timestamp em datetime (aceita 'YYYY-MM-DD HH:MM:SS' ou ISO com 'T').
    """
    if not s:
        return None
    try:
        return datetime.fromisoformat(str(s).replace("T", " "))
    except Exception:
        return None


def row_get(row, key, default=""):
    """
    Acesso seguro a sqlite3.Row com fallback (evita .get inexistente).
    """
    try:
        v = row[key]
        return default if v is None else v
    except Exception:
        return default


# =============================================================================
# Contexto Jinja — injeta CNs ativas (para futuros selects/datalists)
# =============================================================================
@app.context_processor
def inject_cns():
    db = get_db()
    rows = db.execute(
        "SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf "
        "FROM cns WHERE ativo = 1 ORDER BY codigo ASC"
    ).fetchall()
    return {
        "CN_CODES": [r["codigo"] for r in rows],
        "CN_FULL": [dict(codigo=r["codigo"], nome=r["nome"], uf=r["uf"]) for r in rows],
    }


# =============================================================================
# Rotas públicas / sessão
# =============================================================================
@app.get("/")
def index():
    if "user_id" in session:
        return redirect(url_for(f"central_{session.get('role')}"))
    return redirect(url_for("login"))


@app.get("/login")
def login():
    return render_template("login.html")


@app.post("/login")
def login_post():
    email = (request.form.get("email") or "").strip().lower()
    password = request.form.get("password", "")

    db = get_db()
    user = db.execute("SELECT * FROM users WHERE email = ?", (email,)).fetchone()
    if not user or not check_password_hash(user["password_hash"], password):
        flash("Credenciais inválidas.", "danger")
        return redirect(url_for("login"))

    session.clear()
    session["user_id"] = user["id"]
    session["email"] = user["email"]
    session["role"] = user["role"]
    return redirect(url_for(f"central_{user['role']}"))


@app.get("/logout")
def logout():
    session.clear()
    flash("Sessão encerrada.", "info")
    return redirect(url_for("login"))


# =============================================================================
# Admin (registro protegido por código)
# =============================================================================
@app.get("/admin_login")
def admin_login():
    return render_template("admin_login.html")


@app.post("/admin_login")
def admin_login_post():
    code = request.form.get("code", "")
    if code == ADMIN_CODE:
        session["is_admin"] = True
        flash("Login de administrador efetuado.", "success")
        return redirect(url_for("register"))
    flash("Código de administrador inválido.", "danger")
    return redirect(url_for("admin_login"))


@app.get("/register")
@admin_required
def register():
    return render_template("register.html")


@app.post("/register")
@admin_required
def register_post():
    email = (request.form.get("email") or "").strip().lower()
    password = request.form.get("password", "")
    role = (request.form.get("role") or "").strip().lower()

    if not email or not password or role not in ("engenharia", "atacado"):
        flash("Preencha todos os campos corretamente.", "danger")
        return redirect(url_for("register"))

    db = get_db()
    try:
        db.execute(
            "INSERT INTO users (email, password_hash, role) VALUES (?, ?, ?)",
            (email, generate_password_hash(password), role),
        )
        db.commit()
        flash("Usuário criado com sucesso.", "success")
    except sqlite3.IntegrityError:
        flash("Este e-mail já está cadastrado.", "warning")

    return redirect(url_for("register"))


# =============================================================================
# Centrais
# =============================================================================
@app.get("/central_atacado")
@login_required
@role_required("atacado")
def central_atacado():
    return render_template("central_atacado.html")


@app.get("/central_engenharia")
@login_required
@role_required("engenharia")
def central_engenharia():
    return render_template("central_engenharia.html")


# =============================================================================
# Utilitários de filtro/ordenação para as listas
# =============================================================================
def _build_filters(base_sql, params, q=None, status=None, owner_id=None, sort="-created_at"):
    where = []
    if owner_id is not None:
        where.append("owner_id = ?")
        params.append(owner_id)
    if q:
        where.append("(nome_operadora LIKE ?)")
        params.append(f"%{q}%")
    if status:
        where.append("LOWER(COALESCE(status,'')) = LOWER(?)")
        params.append(status)

    if where:
        base_sql += " WHERE " + " AND ".join(where)

    sort_map = {
        "created_at": "created_at ASC",
        "-created_at": "created_at DESC",
        "nome_operadora": "nome_operadora COLLATE NOCASE ASC",
        "-nome_operadora": "nome_operadora COLLATE NOCASE DESC",
        "id": "id ASC",
        "-id": "id DESC",
    }
    base_sql += " ORDER BY " + sort_map.get(sort, "id DESC")
    return base_sql, params


def _status_counters(db, extra_where="", extra_params=()):
    total = db.execute(f"SELECT COUNT(*) c FROM atacado_forms {extra_where}", extra_params).fetchone()["c"]
    rasc = db.execute(
        f"SELECT COUNT(*) c FROM atacado_forms {extra_where} "
        f"{'AND' if 'WHERE' in extra_where else 'WHERE'} LOWER(status)='rascunho'",
        extra_params
    ).fetchone()["c"]
    envi = db.execute(
        f"SELECT COUNT(*) c FROM atacado_forms {extra_where} "
        f"{'AND' if 'WHERE' in extra_where else 'WHERE'} LOWER(status)='enviado'",
        extra_params
    ).fetchone()["c"]
    emrev = db.execute(
        f"SELECT COUNT(*) c FROM atacado_forms {extra_where} "
        f"{'AND' if 'WHERE' in extra_where else 'WHERE'} LOWER(status)='em revisão'",
        extra_params
    ).fetchone()["c"]
    aprov = db.execute(
        f"SELECT COUNT(*) c FROM atacado_forms {extra_where} "
        f"{'AND' if 'WHERE' in extra_where else 'WHERE'} LOWER(status)='aprovado'",
        extra_params
    ).fetchone()["c"]
    return {
        "total": total,
        "rascunho": rasc,
        "enviado": envi,
        "em_revisao": emrev,
        "aprovado": aprov
    }


# =============================================================================
# Atacado - Formulários (CRUD)
# =============================================================================
@app.get("/atacado_formularios")
@login_required
@role_required("atacado")
def atacado_form_list():
    db = get_db()
    q = (request.args.get("q") or "").strip()
    status = (request.args.get("status") or "").strip()
    sort = (request.args.get("sort") or "-created_at").strip()

    params = []
    sql, params = _build_filters(
        "SELECT * FROM atacado_forms", params,
        q=q or None, status=status or None, owner_id=session["user_id"], sort=sort
    )
    forms = db.execute(sql, params).fetchall()

    counters = _status_counters(db, extra_where="WHERE owner_id = ?", extra_params=(session["user_id"],))
    return render_template("atacado_formularios.html", forms=forms, counters=counters)


@app.get("/atacado_formularios/new")
@login_required
@role_required("atacado")
def atacado_form_new():
    db = get_db()
    last = db.execute(
        "SELECT responsavel_atacado FROM atacado_forms WHERE owner_id = ? "
        "AND COALESCE(responsavel_atacado,'')<>'' "
        "ORDER BY created_at DESC LIMIT 1",
        (session["user_id"],)
    ).fetchone()
    preset = (last["responsavel_atacado"] if last else "") or (
        session.get("email", "").split("@")[0].replace(".", " ").title()
    )
    return render_template("formulario_atacado.html", form=None, preset_responsavel_atacado=preset)


@app.post("/atacado_formularios/new")
@login_required
@role_required("atacado")
def atacado_form_create():
    payload = form_payload_from_request()
    payload["owner_id"] = session["user_id"]
    payload["status"] = (payload.get("status") or "rascunho").lower()
    payload["engenharia_params_json"] = "{}"

    # Defesa extra (limite de linhas)
    too_many = False
    for key in ("dados_vivo_json", "dados_operadora_json"):
        try:
            arr = json.loads(payload[key])
            if len(arr) > MAX_TABELA_ROWS:
                arr = arr[:MAX_TABELA_ROWS]
                payload[key] = json.dumps(arr, ensure_ascii=False)
                too_many = True
        except Exception:
            payload[key] = "[]"

    # Regras: finalizar exige responsável do atacado
    if payload["status"] == "enviado" and not (payload.get("responsavel_atacado") or "").strip():
        flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.', "warning")
        return redirect(url_for("atacado_form_new"))

    cols = (
        ["owner_id"]
        + TEXT_FIELDS
        + BOOL_FIELDS
        + ["escopo_flags_json", "dados_vivo_json", "dados_operadora_json", "engenharia_params_json"]
    )
    placeholders = ",".join(["?"] * len(cols))
    sql = f"INSERT INTO atacado_forms ({','.join(cols)}) VALUES ({placeholders})"
    db = get_db()
    db.execute(sql, [payload.get(c) for c in cols])
    db.commit()

    flash("Formulário criado.", "success" if not too_many else f"As tabelas foram limitadas a {MAX_TABELA_ROWS} linhas por seção.")
    return redirect(url_for("atacado_form_list"))


@app.get("/atacado_formularios/<int:form_id>")
@login_required
@role_required("atacado")
def atacado_form_edit(form_id: int):
    db = get_db()
    form = db.execute(
        "SELECT * FROM atacado_forms WHERE id = ? AND owner_id = ?",
        (form_id, session["user_id"]),
    ).fetchone()
    if not form:
        abort(404)
    return render_template("formulario_atacado.html", form=form)


@app.post("/atacado_formularios/<int:form_id>")
@login_required
@role_required("atacado")
def atacado_form_update(form_id: int):
    db = get_db()
    exists = db.execute(
        "SELECT id FROM atacado_forms WHERE id = ? AND owner_id = ?",
        (form_id, session["user_id"]),
    ).fetchone()
    if not exists:
        abort(404)

    payload = form_payload_from_request()
    payload["updated_at"] = datetime.utcnow().isoformat(timespec="seconds")
    new_status = (payload.get("status") or "").lower()

    # Regras: finalizar exige responsável do atacado
    if new_status == "enviado" and not (payload.get("responsavel_atacado") or "").strip():
        flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.', "warning")
        return redirect(url_for("atacado_form_edit", form_id=form_id))

    set_fields = TEXT_FIELDS + BOOL_FIELDS + ["escopo_flags_json", "dados_vivo_json", "dados_operadora_json"]
    set_parts = [f"{k} = ?" for k in set_fields]
    set_parts.append("updated_at = ?")

    too_many = False
    for key in ("dados_vivo_json", "dados_operadora_json"):
        try:
            arr = json.loads(payload[key])
            if len(arr) > MAX_TABELA_ROWS:
                arr = arr[:MAX_TABELA_ROWS]
                payload[key] = json.dumps(arr, ensure_ascii=False)
                too_many = True
        except Exception:
            payload[key] = "[]"

    sql = f"""
        UPDATE atacado_forms
           SET {", ".join(set_parts)}
         WHERE id = ?
           AND owner_id = ?
    """
    params = [payload.get(k) for k in set_fields]
    params.append(payload["updated_at"])
    params.extend([form_id, session["user_id"]])

    db.execute(sql, params)
    db.commit()

    flash("Formulário atualizado.", "success" if not too_many else f"As tabelas foram limitadas a {MAX_TABELA_ROWS} linhas por seção.")
    return redirect(url_for("atacado_form_list"))


@app.post("/atacado_formularios/<int:form_id>/delete")
@login_required
@role_required("atacado")
def atacado_form_delete(form_id: int):
    db = get_db()
    db.execute("DELETE FROM atacado_forms WHERE id = ? AND owner_id = ?", (form_id, session["user_id"]))
    db.commit()
    flash("Formulário excluído.", "info")
    return redirect(url_for("atacado_form_list"))


# =============================================================================
# Engenharia - Formulários (lista + abrir/atualizar Seção 9)
# =============================================================================
@app.get("/engenharia_formularios")
@login_required
@role_required("engenharia")
def engenharia_form_list():
    db = get_db()
    q = (request.args.get("q") or "").strip()
    status = (request.args.get("status") or "").strip()
    sort = (request.args.get("sort") or "-created_at").strip()

    params = []
    sql, params = _build_filters(
        "SELECT * FROM atacado_forms", params,
        q=q or None, status=status or None, owner_id=None, sort=sort
    )
    forms = db.execute(sql, params).fetchall()

    counters = _status_counters(db)

    # (Opcional) carregar exports persistidos – útil se quiser mostrar uma "biblioteca"
    form_filter = request.args.get("form")
    if form_filter and form_filter.isdigit():
        exports = db.execute(
            "SELECT e.id, e.form_id, e.filename, e.size_bytes, e.created_at, f.nome_operadora "
            "FROM exports e JOIN atacado_forms f ON f.id = e.form_id "
            "WHERE e.form_id = ? ORDER BY e.created_at DESC LIMIT 100",
            (int(form_filter),)
        ).fetchall()
    else:
        exports = db.execute(
            "SELECT e.id, e.form_id, e.filename, e.size_bytes, e.created_at, f.nome_operadora "
            "FROM exports e JOIN atacado_forms f ON f.id = e.form_id "
            "ORDER BY e.created_at DESC LIMIT 100"
        ).fetchall()

    show_files = request.args.get("show_files") == "1"

    return render_template(
        "engenharia_formularios.html",
        forms=forms, counters=counters,
        exports=exports, show_files=show_files, form_filter=form_filter
    )


@app.route("/engenharia_formularios/<int:form_id>", methods=["GET", "POST"])
@login_required
@role_required("engenharia")
def engenharia_form_view(form_id: int):
    db = get_db()
    form = db.execute("SELECT * FROM atacado_forms WHERE id = ?", (form_id,)).fetchone()
    if not form:
        abort(404)

    if request.method == "POST":
        # Responsável Eng de ITX é obrigatório p/ salvar a validação
        resp_eng = (request.form.get("responsavel_engenharia") or "").strip()
        if not resp_eng:
            flash('Informe o "Responsável Eng de ITX" para salvar a validação.', "warning")
            return redirect(url_for("engenharia_form_view", form_id=form_id))

        raw = request.form.get("engenharia_params_json", "") or "{}"
        try:
            obj = raw if isinstance(raw, dict) else json.loads(raw)
            eng_json = json.dumps(obj, ensure_ascii=False)
        except Exception:
            eng_json = "{}"

        updated_at = datetime.utcnow().isoformat(timespec="seconds")
        db.execute(
            "UPDATE atacado_forms SET engenharia_params_json = ?, responsavel_engenharia = ?, updated_at = ? WHERE id = ?",
            (eng_json, resp_eng, updated_at, form_id),
        )
        db.commit()
        flash("Validação da Engenharia salva.", "success")
        return redirect(url_for("engenharia_form_list", show_files="1", form=form_id))

    # GET — renderiza o mesmo template do Atacado, mas readonly geral; Seção 9 habilita via template
    return render_template("formulario_atacado.html", form=form, readonly=True)


# =============================================================================
# Excel — Helpers de imagem (PIL)
# =============================================================================
def _parse_xy_pct_str(s, default_tuple):
    """
    Converte string '0.18,0.55' em tupla (0.18, 0.55). Se inválida, retorna default_tuple.
    """
    try:
        if not s:
            return default_tuple
        parts = [p.strip() for p in str(s).split(",")]
        if len(parts) != 2:
            return default_tuple
        x = float(parts[0])
        y = float(parts[1])
        return (max(0.0, min(1.0, x)), max(0.0, min(1.0, y)))
    except Exception:
        return default_tuple


def _render_labels_on_image(
    image_path: str,
    vivo_text: str,
    op_text: str,
    *,
    vivo_xy_pct=(0.18, 0.55),       # posição do "VIVO"
    op_xy_pct=(0.80, 0.55),         # posição do nome da operadora
    font_path: str = None,          # TTF opcional
    font_size_pct: float = 0.06,    # tamanho relativo à ALTURA da imagem (ex.: 0.06 = 6%)
    fill=(255, 255, 255, 255),      # cor do texto (RGBA)
    stroke_fill=(0, 0, 0, 255),     # cor da borda (RGBA)
    stroke_width_pct: float = 0.012,  # espessura da borda como % da ALTURA (ex.: 0.012 = 1.2%)
    extra_labels=None               # lista opcional de rótulos extras com overrides
) -> str:
    """
    Gera uma cópia temporária de image_path com textos desenhados e retorna o caminho do .png gerado.
    Requer Pillow.

    extra_labels: [
        {
            "text": "Roteador OP",
            "xy_pct": (0.64, 0.57),
            "font_size_pct": 0.010,
            "fill": (255,255,255,255),              # opcional
            "stroke_fill": (0,0,0,255),             # opcional
            "stroke_width_pct": 0.0,                # opcional (0.0 remove a borda)
            "anchor": "mm"                          # opcional, default = "mm"
        },
        ...
    ]
    """
    if not _PIL_AVAILABLE:
        raise RuntimeError("Pillow (PIL) não está instalado. Instale com: pip install pillow")

    import tempfile

    img = PILImage.open(image_path).convert("RGBA")
    W, H = img.size
    draw = ImageDraw.Draw(img)

    def _pick_font(size_px: int):
        f = None
        if font_path:
            try:
                f = ImageFont.truetype(font_path, size_px)
            except Exception:
                f = None
        if f is None:
            candidates = [
                "C:\\Windows\\Fonts\\arial.ttf",
                "/System/Library/Fonts/Supplemental/Arial.ttf",
                "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
            ]
            for cand in candidates:
                try:
                    f = ImageFont.truetype(cand, size_px)
                    break
                except Exception:
                    continue
        if f is None:
            f = ImageFont.load_default()
        return f

    def _draw_label(text: str, xy_pct, size_pct: float, *, fill_, stroke_fill_, stroke_w_pct: float, anchor_="mm"):
        if not text:
            return
        x = int(W * float(xy_pct[0]))
        y = int(H * float(xy_pct[1]))
        font_px = max(12, int(H * float(size_pct)))
        stroke_w = max(0, int(H * float(stroke_w_pct)))  # permite 0 p/ remover borda
        font = _pick_font(font_px)
        draw.text(
            (x, y),
            text,
            font=font,
            fill=fill_,
            stroke_width=stroke_w,
            stroke_fill=stroke_fill_,
            anchor=anchor_
        )

    # principais
    _draw_label(
        vivo_text, vivo_xy_pct, font_size_pct,
        fill_=fill, stroke_fill_=stroke_fill, stroke_w_pct=stroke_width_pct
    )
    _draw_label(
        op_text, op_xy_pct, font_size_pct,
        fill_=fill, stroke_fill_=stroke_fill, stroke_w_pct=stroke_width_pct
    )

    # extras (ex.: dois "Roteador OP", com opção de tirar a borda)
    if extra_labels:
        for lbl in extra_labels:
            txt = lbl.get("text", "")
            xy = lbl.get("xy_pct", (0.5, 0.5))
            sz = float(lbl.get("font_size_pct", font_size_pct))
            fill_ex = lbl.get("fill", fill)
            stroke_fill_ex = lbl.get("stroke_fill", stroke_fill)
            stroke_w_ex = float(lbl.get("stroke_width_pct", stroke_width_pct))
            anchor_ex = lbl.get("anchor", "mm")
            _draw_label(
                txt, xy, sz,
                fill_=fill_ex, stroke_fill_=stroke_fill_ex, stroke_w_pct=stroke_w_ex, anchor_=anchor_ex
            )

    fd, tmp_path = tempfile.mkstemp(prefix="vivohub_diagrama_", suffix=".png")
    os.close(fd)
    img.save(tmp_path, format="PNG")
    return tmp_path


def _fit_size_keep_aspect(orig_w: int, orig_h: int, max_w: int, max_h: int):
    """
    Retorna (new_w, new_h) mantendo o aspect ratio dentro de um bounding box (max_w x max_h).
    """
    if orig_w <= 0 or orig_h <= 0:
        return max_w, max_h
    scale = min(max_w / float(orig_w), max_h / float(orig_h))
    return int(orig_w * scale), int(orig_h * scale)


def build_index_excel_wb(form_row):
    """
    Gera um Workbook Excel padronizado com nove abas para o Projeto Técnico de Interligação.
    """
    # =========================================================================
    # VALIDAÇÃO CRÍTICA DE DEPENDÊNCIAS
    # =========================================================================
    if Workbook is None:
        raise RuntimeError(
            "Biblioteca 'openpyxl' não está disponível. "
            "Instale com: pip install openpyxl"
        )
    if not _PIL_AVAILABLE:
        raise RuntimeError(
            "Biblioteca 'Pillow' (PIL) não está disponível para processamento de imagens. "
            "Instale com: pip install pillow"
        )

    # =========================================================================
    # FUNÇÕES AUXILIARES LOCAIS - ESCOPO RESTRITO
    # =========================================================================
    def _col(n: int) -> str:
        """Converte índice numérico para referência de coluna Excel (A, B, C, ...)."""
        return get_column_letter(n)

    def _int_fallback(value, default: int) -> int:
        """
        Converte valor para inteiro com fallback robusto.
        """
        try:
            return int(float(value))
        except (ValueError, TypeError, OverflowError):
            return default

    def _extract_traffic_types():
        """
        Extrai e valida tipos de tráfego selecionados no formulário.
        Retorna lista com até 3 tipos de tráfego para preencher as tabelas.
        """
        raw_data = row_get(form_row, "escopo_flags_json", "[]")
        
        if not raw_data or raw_data == "[]":
            return ["LC", "LD15 + CNG", ""]  # Valores padrão se não houver seleção
        
        try:
            # Parse do JSON
            if isinstance(raw_data, str):
                parsed_data = json.loads(raw_data)
            else:
                parsed_data = raw_data
                
            if not isinstance(parsed_data, list):
                return ["LC", "LD15 + CNG", ""]
                
            # Filtrar e limpar os itens
            valid_items = []
            for item in parsed_data:
                if item and str(item).strip():
                    cleaned_item = str(item).strip()
                    valid_items.append(cleaned_item)
                    
            # Garantir que temos pelo menos 3 itens (preenchendo com vazio se necessário)
            while len(valid_items) < 3:
                valid_items.append("")
                
            return valid_items[:3]  # Retorna no máximo 3 itens
            
        except json.JSONDecodeError:
            return ["LC", "LD15 + CNG", ""]
        except Exception:
            return ["LC", "LD15 + CNG", ""]

    def _get_concentracao_data():
        """
        Extrai dados de concentração do campo dados_operadora_json.
        """
        raw_data = row_get(form_row, "dados_operadora_json", "[]")
        
        try:
            if isinstance(raw_data, str):
                data = json.loads(raw_data)
            else:
                data = raw_data
                
            if not isinstance(data, list):
                return []
                
            # Limpar e validar os dados
            clean_data = []
            for item in data:
                if isinstance(item, dict):
                    clean_item = {
                        "ref": str(item.get("ref", "")).strip(),
                        "cn": str(item.get("cn", "")).strip(),
                        "cnl": str(item.get("cnl", "")).strip(),
                        "cod_cni": str(item.get("cod_cni", "")).strip(),
                        "status_link": str(item.get("status_link", "")).strip()
                    }
                    # Só incluir se tiver pelo menos um campo preenchido
                    if any(clean_item.values()):
                        clean_data.append(clean_item)
                        
            return clean_data[:10]  # Limitar a 10 linhas
            
        except Exception:
            return []

    def _get_vivo_data():
        """
        Extrai dados VIVO do campo dados_vivo_json.
        """
        raw_data = row_get(form_row, "dados_vivo_json", "[]")
        
        try:
            if isinstance(raw_data, str):
                data = json.loads(raw_data)
            else:
                data = raw_data
                
            if not isinstance(data, list):
                return []
                
            return data[:10]  # Limitar a 10 linhas
            
        except Exception:
            return []

    # =========================================================================
    # PALETA DE CORES ROXO PÚRPURA - CORRIGIDA
    # =========================================================================
    # Cores principais da paleta roxo púrpura (usando PatternFill corretamente)
    fill_header_primary = PatternFill("solid", fgColor="4A148C")      # Roxo púrpura principal
    fill_header_secondary = PatternFill("solid", fgColor="6A1B9A")    # Roxo secundário
    fill_header_light = PatternFill("solid", fgColor="7B1FA2")        # Roxo claro
    fill_accent = PatternFill("solid", fgColor="8E24AA")              # Roxo acento
    fill_neutral = PatternFill("solid", fgColor="E1BEE7")             # Roxo neutro claro
    fill_background = PatternFill("solid", fgColor="F3E5F5")          # Fundo roxo muito claro
    
    # Cores de destaque e status
    fill_yellow = PatternFill("solid", fgColor="FFF9C4")              # Amarelo para campos editáveis
    fill_orange = PatternFill("solid", fgColor="FFAB91")              # Laranja para alertas
    fill_green = PatternFill("solid", fgColor="C8E6C9")               # Verde suave para sucesso
    fill_blue = PatternFill("solid", fgColor="B3E5FC")                # Azul claro para informações
    
    # Cores neutras
    fill_gray_light = PatternFill("solid", fgColor="F5F5F5")          # Cinza claro
    fill_gray_medium = PatternFill("solid", fgColor="E0E0E0")         # Cinza médio
    fill_gray_dark = PatternFill("solid", fgColor="616161")           # Cinza escuro
    fill_white = PatternFill("solid", fgColor="FFFFFF")               # Branco

    # =========================================================================
    # SISTEMA DE ESTILOS CORPORATIVOS - CORRIGIDO
    # =========================================================================
    title_font = Font(name="Calibri", size=14, bold=True, color="FFFFFF")
    subtitle_font = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
    brand_font = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
    header_font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
    item_font = Font(name="Calibri", size=11, bold=False)
    small_font = Font(name="Calibri", size=10, bold=False)
    micro_font = Font(name="Calibri", size=8, bold=False)

    thin_side = Side(style="thin", color="DDDDDD")
    box_border = Border(left=thin_side, right=thin_side, top=thin_side, bottom=thin_side)

    center_align = Alignment(horizontal="center", vertical="center")
    left_align = Alignment(horizontal="left", vertical="center")
    wrap_align = Alignment(horizontal="left", vertical="center", wrap_text=True)

    # =========================================================================
    # CONFIGURAÇÕES E PARÂMETROS
    # =========================================================================
    DEFAULT_IMAGE_PATH = r"C:\Users\40418843\Desktop\VIVOHUB\templates\20251007_140942_0000.png"
    image_path = row_get(form_row, "diagram_image_path") or DEFAULT_IMAGE_PATH

    vivo_xy_pct = _parse_xy_pct_str(row_get(form_row, "vivo_label_xy_pct", ""), (0.16, 0.48))
    op_xy_pct = _parse_xy_pct_str(row_get(form_row, "operadora_label_xy_pct", ""), (0.83, 0.44))
    link_vivo_xy_pct = _parse_xy_pct_str(row_get(form_row, "link_vivo_label_xy_pct", ""), (0.49, 0.53))
    link_op_xy_pct = _parse_xy_pct_str(row_get(form_row, "link_op_label_xy_pct", ""), (0.49, 0.35))

    try:
        font_size_pct = float(row_get(form_row, "diagram_font_size_pct", 0.040))
    except (ValueError, TypeError):
        font_size_pct = 0.060

    try:
        link_font_size_pct = float(row_get(form_row, "link_labels_font_size_pct", 0.030))
    except (ValueError, TypeError):
        link_font_size_pct = 0.030

    font_path = row_get(form_row, "diagram_font_path", None)

    roteador1_xy = _parse_xy_pct_str(row_get(form_row, "roteador_op1_label_xy_pct", ""), (0.64, 0.58))
    roteador2_xy = _parse_xy_pct_str(row_get(form_row, "roteador_op2_label_xy_pct", ""), (0.64, 0.39))
    
    try:
        roteador_font_pct = float(row_get(form_row, "roteador_op_font_size_pct", 0.010))
    except (ValueError, TypeError):
        roteador_font_pct = font_size_pct

    no_stroke_flag = str(row_get(form_row, "roteador_op_no_stroke", "")).strip().lower() in ("1", "true", "yes", "sim")
    try:
        roteador_stroke_pct_override = row_get(form_row, "roteador_op_stroke_width_pct", None)
        if roteador_stroke_pct_override not in (None, ""):
            roteador_stroke_pct = float(roteador_stroke_pct_override)
        else:
            roteador_stroke_pct = 0.0 if no_stroke_flag else 0.009
    except (ValueError, TypeError):
        roteador_stroke_pct = 0.0 if no_stroke_flag else 0.009

    bbox_w = _int_fallback(row_get(form_row, "diagram_max_w", 1900), 1900)
    bbox_h = _int_fallback(row_get(form_row, "diagram_max_h", 650), 650)
    anchor_row_offset = _int_fallback(row_get(form_row, "diagram_anchor_row_offset", -3), -3)
    anchor_col_offset = _int_fallback(row_get(form_row, "diagram_anchor_col_offset", +1), +1)

    # =============================================================================
    # EXTRAÇÃO DE DADOS DO FORMULÁRIO
    # =============================================================================
    nome_operadora = (row_get(form_row, "nome_operadora") or "Operadora").strip()
    resp_atacado = (row_get(form_row, "responsavel_atacado") or row_get(form_row, "responsavel_vivo") or "").strip()
    resp_engenharia = (row_get(form_row, "responsavel_engenharia", "") or "").strip()
    asn_op = (row_get(form_row, "asn", "") or "").strip()
    tmr_value = row_get(form_row, "tmr", "")
    csp_value = "C/ CSP" if row_get(form_row, 'csp', 0) == 1 else "S/CSP"

    # Dados dinâmicos
    traffic_types = _extract_traffic_types()
    concentracao_data = _get_concentracao_data()
    vivo_data = _get_vivo_data()

    # Data de criação
    created_raw = row_get(form_row, "created_at", "")
    created_dt = _parse_db_datetime(created_raw)
    created_str = created_dt.strftime("%d/%m/%Y") if created_dt else (str(created_raw) or "")

    # =========================================================================
    # CONFIGURAÇÕES DE LAYOUT
    # =========================================================================
    start_col, end_col = 3, 15
    brand_row_num = 2

    # =========================================================================
    # INICIALIZAÇÃO DO WORKBOOK
    # =========================================================================
    wb = Workbook()

    # =========================================================================
    # ABA 1: ÍNDICE - SUMÁRIO EXECUTIVO (SEM BORDAS)
    # =========================================================================
    ws = wb.active
    ws.title = "Índice"

    # Configuração de colunas mais adequada para o índice
    for col_idx in range(1, 10):
        ws.column_dimensions[_col(col_idx)].width = 12.0
    ws.column_dimensions[_col(3)].width = 8.0  # Coluna principal mais estreita
    ws.column_dimensions[_col(4)].width = 45.0  # Coluna de conteúdo mais larga

    # Cabeçalho corporativo - cobrindo toda a largura útil (SEM BORDA)
    ws.merge_cells(start_row=2, start_column=3, end_row=2, end_column=8)
    brand_cell = ws.cell(row=2, column=3, value=f"VIVO — {nome_operadora}")
    brand_cell.font = Font(name="Calibri", size=14, bold=True, color="FFFFFF")
    brand_cell.alignment = center_align
    brand_cell.fill = fill_header_primary
    # SEM BORDA
    ws.row_dimensions[2].height = 28.0

    # Título principal - cobrindo toda a largura útil (SEM BORDA)
    ws.merge_cells(start_row=4, start_column=3, end_row=4, end_column=8)
    title_cell = ws.cell(row=4, column=3, value="ANEXO 3: PROJETO TÉCNICO")
    title_cell.font = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
    title_cell.alignment = center_align
    title_cell.fill = fill_header_secondary
    # SEM BORDA
    ws.row_dimensions[4].height = 25.0

    # Subtítulo - cobrindo toda a largura útil (SEM BORDA)
    ws.merge_cells(start_row=5, start_column=3, end_row=5, end_column=8)
    subtitle_cell = ws.cell(row=5, column=3, value="PROJETO DE INTERLIGAÇÃO PARA ENCAMINHAMENTO DA TERMINAÇÃO DE CHAMADAS DE VOZ")
    subtitle_cell.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
    subtitle_cell.alignment = center_align
    subtitle_cell.fill = fill_header_secondary
    # SEM BORDA
    ws.row_dimensions[5].height = 30.0

    # Seções do documento - CADA UMA COBRINDO TODA A LARGURA ÚTIL (SEM BORDAS)
    document_sections = [
        "1. Versões",
        "2. Projeto de Interligação", 
        "2.2. Diagrama de Interligação",
        "2.3. Características do Projeto de Interligação e do Plano de Encaminhamento",
        "2.4. Plano de Contingência",
        "2.5. Concentração",
        "2.6. Plan NUM_Operadora",
        "2.7. Dados de MTL",
        "2.8. SE REG III (Vivo STFC Concessionária)",
        "2.9. Parâmetros de Programação",
    ]

    current_row = 7
    for i, section_title in enumerate(document_sections):
        # MESCLAR CÉLULAS PARA COBRIR TODA A LARGURA DO TEXTO (SEM BORDA)
        ws.merge_cells(start_row=current_row, start_column=3, end_row=current_row, end_column=8)
        
        section_cell = ws.cell(row=current_row, column=3, value=section_title)
        section_cell.font = Font(name="Calibri", size=11, bold=False)
        section_cell.alignment = left_align
        # SEM BORDA
        
        # Alternar cores para melhor visualização
        if i % 2 == 0:
            section_cell.fill = fill_background
        else:
            section_cell.fill = fill_neutral
        
        ws.row_dimensions[current_row].height = 22.0
        current_row += 1

    # Configurações de view
    ws.freeze_panes = f"{_col(3)}{7}"  # Congelar no início das seções
    ws.sheet_view.showGridLines = False
    ws.page_setup.paperSize = ws.PAPERSIZE_A4
    ws.page_setup.orientation = ws.ORIENTATION_PORTRAIT
    ws.print_options.horizontalCentered = True
    ws.page_margins.left = ws.page_margins.right = 0.4
    ws.page_margins.top = ws.page_margins.bottom = 0.5


    # =========================================================================
    # ABA 2: VERSÕES - CONTROLE DE REVISÕES (SEM BORDAS)
    # =========================================================================
    wv = wb.create_sheet(title="Versões")

    # Configuração de colunas - MESMA LARGURA QUE O TÍTULO
    column_widths_ver = {
        1: 2.0,   # Espaçamento
        2: 2.0,   # Espaçamento
        3: 8.0,   # Versão
        4: 12.0,  # Data
        5: 28.0,  # Responsável Eng de ITX
        6: 28.0,  # Responsável Gestão de ITX
        7: 28.0,  # Escopo
        8: 10.0,  # CN
        9: 18.0,  # ÁREAS LOCAIS
        10: 10.0, # ATA
        11: 2.0   # Espaçamento
    }

    for col_idx, width in column_widths_ver.items():
        wv.column_dimensions[_col(col_idx)].width = width

    # Cabeçalho corporativo - MESMA LARGURA DA TABELA (ROXO PRINCIPAL) (SEM BORDA)
    wv.merge_cells(start_row=2, start_column=3, end_row=2, end_column=10)
    brand_cell_v = wv.cell(row=2, column=3, value=f"VIVO — {nome_operadora}")
    brand_cell_v.font = Font(name="Calibri", size=14, bold=True, color="FFFFFF")
    brand_cell_v.alignment = center_align
    brand_cell_v.fill = fill_header_primary
    # SEM BORDA
    wv.row_dimensions[2].height = 24.0

    # Título da aba - MESMA LARGURA DA TABELA (ROXO SECUNDÁRIO) (SEM BORDA)
    title_row_v = 4
    wv.merge_cells(start_row=title_row_v, start_column=3, end_row=title_row_v, end_column=10)
    title_cell_v = wv.cell(row=title_row_v, column=3, value="CONTROLE DE VERSÕES DO PTI")
    title_cell_v.font = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
    title_cell_v.alignment = center_align
    title_cell_v.fill = fill_header_secondary
    # SEM BORDA
    wv.row_dimensions[title_row_v].height = 25.0

    # Cabeçalhos da tabela de versões - ALINHADOS COM O TÍTULO (ROXO CLARO) (SEM BORDAS)
    version_headers = [
        "Versão", "Data", "Responsável Eng de ITX", 
        "Responsável Gestão de ITX", "Escopo", "CN", "ÁREAS LOCAIS", "ATA"
    ]

    header_row = title_row_v + 2
    wv.row_dimensions[header_row].height = 25.0

    # Cabeçalhos individuais - cada um na sua coluna (SEM BORDAS)
    for idx, header_text in enumerate(version_headers):
        col_idx = 3 + idx  # Começa na coluna 3
        header_cell = wv.cell(row=header_row, column=col_idx, value=header_text)
        header_cell.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
        header_cell.alignment = center_align
        header_cell.fill = fill_header_light
        # SEM BORDA

    # Dados da tabela de versões - PREENCHER COM DADOS REAIS
    data_start_row = header_row + 1

    for row_index in range(10):
        current_row_num = data_start_row + row_index
        wv.row_dimensions[current_row_num].height = 22.0

        # Padrão de cores alternadas
        if row_index % 2 == 0:
            row_fill = fill_background
        else:
            row_fill = fill_neutral

        # Versão
        version_cell = wv.cell(row=current_row_num, column=3, value=row_index + 1)
        version_cell.font = Font(name="Calibri", size=10)
        version_cell.alignment = center_align
        # SEM BORDA
        version_cell.fill = row_fill

        # Data (apenas na primeira linha com data real)
        date_value = created_str if row_index == 0 else ""
        date_cell = wv.cell(row=current_row_num, column=4, value=date_value)
        date_cell.font = Font(name="Calibri", size=10)
        date_cell.alignment = center_align
        # SEM BORDA
        date_cell.fill = row_fill

        # Responsável Engenharia (apenas na primeira linha)
        eng_value = resp_engenharia if row_index == 0 else ""
        eng_cell = wv.cell(row=current_row_num, column=5, value=eng_value)
        eng_cell.font = Font(name="Calibri", size=10)
        eng_cell.alignment = left_align
        # SEM BORDA
        eng_cell.fill = row_fill

        # Responsável Atacado (apenas na primeira linha)
        atk_value = resp_atacado if row_index == 0 else ""
        atk_cell = wv.cell(row=current_row_num, column=6, value=atk_value)
        atk_cell.font = Font(name="Calibri", size=10)
        atk_cell.alignment = left_align
        # SEM BORDA
        atk_cell.fill = row_fill

        # Colunas vazias (Escopo, CN, ÁREAS LOCAIS, ATA)
        empty_columns = [7, 8, 9, 10]
        for col_idx in empty_columns:
            empty_cell = wv.cell(row=current_row_num, column=col_idx, value="")
            empty_cell.font = Font(name="Calibri", size=10)
            # SEM BORDA
            empty_cell.fill = row_fill

    # Configurações de view
    wv.freeze_panes = f"{_col(3)}{data_start_row}"
    wv.sheet_view.showGridLines = False
    wv.page_setup.paperSize = wv.PAPERSIZE_A4
    wv.page_setup.orientation = wv.ORIENTATION_LANDSCAPE
    wv.print_options.horizontalCentered = True
    wv.page_margins.left = wv.page_margins.right = 0.4
    wv.page_margins.top = wv.page_margins.bottom = 0.5

    # =============================================================================
    # ABA 3: DIAGRAMA DE INTERLIGAÇÃO - COMPLETA COM RAC/RAV/HL4
    # =============================================================================
    wd = wb.create_sheet(title="Diagrama de Interligação")

    # Configuração de colunas - MAIS LARGAS PARA MELHOR ALINHAMENTO
    column_widths_diag = {
        1: 2.0, 2: 2.0, 3: 12.0, 4: 12.0, 5: 12.0, 6: 12.0, 7: 12.0, 
        8: 12.0, 9: 12.0, 10: 12.0, 11: 12.0, 12: 12.0, 13: 12.0, 14: 2.0
    }
    for col_idx, width in column_widths_diag.items():
        wd.column_dimensions[_col(col_idx)].width = width

    # Cabeçalho corporativo - ROXO PRINCIPAL (SEM BORDA)
    wd.merge_cells(start_row=2, start_column=3, end_row=2, end_column=12)
    brand_cell_d = wd.cell(row=2, column=3, value=f"VIVO — {nome_operadora}")
    brand_cell_d.font = Font(name="Calibri", size=14, bold=True, color="FFFFFF")
    brand_cell_d.alignment = center_align
    brand_cell_d.fill = fill_header_primary
    # SEM BORDA
    wd.row_dimensions[2].height = 24.0

    # Título principal (SEM BORDA)
    title_row_d = 4
    wd.merge_cells(start_row=title_row_d, start_column=3, end_row=title_row_d, end_column=12)
    title_cell_d = wd.cell(row=title_row_d, column=3, value=f"Diagrama de Interligação entre a VIVO e a {nome_operadora}")
    title_cell_d.font = Font(name="Calibri", size=12, bold=True)
    title_cell_d.alignment = left_align
    title_cell_d.fill = fill_white
    # SEM BORDA
    wd.row_dimensions[title_row_d].height = 25.0

    # Seção 2.2.1 (SEM BORDA)
    section_221_row = 6
    wd.merge_cells(start_row=section_221_row, start_column=3, end_row=section_221_row, end_column=12)
    section_221_cell = wd.cell(row=section_221_row, column=3, value="2.2.1 DIAGRAMAÇÃO DO PROJETO SIP")
    section_221_cell.font = Font(name="Calibri", size=11, bold=True)
    section_221_cell.alignment = left_align
    section_221_cell.fill = fill_white
    # SEM BORDA
    wd.row_dimensions[section_221_row].height = 22.0

    # Seção 2.2.1.1 (SEM BORDA)
    section_2211_row = 7
    wd.merge_cells(start_row=section_2211_row, start_column=3, end_row=section_2211_row, end_column=12)
    section_2211_cell = wd.cell(row=section_2211_row, column=3, value="2.2.1.1 Anúncio de Redes pelos 2 Links SPC")
    section_2211_cell.font = Font(name="Calibri", size=10, bold=True)
    section_2211_cell.alignment = left_align
    section_2211_cell.fill = fill_white
    # SEM BORDA
    wd.row_dimensions[section_2211_row].height = 22.0

    # Descrição Operadora -> VIVO (SEM BORDA)
    desc_op_vivo_row = 8
    wd.merge_cells(start_row=desc_op_vivo_row, start_column=3, end_row=desc_op_vivo_row, end_column=12)
    desc_op_vivo_cell = wd.cell(row=desc_op_vivo_row, column=3, value=f"A {nome_operadora} abordará os endereços da VIVO conforme abaixo.")
    desc_op_vivo_cell.font = Font(name="Calibri", size=10)
    desc_op_vivo_cell.alignment = wrap_align
    desc_op_vivo_cell.fill = fill_white
    # SEM BORDA
    wd.row_dimensions[desc_op_vivo_row].height = 25.0

    # Descrição VIVO -> Operadora (SEM BORDA)
    desc_vivo_op_row = 9
    wd.merge_cells(start_row=desc_vivo_op_row, start_column=3, end_row=desc_vivo_op_row, end_column=12)
    desc_vivo_op_cell = wd.cell(row=desc_vivo_op_row, column=3, value=f"A VIVO abordará os endereços da {nome_operadora} conforme abaixo.")
    desc_vivo_op_cell.font = Font(name="Calibri", size=10)
    desc_vivo_op_cell.alignment = wrap_align
    desc_vivo_op_cell.fill = fill_white
    # SEM BORDA
    wd.row_dimensions[desc_vivo_op_row].height = 25.0

    # Seção 2.2.1.2 Parâmetros de Configuração do Link (SEM BORDA)
    param_config_row = 11
    wd.merge_cells(start_row=param_config_row, start_column=3, end_row=param_config_row, end_column=12)
    param_config_cell = wd.cell(row=param_config_row, column=3, value="2.2.1.2 Parâmetros de Configuração do Link")
    param_config_cell.font = Font(name="Calibri", size=10, bold=True)
    param_config_cell.alignment = left_align
    param_config_cell.fill = fill_white
    # SEM BORDA
    wd.row_dimensions[param_config_row].height = 22.0

    # ASN VIVO e ASN OPERADORA - SEM CORES E SEM BORDAS
    asn_row = param_config_row + 1
    wd.row_dimensions[asn_row].height = 22.0
    wd.row_dimensions[asn_row + 1].height = 22.0

    # ASN VIVO - MESMA LARGURA DO TÍTULO ACIMA (SEM BORDA)
    wd.merge_cells(start_row=asn_row, start_column=3, end_row=asn_row, end_column=5)
    asn_vivo_label = wd.cell(row=asn_row, column=3, value="ASN VIVO")
    asn_vivo_label.font = Font(name="Calibri", size=10, bold=True)
    asn_vivo_label.alignment = center_align
    asn_vivo_label.fill = fill_white
    # SEM BORDA

    wd.merge_cells(start_row=asn_row, start_column=6, end_row=asn_row, end_column=8)
    asn_vivo_value = wd.cell(row=asn_row, column=6, value="10429 (Público)")
    asn_vivo_value.font = Font(name="Calibri", size=10)
    asn_vivo_value.alignment = center_align
    asn_vivo_value.fill = fill_white
    # SEM BORDA

    # ASN OPERADORA - MESMA LARGURA DO TÍTULO ACIMA (SEM BORDA)
    wd.merge_cells(start_row=asn_row + 1, start_column=3, end_row=asn_row + 1, end_column=5)
    asn_op_label = wd.cell(row=asn_row + 1, column=3, value=f"ASN {nome_operadora}")
    asn_op_label.font = Font(name="Calibri", size=10, bold=True)
    asn_op_label.alignment = center_align
    asn_op_label.fill = fill_white
    # SEM BORDA

    wd.merge_cells(start_row=asn_row + 1, start_column=6, end_row=asn_row + 1, end_column=8)
    asn_op_value = wd.cell(row=asn_row + 1, column=6, value=asn_op)
    asn_op_value.font = Font(name="Calibri", size=10)
    asn_op_value.alignment = center_align
    asn_op_value.fill = fill_white
    # SEM BORDA

    # ESPAÇO entre ASNs e CN/VRF
    espaco_asn_cn_row = asn_row + 2
    wd.row_dimensions[espaco_asn_cn_row].height = 10.0

    # CN e VRF - COM CORES MAS SEM BORDAS
    cn_vrf_row = espaco_asn_cn_row + 1
    wd.row_dimensions[cn_vrf_row].height = 22.0

    # CN - MESMA LARGURA DO TÍTULO ACIMA (SEM BORDA)
    wd.merge_cells(start_row=cn_vrf_row, start_column=3, end_row=cn_vrf_row, end_column=5)
    cn_cell = wd.cell(row=cn_vrf_row, column=3, value="CN")
    cn_cell.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
    cn_cell.alignment = center_align
    cn_cell.fill = fill_header_light
    # SEM BORDA

    # VRF - MESMA LARGURA DO TÍTULO ACIMA (SEM BORDA)
    wd.merge_cells(start_row=cn_vrf_row, start_column=6, end_row=cn_vrf_row, end_column=12)
    vrf_cell = wd.cell(row=cn_vrf_row, column=6, value="VRF: __________________________")
    vrf_cell.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
    vrf_cell.alignment = center_align
    vrf_cell.fill = fill_header_light
    # SEM BORDA

    # ESPAÇO entre CN/VRF e Pontas A/B
    espaco_cn_pontas_row = cn_vrf_row + 1
    wd.row_dimensions[espaco_cn_pontas_row].height = 10.0

    # Pontas A e B - SEM CORES E SEM BORDAS
    pontas_row = espaco_cn_pontas_row + 1
    wd.row_dimensions[pontas_row].height = 22.0
    wd.row_dimensions[pontas_row + 1].height = 22.0

    ponta_configs = [
        (3, 7, "Ponta A", "VIVO"),      # MESMA LARGURA DO TÍTULO
        (8, 12, "Ponta B", nome_operadora), # MESMA LARGURA DO TÍTULO
    ]

    for col_start, col_end, label, value in ponta_configs:
        # Label - SEM COR E SEM BORDA
        wd.merge_cells(start_row=pontas_row, start_column=col_start, end_row=pontas_row, end_column=col_end)
        label_cell = wd.cell(row=pontas_row, column=col_start, value=label)
        label_cell.font = Font(name="Calibri", size=10, bold=True)
        label_cell.alignment = center_align
        label_cell.fill = fill_white
        # SEM BORDA

        # Valor - SEM COR E SEM BORDA
        wd.merge_cells(start_row=pontas_row + 1, start_column=col_start, end_row=pontas_row + 1, end_column=col_end)
        value_cell = wd.cell(row=pontas_row + 1, column=col_start, value=value)
        value_cell.font = Font(name="Calibri", size=10, bold=True)
        value_cell.alignment = center_align
        value_cell.fill = fill_white
        # SEM BORDA

    # ESPAÇO entre Pontas A/B e Tabelas de Tráfego
    espaco_pontas_tabelas_row = pontas_row + 2
    wd.row_dimensions[espaco_pontas_tabelas_row].height = 10.0

    # Tabelas de tráfego - MESMA LARGURA DO TÍTULO ACIMA E SEM BORDAS
    def _create_traffic_table(top_row: int, left_col: int, right_col: int, traffic_data: list = None):
        """
        Cria tabela de tráfego com dados reais do formulário.
        """
        # Configurar larguras das colunas
        col_widths = [18.0, 25.0, 15.0]
        for i, width in enumerate(col_widths):
            wd.column_dimensions[_col(left_col + i)].width = width
        
        # Cabeçalho da tabela
        header_start = left_col
        header_end = right_col
        wd.merge_cells(start_row=top_row, start_column=header_start, end_row=top_row, end_column=header_end)
        
        for col_index, header_text in enumerate(["Tráfego", "Endereço IP", "NET MASK"]):
            header_cell = wd.cell(row=top_row + 1, column=left_col + col_index, value=header_text)
            header_cell.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
            header_cell.alignment = center_align
            header_cell.fill = fill_header_light
            # SEM BORDA
        
        wd.row_dimensions[top_row + 1].height = 25.0

        # Usar os dados reais do formulário
        traffic_list = traffic_data or []
        
        for line_index in range(3):
            current_row = top_row + 2 + line_index
            wd.row_dimensions[current_row].height = 22.0
            
            if line_index % 2 == 0:
                row_fill = fill_background
            else:
                row_fill = fill_neutral
            
            # Preencher com dados reais - tráfego vem do escopo_flags_json
            traffic_type = traffic_list[line_index] if line_index < len(traffic_list) else ""
            traffic_cell = wd.cell(row=current_row, column=left_col, value=traffic_type)
            traffic_cell.font = Font(name="Calibri", size=9)
            traffic_cell.alignment = left_align
            # SEM BORDA
            traffic_cell.fill = row_fill

            # Endereço IP vazio (para preenchimento posterior)
            ip_cell = wd.cell(row=current_row, column=left_col + 1, value="")
            ip_cell.font = Font(name="Calibri", size=9)
            ip_cell.alignment = left_align
            # SEM BORDA
            ip_cell.fill = row_fill

            # NET MASK vazio (para preenchimento posterior)
            mask_cell = wd.cell(row=current_row, column=left_col + 2, value="")
            mask_cell.font = Font(name="Calibri", size=9)
            mask_cell.alignment = center_align
            # SEM BORDA
            mask_cell.fill = row_fill

    # Tabelas alinhadas com a largura total do título (SEM BORDAS)
    table_top_row = espaco_pontas_tabelas_row + 1

    # PASSAR OS DADOS REAIS DO FORMULÁRIO PARA AMBAS AS TABELAS
    _create_traffic_table(table_top_row, 3, 7, traffic_types)   # Tabela esquerda (Ponta A - VIVO)
    _create_traffic_table(table_top_row, 8, 12, traffic_types)  # Tabela direita (Ponta B - Operadora)

    # ESPAÇO entre Tabelas de Tráfego e Endereços
    espaco_tabelas_enderecos_row = table_top_row + 5
    wd.row_dimensions[espaco_tabelas_enderecos_row].height = 10.0

    # Endereços - MESMA LARGURA DO TÍTULO E SEM BORDAS
    addr_row = espaco_tabelas_enderecos_row + 1
    wd.row_dimensions[addr_row].height = 22.0

    address_notes = [
        (3, 7, "Endereço (VIVO): __________________________________________"),
        (8, 12, f"Endereço ({nome_operadora}): __________________________________"),
    ]

    for col_start, col_end, note_text in address_notes:
        wd.merge_cells(start_row=addr_row, start_column=col_start, end_row=addr_row, end_column=col_end)
        note_cell = wd.cell(row=addr_row, column=col_start, value=note_text)
        note_cell.font = Font(name="Calibri", size=9)
        note_cell.alignment = left_align
        note_cell.fill = fill_white
        # SEM BORDA

    # ESPAÇO entre Endereços e Diagrama
    espaco_enderecos_diagrama_row = addr_row + 1
    wd.row_dimensions[espaco_enderecos_diagrama_row].height = 15.0

    # Inserção do diagrama
    diagram_row = espaco_enderecos_diagrama_row + 1

    if not os.path.exists(image_path):
        raise FileNotFoundError(f"Imagem do diagrama não encontrada em: {image_path}")

    link_vivo_text = "Link resp Vivo"
    link_op_text = f"Link resp {nome_operadora}"

    # Definir posições para RAC/RAV/HL4
    rac_rav_xy1 = (0.33, 0.70)  # Primeira posição (inferior)
    rac_rav_xy2 = (0.34, 0.60)  # Segunda posição (superior)

    annotation_labels = [
        {"text": "Roteador OP", "xy_pct": roteador1_xy, "font_size_pct": roteador_font_pct, "stroke_width_pct": roteador_stroke_pct},
        {"text": "Roteador OP", "xy_pct": roteador2_xy, "font_size_pct": roteador_font_pct, "stroke_width_pct": roteador_stroke_pct},
        {"text": link_vivo_text, "xy_pct": link_vivo_xy_pct, "font_size_pct": link_font_size_pct, "fill": (255, 255, 255, 255), "stroke_fill": (0, 0, 0, 255), "stroke_width_pct": 0.006},
        {"text": link_op_text, "xy_pct": link_op_xy_pct, "font_size_pct": link_font_size_pct, "fill": (255, 255, 255, 255), "stroke_fill": (0, 0, 0, 255), "stroke_width_pct": 0.006},
        # Adicionar os dois textos RAC/RAV/HL4
        {"text": "RAC/RAV/HL4", "xy_pct": rac_rav_xy1, "font_size_pct": 0.010, "fill": (255, 255, 255, 255), "stroke_fill": (0, 0, 0, 255), "stroke_width_pct": 0.006},
        {"text": "RAC/RAV/HL4", "xy_pct": rac_rav_xy2, "font_size_pct": 0.010, "fill": (255, 255, 255, 255), "stroke_fill": (0, 0, 0, 255), "stroke_width_pct": 0.006}
    ]

    try:
        annotated_image_path = _render_labels_on_image(
            image_path=image_path,
            vivo_text="VIVO",
            op_text=nome_operadora,
            vivo_xy_pct=vivo_xy_pct,
            op_xy_pct=op_xy_pct,
            font_path=font_path,
            font_size_pct=font_size_pct,
            fill=(255, 255, 255, 255),
            stroke_fill=(0, 0, 0, 255),
            stroke_width_pct=0.009,
            extra_labels=annotation_labels
        )
    except RuntimeError as e:
        raise RuntimeError(f"Falha no processamento da imagem do diagrama: {e}") from e
    except Exception as e:
        raise RuntimeError(f"Erro inesperado ao anotar o diagrama: {e}") from e

    if XLImage is None:
        raise RuntimeError("Suporte a imagens não disponível no openpyxl.")

    try:
        diagram_image = XLImage(annotated_image_path)
    except Exception as e:
        raise FileNotFoundError(f"Imagem anotada não pôde ser carregada: {annotated_image_path}") from e

    try:
        with PILImage.open(annotated_image_path) as source_image:
            original_width, original_height = source_image.size
    except Exception:
        original_width, original_height = (800, 300)

    final_width, final_height = _fit_size_keep_aspect(original_width, original_height, bbox_w, bbox_h)
    diagram_image.width = final_width
    diagram_image.height = final_height

    image_anchor_row = diagram_row + anchor_row_offset
    image_anchor_col = 3 + anchor_col_offset
    wd.add_image(diagram_image, f"{_col(image_anchor_col)}{image_anchor_row}")

    # Configurações de view
    wd.freeze_panes = f"{_col(3)}{table_top_row}"
    wd.sheet_view.showGridLines = False
    wd.page_setup.paperSize = wd.PAPERSIZE_A4
    wd.page_setup.orientation = wd.ORIENTATION_LANDSCAPE
    wd.print_options.horizontalCentered = True
    wd.page_margins.left = wd.page_margins.right = 0.4
    wd.page_margins.top = wd.page_margins.bottom = 0.5

    # =========================================================================
    # ABA 4: ENCAMINHAMENTO - PLANO DE ENCAMINHAMENTO (CÓDIGO COMPLETO)
    # =========================================================================
    we = wb.create_sheet(title="Encaminhamento")

    # Configuração de colunas com larguras adequadas
    column_widths_enc = {
        1: 2.0,    # Espaçamento
        2: 15.0,   # ÁREA LOCAL
        3: 10.0,   # LOCALIZAÇÃO - CN
        4: 15.0,   # LOCALIZAÇÃO - POI/PPI VIVO
        5: 10.0,   # LOCALIZAÇÃO - CN
        6: 18.0,   # LOCALIZAÇÃO - POI/PPI OPERADORA
        7: 15.0,   # DADOS DA ROTA - PONTA A VIVO
        8: 18.0,   # DADOS DA ROTA - PONTA B OPERADORA
        9: 12.0,   # DADOS DA ROTA - TIPO DE TRÁFEGO
        10: 8.0,   # BANDA - EXIST.
        11: 8.0,   # BANDA - PLAN.
        12: 8.0,   # CANAIS - EXIST.
        13: 8.0,   # CANAIS - PLAN.
        14: 8.0,   # CAPS - EXIST.
        15: 8.0,   # CAPS - PLAN.
        16: 12.0,  # ATIVAÇÃO - PREVISTA
        17: 18.0,  # ENCAMINHAMENTO - DE A > B
        18: 12.0,  # ENCAMINHAMENTO - DE B > A
        19: 12.0,  # SINALIZAÇÃO - CODEC
        20: 15.0,  # SINALIZAÇÃO - OBSERVAÇÃO
        21: 12.0,  # ENDEREÇO IP - VIVO - IP ADDRESS
        22: 10.0,  # ENDEREÇO IP - VIVO - NETMASK
        23: 12.0,  # ENDEREÇO IP - OPERADORA - IP ADDRESS
        24: 10.0,  # ENDEREÇO IP - OPERADORA - NETMASK
        25: 2.0,   # Espaçamento
    }

    for col_idx, width in column_widths_enc.items():
        we.column_dimensions[_col(col_idx)].width = width

    # Cabeçalho corporativo (SEM BORDA)
    brand_row_num = 2
    we.merge_cells(start_row=brand_row_num, start_column=2, end_row=brand_row_num, end_column=24)
    brand_cell_we = we.cell(row=brand_row_num, column=2, value=f"VIVO — {nome_operadora}")
    brand_cell_we.font = Font(name="Calibri", size=14, bold=True, color="FFFFFF")
    brand_cell_we.alignment = center_align
    brand_cell_we.fill = fill_header_primary
    # SEM BORDA
    we.row_dimensions[brand_row_num].height = 24.0

    # Título da seção (SEM BORDA)
    title_enc_row = brand_row_num + 2
    we.merge_cells(start_row=title_enc_row, start_column=2, end_row=title_enc_row, end_column=24)
    title_enc_cell = we.cell(row=title_enc_row, column=2, value="2.3. CARACTERÍSTICAS DO PROJETO DE INTERLIGAÇÃO E DO PLANO DE ENCAMINHAMENTO")
    title_enc_cell.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
    title_enc_cell.alignment = center_align
    title_enc_cell.fill = fill_header_secondary
    # SEM BORDA
    we.row_dimensions[title_enc_row].height = 22.0

    # Cabeçalho principal - Linha 1 (TODAS AS COLUNAS MESCLADAS ATÉ A LINHA 3)
    header_row_1 = title_enc_row + 2
    we.row_dimensions[header_row_1].height = 30.0

    # ÁREA LOCAL - mesclada verticalmente até linha 3
    we.merge_cells(start_row=header_row_1, start_column=2, end_row=header_row_1 + 2, end_column=2)
    area_local_cell = we.cell(row=header_row_1, column=2, value="ÁREA LOCAL")
    area_local_cell.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    area_local_cell.alignment = Alignment(horizontal="center", vertical="center", wrap_text=True)
    area_local_cell.fill = fill_header_secondary
    area_local_cell.border = box_border

    # LOCALIZAÇÃO - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=3, end_row=header_row_1, end_column=6)
    localizacao_header = we.cell(row=header_row_1, column=3, value="LOCALIZAÇÃO")
    localizacao_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    localizacao_header.alignment = center_align
    localizacao_header.fill = fill_header_light
    localizacao_header.border = box_border

    # DADOS DA ROTA - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=7, end_row=header_row_1, end_column=9)
    dados_rota_header = we.cell(row=header_row_1, column=7, value="DADOS DA ROTA")
    dados_rota_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    dados_rota_header.alignment = center_align
    dados_rota_header.fill = fill_header_light
    dados_rota_header.border = box_border

    # BANDA - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=10, end_row=header_row_1, end_column=11)
    banda_header = we.cell(row=header_row_1, column=10, value="BANDA")
    banda_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    banda_header.alignment = center_align
    banda_header.fill = fill_header_light
    banda_header.border = box_border

    # CANAIS - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=12, end_row=header_row_1, end_column=13)
    canais_header = we.cell(row=header_row_1, column=12, value="CANAIS")
    canais_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    canais_header.alignment = center_align
    canais_header.fill = fill_header_light
    canais_header.border = box_border

    # CAPS - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=14, end_row=header_row_1, end_column=15)
    caps_header = we.cell(row=header_row_1, column=14, value="CAPS")
    caps_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    caps_header.alignment = center_align
    caps_header.fill = fill_header_light
    caps_header.border = box_border

    # ATIVAÇÃO - mesclada verticalmente até linha 3
    we.merge_cells(start_row=header_row_1, start_column=16, end_row=header_row_1 + 2, end_column=16)
    ativacao_header = we.cell(row=header_row_1, column=16, value="ATIVAÇÃO")
    ativacao_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    ativacao_header.alignment = Alignment(horizontal="center", vertical="center", wrap_text=True)
    ativacao_header.fill = fill_header_light
    ativacao_header.border = box_border

    # ENCAMINHAMENTO - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=17, end_row=header_row_1, end_column=18)
    encaminhamento_header = we.cell(row=header_row_1, column=17, value="ENCAMINHAMENTO")
    encaminhamento_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    encaminhamento_header.alignment = center_align
    encaminhamento_header.fill = fill_header_light
    encaminhamento_header.border = box_border

    # SINALIZAÇÃO - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=19, end_row=header_row_1, end_column=20)
    sinalizacao_header = we.cell(row=header_row_1, column=19, value="SINALIZAÇÃO")
    sinalizacao_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    sinalizacao_header.alignment = center_align
    sinalizacao_header.fill = fill_header_light
    sinalizacao_header.border = box_border

    # ENDEREÇO IP - mesclada horizontalmente linha 1
    we.merge_cells(start_row=header_row_1, start_column=21, end_row=header_row_1, end_column=24)
    endereco_ip_header = we.cell(row=header_row_1, column=21, value="ENDEREÇO IP")
    endereco_ip_header.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    endereco_ip_header.alignment = center_align
    endereco_ip_header.fill = fill_header_light
    endereco_ip_header.border = box_border

    # Cabeçalho Linha 2 - Subcolunas principais
    header_row_2 = header_row_1 + 1
    we.row_dimensions[header_row_2].height = 25.0

    # LOCALIZAÇÃO - Subcolunas (linha 2) - MESCLADAS VERTICALMENTE
    localizacao_subheaders = [
        (3, "CN"),
        (4, "POI/PPI VIVO"),
        (5, "CN"),
        (6, f"POI/PPI {nome_operadora.upper()}")
    ]
    for col_idx, header_text in localizacao_subheaders:
        we.merge_cells(start_row=header_row_2, start_column=col_idx, end_row=header_row_2 + 1, end_column=col_idx)
        sub_cell = we.cell(row=header_row_2, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_accent
        sub_cell.border = box_border

    # DADOS DA ROTA - Subcolunas (linha 2) - MESCLADAS VERTICALMENTE
    dados_rota_subheaders = [
        (7, "PONTA A VIVO"),
        (8, f"PONTA B {nome_operadora.upper()}"),
        (9, "TIPO DE TRÁFEGO")
    ]
    for col_idx, header_text in dados_rota_subheaders:
        we.merge_cells(start_row=header_row_2, start_column=col_idx, end_row=header_row_2 + 1, end_column=col_idx)
        sub_cell = we.cell(row=header_row_2, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_accent
        sub_cell.border = box_border

    # BANDA - Subcolunas (linha 2) - MESCLADAS VERTICALMENTE
    banda_subheaders = [
        (10, "EXIST."),
        (11, "PLAN.")
    ]
    for col_idx, header_text in banda_subheaders:
        we.merge_cells(start_row=header_row_2, start_column=col_idx, end_row=header_row_2 + 1, end_column=col_idx)
        sub_cell = we.cell(row=header_row_2, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_accent
        sub_cell.border = box_border

    # CANAIS - Subcolunas (linha 2) - MESCLADAS VERTICALMENTE
    canais_subheaders = [
        (12, "EXIST."),
        (13, "PLAN.")
    ]
    for col_idx, header_text in canais_subheaders:
        we.merge_cells(start_row=header_row_2, start_column=col_idx, end_row=header_row_2 + 1, end_column=col_idx)
        sub_cell = we.cell(row=header_row_2, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_accent
        sub_cell.border = box_border

    # CAPS - Subcolunas (linha 2) - MESCLADAS VERTICALMENTE
    caps_subheaders = [
        (14, "EXIST."),
        (15, "PLAN.")
    ]
    for col_idx, header_text in caps_subheaders:
        we.merge_cells(start_row=header_row_2, start_column=col_idx, end_row=header_row_2 + 1, end_column=col_idx)
        sub_cell = we.cell(row=header_row_2, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_accent
        sub_cell.border = box_border

    # ENCAMINHAMENTO - Subcolunas (linha 2) - MESCLADAS VERTICALMENTE
    encaminhamento_subheaders = [
        (17, "DE A > B\n(FORMATO DE ENTREGA)"),
        (18, "DE B > A")
    ]
    for col_idx, header_text in encaminhamento_subheaders:
        we.merge_cells(start_row=header_row_2, start_column=col_idx, end_row=header_row_2 + 1, end_column=col_idx)
        sub_cell = we.cell(row=header_row_2, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=7, bold=True, color="FFFFFF")
        sub_cell.alignment = Alignment(horizontal="center", vertical="center", wrap_text=True)
        sub_cell.fill = fill_accent
        sub_cell.border = box_border

    # SINALIZAÇÃO - Subcolunas (linha 2) - MESCLADAS VERTICALMENTE
    sinalizacao_subheaders = [
        (19, "CODEC"),
        (20, "OBSERVAÇÃO")
    ]
    for col_idx, header_text in sinalizacao_subheaders:
        we.merge_cells(start_row=header_row_2, start_column=col_idx, end_row=header_row_2 + 1, end_column=col_idx)
        sub_cell = we.cell(row=header_row_2, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_accent
        sub_cell.border = box_border

    # ENDEREÇO IP - VIVO (linha 2)
    we.merge_cells(start_row=header_row_2, start_column=21, end_row=header_row_2, end_column=22)
    endereco_vivo_header = we.cell(row=header_row_2, column=21, value="VIVO")
    endereco_vivo_header.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
    endereco_vivo_header.alignment = center_align
    endereco_vivo_header.fill = fill_accent
    endereco_vivo_header.border = box_border

    # ENDEREÇO IP - OPERADORA (linha 2)
    we.merge_cells(start_row=header_row_2, start_column=23, end_row=header_row_2, end_column=24)
    endereco_op_header = we.cell(row=header_row_2, column=23, value=f"{nome_operadora.upper()}")
    endereco_op_header.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
    endereco_op_header.alignment = center_align
    endereco_op_header.fill = fill_accent
    endereco_op_header.border = box_border

    # Cabeçalho Linha 3 - APENAS para ENDEREÇO IP (subcolunas mais internas)
    header_row_3 = header_row_2 + 1
    we.row_dimensions[header_row_3].height = 25.0

    # ENDEREÇO IP - VIVO - Subsubcolunas (linha 3)
    endereco_vivo_subsubheaders = [
        (21, "IP ADDRESS"),
        (22, "NETMASK")
    ]
    for col_idx, header_text in endereco_vivo_subsubheaders:
        sub_cell = we.cell(row=header_row_3, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=7, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_header_light
        sub_cell.border = box_border

    # ENDEREÇO IP - OPERADORA - Subsubcolunas (linha 3)
    endereco_op_subsubheaders = [
        (23, "IP ADDRESS"),
        (24, "NETMASK")
    ]
    for col_idx, header_text in endereco_op_subsubheaders:
        sub_cell = we.cell(row=header_row_3, column=col_idx, value=header_text)
        sub_cell.font = Font(name="Calibri", size=7, bold=True, color="FFFFFF")
        sub_cell.alignment = center_align
        sub_cell.fill = fill_header_light
        sub_cell.border = box_border

    # Dados da tabela principal
    data_start_row = header_row_3 + 1
    num_data_rows = 15

    for row_idx in range(num_data_rows):
        we.row_dimensions[data_start_row + row_idx].height = 22.0

    for row_offset in range(num_data_rows):
        current_row = data_start_row + row_offset
        row_fill = fill_background if row_offset % 2 == 0 else fill_white
        
        # Criar células APENAS para as colunas da tabela (2 a 24)
        for col_idx in range(2, 25):
            cell = we.cell(row=current_row, column=col_idx, value="")
            cell.font = Font(name="Calibri", size=9)
            cell.alignment = center_align
            cell.border = box_border
            cell.fill = row_fill

    # Configurações de view
    we.freeze_panes = f"{_col(2)}{data_start_row}"
    we.sheet_view.showGridLines = False
    we.page_setup.paperSize = we.PAPERSIZE_A4
    we.page_setup.orientation = we.ORIENTATION_LANDSCAPE
    we.print_options.horizontalCentered = True
    we.page_margins.left = we.page_margins.right = 0.4
    we.page_margins.top = we.page_margins.bottom = 0.4

    # REMOVER BORDAS DE TODAS AS CÉLULAS FORA DA TABELA
    empty_border = Border()

    # 1. Remover bordas das colunas de espaçamento (1 e 25+)
    for col_idx in [1, 25, 26, 27, 28, 29, 30]:
        we.column_dimensions[_col(col_idx)].width = 2.0
        for row_idx in range(1, 50):
            try:
                cell = we.cell(row=row_idx, column=col_idx)
                cell.fill = PatternFill("solid", fgColor="FFFFFF")
                cell.border = empty_border
            except:
                pass

    # 2. Remover bordas das linhas antes do cabeçalho corporativo
    for row_idx in range(1, brand_row_num):
        we.row_dimensions[row_idx].height = 2.0
        for col_idx in range(1, 30):
            try:
                cell = we.cell(row=row_idx, column=col_idx)
                cell.fill = PatternFill("solid", fgColor="FFFFFF")
                cell.border = empty_border
            except:
                pass

    # 3. Remover bordas da linha entre cabeçalho corporativo e título
    row_between_brand_title = brand_row_num + 1
    we.row_dimensions[row_between_brand_title].height = 2.0
    for col_idx in range(1, 30):
        try:
            cell = we.cell(row=row_between_brand_title, column=col_idx)
            cell.fill = PatternFill("solid", fgColor="FFFFFF")
            cell.border = empty_border
        except:
            pass

    # 4. Remover bordas da linha entre título e cabeçalho da tabela
    row_between_title_header = title_enc_row + 1
    we.row_dimensions[row_between_title_header].height = 2.0
    for col_idx in range(1, 30):
        try:
            cell = we.cell(row=row_between_title_header, column=col_idx)
            cell.fill = PatternFill("solid", fgColor="FFFFFF")
            cell.border = empty_border
        except:
            pass

    # 5. Remover bordas das linhas após os dados da tabela
    first_empty_row_after_data = data_start_row + num_data_rows
    for row_idx in range(first_empty_row_after_data, first_empty_row_after_data + 10):
        we.row_dimensions[row_idx].height = 2.0
        for col_idx in range(1, 30):
            try:
                cell = we.cell(row=row_idx, column=col_idx)
                cell.fill = PatternFill("solid", fgColor="FFFFFF")
                cell.border = empty_border
            except:
                pass

    # 6. Garantir que células específicas fora da área da tabela não tenham bordas
    for row_idx in range(1, brand_row_num):
        for col_idx in range(2, 25):
            try:
                cell = we.cell(row=row_idx, column=col_idx)
                cell.fill = PatternFill("solid", fgColor="FFFFFF")
                cell.border = empty_border
            except:
                pass

    for row_idx in [brand_row_num + 1]:
        for col_idx in range(2, 25):
            try:
                cell = we.cell(row=row_idx, column=col_idx)
                cell.fill = PatternFill("solid", fgColor="FFFFFF")
                cell.border = empty_border
            except:
                pass

    for row_idx in [title_enc_row + 1]:
        for col_idx in range(2, 25):
            try:
                cell = we.cell(row=row_idx, column=col_idx)
                cell.fill = PatternFill("solid", fgColor="FFFFFF")
                cell.border = empty_border
            except:
                pass

    # =========================================================================
    # ABA 5: CONCENTRAÇÃO - TABELA DE CONCENTRAÇÃO DE ALS (SEM BORDAS E SEM DADOS)
    # =========================================================================
    wc = wb.create_sheet(title="Concentração")

    # Configuração precisa de colunas
    column_widths_conc = {
        1: 2.0, 2: 2.0, 3: 8.0, 4: 8.0, 5: 12.0, 6: 12.0, 7: 25.0,
    }

    for col_idx, width in column_widths_conc.items():
        wc.column_dimensions[_col(col_idx)].width = width

    # Título principal (SEM BORDA)
    title_conc_row = 2
    wc.merge_cells(start_row=title_conc_row, start_column=3, end_row=title_conc_row, end_column=7)
    title_conc_cell = wc.cell(row=title_conc_row, column=3, value="2.5 - Concentração")
    title_conc_cell.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
    title_conc_cell.alignment = left_align
    title_conc_cell.fill = fill_header_primary
    # SEM BORDA
    wc.row_dimensions[title_conc_row].height = 22.0  # Aumentada para garantir visibilidade

    # Descrição (SEM BORDA) - ALTURA MAIOR PARA TEXTO COMPLETO
    desc_conc_row = title_conc_row + 1
    wc.merge_cells(start_row=desc_conc_row, start_column=3, end_row=desc_conc_row, end_column=7)
    desc_conc_cell = wc.cell(row=desc_conc_row, column=3, value="A 'Operadora' informou interesse em realizar a concentração de ALs conforme detalhado na tabela abaixo:")
    desc_conc_cell.font = Font(name="Calibri", size=10, bold=False)
    desc_conc_cell.alignment = Alignment(horizontal="left", vertical="center", wrap_text=True)  # Wrap text ativado
    desc_conc_cell.fill = fill_background
    # SEM BORDA
    wc.row_dimensions[desc_conc_row].height = 25.0  # Altura suficiente para o texto

    # Instrução (SEM BORDA) - ALTURA MAIOR PARA TEXTO COMPLETO
    inst_conc_row = desc_conc_row + 1
    wc.merge_cells(start_row=inst_conc_row, start_column=3, end_row=inst_conc_row, end_column=7)
    inst_conc_cell = wc.cell(row=inst_conc_row, column=3, value='Caso necessite de incluir mais ALs, Favor inserir linhas adicionais de acordo com reunião que foi acordado o tema (coluna B - "Ref").')
    inst_conc_cell.font = Font(name="Calibri", size=9, bold=False, italic=True)
    inst_conc_cell.alignment = Alignment(horizontal="left", vertical="center", wrap_text=True)  # Wrap text ativado
    inst_conc_cell.fill = fill_background
    # SEM BORDA
    wc.row_dimensions[inst_conc_row].height = 30.0  # Altura maior para texto mais longo

    # Cabeçalhos da tabela (SEM BORDAS) - MANTIDOS PARA ESTRUTURA
    header_conc_row = inst_conc_row + 2
    wc.row_dimensions[header_conc_row].height = 25.0  # Altura adequada para cabeçalhos

    conc_headers = [
        ("Ref", 3), ("CN", 4), ("CNL", 5), ("Cod. CnI", 6), ("Status Link", 7)
    ]

    for header_text, col_idx in conc_headers:
        header_cell = wc.cell(row=header_conc_row, column=col_idx, value=header_text)
        header_cell.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
        header_cell.alignment = center_align
        header_cell.fill = fill_header_secondary
        # SEM BORDA

    # Área para dados futuros (linhas vazias) - SERÁ PREENCHIDA POSTERIORMENTE COM DADOS DO FORMULÁRIO
    # Dados reais da concentração - SERÃO PREENCHIDOS COM DADOS DO FORMULÁRIO
    data_conc_start_row = header_conc_row + 1
    concentracao_data = _get_concentracao_data()

    # Criar linhas com dados reais ou vazios para estrutura
    for row_offset in range(10):
        current_row = data_conc_start_row + row_offset
        wc.row_dimensions[current_row].height = 22.0
        row_fill = fill_background if row_offset % 2 == 0 else fill_white
        
        # Obter dados reais se existirem
        data_row = concentracao_data[row_offset] if row_offset < len(concentracao_data) else {}
        
        # Ref
        ref_cell = wc.cell(row=current_row, column=3, value=data_row.get("ref", ""))
        ref_cell.font = Font(name="Calibri", size=10)
        ref_cell.alignment = center_align
        # SEM BORDA
        ref_cell.fill = row_fill
        
        # CN
        cn_cell = wc.cell(row=current_row, column=4, value=data_row.get("cn", ""))
        cn_cell.font = Font(name="Calibri", size=10)
        cn_cell.alignment = center_align
        # SEM BORDA
        cn_cell.fill = row_fill
        
        # CNL
        cnl_cell = wc.cell(row=current_row, column=5, value=data_row.get("cnl", ""))
        cnl_cell.font = Font(name="Calibri", size=10)
        cnl_cell.alignment = center_align
        # SEM BORDA
        cnl_cell.fill = row_fill
        
        # Cod. CnI
        cod_cni_cell = wc.cell(row=current_row, column=6, value=data_row.get("cod_cni", ""))
        cod_cni_cell.font = Font(name="Calibri", size=10)
        cod_cni_cell.alignment = center_align
        # SEM BORDA
        cod_cni_cell.fill = row_fill
        
        # Status Link
        status_cell = wc.cell(row=current_row, column=7, value=data_row.get("status_link", ""))
        status_cell.font = Font(name="Calibri", size=10)
        status_cell.alignment = left_align
        # SEM BORDA
        status_cell.fill = row_fill

    # Configurações de view
    wc.freeze_panes = f"{_col(3)}{data_conc_start_row}"
    wc.sheet_view.showGridLines = False
    wc.page_setup.paperSize = wc.PAPERSIZE_A4
    wc.page_setup.orientation = wc.ORIENTATION_PORTRAIT
    wc.print_options.horizontalCentered = True
    wc.page_margins.left = wc.page_margins.right = 0.4
    wc.page_margins.top = wc.page_margins.bottom = 0.5


    # =========================================================================
    # ABA 7: PLAN NUM_OPER - TABELA DE PREFIXOS (SEM DADOS, BORDAS APENAS NAS TABELAS)
    # =========================================================================
    wp = wb.create_sheet(title="Plan Num_Oper")

    # Configuração ultra-precisão de colunas
    plan_num_column_widths = {
        1: 2.0,    # Espaçamento
        2: 4.0,    # Ref
        3: 12.0,   # Município
        4: 12.0,   # ÁREA LOCAL
        5: 12.0,   # CÓDIGO CNL
        6: 6.0,    # CN
        7: 10.0,   # INÍCIO SÉRIE
        8: 10.0,   # FIM SÉRIE
        9: 8.0,    # EOT LOCAL
        10: 15.0,  # SNOA STFC / SMP
        11: 12.0,  # Parâmetros RN1
        12: 12.0,  # Valores parâmetros
        13: 15.0,  # Transbordo UF
        14: 12.0,  # Transbordo Série 1
        15: 12.0,  # Transbordo Série 2,
    }

    for col_idx, width in plan_num_column_widths.items():
        wp.column_dimensions[_col(col_idx)].width = width

    # Cabeçalho corporativo (SEM BORDA)
    wp.merge_cells(start_row=brand_row_num, start_column=2, end_row=brand_row_num, end_column=10)
    brand_cell_p = wp.cell(row=brand_row_num, column=2, value=f"VIVO — {nome_operadora}")
    brand_cell_p.font = brand_font
    brand_cell_p.alignment = center_align
    brand_cell_p.fill = fill_header_primary
    # SEM BORDA
    wp.row_dimensions[brand_row_num].height = 24.0

    # Título principal (SEM BORDA)
    title_row = brand_row_num + 2
    wp.merge_cells(start_row=title_row, start_column=2, end_row=title_row, end_column=10)
    title_cell = wp.cell(row=title_row, column=2, value="2.6. Tabela de Prefixos da IMPERIAL")
    title_cell.font = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
    title_cell.alignment = left_align
    title_cell.fill = fill_header_secondary
    # SEM BORDA
    wp.row_dimensions[title_row].height = 22.0

    # Observação (SEM BORDA)
    obs_row = title_row + 1
    wp.merge_cells(start_row=obs_row, start_column=2, end_row=obs_row, end_column=10)
    obs_cell = wp.cell(row=obs_row, column=2, 
                    value='OBS: Caso necessite de incluir mais faixas, favor inserir linhas adicionais de acordo com reunião que foi acordado o tema (coluna B - "Ref").')
    obs_cell.font = Font(name="Calibri", size=9, italic=True)
    obs_cell.alignment = Alignment(horizontal="left", vertical="center", wrap_text=True)
    obs_cell.fill = fill_background
    # SEM BORDA
    wp.row_dimensions[obs_row].height = 25.0

    # Cabeçalho da tabela principal - Linha 1 (COM BORDAS)
    header1_row = obs_row + 2
    wp.row_dimensions[header1_row].height = 25.0

    # Cabeçalhos individuais (COM BORDAS)
    headers_line1 = [
        (2, "Ref"),
        (3, "Município"),
        (4, "ÁREA LOCAL"),
        (5, "CÓDIGO CNL"),
        (6, "CN"),
        (7, "SERIE AUTORIZADA"),
        (9, "EOT LOCAL"),
        (10, "SNOA STFC / SMP")
    ]

    for col_idx, header_text in headers_line1:
        cell = wp.cell(row=header1_row, column=col_idx, value=header_text)
        cell.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
        cell.alignment = center_align
        cell.fill = fill_header_secondary
        cell.border = box_border  # COM BORDA

    # Mesclagens
    wp.merge_cells(start_row=header1_row, start_column=2, end_row=header1_row + 1, end_column=2)  # Ref
    wp.merge_cells(start_row=header1_row, start_column=3, end_row=header1_row + 1, end_column=3)  # Município
    wp.merge_cells(start_row=header1_row, start_column=4, end_row=header1_row + 1, end_column=4)  # ÁREA LOCAL
    wp.merge_cells(start_row=header1_row, start_column=5, end_row=header1_row + 1, end_column=5)  # CÓDIGO CNL
    wp.merge_cells(start_row=header1_row, start_column=6, end_row=header1_row + 1, end_column=6)  # CN
    wp.merge_cells(start_row=header1_row, start_column=7, end_row=header1_row, end_column=8)     # SÉRIE AUTORIZADA
    wp.merge_cells(start_row=header1_row, start_column=9, end_row=header1_row + 1, end_column=9)  # EOT LOCAL
    wp.merge_cells(start_row=header1_row, start_column=10, end_row=header1_row + 1, end_column=10) # SNOA STFC / SMP

    # Cabeçalho linha 2 (subcabeçalhos da SÉRIE AUTORIZADA) (COM BORDAS)
    header2_row = header1_row + 1
    wp.row_dimensions[header2_row].height = 30.0

    serie_headers = [
        (7, "INCLUÍNOS\nN8 N7 N6 N5"),
        (8, "FINAL\nN4 N3 N2 N1")
    ]

    for col_idx, header_text in serie_headers:
        cell = wp.cell(row=header2_row, column=col_idx, value=header_text[1])
        cell.font = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        cell.alignment = Alignment(horizontal="center", vertical="center", wrap_text=True)
        cell.fill = fill_accent
        cell.border = box_border  # COM BORDA

    # Dados da tabela principal - LINHAS VAZIAS PARA PREENCHER POSTERIORMENTE (COM BORDAS)
    data_start_row = header2_row + 1

    # Criar 5 linhas vazias para estrutura da tabela principal
    for row_idx in range(5):
        current_row = data_start_row + row_idx
        wp.row_dimensions[current_row].height = 20.0
        row_fill = fill_background if row_idx % 2 == 0 else fill_white
        
        for col_idx in range(2, 11):  # Colunas 2 a 10
            cell = wp.cell(row=current_row, column=col_idx, value="")
            cell.font = Font(name="Calibri", size=9)
            cell.alignment = center_align
            cell.border = box_border  # COM BORDA
            cell.fill = row_fill

    # Tabela de parâmetros - ABAIXO DA TABELA PRINCIPAL (COM BORDAS)
    params_start_row = data_start_row + 6

    # Primeira coluna de parâmetros - ESTRUTURA COM CÉLULAS AO LADO
    param_col1 = [
        ("RN1", ""),
        ("EOT Local", ""),
        ("EOT LDN/LDI", ""),
        ("CSP", ""),
        ("Código Especial:", ""),
        ("CNG:", ""),
        ("RN2", "")
    ]

    for row_offset, (param, value) in enumerate(param_col1):
        current_row = params_start_row + row_offset
        wp.row_dimensions[current_row].height = 20.0
        row_fill = fill_background if row_offset % 2 == 0 else fill_white
        
        # Parâmetro (COM BORDA)
        param_cell = wp.cell(row=current_row, column=2, value=param)
        param_cell.font = Font(name="Calibri", size=9, bold=True)
        param_cell.alignment = left_align
        param_cell.border = box_border  # COM BORDA
        param_cell.fill = row_fill
        
        # Valor (COM BORDA) - CÉLULA VAZIA AO LADO PARA ESCREVER
        value_cell = wp.cell(row=current_row, column=3, value=value)
        value_cell.font = Font(name="Calibri", size=9)
        value_cell.alignment = left_align
        value_cell.border = box_border  # COM BORDA
        value_cell.fill = row_fill

    # Textos ao lado direito da tabela de parâmetros
    transporte_row = params_start_row
    wp.merge_cells(start_row=transporte_row, start_column=5, end_row=transporte_row, end_column=10)
    transporte_cell = wp.cell(row=transporte_row, column=5, value="Operadora de transporte nas localidades onde não existir rota direta")
    transporte_cell.font = Font(name="Calibri", size=9, bold=True)
    transporte_cell.alignment = left_align
    transporte_cell.fill = fill_white  # SEM COR (BRANCO)
    # SEM BORDA
    wp.row_dimensions[transporte_row].height = 20.0

    # Texto sobre transbordo (SEM BORDA)
    transbordo_row = transporte_row + 1
    wp.merge_cells(start_row=transbordo_row, start_column=5, end_row=transbordo_row, end_column=10)
    transbordo_cell = wp.cell(row=transbordo_row, column=5, value="Transbordo nas localidades onde existe rota direta:")
    transbordo_cell.font = Font(name="Calibri", size=9, bold=True)
    transbordo_cell.alignment = left_align
    transbordo_cell.fill = fill_white  # SEM COR (BRANCO)
    # SEM BORDA
    wp.row_dimensions[transbordo_row].height = 20.0

    # Tabela de transbordo - ABAIXO DA TABELA DE PARÂMETROS (COM BORDAS)
    transbordo_table_row = params_start_row + len(param_col1) + 1
    wp.row_dimensions[transbordo_table_row].height = 20.0

    # Cabeçalho transbordo - UF (COM BORDA)
    wp.merge_cells(start_row=transbordo_table_row, start_column=2, end_row=transbordo_table_row, end_column=3)
    transbordo_header1 = wp.cell(row=transbordo_table_row, column=2, value="UF")
    transbordo_header1.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    transbordo_header1.alignment = center_align
    transbordo_header1.fill = fill_header_secondary
    transbordo_header1.border = box_border  # COM BORDA

    # Cabeçalho transbordo - SÉRIE AUTORIZADA (COM BORDA)
    wp.merge_cells(start_row=transbordo_table_row, start_column=4, end_row=transbordo_table_row, end_column=7)
    transbordo_header2 = wp.cell(row=transbordo_table_row, column=4, value="SERIE AUTORIZADA")
    transbordo_header2.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
    transbordo_header2.alignment = center_align
    transbordo_header2.fill = fill_header_secondary
    transbordo_header2.border = box_border  # COM BORDA

    # Subcabeçalhos transbordo - CÉLULAS VAZIAS PARA ESCREVER (COM BORDAS)
    transbordo_subheader_row = transbordo_table_row + 1
    wp.row_dimensions[transbordo_subheader_row].height = 20.0

    # UF (COM BORDA) - CÉLULA VAZIA PARA ESCREVER
    wp.merge_cells(start_row=transbordo_subheader_row, start_column=2, end_row=transbordo_subheader_row, end_column=3)
    uf_cell = wp.cell(row=transbordo_subheader_row, column=2, value="")
    uf_cell.font = Font(name="Calibri", size=9)
    uf_cell.alignment = center_align
    uf_cell.border = box_border  # COM BORDA
    uf_cell.fill = fill_background

    # Série 1 (COM BORDA) - CÉLULA VAZIA PARA ESCREVER
    wp.merge_cells(start_row=transbordo_subheader_row, start_column=4, end_row=transbordo_subheader_row, end_column=5)
    n11_n8_cell = wp.cell(row=transbordo_subheader_row, column=4, value="")
    n11_n8_cell.font = Font(name="Calibri", size=9)
    n11_n8_cell.alignment = center_align
    n11_n8_cell.border = box_border  # COM BORDA
    n11_n8_cell.fill = fill_background

    # Série 2 (COM BORDA) - CÉLULA VAZIA PARA ESCREVER
    wp.merge_cells(start_row=transbordo_subheader_row, start_column=6, end_row=transbordo_subheader_row, end_column=7)
    n7_n1_cell = wp.cell(row=transbordo_subheader_row, column=6, value="")
    n7_n1_cell.font = Font(name="Calibri", size=9)
    n7_n1_cell.alignment = center_align
    n7_n1_cell.border = box_border  # COM BORDA
    n7_n1_cell.fill = fill_background

    # Texto sobre CNG (SEM BORDA E SEM COR)
    cng_row = transbordo_subheader_row + 2
    wp.merge_cells(start_row=cng_row, start_column=2, end_row=cng_row, end_column=10)
    cng_cell = wp.cell(row=cng_row, column=2, value="A programação de CNG ocorre pelo processo de portabilidade intrínseca - via ABR")
    cng_cell.font = Font(name="Calibri", size=9)
    cng_cell.alignment = left_align
    cng_cell.fill = fill_white  # SEM COR (BRANCO)
    # SEM BORDA
    wp.row_dimensions[cng_row].height = 18.0

    cng_row2 = cng_row + 1
    wp.merge_cells(start_row=cng_row2, start_column=2, end_row=cng_row2, end_column=10)
    cng_cell2 = wp.cell(row=cng_row2, column=2, value="O encaminhamento do CNG será programado nas localidades onde o ponto de entrega for informado")
    cng_cell2.font = Font(name="Calibri", size=9)
    cng_cell2.alignment = left_align
    cng_cell2.fill = fill_white  # SEM COR (BRANCO)
    # SEM BORDA
    wp.row_dimensions[cng_row2].height = 18.0

    # Título Vivo SMP (SEM BORDA E SEM COR)
    vivo_smp_row = cng_row2 + 2
    wp.merge_cells(start_row=vivo_smp_row, start_column=2, end_row=vivo_smp_row, end_column=10)
    vivo_smp_cell = wp.cell(row=vivo_smp_row, column=2, value="2.12 Tabela de Prefixos Vivo SMP")
    vivo_smp_cell.font = Font(name="Calibri", size=10, bold=True)
    vivo_smp_cell.alignment = left_align
    vivo_smp_cell.fill = fill_white  # SEM COR (BRANCO)
    # SEM BORDA
    wp.row_dimensions[vivo_smp_row].height = 20.0

    # Texto Vivo SMP (SEM BORDA E SEM COR)
    vivo_smp_text_row = vivo_smp_row + 1
    wp.merge_cells(start_row=vivo_smp_text_row, start_column=2, end_row=vivo_smp_text_row, end_column=10)
    vivo_smp_text_cell = wp.cell(row=vivo_smp_text_row, column=2, 
                                value="Os prefixos da VIVO SMP, poderão ser obtidos acessando o site www.telefonica.net.br/sp/transfer/")
    vivo_smp_text_cell.font = Font(name="Calibri", size=9)
    vivo_smp_text_cell.alignment = left_align
    vivo_smp_text_cell.fill = fill_white  # SEM COR (BRANCO)
    # SEM BORDA
    wp.row_dimensions[vivo_smp_text_row].height = 18.0

    # Configurações de view
    wp.freeze_panes = f"{_col(2)}{header1_row}"
    wp.sheet_view.showGridLines = False
    wp.page_setup.paperSize = wp.PAPERSIZE_A4
    wp.page_setup.orientation = wp.ORIENTATION_LANDSCAPE
    wp.print_options.horizontalCentered = True
    wp.page_margins.left = wp.page_margins.right = 0.4
    wp.page_margins.top = wp.page_margins.bottom = 0.5

    # =========================================================================
    # ABA 8: DADOS MTL - TABELA DE DADOS MTL (CORRIGIDA - SEM AMARELO)
    # =========================================================================
    wmtl = wb.create_sheet(title="Dados MTL")

    # Configuração de colunas
    mtl_column_widths = {
        1: 2.0,    # Espaçamento
        2: 6.0,    # Ref
        3: 8.0,    # Cn
        4: 10.0,   # ID
        5: 12.0,   # MTL
        6: 15.0,   # PONTA VIVO
        7: 15.0,   # PONTA IMPERIAL
        8: 20.0,   # DESIGNADOR DO CIRCUITO
        9: 15.0,   # PROVEDOR
        10: 18.0,  # Posição Física VIVO
        11: 18.0,  # Posição Física OPERADORA
        12: 20.0,  # OBSERVAÇÕES
    }

    for col_idx, width in mtl_column_widths.items():
        wmtl.column_dimensions[_col(col_idx)].width = width

    # Cabeçalho corporativo (SEM BORDA)
    wmtl.merge_cells(start_row=brand_row_num, start_column=2, end_row=brand_row_num, end_column=11)
    brand_cell_mtl = wmtl.cell(row=brand_row_num, column=2, value=f"VIVO — {nome_operadora}")
    brand_cell_mtl.font = brand_font
    brand_cell_mtl.alignment = center_align
    brand_cell_mtl.fill = fill_header_primary
    # SEM BORDA
    wmtl.row_dimensions[brand_row_num].height = 24.0

    # Título principal (SEM BORDA)
    title_mtl_row = brand_row_num + 2
    wmtl.merge_cells(start_row=title_mtl_row, start_column=2, end_row=title_mtl_row, end_column=11)
    title_mtl_cell = wmtl.cell(row=title_mtl_row, column=2, value="2.7. Dados de MTL")
    title_mtl_cell.font = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
    title_mtl_cell.alignment = left_align
    title_mtl_cell.fill = fill_header_secondary
    # SEM BORDA
    wmtl.row_dimensions[title_mtl_row].height = 22.0

    # Descrição (SEM BORDA)
    desc_mtl_row = title_mtl_row + 1
    wmtl.merge_cells(start_row=desc_mtl_row, start_column=2, end_row=desc_mtl_row, end_column=11)
    desc_mtl_cell = wmtl.cell(row=desc_mtl_row, column=2, 
                            value="Tabela de dados MTL (Meio de Transmissão de Longa Distância) para interligação entre VIVO e operadora.")
    desc_mtl_cell.font = Font(name="Calibri", size=10, bold=False)
    desc_mtl_cell.alignment = left_align
    desc_mtl_cell.fill = fill_background
    # SEM BORDA
    wmtl.row_dimensions[desc_mtl_row].height = 18.0

    # Instrução (SEM BORDA)
    inst_mtl_row = desc_mtl_row + 1
    wmtl.merge_cells(start_row=inst_mtl_row, start_column=2, end_row=inst_mtl_row, end_column=11)
    inst_mtl_cell = wmtl.cell(row=inst_mtl_row, column=2, 
                            value="Preencher com os dados técnicos dos circuitos de interligação.")
    inst_mtl_cell.font = Font(name="Calibri", size=9, bold=False, italic=True)
    inst_mtl_cell.alignment = left_align
    inst_mtl_cell.fill = fill_background
    # SEM BORDA
    wmtl.row_dimensions[inst_mtl_row].height = 16.0

    # Cabeçalhos da tabela MTL (COM BORDAS)
    header_mtl_row = inst_mtl_row + 2
    wmtl.row_dimensions[header_mtl_row].height = 25.0

    mtl_headers = [
        ("Ref", 2),
        ("Cn", 3),
        ("ID", 4),
        ("MTL", 5),
        ("PONTA VIVO", 6),
        ("PONTA IMPERIAL", 7),
        ("DESIGNADOR DO CIRCUITO", 8),
        ("PROVEDOR", 9),
        ("Posição Física VIVO", 10),
        ("Posição Física OPERADORA", 11),
        ("OBSERVAÇÕES", 12)
    ]

    for header_text, col_idx in mtl_headers:
        header_cell = wmtl.cell(row=header_mtl_row, column=col_idx, value=header_text)
        header_cell.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
        header_cell.alignment = center_align
        header_cell.fill = fill_header_secondary
        header_cell.border = box_border

    # Dados da tabela MTL (10 linhas vazias) - COM PADRÃO ROXO PÚRPURA
    data_mtl_start_row = header_mtl_row + 1
    num_mtl_rows = 10

    for row_offset in range(num_mtl_rows):
        current_row = data_mtl_start_row + row_offset
        wmtl.row_dimensions[current_row].height = 20.0
        row_fill = fill_background if row_offset % 2 == 0 else fill_white
        
        # Ref
        ref_cell = wmtl.cell(row=current_row, column=2, value=row_offset + 1)
        ref_cell.font = Font(name="Calibri", size=9)
        ref_cell.alignment = center_align
        ref_cell.border = box_border
        ref_cell.fill = row_fill
        
        # Cn - MESMA COR DA LINHA (REMOVIDO AMARELO)
        cn_cell = wmtl.cell(row=current_row, column=3, value="")
        cn_cell.font = Font(name="Calibri", size=9)
        cn_cell.alignment = center_align
        cn_cell.border = box_border
        cn_cell.fill = row_fill
        
        # ID - MESMA COR DA LINHA (REMOVIDO AMARELO)
        id_cell = wmtl.cell(row=current_row, column=4, value="")
        id_cell.font = Font(name="Calibri", size=9)
        id_cell.alignment = center_align
        id_cell.border = box_border
        id_cell.fill = row_fill
        
        # MTL - MESMA COR DA LINHA (REMOVIDO AMARELO)
        mtl_cell = wmtl.cell(row=current_row, column=5, value="")
        mtl_cell.font = Font(name="Calibri", size=9)
        mtl_cell.alignment = center_align
        mtl_cell.border = box_border
        mtl_cell.fill = row_fill
        
        # PONTA VIVO - MESMA COR DA LINHA (REMOVIDO AMARELO)
        ponta_vivo_cell = wmtl.cell(row=current_row, column=6, value="")
        ponta_vivo_cell.font = Font(name="Calibri", size=9)
        ponta_vivo_cell.alignment = center_align
        ponta_vivo_cell.border = box_border
        ponta_vivo_cell.fill = row_fill
        
        # PONTA IMPERIAL - MESMA COR DA LINHA (REMOVIDO AMARELO)
        ponta_imperial_cell = wmtl.cell(row=current_row, column=7, value="")
        ponta_imperial_cell.font = Font(name="Calibri", size=9)
        ponta_imperial_cell.alignment = center_align
        ponta_imperial_cell.border = box_border
        ponta_imperial_cell.fill = row_fill
        
        # DESIGNADOR DO CIRCUITO - MESMA COR DA LINHA (REMOVIDO AMARELO)
        designador_cell = wmtl.cell(row=current_row, column=8, value="")
        designador_cell.font = Font(name="Calibri", size=9)
        designador_cell.alignment = center_align
        designador_cell.border = box_border
        designador_cell.fill = row_fill
        
        # PROVEDOR - MESMA COR DA LINHA (REMOVIDO AMARELO)
        provedor_cell = wmtl.cell(row=current_row, column=9, value="")
        provedor_cell.font = Font(name="Calibri", size=9)
        provedor_cell.alignment = center_align
        provedor_cell.border = box_border
        provedor_cell.fill = row_fill
        
        # Posição Física VIVO - MESMA COR DA LINHA (REMOVIDO AMARELO)
        pos_vivo_cell = wmtl.cell(row=current_row, column=10, value="")
        pos_vivo_cell.font = Font(name="Calibri", size=9)
        pos_vivo_cell.alignment = center_align
        pos_vivo_cell.border = box_border
        pos_vivo_cell.fill = row_fill
        
        # Posição Física OPERADORA - MESMA COR DA LINHA (REMOVIDO AMARELO)
        pos_op_cell = wmtl.cell(row=current_row, column=11, value="")
        pos_op_cell.font = Font(name="Calibri", size=9)
        pos_op_cell.alignment = center_align
        pos_op_cell.border = box_border
        pos_op_cell.fill = row_fill
        
        # OBSERVAÇÕES - MESMA COR DA LINHA (REMOVIDO AMARELO)
        obs_cell = wmtl.cell(row=current_row, column=12, value="")
        obs_cell.font = Font(name="Calibri", size=9)
        obs_cell.alignment = left_align
        obs_cell.border = box_border
        obs_cell.fill = row_fill

    # REMOVIDO O AVISO SOBRE CAMPOS AMARELOS (não se aplica mais)

    # Configurações de view
    wmtl.freeze_panes = f"{_col(2)}{data_mtl_start_row}"
    wmtl.sheet_view.showGridLines = False
    wmtl.page_setup.paperSize = wmtl.PAPERSIZE_A4
    wmtl.page_setup.orientation = wmtl.ORIENTATION_LANDSCAPE
    wmtl.print_options.horizontalCentered = True
    wmtl.page_margins.left = wmtl.page_margins.right = 0.4
    wmtl.page_margins.top = wmtl.page_margins.bottom = 0.5

    # =========================================================================
    # ABA 9: PARÂMETROS DE PROGRAMAÇÃO (CORRIGIDA - PADRÃO ROXO PÚRPURA)
    # =========================================================================
    wparam = wb.create_sheet(title="Parâmetros de Programação")

    # Configuração de colunas
    param_column_widths = {
        1: 2.0,    # Espaçamento
        2: 50.0,   # Parâmetro
        3: 35.0,   # TELEFÔNICA VIVO <> IMPERIAL
        4: 15.0,   # Categoria
        5: 12.0,   # Cumpre?
        6: 35.0    # Observação
    }
    for col_idx, width in param_column_widths.items():
        wparam.column_dimensions[_col(col_idx)].width = width

    # Cabeçalho corporativo (SEM BORDA)
    wparam.merge_cells(start_row=brand_row_num, start_column=2, end_row=brand_row_num, end_column=6)
    brand_cell_param = wparam.cell(row=brand_row_num, column=2, value=f"VIVO — {nome_operadora}")
    brand_cell_param.font = brand_font
    brand_cell_param.alignment = center_align
    brand_cell_param.fill = fill_header_primary
    # SEM BORDA
    wparam.row_dimensions[brand_row_num].height = 24.0

    # Título principal (SEM BORDA)
    title_param_row = brand_row_num + 2
    wparam.merge_cells(start_row=title_param_row, start_column=2, end_row=title_param_row, end_column=6)
    title_param_cell = wparam.cell(row=title_param_row, column=2, value="2.9 - Parâmetros de Programação")
    title_param_cell.font = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
    title_param_cell.alignment = left_align
    title_param_cell.fill = fill_header_secondary
    # SEM BORDA
    wparam.row_dimensions[title_param_row].height = 22.0

    # Descrição (SEM BORDA)
    desc_param_row = title_param_row + 1
    wparam.merge_cells(start_row=desc_param_row, start_column=2, end_row=desc_param_row, end_column=6)
    desc_param_cell = wparam.cell(row=desc_param_row, column=2, 
        value="Na tabela abaixo, possui a relação dos parâmetros de configuração de rotas SIP entre a TELEFÔNICA-VIVO e a Operadora IMPERIAL.")
    desc_param_cell.font = Font(name="Calibri", size=10)
    desc_param_cell.alignment = left_align
    desc_param_cell.fill = fill_background
    # SEM BORDA
    wparam.row_dimensions[desc_param_row].height = 18.0

    # Cabeçalho roxo púrpura (SUBSTITUINDO O VERDE)
    header_row = desc_param_row + 2
    wparam.merge_cells(start_row=header_row, start_column=2, end_row=header_row, end_column=6)
    header_cell = wparam.cell(row=header_row, column=2, value="Interconexão Nacional")
    header_cell.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
    header_cell.alignment = center_align
    header_cell.fill = fill_header_primary  # ROXO PÚRPURA PRINCIPAL
    header_cell.border = box_border
    wparam.row_dimensions[header_row].height = 22.0

    # Funções auxiliares atualizadas para usar roxo púrpura
    def criar_cabecalho_roxo(row, texto):
        wparam.merge_cells(start_row=row, start_column=2, end_row=row, end_column=6)
        cell = wparam.cell(row=row, column=2, value=texto)
        cell.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
        cell.alignment = center_align
        cell.fill = fill_header_light  # ROXO CLARO
        cell.border = box_border
        wparam.row_dimensions[row].height = 20.0

    def preencher_linhas(start_row, dados):
        for i, linha in enumerate(dados):
            current_row = start_row + i
            fill_color = fill_background if i % 2 == 0 else fill_white
            for col_idx, valor in enumerate(linha, start=2):
                cell = wparam.cell(row=current_row, column=col_idx, value=valor)
                cell.font = Font(name="Calibri", size=9)
                cell.alignment = center_align if col_idx != 2 else left_align
                cell.fill = fill_color
                cell.border = box_border
                if col_idx == 5 and valor == "X":
                    cell.font = Font(name="Calibri", size=11, bold=True, color="FF0000")

    # Cabeçalho Informação do Gateway (ROXO CLARO)
    subheader_row = header_row + 1
    criar_cabecalho_roxo(subheader_row, "Informação do Gateway")

    # Cabeçalhos da tabela (ROXO SECUNDÁRIO)
    header_table_row = subheader_row + 1
    param_headers = ["Parâmetro", "TELEFÔNICA VIVO <> IMPERIAL", "Categoria", "Cumpre?", "Observação"]
    for col_idx, header_text in enumerate(param_headers, start=2):
        cell = wparam.cell(row=header_table_row, column=col_idx, value=header_text)
        cell.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
        cell.alignment = center_align
        cell.fill = fill_header_secondary  # ROXO SECUNDÁRIO
        cell.border = box_border
    wparam.row_dimensions[header_table_row].height = 25.0

    # Dados Informação do Gateway
    dados_gateway = [
        ("Fabricante / Fornecedor", "sipwise", "Mandatório", "X", ""),
        ("Modelo - Versão", "Elementos Rede Fixa, Fixa I e Móvel", "Mandatório", "X", "")
    ]
    linha_atual = header_table_row + 1
    preencher_linhas(linha_atual, dados_gateway)

    # Tipo de Serviço (ROXO CLARO)
    linha_atual += len(dados_gateway) + 1
    criar_cabecalho_roxo(linha_atual, "Tipo de Serviço")
    dados_tipo_servico = [
        ("Voz", "SIM", "Mandatório", "X", ""),
        ("Fax", "SIM", "Mandatório", "X", ""),
        ("DTMF", "SIM", "Mandatório", "X", "")
    ]
    preencher_linhas(linha_atual + 1, dados_tipo_servico)

    # Protocolo (ROXO CLARO)
    linha_atual += len(dados_tipo_servico) + 2
    criar_cabecalho_roxo(linha_atual, "Protocolo")
    dados_protocolo = [("Tipo de Protocolo", "SIP-I (Q.1912.5)", "Mandatório", "X", "")]
    preencher_linhas(linha_atual + 1, dados_protocolo)

    # Atributos SIP (ROXO CLARO)
    linha_atual += len(dados_protocolo) + 2
    criar_cabecalho_roxo(linha_atual, "Atributos SIP")
    dados_atributos_sip = [
        ("SIP Version", "SIP v2.1", "Mandatório", "X", ""),
        ("Protocolo de Transporte", "UDP", "Mandatório", "X", ""),
        ("Endereço IP Gateway SIP", "Gateway de Gateway - NNI", "Mandatório", "X", ""),
        ("IP ADDR do RTP", "IP ADDR do RTP", "Mandatório", "X", ""),
        ("FW ou SBC antes do Gateway", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN na URI", "P-Asserted-Identity e Diversion", "Mandatório", "X", ""),
        ("Envia DOMAIN IP", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP para Controle", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP para Controle de QoS", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP para Controle de QoS e Segurança", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Chamada", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Sessão", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Recursos", "Mandatório", "Mandatório", "X", ""),
        ("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Política", "Mandatório", "Mandatório", "X", "")
    ]
    preencher_linhas(linha_atual + 1, dados_atributos_sip)

    # Codec Rede Móvel (ROXO CLARO)
    linha_atual += len(dados_atributos_sip) + 2
    criar_cabecalho_roxo(linha_atual, "Codec Rede Móvel")
    dados_codec_movel = [
        ("AMR encaminhamento", "", "Mandatório", "X", ""),
        ("2ª opção G.711a 20ms", "", "Mandatório", "X", "Não aceita codec g.711u")
    ]
    preencher_linhas(linha_atual + 1, dados_codec_movel)

    # Codec Rede Fixa (ROXO CLARO)
    linha_atual += len(dados_codec_movel) + 2
    criar_cabecalho_roxo(linha_atual, "Codec Rede Fixa")
    dados_codec_fixa = [
        ("G.711a 20ms", "", "Mandatório", "X", ""),
        ("2ª opção G.729a 20ms", "", "Mandatório", "X", "Não aceita codec g.711u")
    ]
    preencher_linhas(linha_atual + 1, dados_codec_fixa)

    # DTMF (ROXO CLARO)
    linha_atual += len(dados_codec_fixa) + 2
    criar_cabecalho_roxo(linha_atual, "DTMF")
    dados_dtmf = [
        ("1ª Opção: RFC 2833 (Outband) com valor payload = 100", "", "Mandatório", "X", ""),
        ("Todos os payloads de DTMF definidos no padrão (96-125) podem ser usados", "", "Mandatório", "X", "")
    ]
    preencher_linhas(linha_atual + 1, dados_dtmf)

    # FAX (ROXO CLARO)
    linha_atual += len(dados_dtmf) + 2
    criar_cabecalho_roxo(linha_atual, "FAX")
    dados_fax = [("Protocolo T.38 suportado", "", "Mandatório", "X", "")]
    preencher_linhas(linha_atual + 1, dados_fax)

    # POS (ROXO CLARO)
    linha_atual += len(dados_fax) + 2
    criar_cabecalho_roxo(linha_atual, "POS")
    dados_pos = [
        ("Detecção do UPSEEED", "Utilização do g711a em re-INVITE", "Mandatório", "X", "Dados necessários no SDP"),
        ("Detecção do UPSEEED", "Utilização do g711u em re-INVITE", "Mandatório", "X", "Dados necessários no SDP")
    ]
    preencher_linhas(linha_atual + 1, dados_pos)

    # Encaminhamento Enfrante (ROXO CLARO)
    linha_atual += len(dados_pos) + 2
    criar_cabecalho_roxo(linha_atual, "Encaminhamento Enfrante")
    dados_encaminhamento = [
        ("P-ASSERTED Identity (RFC 3325)", "SIM - Fomento E.164", "Mandatório", "X", ""),
        ("Nature of address of Calling", "SUB, NAT, UNKW", "Mandatório", "X", ""),
        ("B-NUMBER (Called Number)", "E.164 - Padrão Roteamento", "Mandatório", "X", ""),
        ("Nature of address", "SUB, NAT, UNKW", "Mandatório", "X", "")
    ]
    preencher_linhas(linha_atual + 1, dados_encaminhamento)

    # Encaminhamento Sainte (ROXO CLARO)
    linha_atual += len(dados_encaminhamento) + 2
    criar_cabecalho_roxo(linha_atual, "Encaminhamento Sainte")
    dados_encaminhamento_sainte = [
        ("A-NUMBER (Calling Number)", "E.164 - Nacional", "Mandatório", "X", ""),
        ("P-Asserted Identity (RFC 3325)", "SIM - Formato E.164", "Mandatório", "X", ""),
        ("Nature of address of Calling", "SUB, NAT, UNKW", "Mandatório", "X", ""),
        ("B-NUMBER (Called Number)", "E.164 - Padrão Roteamento", "Mandatório", "X", ""),
        ("Nature of address", "SUB, NAT, UNKW", "Mandatório", "X", "")
    ]
    preencher_linhas(linha_atual + 1, dados_encaminhamento_sainte)

    # Principais RFC's (ROXO CLARO)
    linha_atual += len(dados_encaminhamento_sainte) + 2
    criar_cabecalho_roxo(linha_atual, "Principais RFC's que devem estar habilitadas e/ou suportadas")
    dados_rfcs = [
        ("3261 - SIP Base", "", "", "", ""),
        ("3311 - UPDATE (PRACK)", "", "", "", ""),
        ("3264/4028 - Offer/Answer - SDP", "", "", "", ""),
        ("2833 - DTMF - RTP", "", "", "", ""),
        ("RFC 3398 - SIP-I interworking", "", "", "", ""),
        ("RFC 4033/4035 - DNSSEC", "", "", "", ""),
        ("RFC 4566 - SDP", "", "", "", ""),
        ("RFC 3606 - Early Media & Tone Generation", "", "", "", "SIP 180Ring (puro), o Proxy deverá gerar o RBT localmente")
    ]
    preencher_linhas(linha_atual + 1, dados_rfcs)

    # Método Update (ROXO CLARO)
    linha_atual += len(dados_rfcs) + 2
    criar_cabecalho_roxo(linha_atual, "Método para alteração dos dados do SDP")
    dados_metodo_update = [
        ("Método Update", "", "Mandatório", "", "Antes do Atendimento"),
        ("Método Update", "", "Mandatório", "", "Após atendimento")
    ]
    preencher_linhas(linha_atual + 1, dados_metodo_update)

    # Negociação de Codec (ROXO CLARO)
    linha_atual += len(dados_metodo_update) + 2
    criar_cabecalho_roxo(linha_atual, "Negociação de Codec")
    dados_negociacao_codec = [
        ("É mandatório a utilização de re-invites pelo Origem para a definição de codecs, nas chamadas em que o destino não define um codec único, ou seja, nos casos em que o destino devolve uma lista de codecs para a origem.", "", "Mandatório", "X", ""),
        ("Primeira/Multiple: O ptimo e a magnitude presente no SDP answer deve ser um múltiplo inteiro de 20 para os codecs dos dispositivos móveis; 3GPP 26.114", "", "Mandatório", "X", ""),
        ("Todos os pacotes são suportados às marcações do tipo USR, DSCP 46 ou EF para os pacotes RTP", "", "Mandatório", "X", "")
    ]
    preencher_linhas(linha_atual + 1, dados_negociacao_codec)

    # Configurações de view
    wparam.freeze_panes = f"{_col(2)}{header_table_row + 1}"
    wparam.sheet_view.showGridLines = False
    wparam.page_setup.paperSize = wparam.PAPERSIZE_A4
    wparam.page_setup.orientation = wparam.ORIENTATION_LANDSCAPE
    wparam.print_options.horizontalCentered = True
    wparam.page_margins.left = wparam.page_margins.right = 0.4
    wparam.page_margins.top = wparam.page_margins.bottom = 0.5

    # =========================================================================
    # METADADOS DO DOCUMENTO E FINALIZAÇÃO
    # =========================================================================
    wb.properties.title = f"PTI — Completo — {nome_operadora}"
    wb.properties.subject = "Projeto Técnico de Interligação"
    wb.properties.creator = "VIVOHUB"
    wb.properties.description = "Documento técnico padronizado com 9 abas para interligação entre operadoras."
    wb.properties.keywords = f"PTI, VIVO, {nome_operadora}, Interligação, Telecomunicações"
    wb.properties.category = "Projeto Técnico"
    wb.properties.lastModifiedBy = "VIVOHUB Sistema"

    # Definir aba inicial como Índice
    wb.active = 0

    return wb

# =============================================================================
# Exportação Excel
# =============================================================================
@app.get("/formularios/<int:form_id>/excel_index")
@login_required
def exportar_form_excel_index(form_id: int):
    """
    Exporta Workbook com sete abas completas do PTI.

    Permissões:
    - Atacado: somente o próprio formulário
    - Engenharia: pode exportar qualquer formulário
    """
    if Workbook is None:
        flash("Biblioteca 'openpyxl' não instalada. Peça ao admin para instalar.", "warning")
        return redirect(url_for("index"))

    db = get_db()
    form = db.execute("SELECT * FROM atacado_forms WHERE id = ?", (form_id,)).fetchone()
    if not form:
        abort(404)

    role = session.get("role")
    if role == "atacado" and form["owner_id"] != session.get("user_id"):
        abort(403)

    wb = build_index_excel_wb(form)
    buf = BytesIO()
    wb.save(buf)
    buf.seek(0)

    nome_op = _safe_name(form["nome_operadora"] or "Operadora")
    download_name = f"PTI_Completo_{nome_op}_ID{form['id']}.xlsx"

    return send_file(
        buf,
        mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        as_attachment=True,
        download_name=download_name
    )


# (Opcional) geração de Excel vazio e registro na tabela "exports"
def build_empty_index_wb():
    if Workbook is None:
        raise RuntimeError("Biblioteca 'openpyxl' não instalada.")
    wb = Workbook()
    ws = wb.active
    ws.title = "Índice"
    return wb


@app.post("/engenharia_formularios/<int:form_id>/generate_excel")
@login_required
@role_required("engenharia")
def engenharia_generate_excel(form_id: int):
    if Workbook is None:
        flash("Biblioteca 'openpyxl' não instalada. Peça ao admin para instalar.", "warning")
        return redirect(url_for("engenharia_form_list"))

    db = get_db()
    form = db.execute("SELECT id, nome_operadora FROM atacado_forms WHERE id = ?", (form_id,)).fetchone()
    if not form:
        abort(404)

    wb = build_empty_index_wb()
    ts = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    safe_op = _safe_name(form["nome_operadora"] or "Operadora")
    filename = f"PTI_Indice_{safe_op}_ID{form_id}_{ts}.xlsx"
    filepath = os.path.join(EXPORT_DIR, filename)

    wb.save(filepath)
    size_bytes = os.path.getsize(filepath)

    db.execute(
        "INSERT INTO exports (form_id, filename, filepath, size_bytes) VALUES (?, ?, ?, ?)",
        (form_id, filename, filepath, size_bytes)
    )
    db.commit()

    flash(f"Excel gerado: {filename}", "success")
    return redirect(url_for("engenharia_form_list", show_files="1", form=form_id))


@app.get("/engenharia_exports/<int:export_id>/download")
@login_required
@role_required("engenharia")
def engenharia_export_download(export_id: int):
    db = get_db()
    row = db.execute("SELECT filename, filepath FROM exports WHERE id = ?", (export_id,)).fetchone()
    if not row:
        abort(404)
    if not os.path.exists(row["filepath"]):
        flash("Arquivo não encontrado no disco.", "warning")
        return redirect(url_for("engenharia_form_list", show_files="1"))
    return send_file(
        row["filepath"],
        mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        as_attachment=True,
        download_name=row["filename"]
    )


@app.post("/engenharia_exports/<int:export_id>/delete")
@login_required
@role_required("engenharia")
def engenharia_export_delete(export_id: int):
    db = get_db()
    row = db.execute("SELECT filepath FROM exports WHERE id = ?", (export_id,)).fetchone()
    if not row:
        abort(404)
    try:
        if os.path.exists(row["filepath"]):
            os.remove(row["filepath"])
    except Exception:
        pass
    db.execute("DELETE FROM exports WHERE id = ?", (export_id,))
    db.commit()
    flash("Arquivo removido.", "info")
    return redirect(url_for("engenharia_form_list", show_files="1"))


# =============================================================================
# API interna — CNs
# =============================================================================
@app.get("/api/cns")
@login_required
def api_cns():
    """
    Retorna a lista de CNs ativas carregadas internamente.
    Estrutura: [{"codigo":"11","nome":"","uf":""}, ...]
    """
    db = get_db()
    rows = db.execute(
        "SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf "
        "FROM cns WHERE ativo=1 ORDER BY codigo ASC"
    ).fetchall()
    return jsonify([dict(codigo=r["codigo"], nome=r["nome"], uf=r["uf"]) for r in rows])


# =============================================================================
# Jinja filters
# =============================================================================
@app.template_filter("date_br")
def date_br_filter(val):
    """
    Converte datetime/str -> 'DD/MM/YYYY HH:MM' (se possível).
    Evita TypeError no template (ex.: f.created_at[:16] em datetime).
    """
    if not val:
        return ""
    if hasattr(val, "strftime"):
        try:
            return val.strftime("%d/%m/%Y %H:%M")
        except Exception:
            return str(val)
    try:
        s = str(val).replace("T", " ")
        dt = datetime.fromisoformat(s)
        return dt.strftime("%d/%m/%Y %H:%M")
    except Exception:
        return str(val)


# =============================================================================
# CLI (opcional)
# =============================================================================
import click


@app.cli.command("create-user")
@click.argument("email")
@click.argument("password")
@click.argument("role")
def create_user_cmd(email, password, role):
    """Uso:
    flask --app app.py create-user EMAIL SENHA ROLE(engenharia|atacado)
    """
    role = role.strip().lower()
    if role not in ("engenharia", "atacado"):
        click.echo("Role inválida. Use: engenharia ou atacado.")
        return
    db = get_db()
    try:
        db.execute(
            "INSERT INTO users (email, password_hash, role) VALUES (?, ?, ?)",
            (email.strip().lower(), generate_password_hash(password), role),
        )
        db.commit()
        click.echo("Usuário criado com sucesso.")
    except sqlite3.IntegrityError:
        click.echo("E-mail já cadastrado.")


@app.cli.command("seed-cns")
def seed_cns_cmd():
    """Reaplica o seed de CNs (somente códigos) e os metadados (cidade/UF)."""
    db = get_db()
    _seed_cns_if_needed(db)
    _apply_cn_metadata(db)
    click.echo("Seed de CNs aplicado/atualizado com metadados.")


# =============================================================================
# Run
# =============================================================================
if __name__ == "__main__":  # Em produção use gunicorn/uwsgi e variáveis de ambiente
    app.run(debug=True)
