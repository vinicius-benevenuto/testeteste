# =============================================================================
# app.py — VIVOHUB: Sistema de Gestão de Projetos Técnicos de Interligação
# =============================================================================
#
# Aplicação Flask para gerenciar formulários PTI (Projeto Técnico de
# Interligação) entre operadoras de telecomunicações e a VIVO.
#
# Arquitetura:
#   ┌─────────────┐     ┌──────────────┐     ┌──────────┐
#   │  Flask Web   │────▶│  SQLite DB   │     │  Excel   │
#   │  (Rotas)     │     │  (Dados)     │     │ (Export) │
#   └──────┬──────┘     └──────────────┘     └────┬─────┘
#          │                                       │
#          │            ┌──────────────┐           │
#          └───────────▶│  Pillow      │◀──────────┘
#                       │  (Imagens)   │
#                       └──────────────┘
#
# Perfis de Usuário:
#   - Atacado:     Cria e gerencia formulários (CRUD completo)
#   - Engenharia:  Visualiza todos os formulários + edita Seção 9
#   - Admin:       Registra novos usuários (protegido por código)
#
# -*- coding: utf-8 -*-
# =============================================================================

import os
import json
import sqlite3
import click
import tempfile
import logging
from pathlib import Path
from functools import wraps
from datetime import datetime
from io import BytesIO
from typing import Any, Optional

from flask import (
    Flask, render_template, request, redirect, url_for,
    session, flash, g, abort, send_file, jsonify,
)
from werkzeug.security import generate_password_hash, check_password_hash

# =============================================================================
# IMPORTAÇÕES OPCIONAIS (graceful degradation)
# =============================================================================
try:
    from openpyxl import Workbook
    from openpyxl.styles import Font, Alignment, PatternFill, Border, Side
    from openpyxl.utils import get_column_letter
    from openpyxl.drawing.image import Image as XLImage
    OPENPYXL_AVAILABLE = True
except ImportError:
    Workbook = None
    XLImage = None
    OPENPYXL_AVAILABLE = False

try:
    from PIL import Image as PILImage, ImageDraw, ImageFont
    PIL_AVAILABLE = True
except ImportError:
    PIL_AVAILABLE = False

# =============================================================================
# LOGGING
# =============================================================================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
)
logger = logging.getLogger("vivohub")

# =============================================================================
# CONSTANTES DA APLICAÇÃO
# =============================================================================
MAX_TABLE_ROWS = 10
DEFAULT_ADMIN_CODE = "v28B112004"
DEFAULT_DIAGRAM_IMAGE = (
    r"C:\Users\40418843\Desktop\VIVOHUB\templates\20251007_140942_0000.png"
)

# Whitelist de flags permitidas para o campo "Escopo"
ALLOWED_SCOPE_FLAGS = frozenset({
    "LC", "LD15 + CNG", "LDS/CSP + CNG", "Transporte", "VC1", "Concentração"
})

# Campos do formulário organizados por tipo
BOOLEAN_FIELDS = (
    "csp", "servicos_especiais", "cng",
    "sbc_ativo", "ip_reservado", "vivo_reserva",
    "operadora_ciente", "lcr_nacional", "white_list",
    "prefixos_liberados_abr", "premissas_ok",
)

TEXT_FIELDS = (
    "nome_operadora", "rn1", "atendimento", "redes", "qual", "tmr",
    "responsavel_operadora", "responsavel_vivo", "asn",
    "responsavel_infra", "aprovado_por",
    "status", "escopo_text",
    "responsavel_atacado", "responsavel_engenharia",
)

JSON_FIELDS = (
    "escopo_flags_json", "dados_vivo_json",
    "dados_operadora_json", "engenharia_params_json",
)

# =============================================================================
# SEED DE CNs (Códigos Nacionais de telefonia)
# =============================================================================
_CN_SEED_RAW = """
68 82 97 92 96 77 75 74 73 71 88 85 61 27 28 64 62 61 98 99
34 37 31 35 32 38 33 67 66 65 91 94 93 83 81 87 86 89
43 44 45 46 41 24 22 21 84 69 95 51 53 54 55 47 48 49 79
18 14 15 16 13 19 17 11 12 63
"""

CN_METADATA = {
    "11": ("São Paulo", "SP"), "12": ("São José dos Campos", "SP"),
    "13": ("Santos", "SP"), "14": ("Bauru", "SP"),
    "15": ("Sorocaba", "SP"), "16": ("Ribeirão Preto", "SP"),
    "17": ("São José do Rio Preto", "SP"), "18": ("Presidente Prudente", "SP"),
    "19": ("Campinas", "SP"),
    "21": ("Rio de Janeiro", "RJ"), "22": ("Campos dos Goytacazes", "RJ"),
    "24": ("Volta Redonda", "RJ"), "27": ("Vitória", "ES"),
    "28": ("Cachoeiro de Itapemirim", "ES"),
    "31": ("Belo Horizonte", "MG"), "32": ("Juiz de Fora", "MG"),
    "33": ("Governador Valadares", "MG"), "34": ("Uberlândia", "MG"),
    "35": ("Poços de Caldas", "MG"), "37": ("Divinópolis", "MG"),
    "38": ("Montes Claros", "MG"),
    "41": ("Curitiba", "PR"), "42": ("Ponta Grossa", "PR"),
    "43": ("Londrina", "PR"), "44": ("Maringá", "PR"),
    "45": ("Foz do Iguaçu", "PR"), "46": ("Francisco Beltrão", "PR"),
    "47": ("Joinville", "SC"), "48": ("Florianópolis", "SC"),
    "49": ("Chapecó", "SC"),
    "51": ("Porto Alegre", "RS"), "53": ("Pelotas", "RS"),
    "54": ("Caxias do Sul", "RS"), "55": ("Santa Maria", "RS"),
    "61": ("Brasília", "DF"), "62": ("Goiânia", "GO"),
    "63": ("Palmas", "TO"), "64": ("Rio Verde", "GO"),
    "65": ("Cuiabá", "MT"), "66": ("Rondonópolis", "MT"),
    "67": ("Campo Grande", "MS"),
    "71": ("Salvador", "BA"), "73": ("Ilhéus", "BA"),
    "74": ("Juazeiro", "BA"), "75": ("Feira de Santana", "BA"),
    "77": ("Vitória da Conquista", "BA"),
    "79": ("Aracaju", "SE"), "81": ("Recife", "PE"),
    "82": ("Maceió", "AL"), "83": ("João Pessoa", "PB"),
    "84": ("Natal", "RN"), "85": ("Fortaleza", "CE"),
    "86": ("Teresina", "PI"), "87": ("Petrolina", "PE"),
    "88": ("Juazeiro do Norte", "CE"), "89": ("Picos", "PI"),
    "91": ("Belém", "PA"), "92": ("Manaus", "AM"),
    "93": ("Santarém", "PA"), "94": ("Marabá", "PA"),
    "95": ("Boa Vista", "RR"), "96": ("Macapá", "AP"),
    "97": ("Coari", "AM"),
    "98": ("São Luís", "MA"), "99": ("Imperatriz", "MA"),
    "68": ("Rio Branco", "AC"), "69": ("Porto Velho", "RO"),
}

# =============================================================================
# FLASK APP FACTORY
# =============================================================================
def create_app() -> Flask:
    """
    Application Factory Pattern (recomendado pelo Flask).
    Permite múltiplas instâncias (testes), configuração por ambiente,
    e registro limpo de componentes.
    """
    app = Flask(__name__, instance_relative_config=True)

    app.config["SECRET_KEY"] = os.environ.get("SECRET_KEY", "dev-secret-change-me")
    app.config["DATABASE"] = os.path.join(app.instance_path, "vivohub.db")
    app.config["ADMIN_CODE"] = os.environ.get("ADMIN_CODE", DEFAULT_ADMIN_CODE)

    Path(app.instance_path).mkdir(parents=True, exist_ok=True)
    export_dir = os.path.join(app.instance_path, "exports")
    Path(export_dir).mkdir(parents=True, exist_ok=True)
    app.config["EXPORT_DIR"] = export_dir

    _register_db_hooks(app)
    _register_security_headers(app)
    _register_context_processors(app)
    _register_template_filters(app)
    _register_routes(app)
    _register_cli_commands(app)

    return app

# =============================================================================
# BANCO DE DADOS
# =============================================================================
def get_db() -> sqlite3.Connection:
    """Retorna conexão SQLite do request atual (padrão per-request)."""
    if "db" not in g:
        from flask import current_app
        g.db = sqlite3.connect(current_app.config["DATABASE"])
        g.db.row_factory = sqlite3.Row
        g.db.execute("PRAGMA foreign_keys = ON;")
    return g.db


def _register_db_hooks(app: Flask) -> None:
    """Registra hooks de ciclo de vida do banco."""
    @app.teardown_appcontext
    def close_db(exception=None):
        db = g.pop("db", None)
        if db is not None:
            db.close()

    @app.before_request
    def ensure_schema():
        _init_db()


def _init_db() -> None:
    """Cria/atualiza o schema de forma idempotente."""
    db = get_db()
    db.executescript("""
        CREATE TABLE IF NOT EXISTS users (
            id            INTEGER PRIMARY KEY AUTOINCREMENT,
            email         TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            role          TEXT NOT NULL CHECK (role IN ('engenharia', 'atacado')),
            created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        CREATE TABLE IF NOT EXISTS atacado_forms (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            owner_id INTEGER NOT NULL,
            status TEXT DEFAULT 'rascunho',
            nome_operadora TEXT, rn1 TEXT,
            csp INTEGER DEFAULT 0, servicos_especiais INTEGER DEFAULT 0,
            cng INTEGER DEFAULT 0, atendimento TEXT, redes TEXT,
            qual TEXT, tmr TEXT,
            responsavel_operadora TEXT, responsavel_vivo TEXT,
            sbc_ativo INTEGER DEFAULT 0, ip_reservado INTEGER DEFAULT 0,
            vivo_reserva INTEGER DEFAULT 0, asn TEXT,
            escopo_text TEXT, escopo_flags_json TEXT, dados_vivo_json TEXT,
            dados_operadora_json TEXT,
            operadora_ciente INTEGER DEFAULT 0, responsavel_infra TEXT,
            lcr_nacional INTEGER DEFAULT 0, white_list INTEGER DEFAULT 0,
            prefixos_liberados_abr INTEGER DEFAULT 0,
            premissas_ok INTEGER DEFAULT 0, aprovado_por TEXT,
            engenharia_params_json TEXT,
            responsavel_atacado TEXT, responsavel_engenharia TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(owner_id) REFERENCES users(id)
        );
        CREATE TABLE IF NOT EXISTS exports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            form_id INTEGER NOT NULL,
            filename TEXT NOT NULL, filepath TEXT NOT NULL,
            size_bytes INTEGER DEFAULT 0,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY(form_id) REFERENCES atacado_forms(id)
        );
    """)

    # Migrações leves
    for table, col, coldef in [
        ("atacado_forms", "owner_id", "INTEGER NOT NULL DEFAULT 0"),
        ("atacado_forms", "engenharia_params_json", "TEXT"),
        ("atacado_forms", "dados_vivo_json", "TEXT"),
        ("atacado_forms", "dados_operadora_json", "TEXT"),
        ("atacado_forms", "escopo_text", "TEXT"),
        ("atacado_forms", "escopo_flags_json", "TEXT"),
        ("atacado_forms", "responsavel_atacado", "TEXT"),
        ("atacado_forms", "responsavel_engenharia", "TEXT"),
    ]:
        _ensure_column(db, table, col, coldef)
    db.commit()

    _seed_cns(db)
    _apply_cn_metadata(db)


def _ensure_column(db, table, column, coldef):
    """Adiciona coluna se não existir (migração idempotente)."""
    existing = {r["name"] for r in db.execute(f"PRAGMA table_info({table});").fetchall()}
    if column not in existing:
        db.execute(f"ALTER TABLE {table} ADD COLUMN {column} {coldef};")
        logger.info(f"Coluna '{column}' adicionada à tabela '{table}'.")


def _parse_cn_seed(raw: str) -> list[str]:
    """Converte string bruta de códigos CN em lista sem duplicatas."""
    seen, result = set(), []
    for tok in raw.split():
        code = tok.strip().zfill(2)
        if code.isdigit() and code not in seen:
            seen.add(code)
            result.append(code)
    return result


def _seed_cns(db):
    """Insere códigos CN iniciais se a tabela estiver vazia."""
    db.execute("""
        CREATE TABLE IF NOT EXISTS cns (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            codigo TEXT UNIQUE NOT NULL,
            nome TEXT, uf TEXT,
            ativo INTEGER NOT NULL DEFAULT 1
        )
    """)
    if db.execute("SELECT COUNT(*) AS c FROM cns").fetchone()["c"] == 0:
        codes = _parse_cn_seed(_CN_SEED_RAW)
        db.executemany("INSERT OR IGNORE INTO cns (codigo, ativo) VALUES (?, 1)", [(c,) for c in codes])
        db.commit()


def _apply_cn_metadata(db):
    """Atualiza nomes/UFs de CNs via UPSERT (idempotente)."""
    for code, (nome, uf) in CN_METADATA.items():
        db.execute("""
            INSERT INTO cns (codigo, nome, uf, ativo) VALUES (?, ?, ?, 1)
            ON CONFLICT(codigo) DO UPDATE SET nome=excluded.nome, uf=excluded.uf, ativo=1
        """, (code, nome, uf))
    db.commit()

# =============================================================================
# SEGURANÇA — Headers e decoradores de autenticação
# =============================================================================
def _register_security_headers(app):
    @app.after_request
    def add_headers(resp):
        resp.headers.setdefault("X-Content-Type-Options", "nosniff")
        resp.headers.setdefault("X-Frame-Options", "SAMEORIGIN")
        resp.headers.setdefault("X-XSS-Protection", "0")
        return resp


def login_required(view):
    """Exige autenticação."""
    @wraps(view)
    def wrapped(*a, **kw):
        if "user_id" not in session:
            return redirect(url_for("login"))
        return view(*a, **kw)
    return wrapped


def role_required(required_role: str):
    """Exige papel específico (atacado ou engenharia)."""
    def decorator(view):
        @wraps(view)
        def wrapped(*a, **kw):
            if "user_id" not in session:
                return redirect(url_for("login"))
            if session.get("role") != required_role:
                flash("Acesso negado para esta área.", "danger")
                return redirect(url_for(f"central_{session.get('role')}"))
            return view(*a, **kw)
        return wrapped
    return decorator


def admin_required(view):
    """Exige sessão de administrador."""
    @wraps(view)
    def wrapped(*a, **kw):
        if not session.get("is_admin"):
            flash("Área restrita ao administrador.", "danger")
            return redirect(url_for("admin_login"))
        return view(*a, **kw)
    return wrapped

# =============================================================================
# HELPERS — Funções utilitárias
# =============================================================================
def row_get(row, key, default=""):
    """Acesso seguro a sqlite3.Row com fallback."""
    try:
        v = row[key]
        return default if v is None else v
    except (KeyError, IndexError):
        return default


def safe_filename(name: str) -> str:
    """Sanitiza string para nome de arquivo seguro."""
    name = (name or "").strip().replace(" ", "_")
    for ch in ("..", "/", "\\", ":", "*", "?", '"', "<", ">", "|"):
        name = name.replace(ch, "_")
    return name or "Operadora"


def parse_db_datetime(value) -> Optional[datetime]:
    """Converte timestamp SQLite em datetime (None se falhar)."""
    if not value:
        return None
    try:
        return datetime.fromisoformat(str(value).replace("T", " "))
    except (ValueError, TypeError):
        return None


def truncate_json_list(raw: str, default="[]") -> str:
    """Valida e trunca lista JSON para MAX_TABLE_ROWS itens."""
    if not raw:
        return default
    try:
        parsed = json.loads(raw) if isinstance(raw, str) else raw
        if not isinstance(parsed, list):
            return default
        return json.dumps(parsed[:MAX_TABLE_ROWS], ensure_ascii=False)
    except (json.JSONDecodeError, TypeError):
        return default


def parse_bool_field(value) -> int:
    """Converte checkbox HTML para int (0/1)."""
    return 1 if str(value).lower() in ("on", "1", "true", "sim") else 0
# =============================================================================
# PROCESSAMENTO DE FORMULÁRIO
# =============================================================================
def extract_scope_flags_from_request() -> str:
    """Extrai flags de escopo dos checkboxes com whitelist de segurança."""
    raw = request.form.getlist("escopo_flags") or []
    valid = [f for f in raw if f in ALLOWED_SCOPE_FLAGS]
    unique = list(dict.fromkeys(valid))  # Remove duplicatas, preserva ordem
    return json.dumps(unique, ensure_ascii=False)


def _parse_json_dict(raw: str) -> str:
    """Valida string JSON como dicionário. Retorna '{}' se inválido."""
    if not raw:
        return "{}"
    try:
        parsed = json.loads(raw) if isinstance(raw, str) else raw
        if isinstance(parsed, dict):
            return json.dumps(parsed, ensure_ascii=False)
    except (json.JSONDecodeError, TypeError):
        pass
    return "{}"


def extract_form_payload() -> dict:
    """
    Extrai e normaliza todos os dados do formulário HTML.
    Processa: texto (strip), booleanos (0/1), JSON (validação + truncamento).
    """
    payload = {}

    for field in TEXT_FIELDS:
        payload[field] = (request.form.get(field) or "").strip()

    for field in BOOLEAN_FIELDS:
        payload[field] = parse_bool_field(request.form.get(field))

    payload["escopo_flags_json"] = extract_scope_flags_from_request()

    for field in JSON_FIELDS:
        if field == "escopo_flags_json":
            continue
        raw = request.form.get(field, "")
        if field == "engenharia_params_json":
            payload[field] = _parse_json_dict(raw)
        else:
            payload[field] = truncate_json_list(raw, "[]")

    return payload


def validate_table_rows(payload: dict) -> bool:
    """Trunca tabelas que excedem o limite. Retorna True se truncou."""
    truncated = False
    for key in ("dados_vivo_json", "dados_operadora_json"):
        try:
            rows = json.loads(payload[key])
            if isinstance(rows, list) and len(rows) > MAX_TABLE_ROWS:
                payload[key] = json.dumps(rows[:MAX_TABLE_ROWS], ensure_ascii=False)
                truncated = True
        except (json.JSONDecodeError, TypeError, KeyError):
            payload[key] = "[]"
    return truncated

# =============================================================================
# CONTEXT PROCESSORS E TEMPLATE FILTERS
# =============================================================================
def _register_context_processors(app):
    @app.context_processor
    def inject_cn_codes():
        """Disponibiliza CNs ativos em todos os templates."""
        db = get_db()
        rows = db.execute(
            "SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf "
            "FROM cns WHERE ativo = 1 ORDER BY codigo ASC"
        ).fetchall()
        return {
            "CN_CODES": [r["codigo"] for r in rows],
            "CN_FULL": [{"codigo": r["codigo"], "nome": r["nome"], "uf": r["uf"]} for r in rows],
        }


def _register_template_filters(app):
    @app.template_filter("date_br")
    def date_br_filter(value):
        """Formata datetime para DD/MM/YYYY HH:MM."""
        if not value:
            return ""
        if hasattr(value, "strftime"):
            try:
                return value.strftime("%d/%m/%Y %H:%M")
            except (ValueError, OSError):
                return str(value)
        dt = parse_db_datetime(value)
        return dt.strftime("%d/%m/%Y %H:%M") if dt else str(value)

# =============================================================================
# QUERIES — Filtros e contadores para listagens
# =============================================================================
def build_list_query(base_sql, *, search_term=None, status_filter=None,
                     owner_id=None, sort_key="-created_at"):
    """
    Constrói query dinâmica com filtros e ordenação segura.
    Retorna (sql, params) com placeholders para prevenir SQL injection.
    """
    conditions, params = [], []

    if owner_id is not None:
        conditions.append("owner_id = ?")
        params.append(owner_id)
    if search_term:
        conditions.append("nome_operadora LIKE ?")
        params.append(f"%{search_term}%")
    if status_filter:
        conditions.append("LOWER(COALESCE(status, '')) = LOWER(?)")
        params.append(status_filter)

    if conditions:
        base_sql += " WHERE " + " AND ".join(conditions)

    # Mapa seguro de ordenação (previne injection via sort param)
    sort_map = {
        "created_at": "created_at ASC", "-created_at": "created_at DESC",
        "nome_operadora": "nome_operadora COLLATE NOCASE ASC",
        "-nome_operadora": "nome_operadora COLLATE NOCASE DESC",
        "id": "id ASC", "-id": "id DESC",
    }
    base_sql += f" ORDER BY {sort_map.get(sort_key, 'id DESC')}"
    return base_sql, params


def get_status_counters(db, extra_where="", extra_params=()):
    """Calcula contadores por status para badges na UI."""
    connector = "AND" if "WHERE" in extra_where else "WHERE"

    def _count(condition=""):
        sql = f"SELECT COUNT(*) AS c FROM atacado_forms {extra_where}"
        if condition:
            sql += f" {connector} {condition}"
        return db.execute(sql, extra_params).fetchone()["c"]

    return {
        "total": _count(),
        "rascunho": _count("LOWER(status) = 'rascunho'"),
        "enviado": _count("LOWER(status) = 'enviado'"),
        "em_revisao": _count("LOWER(status) = 'em revisão'"),
        "aprovado": _count("LOWER(status) = 'aprovado'"),
    }

# =============================================================================
# PROCESSAMENTO DE IMAGENS (Pillow)
# =============================================================================
def parse_xy_percent(raw, default):
    """Converte '0.18,0.55' em tupla (0.18, 0.55), clamped a [0,1]."""
    if not raw:
        return default
    try:
        parts = [p.strip() for p in str(raw).split(",")]
        if len(parts) != 2:
            return default
        return (max(0.0, min(1.0, float(parts[0]))), max(0.0, min(1.0, float(parts[1]))))
    except (ValueError, TypeError):
        return default


def fit_size_keep_aspect(orig_w, orig_h, max_w, max_h):
    """Calcula dimensões proporcionais dentro de bounding box."""
    if orig_w <= 0 or orig_h <= 0:
        return max_w, max_h
    scale = min(max_w / orig_w, max_h / orig_h)
    return int(orig_w * scale), int(orig_h * scale)


def _find_system_font(size_px, preferred_path=None):
    """Tenta carregar fonte TrueType do sistema (Windows/macOS/Linux)."""
    if not PIL_AVAILABLE:
        return None
    if preferred_path:
        try:
            return ImageFont.truetype(preferred_path, size_px)
        except (OSError, IOError):
            pass
    for path in [
        "C:\\Windows\\Fonts\\arial.ttf",
        "/System/Library/Fonts/Supplemental/Arial.ttf",
        "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
    ]:
        try:
            return ImageFont.truetype(path, size_px)
        except (OSError, IOError):
            continue
    return ImageFont.load_default()


def render_labels_on_image(
    image_path, vivo_text, operator_text, *,
    vivo_xy_pct=(0.18, 0.55), operator_xy_pct=(0.80, 0.55),
    font_path=None, font_size_pct=0.06,
    fill=(255, 255, 255, 255), stroke_fill=(0, 0, 0, 255),
    stroke_width_pct=0.012, extra_labels=None,
) -> str:
    """
    Gera cópia da imagem com rótulos sobrepostos (coordenadas percentuais).
    Retorna caminho do PNG temporário.
    """
    if not PIL_AVAILABLE:
        raise RuntimeError("Pillow não instalado. pip install pillow")

    img = PILImage.open(image_path).convert("RGBA")
    W, H = img.size
    draw = ImageDraw.Draw(img)

    def _draw(text, xy_pct, size_pct, *, lbl_fill=fill, lbl_stroke=stroke_fill,
              lbl_stroke_w=stroke_width_pct, anchor="mm"):
        if not text:
            return
        x, y = int(W * xy_pct[0]), int(H * xy_pct[1])
        font = _find_system_font(max(12, int(H * size_pct)), font_path)
        draw.text((x, y), text, font=font, fill=lbl_fill,
                  stroke_width=max(0, int(H * lbl_stroke_w)),
                  stroke_fill=lbl_stroke, anchor=anchor)

    _draw(vivo_text, vivo_xy_pct, font_size_pct)
    _draw(operator_text, operator_xy_pct, font_size_pct)

    if extra_labels:
        for lbl in extra_labels:
            _draw(lbl.get("text", ""), lbl.get("xy_pct", (0.5, 0.5)),
                  float(lbl.get("font_size_pct", font_size_pct)),
                  lbl_fill=lbl.get("fill", fill),
                  lbl_stroke=lbl.get("stroke_fill", stroke_fill),
                  lbl_stroke_w=float(lbl.get("stroke_width_pct", stroke_width_pct)),
                  anchor=lbl.get("anchor", "mm"))

    fd, tmp = tempfile.mkstemp(prefix="vivohub_diagrama_", suffix=".png")
    os.close(fd)
    img.save(tmp, format="PNG")
    return tmp
# =============================================================================
# GERAÇÃO DE EXCEL — Estilos e Builder do PTI
# =============================================================================
class ExcelStylePalette:
    """Paleta centralizada de estilos para consistência visual no Excel."""

    def __init__(self):
        # Cores (paleta roxo púrpura corporativa)
        self.fill_primary = PatternFill("solid", fgColor="4A148C")
        self.fill_secondary = PatternFill("solid", fgColor="6A1B9A")
        self.fill_light = PatternFill("solid", fgColor="7B1FA2")
        self.fill_accent = PatternFill("solid", fgColor="8E24AA")
        self.fill_neutral = PatternFill("solid", fgColor="E1BEE7")
        self.fill_background = PatternFill("solid", fgColor="F3E5F5")
        self.fill_white = PatternFill("solid", fgColor="FFFFFF")
        # Fontes
        self.font_title = Font(name="Calibri", size=14, bold=True, color="FFFFFF")
        self.font_subtitle = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
        self.font_brand = Font(name="Calibri", size=12, bold=True, color="FFFFFF")
        self.font_header = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
        self.font_subheader = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
        self.font_body = Font(name="Calibri", size=10)
        self.font_small = Font(name="Calibri", size=9)
        self.font_micro = Font(name="Calibri", size=8, bold=True, color="FFFFFF")
        self.font_check = Font(name="Calibri", size=11, bold=True, color="FF0000")
        # Bordas
        thin = Side(style="thin", color="DDDDDD")
        self.box_border = Border(left=thin, right=thin, top=thin, bottom=thin)
        self.no_border = Border()
        # Alinhamentos
        self.align_center = Alignment(horizontal="center", vertical="center")
        self.align_left = Alignment(horizontal="left", vertical="center")
        self.align_wrap = Alignment(horizontal="left", vertical="center", wrap_text=True)
        self.align_center_wrap = Alignment(horizontal="center", vertical="center", wrap_text=True)

    def alternating_fill(self, idx):
        """Cor alternada para linhas (zebra striping)."""
        return self.fill_background if idx % 2 == 0 else self.fill_white


class PTIWorkbookBuilder:
    """
    Builder para Workbook Excel do PTI.
    Cada _build_*_sheet() constrói uma aba independente.
    build() orquestra tudo.
    """

    def __init__(self, form_row):
        if not OPENPYXL_AVAILABLE:
            raise RuntimeError("openpyxl não instalado.")
        if not PIL_AVAILABLE:
            raise RuntimeError("Pillow não instalado.")

        self.form = form_row
        self.style = ExcelStylePalette()
        self.wb = Workbook()

        # Extrai dados do formulário uma vez
        self.nome_op = (row_get(form_row, "nome_operadora") or "Operadora").strip()
        self.resp_atacado = (row_get(form_row, "responsavel_atacado") or row_get(form_row, "responsavel_vivo") or "").strip()
        self.resp_eng = (row_get(form_row, "responsavel_engenharia", "") or "").strip()
        self.asn_op = (row_get(form_row, "asn", "") or "").strip()

        created_dt = parse_db_datetime(row_get(form_row, "created_at", ""))
        self.created_str = created_dt.strftime("%d/%m/%Y") if created_dt else ""

        self.traffic_types = self._extract_traffic_types()
        self.concentracao = self._extract_concentracao()

    # --- Extração de dados ---
    def _extract_traffic_types(self):
        raw = row_get(self.form, "escopo_flags_json", "[]")
        try:
            parsed = json.loads(raw) if isinstance(raw, str) else raw
            items = [str(x).strip() for x in (parsed if isinstance(parsed, list) else []) if str(x).strip()]
        except (json.JSONDecodeError, TypeError):
            items = []
        while len(items) < 3:
            items.append("")
        return items[:3]

    def _extract_concentracao(self):
        raw = row_get(self.form, "dados_operadora_json", "[]")
        try:
            data = json.loads(raw) if isinstance(raw, str) else raw
            if not isinstance(data, list):
                return []
            result = []
            for item in data:
                if not isinstance(item, dict):
                    continue
                clean = {k: str(item.get(k, "")).strip() for k in ("ref", "cn", "cnl", "cod_cni", "status_link")}
                if any(clean.values()):
                    result.append(clean)
            return result[:MAX_TABLE_ROWS]
        except (json.JSONDecodeError, TypeError):
            return []

    def _get_diagram_config(self):
        """Extrai configurações de posicionamento do diagrama."""
        f = self.form
        def _fl(key, d):
            try: return float(row_get(f, key, d))
            except (ValueError, TypeError): return d
        def _it(key, d):
            try: return int(float(row_get(f, key, d)))
            except (ValueError, TypeError): return d

        no_stroke = str(row_get(f, "roteador_op_no_stroke", "")).lower() in ("1", "true", "yes", "sim")
        rot_stroke = 0.0 if no_stroke else 0.009
        override = row_get(f, "roteador_op_stroke_width_pct", None)
        if override not in (None, ""):
            try: rot_stroke = float(override)
            except (ValueError, TypeError): pass

        return {
            "image_path": row_get(f, "diagram_image_path") or DEFAULT_DIAGRAM_IMAGE,
            "vivo_xy": parse_xy_percent(row_get(f, "vivo_label_xy_pct", ""), (0.16, 0.48)),
            "op_xy": parse_xy_percent(row_get(f, "operadora_label_xy_pct", ""), (0.83, 0.44)),
            "link_vivo_xy": parse_xy_percent(row_get(f, "link_vivo_label_xy_pct", ""), (0.49, 0.53)),
            "link_op_xy": parse_xy_percent(row_get(f, "link_op_label_xy_pct", ""), (0.49, 0.35)),
            "font_size_pct": _fl("diagram_font_size_pct", 0.040),
            "link_font_pct": _fl("link_labels_font_size_pct", 0.030),
            "font_path": row_get(f, "diagram_font_path", None),
            "rot1_xy": parse_xy_percent(row_get(f, "roteador_op1_label_xy_pct", ""), (0.64, 0.58)),
            "rot2_xy": parse_xy_percent(row_get(f, "roteador_op2_label_xy_pct", ""), (0.64, 0.39)),
            "rot_font_pct": _fl("roteador_op_font_size_pct", 0.010),
            "rot_stroke_pct": rot_stroke,
            "bbox_w": _it("diagram_max_w", 1900),
            "bbox_h": _it("diagram_max_h", 650),
            "anchor_row_off": _it("diagram_anchor_row_offset", -3),
            "anchor_col_off": _it("diagram_anchor_col_offset", 1),
        }

    # --- Helpers de construção ---
    def _set_col_widths(self, ws, widths):
        for col, w in widths.items():
            ws.column_dimensions[get_column_letter(col)].width = w

    def _brand_header(self, ws, row, sc, ec):
        ws.merge_cells(start_row=row, start_column=sc, end_row=row, end_column=ec)
        c = ws.cell(row=row, column=sc, value=f"VIVO — {self.nome_op}")
        c.font = self.style.font_title
        c.alignment = self.style.align_center
        c.fill = self.style.fill_primary
        ws.row_dimensions[row].height = 24.0

    def _section_title(self, ws, row, sc, ec, text):
        ws.merge_cells(start_row=row, start_column=sc, end_row=row, end_column=ec)
        c = ws.cell(row=row, column=sc, value=text)
        c.font = self.style.font_subtitle
        c.alignment = self.style.align_center
        c.fill = self.style.fill_secondary
        ws.row_dimensions[row].height = 25.0

    def _print_settings(self, ws, *, landscape=False):
        ws.sheet_view.showGridLines = False
        ws.page_setup.paperSize = ws.PAPERSIZE_A4
        ws.page_setup.orientation = ws.ORIENTATION_LANDSCAPE if landscape else ws.ORIENTATION_PORTRAIT
        ws.print_options.horizontalCentered = True
        ws.page_margins.left = ws.page_margins.right = 0.4
        ws.page_margins.top = ws.page_margins.bottom = 0.5

    def _clear_area(self, ws, r1, r2, c1, c2):
        for r in range(r1, r2 + 1):
            for c in range(c1, c2 + 1):
                cell = ws.cell(row=r, column=c)
                cell.fill = self.style.fill_white
                cell.border = self.style.no_border

    # --- ABA 1: ÍNDICE ---
    def _build_index(self):
        s = self.style
        ws = self.wb.active
        ws.title = "Índice"
        for c in range(1, 10):
            ws.column_dimensions[get_column_letter(c)].width = 12.0
        ws.column_dimensions["C"].width = 8.0
        ws.column_dimensions["D"].width = 45.0

        self._brand_header(ws, 2, 3, 8)
        ws.row_dimensions[2].height = 28.0

        ws.merge_cells(start_row=4, start_column=3, end_row=4, end_column=8)
        c = ws.cell(row=4, column=3, value="ANEXO 3: PROJETO TÉCNICO")
        c.font = s.font_subtitle; c.alignment = s.align_center; c.fill = s.fill_secondary

        ws.merge_cells(start_row=5, start_column=3, end_row=5, end_column=8)
        c = ws.cell(row=5, column=3, value="PROJETO DE INTERLIGAÇÃO PARA ENCAMINHAMENTO DA TERMINAÇÃO DE CHAMADAS DE VOZ")
        c.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
        c.alignment = s.align_center; c.fill = s.fill_secondary
        ws.row_dimensions[5].height = 30.0

        sections = [
            "1. Versões", "2. Projeto de Interligação",
            "2.2. Diagrama de Interligação",
            "2.3. Características do Projeto de Interligação e do Plano de Encaminhamento",
            "2.4. Plano de Contingência", "2.5. Concentração",
            "2.6. Plan NUM_Operadora", "2.7. Dados de MTL",
            "2.8. SE REG III (Vivo STFC Concessionária)",
            "2.9. Parâmetros de Programação",
        ]
        for i, title in enumerate(sections):
            r = 7 + i
            ws.merge_cells(start_row=r, start_column=3, end_row=r, end_column=8)
            c = ws.cell(row=r, column=3, value=title)
            c.font = Font(name="Calibri", size=11)
            c.alignment = s.align_left
            c.fill = s.alternating_fill(i)
            ws.row_dimensions[r].height = 22.0

        ws.freeze_panes = "C7"
        self._print_settings(ws)

    # --- ABA 2: VERSÕES ---
    def _build_versions(self):
        s = self.style
        ws = self.wb.create_sheet(title="Versões")
        self._set_col_widths(ws, {1:2, 2:2, 3:8, 4:12, 5:28, 6:28, 7:28, 8:10, 9:18, 10:10, 11:2})
        self._brand_header(ws, 2, 3, 10)
        self._section_title(ws, 4, 3, 10, "CONTROLE DE VERSÕES DO PTI")

        headers = ["Versão", "Data", "Responsável Eng de ITX", "Responsável Gestão de ITX",
                    "Escopo", "CN", "ÁREAS LOCAIS", "ATA"]
        hr = 6
        ws.row_dimensions[hr].height = 25.0
        for i, txt in enumerate(headers):
            c = ws.cell(row=hr, column=3+i, value=txt)
            c.font = Font(name="Calibri", size=9, bold=True, color="FFFFFF")
            c.alignment = s.align_center; c.fill = s.fill_light

        first_data = {4: self.created_str, 5: self.resp_eng, 6: self.resp_atacado}
        for i in range(10):
            r = hr + 1 + i
            ws.row_dimensions[r].height = 22.0
            fill = s.alternating_fill(i)
            ws.cell(row=r, column=3, value=i+1).fill = fill
            ws.cell(row=r, column=3).font = s.font_body
            ws.cell(row=r, column=3).alignment = s.align_center
            for col in range(4, 11):
                c = ws.cell(row=r, column=col)
                c.value = first_data.get(col, "") if i == 0 else ""
                c.font = s.font_body; c.fill = fill
                c.alignment = s.align_left if col in (5, 6) else s.align_center

        ws.freeze_panes = f"C{hr+1}"
        self._print_settings(ws, landscape=True)
    # --- ABA 3: DIAGRAMA DE INTERLIGAÇÃO ---
    def _build_diagram(self):
        s = self.style
        ws = self.wb.create_sheet(title="Diagrama de Interligação")
        cfg = self._get_diagram_config()
        nome = self.nome_op

        self._set_col_widths(ws, {i: (2.0 if i in (1,14) else 12.0) for i in range(1, 15)})
        self._brand_header(ws, 2, 3, 12)

        # Título e seções de texto
        for r, txt, bold in [
            (4, f"Diagrama de Interligação entre a VIVO e a {nome}", True),
            (6, "2.2.1 DIAGRAMAÇÃO DO PROJETO SIP", True),
            (7, "2.2.1.1 Anúncio de Redes pelos 2 Links SPC", True),
            (8, f"A {nome} abordará os endereços da VIVO conforme abaixo.", False),
            (9, f"A VIVO abordará os endereços da {nome} conforme abaixo.", False),
            (11, "2.2.1.2 Parâmetros de Configuração do Link", True),
        ]:
            ws.merge_cells(start_row=r, start_column=3, end_row=r, end_column=12)
            c = ws.cell(row=r, column=3, value=txt)
            c.font = Font(name="Calibri", size=10 if not bold else 11, bold=bold)
            c.alignment = s.align_left if bold else s.align_wrap
            ws.row_dimensions[r].height = 22.0 if bold else 25.0

        # ASN VIVO / Operadora
        for off, label, val in [(0, "ASN VIVO", "10429 (Público)"), (1, f"ASN {nome}", self.asn_op)]:
            r = 12 + off
            ws.merge_cells(start_row=r, start_column=3, end_row=r, end_column=5)
            ws.cell(row=r, column=3, value=label).font = Font(name="Calibri", size=10, bold=True)
            ws.cell(row=r, column=3).alignment = s.align_center
            ws.merge_cells(start_row=r, start_column=6, end_row=r, end_column=8)
            ws.cell(row=r, column=6, value=val).font = s.font_body
            ws.cell(row=r, column=6).alignment = s.align_center

        # CN / VRF
        for sc, ec, txt in [(3, 5, "CN"), (6, 12, "VRF: __________________________")]:
            ws.merge_cells(start_row=15, start_column=sc, end_row=15, end_column=ec)
            c = ws.cell(row=15, column=sc, value=txt)
            c.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c.alignment = s.align_center; c.fill = s.fill_light

        # Pontas A / B
        for sc, ec, lbl, val in [(3,7,"Ponta A","VIVO"), (8,12,"Ponta B",nome)]:
            ws.merge_cells(start_row=17, start_column=sc, end_row=17, end_column=ec)
            ws.cell(row=17, column=sc, value=lbl).font = Font(name="Calibri", size=10, bold=True)
            ws.cell(row=17, column=sc).alignment = s.align_center
            ws.merge_cells(start_row=18, start_column=sc, end_row=18, end_column=ec)
            ws.cell(row=18, column=sc, value=val).font = Font(name="Calibri", size=10, bold=True)
            ws.cell(row=18, column=sc).alignment = s.align_center

        # Tabelas de tráfego
        for lc in (3, 8):
            for i, h in enumerate(["Tráfego", "Endereço IP", "NET MASK"]):
                c = ws.cell(row=20, column=lc+i, value=h)
                c.font = s.font_subheader; c.alignment = s.align_center; c.fill = s.fill_light
            ws.row_dimensions[20].height = 25.0
            for j in range(3):
                r = 21 + j
                ws.row_dimensions[r].height = 22.0
                fill = s.alternating_fill(j)
                ws.cell(row=r, column=lc, value=self.traffic_types[j] if j < len(self.traffic_types) else "").fill = fill
                ws.cell(row=r, column=lc).font = s.font_small
                ws.cell(row=r, column=lc+1, value="").fill = fill
                ws.cell(row=r, column=lc+2, value="").fill = fill

        # Endereços
        for sc, ec, txt in [
            (3, 7, "Endereço (VIVO): __________________________________________"),
            (8, 12, f"Endereço ({nome}): __________________________________"),
        ]:
            ws.merge_cells(start_row=25, start_column=sc, end_row=25, end_column=ec)
            ws.cell(row=25, column=sc, value=txt).font = s.font_small
            ws.cell(row=25, column=sc).alignment = s.align_left

        # Diagrama (imagem anotada)
        if not os.path.exists(cfg["image_path"]):
            raise FileNotFoundError(f"Imagem não encontrada: {cfg['image_path']}")

        extra_labels = [
            {"text": "Roteador OP", "xy_pct": cfg["rot1_xy"], "font_size_pct": cfg["rot_font_pct"], "stroke_width_pct": cfg["rot_stroke_pct"]},
            {"text": "Roteador OP", "xy_pct": cfg["rot2_xy"], "font_size_pct": cfg["rot_font_pct"], "stroke_width_pct": cfg["rot_stroke_pct"]},
            {"text": "Link resp Vivo", "xy_pct": cfg["link_vivo_xy"], "font_size_pct": cfg["link_font_pct"], "stroke_width_pct": 0.006},
            {"text": f"Link resp {nome}", "xy_pct": cfg["link_op_xy"], "font_size_pct": cfg["link_font_pct"], "stroke_width_pct": 0.006},
            {"text": "RAC/RAV/HL4", "xy_pct": (0.33, 0.70), "font_size_pct": 0.010, "stroke_width_pct": 0.006},
            {"text": "RAC/RAV/HL4", "xy_pct": (0.34, 0.60), "font_size_pct": 0.010, "stroke_width_pct": 0.006},
        ]
        annotated = render_labels_on_image(
            image_path=cfg["image_path"], vivo_text="VIVO", operator_text=nome,
            vivo_xy_pct=cfg["vivo_xy"], operator_xy_pct=cfg["op_xy"],
            font_path=cfg["font_path"], font_size_pct=cfg["font_size_pct"],
            stroke_width_pct=0.009, extra_labels=extra_labels,
        )
        xl_img = XLImage(annotated)
        try:
            with PILImage.open(annotated) as src:
                ow, oh = src.size
        except Exception:
            ow, oh = 800, 300
        fw, fh = fit_size_keep_aspect(ow, oh, cfg["bbox_w"], cfg["bbox_h"])
        xl_img.width, xl_img.height = fw, fh
        anchor_r = 27 + cfg["anchor_row_off"]
        anchor_c = 3 + cfg["anchor_col_off"]
        ws.add_image(xl_img, f"{get_column_letter(anchor_c)}{anchor_r}")
        self._print_settings(ws, landscape=True)

    # --- ABA 4: ENCAMINHAMENTO ---
    def _build_routing(self):
        s = self.style
        ws = self.wb.create_sheet(title="Encaminhamento")
        nome = self.nome_op

        cw = {1:2, 2:15, 3:10, 4:15, 5:10, 6:18, 7:15, 8:18, 9:12, 10:8, 11:8,
              12:8, 13:8, 14:8, 15:8, 16:12, 17:18, 18:12, 19:12, 20:15, 21:12, 22:10, 23:12, 24:10, 25:2}
        self._set_col_widths(ws, cw)
        self._brand_header(ws, 2, 2, 24)
        self._section_title(ws, 4, 2, 24, "2.3. CARACTERÍSTICAS DO PROJETO DE INTERLIGAÇÃO E DO PLANO DE ENCAMINHAMENTO")

        # Nível 1 do cabeçalho
        h1 = 6
        ws.row_dimensions[h1].height = 30.0
        for sc, ec, txt, vert in [
            (2,2,"ÁREA LOCAL",True), (3,6,"LOCALIZAÇÃO",False),
            (7,9,"DADOS DA ROTA",False), (10,11,"BANDA",False),
            (12,13,"CANAIS",False), (14,15,"CAPS",False),
            (16,16,"ATIVAÇÃO",True), (17,18,"ENCAMINHAMENTO",False),
            (19,20,"SINALIZAÇÃO",False), (21,24,"ENDEREÇO IP",False),
        ]:
            if vert:
                ws.merge_cells(start_row=h1, start_column=sc, end_row=h1+2, end_column=ec)
            else:
                ws.merge_cells(start_row=h1, start_column=sc, end_row=h1, end_column=ec)
            c = ws.cell(row=h1, column=sc, value=txt)
            c.font = s.font_subheader; c.alignment = s.align_center_wrap
            c.fill = s.fill_secondary if vert else s.fill_light; c.border = s.box_border

        # Nível 2
        h2 = h1 + 1
        ws.row_dimensions[h2].height = 25.0
        for col, txt in [
            (3,"CN"), (4,"POI/PPI VIVO"), (5,"CN"), (6,f"POI/PPI {nome.upper()}"),
            (7,"PONTA A VIVO"), (8,f"PONTA B {nome.upper()}"), (9,"TIPO DE TRÁFEGO"),
            (10,"EXIST."), (11,"PLAN."), (12,"EXIST."), (13,"PLAN."),
            (14,"EXIST."), (15,"PLAN."),
            (17,"DE A > B\n(FORMATO DE ENTREGA)"), (18,"DE B > A"),
            (19,"CODEC"), (20,"OBSERVAÇÃO"),
        ]:
            ws.merge_cells(start_row=h2, start_column=col, end_row=h2+1, end_column=col)
            c = ws.cell(row=h2, column=col, value=txt)
            c.font = Font(name="Calibri", size=7 if col==17 else 8, bold=True, color="FFFFFF")
            c.alignment = s.align_center_wrap; c.fill = s.fill_accent; c.border = s.box_border

        for sc, ec, txt in [(21,22,"VIVO"), (23,24,nome.upper())]:
            ws.merge_cells(start_row=h2, start_column=sc, end_row=h2, end_column=ec)
            c = ws.cell(row=h2, column=sc, value=txt)
            c.font = s.font_micro; c.alignment = s.align_center; c.fill = s.fill_accent; c.border = s.box_border

        # Nível 3 (IP)
        h3 = h2 + 1
        ws.row_dimensions[h3].height = 25.0
        for col, txt in [(21,"IP ADDRESS"), (22,"NETMASK"), (23,"IP ADDRESS"), (24,"NETMASK")]:
            c = ws.cell(row=h3, column=col, value=txt)
            c.font = Font(name="Calibri", size=7, bold=True, color="FFFFFF")
            c.alignment = s.align_center; c.fill = s.fill_light; c.border = s.box_border

        # Dados
        ds = h3 + 1
        for i in range(15):
            r = ds + i
            ws.row_dimensions[r].height = 22.0
            fill = s.alternating_fill(i)
            for col in range(2, 25):
                c = ws.cell(row=r, column=col, value="")
                c.font = s.font_small; c.alignment = s.align_center
                c.border = s.box_border; c.fill = fill

        # Limpar fora da tabela
        self._clear_area(ws, 1, 1, 1, 30)
        self._clear_area(ws, 3, 3, 1, 30)
        self._clear_area(ws, 5, 5, 1, 30)
        for col in (1, 25):
            for r in range(1, ds+20):
                ws.cell(row=r, column=col).fill = s.fill_white
                ws.cell(row=r, column=col).border = s.no_border

        ws.freeze_panes = f"B{ds}"
        self._print_settings(ws, landscape=True)

    # --- ABA 5: CONCENTRAÇÃO ---
    def _build_concentration(self):
        s = self.style
        ws = self.wb.create_sheet(title="Concentração")
        self._set_col_widths(ws, {1:2, 2:2, 3:8, 4:8, 5:12, 6:12, 7:25})

        ws.merge_cells(start_row=2, start_column=3, end_row=2, end_column=7)
        c = ws.cell(row=2, column=3, value="2.5 - Concentração")
        c.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
        c.fill = s.fill_primary; ws.row_dimensions[2].height = 22.0

        for r, txt, it in [
            (3, "A 'Operadora' informou interesse em realizar a concentração de ALs conforme detalhado na tabela abaixo:", False),
            (4, 'Caso necessite de incluir mais ALs, Favor inserir linhas adicionais de acordo com reunião que foi acordado o tema (coluna B - "Ref").', True),
        ]:
            ws.merge_cells(start_row=r, start_column=3, end_row=r, end_column=7)
            c = ws.cell(row=r, column=3, value=txt)
            c.font = Font(name="Calibri", size=9 if it else 10, italic=it)
            c.alignment = s.align_wrap; c.fill = s.fill_background

        hr = 6
        for txt, col in [("Ref",3), ("CN",4), ("CNL",5), ("Cod. CnI",6), ("Status Link",7)]:
            c = ws.cell(row=hr, column=col, value=txt)
            c.font = Font(name="Calibri", size=10, bold=True, color="FFFFFF")
            c.alignment = s.align_center; c.fill = s.fill_secondary
        ws.row_dimensions[hr].height = 25.0

        for i in range(10):
            r = hr + 1 + i
            ws.row_dimensions[r].height = 22.0
            fill = s.alternating_fill(i)
            data = self.concentracao[i] if i < len(self.concentracao) else {}
            for col, key in [(3,"ref"), (4,"cn"), (5,"cnl"), (6,"cod_cni"), (7,"status_link")]:
                c = ws.cell(row=r, column=col, value=data.get(key, ""))
                c.font = s.font_body; c.fill = fill
                c.alignment = s.align_center if col != 7 else s.align_left

        ws.freeze_panes = f"C{hr+1}"
        self._print_settings(ws)

    # --- ABA 6: PLAN NUM_OPER ---
    def _build_prefixes(self):
        s = self.style
        ws = self.wb.create_sheet(title="Plan Num_Oper")
        self._set_col_widths(ws, {1:2, 2:4, 3:12, 4:12, 5:12, 6:6, 7:10, 8:10, 9:8, 10:15})
        self._brand_header(ws, 2, 2, 10)
        self._section_title(ws, 4, 2, 10, f"2.6. Tabela de Prefixos da {self.nome_op.upper()}")

        ws.merge_cells(start_row=5, start_column=2, end_row=5, end_column=10)
        ws.cell(row=5, column=2, value='OBS: Caso necessite incluir mais faixas, inserir linhas adicionais (coluna B - "Ref").').font = Font(name="Calibri", size=9, italic=True)
        ws.cell(row=5, column=2).alignment = s.align_wrap; ws.cell(row=5, column=2).fill = s.fill_background

        h1 = 7; ws.row_dimensions[h1].height = 25.0
        for sc, ec, txt, vert in [
            (2,2,"Ref",True), (3,3,"Município",True), (4,4,"ÁREA LOCAL",True),
            (5,5,"CÓDIGO CNL",True), (6,6,"CN",True), (7,8,"SERIE AUTORIZADA",False),
            (9,9,"EOT LOCAL",True), (10,10,"SNOA STFC / SMP",True),
        ]:
            if vert: ws.merge_cells(start_row=h1, start_column=sc, end_row=h1+1, end_column=ec)
            else: ws.merge_cells(start_row=h1, start_column=sc, end_row=h1, end_column=ec)
            c = ws.cell(row=h1, column=sc, value=txt)
            c.font = s.font_subheader; c.alignment = s.align_center; c.fill = s.fill_secondary; c.border = s.box_border

        h2 = h1 + 1; ws.row_dimensions[h2].height = 30.0
        for col, txt in [(7, "INCLUÍNOS\nN8 N7 N6 N5"), (8, "FINAL\nN4 N3 N2 N1")]:
            c = ws.cell(row=h2, column=col, value=txt)
            c.font = s.font_micro; c.alignment = s.align_center_wrap; c.fill = s.fill_accent; c.border = s.box_border

        ds = h2 + 1
        for i in range(5):
            r = ds + i; ws.row_dimensions[r].height = 20.0
            fill = s.alternating_fill(i)
            for col in range(2, 11):
                c = ws.cell(row=r, column=col, value="")
                c.font = s.font_small; c.alignment = s.align_center; c.border = s.box_border; c.fill = fill

        # Parâmetros
        ps = ds + 6
        for i, p in enumerate(["RN1", "EOT Local", "EOT LDN/LDI", "CSP", "Código Especial:", "CNG:", "RN2"]):
            r = ps + i; ws.row_dimensions[r].height = 20.0; fill = s.alternating_fill(i)
            ws.cell(row=r, column=2, value=p).font = Font(name="Calibri", size=9, bold=True)
            ws.cell(row=r, column=2).border = s.box_border; ws.cell(row=r, column=2).fill = fill
            ws.cell(row=r, column=3, value="").border = s.box_border; ws.cell(row=r, column=3).fill = fill

        for off, txt in [(0, "Operadora de transporte nas localidades onde não existir rota direta"),
                          (1, "Transbordo nas localidades onde existe rota direta:")]:
            ws.merge_cells(start_row=ps+off, start_column=5, end_row=ps+off, end_column=10)
            ws.cell(row=ps+off, column=5, value=txt).font = Font(name="Calibri", size=9, bold=True)

        # Transbordo
        tr = ps + len(["RN1","EOT Local","EOT LDN/LDI","CSP","Código Especial:","CNG:","RN2"]) + 1
        for sc, ec, txt in [(2,3,"UF"), (4,7,"SERIE AUTORIZADA")]:
            ws.merge_cells(start_row=tr, start_column=sc, end_row=tr, end_column=ec)
            c = ws.cell(row=tr, column=sc, value=txt)
            c.font = s.font_subheader; c.alignment = s.align_center; c.fill = s.fill_secondary; c.border = s.box_border
        sr = tr + 1
        for sc, ec in [(2,3), (4,5), (6,7)]:
            ws.merge_cells(start_row=sr, start_column=sc, end_row=sr, end_column=ec)
            ws.cell(row=sr, column=sc, value="").border = s.box_border
            ws.cell(row=sr, column=sc).fill = s.fill_background

        # Notas
        cr = sr + 2
        for off, txt in [
            (0, "A programação de CNG ocorre pelo processo de portabilidade intrínseca - via ABR"),
            (1, "O encaminhamento do CNG será programado nas localidades onde o ponto de entrega for informado"),
        ]:
            ws.merge_cells(start_row=cr+off, start_column=2, end_row=cr+off, end_column=10)
            ws.cell(row=cr+off, column=2, value=txt).font = s.font_small

        smr = cr + 3
        ws.merge_cells(start_row=smr, start_column=2, end_row=smr, end_column=10)
        ws.cell(row=smr, column=2, value="2.12 Tabela de Prefixos Vivo SMP").font = Font(name="Calibri", size=10, bold=True)
        ws.merge_cells(start_row=smr+1, start_column=2, end_row=smr+1, end_column=10)
        ws.cell(row=smr+1, column=2, value="Os prefixos da VIVO SMP, poderão ser obtidos acessando o site www.telefonica.net.br/sp/transfer/").font = s.font_small

        ws.freeze_panes = f"B{h1}"
        self._print_settings(ws, landscape=True)

    # --- ABA 7: DADOS MTL ---
    def _build_mtl(self):
        s = self.style
        ws = self.wb.create_sheet(title="Dados MTL")
        self._set_col_widths(ws, {1:2, 2:6, 3:8, 4:10, 5:12, 6:15, 7:15, 8:20, 9:15, 10:18, 11:18, 12:20})
        self._brand_header(ws, 2, 2, 11)
        self._section_title(ws, 4, 2, 11, "2.7. Dados de MTL")

        ws.merge_cells(start_row=5, start_column=2, end_row=5, end_column=11)
        ws.cell(row=5, column=2, value="Tabela de dados MTL para interligação entre VIVO e operadora.").font = s.font_body
        ws.cell(row=5, column=2).fill = s.fill_background
        ws.merge_cells(start_row=6, start_column=2, end_row=6, end_column=11)
        ws.cell(row=6, column=2, value="Preencher com os dados técnicos dos circuitos de interligação.").font = Font(name="Calibri", size=9, italic=True)
        ws.cell(row=6, column=2).fill = s.fill_background

        hr = 8
        for col, txt in [(2,"Ref"), (3,"Cn"), (4,"ID"), (5,"MTL"), (6,"PONTA VIVO"), (7,"PONTA IMPERIAL"),
                          (8,"DESIGNADOR DO CIRCUITO"), (9,"PROVEDOR"), (10,"Posição Física VIVO"),
                          (11,"Posição Física OPERADORA"), (12,"OBSERVAÇÕES")]:
            c = ws.cell(row=hr, column=col, value=txt)
            c.font = s.font_subheader; c.alignment = s.align_center; c.fill = s.fill_secondary; c.border = s.box_border
        ws.row_dimensions[hr].height = 25.0

        for i in range(10):
            r = hr + 1 + i; ws.row_dimensions[r].height = 20.0
            fill = s.alternating_fill(i)
            ws.cell(row=r, column=2, value=i+1)
            for col in range(2, 13):
                c = ws.cell(row=r, column=col)
                if col != 2: c.value = ""
                c.font = s.font_small; c.border = s.box_border; c.fill = fill
                c.alignment = s.align_left if col == 12 else s.align_center

        ws.freeze_panes = f"B{hr+1}"
        self._print_settings(ws, landscape=True)
    # --- ABA 8: PARÂMETROS DE PROGRAMAÇÃO ---
    def _build_parameters(self):
        s = self.style
        ws = self.wb.create_sheet(title="Parâmetros de Programação")
        nome = self.nome_op
        self._set_col_widths(ws, {1:2, 2:50, 3:35, 4:15, 5:12, 6:35})
        self._brand_header(ws, 2, 2, 6)
        self._section_title(ws, 4, 2, 6, "2.9 - Parâmetros de Programação")

        ws.merge_cells(start_row=5, start_column=2, end_row=5, end_column=6)
        ws.cell(row=5, column=2, value=f"Relação dos parâmetros de configuração de rotas SIP entre a TELEFÔNICA-VIVO e a Operadora {nome.upper()}.").font = s.font_body
        ws.cell(row=5, column=2).fill = s.fill_background

        cur = [7]  # Linha atual (lista para mutabilidade em closures)

        def _header(text):
            r = cur[0]
            ws.merge_cells(start_row=r, start_column=2, end_row=r, end_column=6)
            c = ws.cell(row=r, column=2, value=text)
            c.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
            c.alignment = s.align_center; c.fill = s.fill_light; c.border = s.box_border
            ws.row_dimensions[r].height = 20.0
            cur[0] += 1

        def _col_headers():
            r = cur[0]
            for ci, txt in enumerate(["Parâmetro", f"TELEFÔNICA VIVO <> {nome.upper()}", "Categoria", "Cumpre?", "Observação"], 2):
                c = ws.cell(row=r, column=ci, value=txt)
                c.font = s.font_subheader; c.alignment = s.align_center; c.fill = s.fill_secondary; c.border = s.box_border
            ws.row_dimensions[r].height = 25.0
            cur[0] += 1

        def _rows(data):
            for i, row_data in enumerate(data):
                r = cur[0]; fill = s.alternating_fill(i)
                for ci, val in enumerate(row_data, 2):
                    c = ws.cell(row=r, column=ci, value=val)
                    c.font = s.font_check if ci == 5 and val == "X" else s.font_small
                    c.alignment = s.align_left if ci == 2 else s.align_center
                    c.fill = fill; c.border = s.box_border
                cur[0] += 1

        def _section(title, data, col_hdrs=False):
            _header(title)
            if col_hdrs: _col_headers()
            _rows(data); cur[0] += 1

        # Cabeçalho principal
        ws.merge_cells(start_row=cur[0], start_column=2, end_row=cur[0], end_column=6)
        c = ws.cell(row=cur[0], column=2, value="Interconexão Nacional")
        c.font = Font(name="Calibri", size=11, bold=True, color="FFFFFF")
        c.alignment = s.align_center; c.fill = s.fill_primary; c.border = s.box_border
        cur[0] += 1

        # Todas as seções de parâmetros
        _section("Informação do Gateway", [
            ("Fabricante / Fornecedor", "sipwise", "Mandatório", "X", ""),
            ("Modelo - Versão", "Elementos Rede Fixa, Fixa I e Móvel", "Mandatório", "X", ""),
        ], col_hdrs=True)

        _section("Tipo de Serviço", [
            ("Voz", "SIM", "Mandatório", "X", ""),
            ("Fax", "SIM", "Mandatório", "X", ""),
            ("DTMF", "SIM", "Mandatório", "X", ""),
        ])

        _section("Protocolo", [("Tipo de Protocolo", "SIP-I (Q.1912.5)", "Mandatório", "X", "")])

        _section("Atributos SIP", [
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
            ("Envia DOMAIN IP no SDP para Controle de QoS e Segurança e Controle de Política", "Mandatório", "Mandatório", "X", ""),
        ])

        _section("Codec Rede Móvel", [
            ("AMR encaminhamento", "", "Mandatório", "X", ""),
            ("2ª opção G.711a 20ms", "", "Mandatório", "X", "Não aceita codec g.711u"),
        ])
        _section("Codec Rede Fixa", [
            ("G.711a 20ms", "", "Mandatório", "X", ""),
            ("2ª opção G.729a 20ms", "", "Mandatório", "X", "Não aceita codec g.711u"),
        ])
        _section("DTMF", [
            ("1ª Opção: RFC 2833 (Outband) com valor payload = 100", "", "Mandatório", "X", ""),
            ("Todos os payloads de DTMF definidos no padrão (96-125) podem ser usados", "", "Mandatório", "X", ""),
        ])
        _section("FAX", [("Protocolo T.38 suportado", "", "Mandatório", "X", "")])
        _section("POS", [
            ("Detecção do UPSEEED", "Utilização do g711a em re-INVITE", "Mandatório", "X", "Dados necessários no SDP"),
            ("Detecção do UPSEEED", "Utilização do g711u em re-INVITE", "Mandatório", "X", "Dados necessários no SDP"),
        ])
        _section("Encaminhamento Enfrante", [
            ("P-ASSERTED Identity (RFC 3325)", "SIM - Fomento E.164", "Mandatório", "X", ""),
            ("Nature of address of Calling", "SUB, NAT, UNKW", "Mandatório", "X", ""),
            ("B-NUMBER (Called Number)", "E.164 - Padrão Roteamento", "Mandatório", "X", ""),
            ("Nature of address", "SUB, NAT, UNKW", "Mandatório", "X", ""),
        ])
        _section("Encaminhamento Sainte", [
            ("A-NUMBER (Calling Number)", "E.164 - Nacional", "Mandatório", "X", ""),
            ("P-Asserted Identity (RFC 3325)", "SIM - Formato E.164", "Mandatório", "X", ""),
            ("Nature of address of Calling", "SUB, NAT, UNKW", "Mandatório", "X", ""),
            ("B-NUMBER (Called Number)", "E.164 - Padrão Roteamento", "Mandatório", "X", ""),
            ("Nature of address", "SUB, NAT, UNKW", "Mandatório", "X", ""),
        ])
        _section("Principais RFC's que devem estar habilitadas e/ou suportadas", [
            ("3261 - SIP Base", "", "", "", ""),
            ("3311 - UPDATE (PRACK)", "", "", "", ""),
            ("3264/4028 - Offer/Answer - SDP", "", "", "", ""),
            ("2833 - DTMF - RTP", "", "", "", ""),
            ("RFC 3398 - SIP-I interworking", "", "", "", ""),
            ("RFC 4033/4035 - DNSSEC", "", "", "", ""),
            ("RFC 4566 - SDP", "", "", "", ""),
            ("RFC 3606 - Early Media & Tone Generation", "", "", "", "SIP 180Ring (puro), o Proxy deverá gerar o RBT localmente"),
        ])
        _section("Método para alteração dos dados do SDP", [
            ("Método Update", "", "Mandatório", "", "Antes do Atendimento"),
            ("Método Update", "", "Mandatório", "", "Após atendimento"),
        ])
        _section("Negociação de Codec", [
            ("É mandatório a utilização de re-invites pelo Origem para a definição de codecs, nas chamadas em que o destino não define um codec único.", "", "Mandatório", "X", ""),
            ("O ptime presente no SDP answer deve ser múltiplo inteiro de 20 para codecs móveis; 3GPP 26.114", "", "Mandatório", "X", ""),
            ("Pacotes suportados às marcações do tipo USR, DSCP 46 ou EF para pacotes RTP", "", "Mandatório", "X", ""),
        ])
        self._print_settings(ws, landscape=True)

    # === ORQUESTRADOR ===
    def build(self) -> "Workbook":
        """Constrói o Workbook completo com todas as abas do PTI."""
        logger.info(f"Gerando PTI Excel para '{self.nome_op}'...")
        self._build_index()
        self._build_versions()
        self._build_diagram()
        self._build_routing()
        self._build_concentration()
        self._build_prefixes()
        self._build_mtl()
        self._build_parameters()

        self.wb.properties.title = f"PTI — Completo — {self.nome_op}"
        self.wb.properties.subject = "Projeto Técnico de Interligação"
        self.wb.properties.creator = "VIVOHUB"
        self.wb.properties.keywords = f"PTI, VIVO, {self.nome_op}, Interligação"
        self.wb.active = 0

        logger.info("PTI Excel gerado com sucesso.")
        return self.wb


# =============================================================================
# REGISTRO DE ROTAS HTTP
# =============================================================================
def _register_routes(app):

    # --- Públicas ---
    @app.get("/")
    def index():
        if "user_id" in session:
            return redirect(url_for(f"central_{session['role']}"))
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

    # --- Admin ---
    @app.get("/admin_login")
    def admin_login():
        return render_template("admin_login.html")

    @app.post("/admin_login")
    def admin_login_post():
        if request.form.get("code", "") == app.config["ADMIN_CODE"]:
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
            db.execute("INSERT INTO users (email, password_hash, role) VALUES (?, ?, ?)",
                        (email, generate_password_hash(password), role))
            db.commit()
            flash("Usuário criado com sucesso.", "success")
        except sqlite3.IntegrityError:
            flash("Este e-mail já está cadastrado.", "warning")
        return redirect(url_for("register"))

    # --- Centrais ---
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

    # --- Atacado CRUD ---
    @app.get("/atacado_formularios")
    @login_required
    @role_required("atacado")
    def atacado_form_list():
        db = get_db()
        q = (request.args.get("q") or "").strip()
        status = (request.args.get("status") or "").strip()
        sort = (request.args.get("sort") or "-created_at").strip()
        sql, params = build_list_query("SELECT * FROM atacado_forms",
            search_term=q or None, status_filter=status or None,
            owner_id=session["user_id"], sort_key=sort)
        forms = db.execute(sql, params).fetchall()
        counters = get_status_counters(db, "WHERE owner_id = ?", (session["user_id"],))
        return render_template("atacado_formularios.html", forms=forms, counters=counters)

    @app.get("/atacado_formularios/new")
    @login_required
    @role_required("atacado")
    def atacado_form_new():
        db = get_db()
        last = db.execute(
            "SELECT responsavel_atacado FROM atacado_forms WHERE owner_id = ? AND COALESCE(responsavel_atacado,'')<>'' ORDER BY created_at DESC LIMIT 1",
            (session["user_id"],)).fetchone()
        preset = (last["responsavel_atacado"] if last else "") or session.get("email", "").split("@")[0].replace(".", " ").title()
        return render_template("formulario_atacado.html", form=None, preset_responsavel_atacado=preset)

    @app.post("/atacado_formularios/new")
    @login_required
    @role_required("atacado")
    def atacado_form_create():
        payload = extract_form_payload()
        payload["owner_id"] = session["user_id"]
        payload["status"] = (payload.get("status") or "rascunho").lower()
        payload["engenharia_params_json"] = "{}"
        truncated = validate_table_rows(payload)

        if payload["status"] == "enviado" and not (payload.get("responsavel_atacado") or "").strip():
            flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.', "warning")
            return redirect(url_for("atacado_form_new"))

        cols = ["owner_id"] + list(TEXT_FIELDS) + list(BOOLEAN_FIELDS) + ["escopo_flags_json", "dados_vivo_json", "dados_operadora_json", "engenharia_params_json"]
        sql = f"INSERT INTO atacado_forms ({','.join(cols)}) VALUES ({','.join(['?']*len(cols))})"
        db = get_db()
        db.execute(sql, [payload.get(c) for c in cols])
        db.commit()

        msg = "Formulário criado."
        if truncated: msg += f" Tabelas limitadas a {MAX_TABLE_ROWS} linhas."
        flash(msg, "success")
        return redirect(url_for("atacado_form_list"))

    @app.get("/atacado_formularios/<int:form_id>")
    @login_required
    @role_required("atacado")
    def atacado_form_edit(form_id):
        db = get_db()
        form = db.execute("SELECT * FROM atacado_forms WHERE id = ? AND owner_id = ?", (form_id, session["user_id"])).fetchone()
        if not form: abort(404)
        return render_template("formulario_atacado.html", form=form)

    @app.post("/atacado_formularios/<int:form_id>")
    @login_required
    @role_required("atacado")
    def atacado_form_update(form_id):
        db = get_db()
        if not db.execute("SELECT id FROM atacado_forms WHERE id = ? AND owner_id = ?", (form_id, session["user_id"])).fetchone():
            abort(404)
        payload = extract_form_payload()
        payload["updated_at"] = datetime.utcnow().isoformat(timespec="seconds")
        truncated = validate_table_rows(payload)

        if (payload.get("status") or "").lower() == "enviado" and not (payload.get("responsavel_atacado") or "").strip():
            flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.', "warning")
            return redirect(url_for("atacado_form_edit", form_id=form_id))

        fields = list(TEXT_FIELDS) + list(BOOLEAN_FIELDS) + ["escopo_flags_json", "dados_vivo_json", "dados_operadora_json"]
        set_clause = ", ".join([f"{f} = ?" for f in fields] + ["updated_at = ?"])
        params = [payload.get(f) for f in fields] + [payload["updated_at"], form_id, session["user_id"]]
        db.execute(f"UPDATE atacado_forms SET {set_clause} WHERE id = ? AND owner_id = ?", params)
        db.commit()

        msg = "Formulário atualizado."
        if truncated: msg += f" Tabelas limitadas a {MAX_TABLE_ROWS} linhas."
        flash(msg, "success")
        return redirect(url_for("atacado_form_list"))

    @app.post("/atacado_formularios/<int:form_id>/delete")
    @login_required
    @role_required("atacado")
    def atacado_form_delete(form_id):
        db = get_db()
        db.execute("DELETE FROM atacado_forms WHERE id = ? AND owner_id = ?", (form_id, session["user_id"]))
        db.commit()
        flash("Formulário excluído.", "info")
        return redirect(url_for("atacado_form_list"))

    # --- Engenharia ---
    @app.get("/engenharia_formularios")
    @login_required
    @role_required("engenharia")
    def engenharia_form_list():
        db = get_db()
        q = (request.args.get("q") or "").strip()
        status = (request.args.get("status") or "").strip()
        sort = (request.args.get("sort") or "-created_at").strip()
        sql, params = build_list_query("SELECT * FROM atacado_forms",
            search_term=q or None, status_filter=status or None, sort_key=sort)
        forms = db.execute(sql, params).fetchall()
        counters = get_status_counters(db)

        form_filter = request.args.get("form")
        if form_filter and form_filter.isdigit():
            exports = db.execute(
                "SELECT e.id, e.form_id, e.filename, e.size_bytes, e.created_at, f.nome_operadora "
                "FROM exports e JOIN atacado_forms f ON f.id = e.form_id WHERE e.form_id = ? ORDER BY e.created_at DESC LIMIT 100",
                (int(form_filter),)).fetchall()
        else:
            exports = db.execute(
                "SELECT e.id, e.form_id, e.filename, e.size_bytes, e.created_at, f.nome_operadora "
                "FROM exports e JOIN atacado_forms f ON f.id = e.form_id ORDER BY e.created_at DESC LIMIT 100").fetchall()

        return render_template("engenharia_formularios.html", forms=forms, counters=counters,
                               exports=exports, show_files=request.args.get("show_files")=="1", form_filter=form_filter)

    @app.route("/engenharia_formularios/<int:form_id>", methods=["GET", "POST"])
    @login_required
    @role_required("engenharia")
    def engenharia_form_view(form_id):
        db = get_db()
        form = db.execute("SELECT * FROM atacado_forms WHERE id = ?", (form_id,)).fetchone()
        if not form: abort(404)

        if request.method == "POST":
            resp_eng = (request.form.get("responsavel_engenharia") or "").strip()
            if not resp_eng:
                flash('Informe o "Responsável Eng de ITX" para salvar.', "warning")
                return redirect(url_for("engenharia_form_view", form_id=form_id))

            raw = request.form.get("engenharia_params_json", "") or "{}"
            eng_json = _parse_json_dict(raw)
            updated = datetime.utcnow().isoformat(timespec="seconds")
            db.execute("UPDATE atacado_forms SET engenharia_params_json = ?, responsavel_engenharia = ?, updated_at = ? WHERE id = ?",
                        (eng_json, resp_eng, updated, form_id))
            db.commit()
            flash("Validação da Engenharia salva.", "success")
            return redirect(url_for("engenharia_form_list", show_files="1", form=form_id))

        return render_template("formulario_atacado.html", form=form, readonly=True)

    # --- Exportação Excel ---
    @app.get("/formularios/<int:form_id>/excel_index")
    @login_required
    def exportar_form_excel_index(form_id):
        """Exporta PTI completo em Excel."""
        if not OPENPYXL_AVAILABLE:
            flash("openpyxl não instalado.", "warning")
            return redirect(url_for("index"))
        db = get_db()
        form = db.execute("SELECT * FROM atacado_forms WHERE id = ?", (form_id,)).fetchone()
        if not form: abort(404)
        if session.get("role") == "atacado" and form["owner_id"] != session.get("user_id"):
            abort(403)

        wb = PTIWorkbookBuilder(form).build()
        buf = BytesIO()
        wb.save(buf); buf.seek(0)
        nome = safe_filename(form["nome_operadora"] or "Operadora")
        return send_file(buf, mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                         as_attachment=True, download_name=f"PTI_Completo_{nome}_ID{form['id']}.xlsx")

    @app.post("/engenharia_formularios/<int:form_id>/generate_excel")
    @login_required
    @role_required("engenharia")
    def engenharia_generate_excel(form_id):
        if not OPENPYXL_AVAILABLE:
            flash("openpyxl não instalado.", "warning")
            return redirect(url_for("engenharia_form_list"))
        db = get_db()
        form = db.execute("SELECT id, nome_operadora FROM atacado_forms WHERE id = ?", (form_id,)).fetchone()
        if not form: abort(404)

        wb = Workbook(); wb.active.title = "Índice"
        ts = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        fname = f"PTI_Indice_{safe_filename(form['nome_operadora'] or 'Operadora')}_ID{form_id}_{ts}.xlsx"
        fpath = os.path.join(app.config["EXPORT_DIR"], fname)
        wb.save(fpath)
        db.execute("INSERT INTO exports (form_id, filename, filepath, size_bytes) VALUES (?, ?, ?, ?)",
                    (form_id, fname, fpath, os.path.getsize(fpath)))
        db.commit()
        flash(f"Excel gerado: {fname}", "success")
        return redirect(url_for("engenharia_form_list", show_files="1", form=form_id))

    @app.get("/engenharia_exports/<int:export_id>/download")
    @login_required
    @role_required("engenharia")
    def engenharia_export_download(export_id):
        db = get_db()
        row = db.execute("SELECT filename, filepath FROM exports WHERE id = ?", (export_id,)).fetchone()
        if not row: abort(404)
        if not os.path.exists(row["filepath"]):
            flash("Arquivo não encontrado.", "warning")
            return redirect(url_for("engenharia_form_list", show_files="1"))
        return send_file(row["filepath"], mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                         as_attachment=True, download_name=row["filename"])

    @app.post("/engenharia_exports/<int:export_id>/delete")
    @login_required
    @role_required("engenharia")
    def engenharia_export_delete(export_id):
        db = get_db()
        row = db.execute("SELECT filepath FROM exports WHERE id = ?", (export_id,)).fetchone()
        if not row: abort(404)
        try:
            if os.path.exists(row["filepath"]): os.remove(row["filepath"])
        except OSError: pass
        db.execute("DELETE FROM exports WHERE id = ?", (export_id,))
        db.commit()
        flash("Arquivo removido.", "info")
        return redirect(url_for("engenharia_form_list", show_files="1"))

    # --- API interna ---
    @app.get("/api/cns")
    @login_required
    def api_cns():
        """Lista CNs ativos em formato JSON."""
        db = get_db()
        rows = db.execute("SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf FROM cns WHERE ativo=1 ORDER BY codigo ASC").fetchall()
        return jsonify([{"codigo": r["codigo"], "nome": r["nome"], "uf": r["uf"]} for r in rows])


# =============================================================================
# CLI COMMANDS
# =============================================================================
def _register_cli_commands(app):
    @app.cli.command("create-user")
    @click.argument("email")
    @click.argument("password")
    @click.argument("role")
    def create_user_cmd(email, password, role):
        """Cria usuário via CLI: flask create-user EMAIL SENHA ROLE"""
        role = role.strip().lower()
        if role not in ("engenharia", "atacado"):
            click.echo("Role inválida. Use: engenharia ou atacado.")
            return
        db = get_db()
        try:
            db.execute("INSERT INTO users (email, password_hash, role) VALUES (?, ?, ?)",
                        (email.strip().lower(), generate_password_hash(password), role))
            db.commit()
            click.echo("Usuário criado com sucesso.")
        except sqlite3.IntegrityError:
            click.echo("E-mail já cadastrado.")

    @app.cli.command("seed-cns")
    def seed_cns_cmd():
        """Reaplica seed de CNs com metadados."""
        db = get_db()
        _seed_cns(db)
        _apply_cn_metadata(db)
        click.echo("Seed de CNs aplicado/atualizado.")


# =============================================================================
# ENTRYPOINT
# =============================================================================
app = create_app()

if __name__ == "__main__":
    app.run(debug=True)
