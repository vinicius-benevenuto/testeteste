"""auth.py — Decoradores de autenticação e headers de segurança."""
from functools import wraps
from flask import Flask, flash, redirect, session, url_for


# =============================================================================
# HEADERS DE SEGURANÇA
# =============================================================================
def _register_security_headers(app: Flask) -> None:
    @app.after_request
    def add_headers(resp):
        resp.headers.setdefault("X-Content-Type-Options", "nosniff")
        resp.headers.setdefault("X-Frame-Options", "SAMEORIGIN")
        resp.headers.setdefault("X-XSS-Protection", "0")
        return resp


# =============================================================================
# DECORADORES
# =============================================================================
def login_required(view):
    @wraps(view)
    def wrapped(*args, **kwargs):
        if "user_id" not in session:
            return redirect(url_for("login"))
        return view(*args, **kwargs)
    return wrapped


def role_required(required_role: str):
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
