"""routes/atacado.py — CRUD de formulários do perfil Atacado."""
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


# ---------------------------------------------------------------------------
# LISTAGEM
# ---------------------------------------------------------------------------
@bp.get("")
@login_required
@role_required("atacado")
def form_list():
    db     = get_db()
    uid    = session["user_id"]
    q      = (request.args.get("q") or "").strip()
    status = (request.args.get("status") or "").strip()
    sort   = (request.args.get("sort") or "-created_at").strip()

    sort_map = {
        "created_at":      "created_at ASC",
        "-created_at":     "created_at DESC",
        "nome_operadora":  "nome_operadora COLLATE NOCASE ASC",
        "-nome_operadora": "nome_operadora COLLATE NOCASE DESC",
        "id":  "id ASC",
        "-id": "id DESC",
    }
    order = sort_map.get(sort, "id DESC")

    sql    = "SELECT * FROM atacado_forms WHERE CAST(owner_id AS TEXT) = CAST(? AS TEXT)"
    params = [uid]
    if status:
        sql += " AND LOWER(COALESCE(status,'')) = LOWER(?)"
        params.append(status)
    if q:
        sql += " AND nome_operadora LIKE ?"
        params.append(f"%{q}%")
    sql += f" ORDER BY {order}"

    logger.info("form_list | uid=%r status=%r q=%r | sql: %s | params: %s",
                uid, status, q, sql, params)
    forms = db.execute(sql, params).fetchall()
    logger.info("form_list | %d formulário(s) encontrado(s)", len(forms))

    # Contadores
    counters = {}
    for key, cond in [
        ("total",     ""),
        ("rascunho",  "AND LOWER(status)='rascunho'"),
        ("enviado",   "AND LOWER(status)='enviado'"),
        ("em_revisao","AND LOWER(status)='em revisão'"),
        ("aprovado",  "AND LOWER(status)='aprovado'"),
    ]:
        row = db.execute(
            f"SELECT COUNT(*) AS c FROM atacado_forms "
            f"WHERE CAST(owner_id AS TEXT)=CAST(? AS TEXT) {cond}", (uid,)
        ).fetchone()
        counters[key] = row["c"] if row else 0

    return render_template(
        "atacado_formularios.html",
        forms=forms,
        counters=counters,
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
    payload                        = extract_form_payload()
    payload["owner_id"]            = session["user_id"]
    payload["status"]              = (payload.get("status") or "rascunho").lower()
    payload["engenharia_params_json"] = "{}"
    truncated                      = validate_table_rows(payload)

    if payload["status"] == "enviado" and not (
        payload.get("responsavel_atacado") or ""
    ).strip():
        flash(
            'Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.',
            "warning",
        )
        return redirect(url_for("atacado.form_new"))

    cols = (
        ["owner_id"]
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

    payload              = extract_form_payload()
    payload["updated_at"] = datetime.utcnow().isoformat(timespec="seconds")
    truncated            = validate_table_rows(payload)

    if (payload.get("status") or "").lower() == "enviado" and not (
        payload.get("responsavel_atacado") or ""
    ).strip():
        flash(
            'Informe o "Responsável Gestão de ITX (Atacado)" antes de finalizar.',
            "warning",
        )
        return redirect(url_for("atacado.form_edit", form_id=form_id))

    fields = (
        list(TEXT_FIELDS)
        + list(BOOLEAN_FIELDS)
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

