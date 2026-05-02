"""routes/engenharia.py — Formulários, validação de engenharia e geração de Excel."""
import logging
from datetime import datetime
from io import BytesIO

from flask import (
    Blueprint, abort, flash, redirect, render_template,
    request, send_file, session, url_for, current_app,
)

from auth import login_required, role_required
from db import get_db
from forms import _parse_json_dict, truncate_json_list
from helpers import safe_filename
from queries import build_list_query, get_status_counters

logger = logging.getLogger(__name__)
bp = Blueprint("engenharia", __name__)


# ---------------------------------------------------------------------------
# LISTAGEM
# ---------------------------------------------------------------------------
@bp.get("/engenharia_formularios")
@login_required
@role_required("engenharia")
def form_list():
    db     = get_db()
    q      = (request.args.get("q") or "").strip()
    status = (request.args.get("status") or "").strip()
    sort   = (request.args.get("sort") or "-created_at").strip()

    sql, params = build_list_query(
        "SELECT * FROM atacado_forms",
        search_term=q or None,
        status_filter=status or None,
        sort_key="nome_operadora",
    )
    forms    = db.execute(sql, params).fetchall()
    counters = get_status_counters(db)

    # Agrupar por operadora
    grupos: dict[str, list] = {}
    for f in forms:
        op = (f["nome_operadora"] or "Sem operadora").strip()
        grupos.setdefault(op, []).append(f)
    # Ordenar versões dentro de cada grupo
    for op in grupos:
        grupos[op].sort(key=lambda r: int(r["version"] or 1))

    return render_template(
        "engenharia_formularios.html",
        grupos=grupos,
        counters=counters,
        q=q,
        status=status,
    )


# ---------------------------------------------------------------------------
# VISUALIZAR / VALIDAR
# ---------------------------------------------------------------------------
@bp.route("/engenharia_formularios/<int:form_id>", methods=["GET", "POST"])
@login_required
@role_required("engenharia")
def form_view(form_id: int):
    db   = get_db()
    form = db.execute(
        "SELECT * FROM atacado_forms WHERE id = ?", (form_id,)
    ).fetchone()
    if not form:
        abort(404)

    if request.method == "POST":
        resp_eng = (request.form.get("responsavel_engenharia") or "").strip()
        if not resp_eng:
            flash('Informe o "Responsável Eng de ITX" para salvar.', "warning")
            return redirect(url_for("engenharia.form_view", form_id=form_id))

        eng_json  = _parse_json_dict(
            request.form.get("engenharia_params_json", "") or "{}"
        )
        vivo_json = truncate_json_list(
            request.form.get("dados_vivo_json", "[]"), "[]"
        )
        op_json   = truncate_json_list(
            request.form.get("dados_operadora_json", "[]"), "[]"
        )
        db.execute(
            "UPDATE atacado_forms "
            "SET engenharia_params_json=?, responsavel_engenharia=?, "
            "dados_vivo_json=?, dados_operadora_json=?, updated_at=? "
            "WHERE id=?",
            (
                eng_json, resp_eng, vivo_json, op_json,
                datetime.utcnow().isoformat(timespec="seconds"),
                form_id,
            ),
        )
        db.commit()
        flash("Validação da Engenharia salva.", "success")
        return redirect(url_for("engenharia.form_list"))

    return render_template("formulario_atacado.html", form=form, readonly=True)


# ---------------------------------------------------------------------------
# VALIDAR (APROVAR) PTI
# ---------------------------------------------------------------------------
@bp.post("/engenharia_formularios/<int:form_id>/validar")
@login_required
@role_required("engenharia")
def validar(form_id: int):
    db = get_db()
    form = db.execute(
        "SELECT id, nome_operadora FROM atacado_forms WHERE id = ?", (form_id,)
    ).fetchone()
    if not form:
        abort(404)
    db.execute(
        "UPDATE atacado_forms SET status='aprovado', updated_at=? WHERE id=?",
        (datetime.utcnow().isoformat(timespec="seconds"), form_id),
    )
    db.commit()
    flash(f"PTI #{form_id} — {form['nome_operadora'] or ''} aprovado.", "success")
    return redirect(url_for("engenharia.form_list"))


def _get_all_versions(db, form) -> list:
    """Retorna todas as versões relacionadas ao PTI (ancestors + current), ordenadas por versão."""
    rows = []
    # Subir até a raiz pelo parent_id
    cur = form
    visited = set()
    while cur and cur["id"] not in visited:
        visited.add(cur["id"])
        rows.append(cur)
        if cur["parent_id"]:
            parent = db.execute(
                "SELECT * FROM atacado_forms WHERE id = ?", (cur["parent_id"],)
            ).fetchone()
            cur = parent
        else:
            break
    # Descer: buscar todos os filhos/netos
    def _children(fid):
        for child in db.execute(
            "SELECT * FROM atacado_forms WHERE parent_id = ?", (fid,)
        ).fetchall():
            if child["id"] not in visited:
                visited.add(child["id"])
                rows.append(child)
                _children(child["id"])
    root_id = rows[-1]["id"] if rows else form["id"]
    _children(root_id)
    # Ordenar por version ASC
    rows.sort(key=lambda r: int(r["version"] or 1) if r["version"] else 1)
    return rows


# ---------------------------------------------------------------------------
# EXPORTAR EXCEL COMPLETO (PTI)
# ---------------------------------------------------------------------------
@bp.get("/formularios/<int:form_id>/excel_index")
@login_required
def exportar_excel(form_id: int):
    try:
        from excel.builder import PTIWorkbookBuilder
    except ImportError:
        flash("openpyxl não instalado.", "warning")
        return redirect(url_for("central.index"))

    db   = get_db()
    form = db.execute(
        "SELECT * FROM atacado_forms WHERE id = ?", (form_id,)
    ).fetchone()
    if not form:
        abort(404)
    if session.get("role") == "atacado" and form["owner_id"] != session.get("user_id"):
        abort(403)

    wb  = PTIWorkbookBuilder(form, all_versions=_get_all_versions(db, form)).build()
    buf = BytesIO()
    wb.save(buf)
    buf.seek(0)
    nome = safe_filename(form["nome_operadora"] or "Operadora")
    ver  = form["version"] if "version" in form.keys() else 1
    return send_file(
        buf,
        mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        as_attachment=True,
        download_name=f"PTI_{nome}_v{ver}_ID{form['id']}.xlsx",
    )

