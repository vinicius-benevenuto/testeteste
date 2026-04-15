"""routes/atacado.py — CRUD de formulários do perfil Atacado + versionamento."""
import json
import logging
from datetime import datetime

from flask import (
    Blueprint, abort, flash, redirect, render_template,
    request, session, url_for,
)

from auth import login_required, role_required
from config import BOOLEAN_FIELDS, MAX_TABLE_ROWS, TEXT_FIELDS
from db import get_db
from forms import extract_form_payload, validate_table_rows
from queries import build_list_query, get_status_counters

logger = logging.getLogger(__name__)
bp = Blueprint("atacado", __name__, url_prefix="/atacado_formularios")

# Valores proibidos nos campos de texto
_FORBIDDEN = {"n/a", "na", "não se aplica", "nao se aplica", "-", "--", "none", "null", ""}


def _field_invalid(val: str) -> bool:
    """Retorna True se o valor é proibido ou genérico."""
    return (val or "").strip().lower() in _FORBIDDEN


# ---------------------------------------------------------------------------
# LISTAGEM — agrupada por operadora
# ---------------------------------------------------------------------------
@bp.get("")
@login_required
@role_required("atacado")
def form_list():
    db     = get_db()
    q      = (request.args.get("q") or "").strip()
    status = (request.args.get("status") or "").strip()
    sort   = (request.args.get("sort") or "-created_at").strip()

    sql, params = build_list_query(
        "SELECT * FROM atacado_forms",
        search_term=q or None,
        status_filter=status or None,
        owner_id=session["user_id"],
        sort_key=sort,
    )
    forms = db.execute(sql, params).fetchall()

    # Agrupar por operadora
    grupos: dict[str, list] = {}
    for f in forms:
        op = (f["nome_operadora"] or "Sem operadora").strip()
        grupos.setdefault(op, []).append(f)

    return render_template(
        "atacado_formularios.html",
        grupos=grupos,
        counters=get_status_counters(db, "WHERE owner_id = ?", (session["user_id"],)),
    )


# ---------------------------------------------------------------------------
# CRIAR
# ---------------------------------------------------------------------------
@bp.get("/new")
@login_required
@role_required("atacado")
def form_new():
    db   = get_db()
    last = db.execute(
        "SELECT responsavel_atacado FROM atacado_forms "
        "WHERE owner_id = ? AND COALESCE(responsavel_atacado,'') <> '' "
        "ORDER BY created_at DESC LIMIT 1",
        (session["user_id"],),
    ).fetchone()
    preset = (last["responsavel_atacado"] if last else "") or (
        session.get("email", "").split("@")[0].replace(".", " ").title()
    )
    return render_template(
        "formulario_atacado.html",
        form=None,
        preset_responsavel_atacado=preset,
    )


@bp.post("/new")
@login_required
@role_required("atacado")
def form_create():
    payload = extract_form_payload()
    payload["owner_id"] = session["user_id"]
    payload["status"]   = (payload.get("status") or "rascunho").lower()
    payload["engenharia_params_json"] = "{}"
    payload["version"]   = 1
    payload["parent_id"] = None
    truncated = validate_table_rows(payload)

    if payload["status"] == "enviado" and _field_invalid(
        payload.get("responsavel_atacado", "")
    ):
        flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.', "warning")
        return redirect(url_for("atacado.form_new"))

    cols = (
        ["owner_id", "version", "parent_id"]
        + list(TEXT_FIELDS)
        + list(BOOLEAN_FIELDS)
        + ["escopo_flags_json", "dados_vivo_json", "dados_operadora_json",
           "engenharia_params_json"]
    )
    db = get_db()
    db.execute(
        f"INSERT INTO atacado_forms ({','.join(cols)}) "
        f"VALUES ({','.join(['?'] * len(cols))})",
        [payload.get(c) for c in cols],
    )
    db.commit()

    msg = "Formulário criado."
    if truncated:
        msg += f" Tabelas limitadas a {MAX_TABLE_ROWS} linhas."
    flash(msg, "success")
    return redirect(url_for("atacado.form_list"))


# ---------------------------------------------------------------------------
# NOVA VERSÃO
# ---------------------------------------------------------------------------
@bp.post("/<int:form_id>/new_version")
@login_required
@role_required("atacado")
def form_new_version(form_id: int):
    db   = get_db()
    orig = db.execute(
        "SELECT * FROM atacado_forms WHERE id = ? AND owner_id = ?",
        (form_id, session["user_id"]),
    ).fetchone()
    if not orig:
        abort(404)

    # Calcular próxima versão para esta operadora/rn1
    root_id = orig["parent_id"] or orig["id"]
    max_ver = db.execute(
        "SELECT MAX(version) FROM atacado_forms "
        "WHERE owner_id = ? AND (id = ? OR parent_id = ?)",
        (session["user_id"], root_id, root_id),
    ).fetchone()[0] or 1

    next_version = max_ver + 1

    # Copiar todos os campos TEXT + BOOLEAN + JSON
    copy_cols = (
        list(TEXT_FIELDS) + list(BOOLEAN_FIELDS)
        + ["escopo_flags_json", "dados_vivo_json", "dados_operadora_json"]
    )
    row_dict = dict(orig)

    db.execute(
        f"INSERT INTO atacado_forms "
        f"(owner_id, version, parent_id, status, {','.join(copy_cols)}, engenharia_params_json) "
        f"VALUES (?, ?, ?, 'rascunho', {','.join(['?']*len(copy_cols))}, '{{}}')",
        [session["user_id"], next_version, root_id]
        + [row_dict.get(c) for c in copy_cols],
    )
    db.commit()

    flash(f"Versão {next_version} criada a partir do formulário #{form_id}.", "success")
    return redirect(url_for("atacado.form_list"))


# ---------------------------------------------------------------------------
# EDITAR
# ---------------------------------------------------------------------------
@bp.get("/<int:form_id>")
@login_required
@role_required("atacado")
def form_edit(form_id: int):
    db   = get_db()
    form = db.execute(
        "SELECT * FROM atacado_forms WHERE id = ? AND owner_id = ?",
        (form_id, session["user_id"]),
    ).fetchone()
    if not form:
        abort(404)
    return render_template("formulario_atacado.html", form=form)


@bp.post("/<int:form_id>")
@login_required
@role_required("atacado")
def form_update(form_id: int):
    db = get_db()
    if not db.execute(
        "SELECT id FROM atacado_forms WHERE id = ? AND owner_id = ?",
        (form_id, session["user_id"]),
    ).fetchone():
        abort(404)

    payload               = extract_form_payload()
    payload["updated_at"] = datetime.utcnow().isoformat(timespec="seconds")
    truncated             = validate_table_rows(payload)

    if (payload.get("status") or "").lower() == "enviado" and _field_invalid(
        payload.get("responsavel_atacado", "")
    ):
        flash('Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.', "warning")
        return redirect(url_for("atacado.form_edit", form_id=form_id))

    fields = (
        list(TEXT_FIELDS) + list(BOOLEAN_FIELDS)
        + ["escopo_flags_json", "dados_vivo_json", "dados_operadora_json"]
    )
    params = (
        [payload.get(f) for f in fields]
        + [payload["updated_at"], form_id, session["user_id"]]
    )
    db.execute(
        f"UPDATE atacado_forms SET "
        f"{', '.join([f'{f} = ?' for f in fields] + ['updated_at = ?'])} "
        f"WHERE id = ? AND owner_id = ?",
        params,
    )
    db.commit()

    msg = "Formulário atualizado."
    if truncated:
        msg += f" Tabelas limitadas a {MAX_TABLE_ROWS} linhas."
    flash(msg, "success")
    return redirect(url_for("atacado.form_list"))


# ---------------------------------------------------------------------------
# DELETAR
# ---------------------------------------------------------------------------
@bp.post("/<int:form_id>/delete")
@login_required
@role_required("atacado")
def form_delete(form_id: int):
    db = get_db()
    db.execute(
        "DELETE FROM atacado_forms WHERE id = ? AND owner_id = ?",
        (form_id, session["user_id"]),
    )
    db.commit()
    flash("Formulário excluído.", "info")
    return redirect(url_for("atacado.form_list"))
