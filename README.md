"""routes/central.py — Rotas das centrais (landing pages por role)."""
from flask import Blueprint, render_template
from auth import login_required, role_required

bp = Blueprint("central", __name__)


@bp.get("/")
@login_required
def index():
    from flask import session, redirect, url_for
    return redirect(url_for(f"central.central_{session['role']}"))


@bp.get("/central_atacado")
@login_required
@role_required("atacado")
def central_atacado():
    return render_template("central_atacado.html")


@bp.get("/central_engenharia")
@login_required
@role_required("engenharia")
def central_engenharia():
    return render_template("central_engenharia.html")
