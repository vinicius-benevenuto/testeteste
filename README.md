"""routes/engenharia.py — Formulários, validação de engenharia e geração de Excel."""
import logging
import os
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
    ff     = request.args.get("form")

    sql, params = build_list_query(
        "SELECT * FROM atacado_forms",
        search_term=q or None,
        status_filter=status or None,
        sort_key=sort,
    )
    forms    = db.execute(sql, params).fetchall()
    counters = get_status_counters(db)

    if ff and ff.isdigit():
        exports = db.execute(
            "SELECT e.id, e.form_id, e.filename, e.size_bytes, e.created_at, "
            "f.nome_operadora "
            "FROM exports e JOIN atacado_forms f ON f.id = e.form_id "
            "WHERE e.form_id = ? ORDER BY e.created_at DESC LIMIT 100",
            (int(ff),),
        ).fetchall()
    else:
        exports = db.execute(
            "SELECT e.id, e.form_id, e.filename, e.size_bytes, e.created_at, "
            "f.nome_operadora "
            "FROM exports e JOIN atacado_forms f ON f.id = e.form_id "
            "ORDER BY e.created_at DESC LIMIT 100"
        ).fetchall()

    return render_template(
        "engenharia_formularios.html",
        forms=forms,
        counters=counters,
        exports=exports,
        show_files=request.args.get("show_files") == "1",
        form_filter=ff,
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
        db.execute(
            "UPDATE atacado_forms "
            "SET engenharia_params_json=?, responsavel_engenharia=?, "
            "dados_vivo_json=?, updated_at=? "
            "WHERE id=?",
            (
                eng_json, resp_eng, vivo_json,
                datetime.utcnow().isoformat(timespec="seconds"),
                form_id,
            ),
        )
        db.commit()
        flash("Validação da Engenharia salva.", "success")
        return redirect(
            url_for("engenharia.form_list", show_files="1", form=form_id)
        )

    return render_template("formulario_atacado.html", form=form, readonly=True)


# ---------------------------------------------------------------------------
# EXPORTAR EXCEL COMPLETO (PTI)
# ---------------------------------------------------------------------------
@bp.get("/formularios/<int:form_id>/excel_index")
@login_required
def exportar_excel(form_id: int):
    try:
        from openpyxl import Workbook as _WB  # noqa – verifica disponibilidade
        from excel.builder import PTIWorkbookBuilder
        OPENPYXL_OK = True
    except ImportError:
        OPENPYXL_OK = False

    if not OPENPYXL_OK:
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

    wb  = PTIWorkbookBuilder(
        form, sbc_analyzer=current_app.config.get("SBC_ANALYZER")
    ).build()
    buf = BytesIO()
    wb.save(buf)
    buf.seek(0)
    nome = safe_filename(form["nome_operadora"] or "Operadora")
    return send_file(
        buf,
        mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        as_attachment=True,
        download_name=f"PTI_Completo_{nome}_ID{form['id']}.xlsx",
    )


# ---------------------------------------------------------------------------
# GERAR EXCEL SIMPLES (Índice salvo em disco)
# ---------------------------------------------------------------------------
@bp.post("/engenharia_formularios/<int:form_id>/generate_excel")
@login_required
@role_required("engenharia")
def generate_excel(form_id: int):
    try:
        from openpyxl import Workbook
        OPENPYXL_OK = True
    except ImportError:
        OPENPYXL_OK = False

    if not OPENPYXL_OK:
        flash("openpyxl não instalado.", "warning")
        return redirect(url_for("engenharia.form_list"))

    from openpyxl import Workbook

    db   = get_db()
    form = db.execute(
        "SELECT id, nome_operadora FROM atacado_forms WHERE id = ?", (form_id,)
    ).fetchone()
    if not form:
        abort(404)

    wb = Workbook()
    wb.active.title = "Índice"
    ts    = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    fname = (
        f"PTI_Indice_{safe_filename(form['nome_operadora'] or 'Operadora')}"
        f"_ID{form_id}_{ts}.xlsx"
    )
    fpath = os.path.join(current_app.config["EXPORT_DIR"], fname)
    wb.save(fpath)

    db.execute(
        "INSERT INTO exports (form_id, filename, filepath, size_bytes) "
        "VALUES (?, ?, ?, ?)",
        (form_id, fname, fpath, os.path.getsize(fpath)),
    )
    db.commit()
    flash(f"Excel gerado: {fname}", "success")
    return redirect(
        url_for("engenharia.form_list", show_files="1", form=form_id)
    )


# ---------------------------------------------------------------------------
# DOWNLOAD DE EXPORT
# ---------------------------------------------------------------------------
@bp.get("/engenharia_exports/<int:export_id>/download")
@login_required
@role_required("engenharia")
def export_download(export_id: int):
    db  = get_db()
    row = db.execute(
        "SELECT filename, filepath FROM exports WHERE id = ?", (export_id,)
    ).fetchone()
    if not row:
        abort(404)
    if not os.path.exists(row["filepath"]):
        flash("Arquivo não encontrado.", "warning")
        return redirect(url_for("engenharia.form_list", show_files="1"))
    return send_file(
        row["filepath"],
        mimetype="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        as_attachment=True,
        download_name=row["filename"],
    )


# ---------------------------------------------------------------------------
# DELETAR EXPORT
# ---------------------------------------------------------------------------
@bp.post("/engenharia_exports/<int:export_id>/delete")
@login_required
@role_required("engenharia")
def export_delete(export_id: int):
    db  = get_db()
    row = db.execute(
        "SELECT filepath FROM exports WHERE id = ?", (export_id,)
    ).fetchone()
    if not row:
        abort(404)
    try:
        if os.path.exists(row["filepath"]):
            os.remove(row["filepath"])
    except OSError:
        pass
    db.execute("DELETE FROM exports WHERE id = ?", (export_id,))
    db.commit()
    flash("Arquivo removido.", "info")
    return redirect(url_for("engenharia.form_list", show_files="1"))
