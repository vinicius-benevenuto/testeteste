"""run.py — Entrypoint para desenvolvimento local.

Uso:
    python run.py
    # ou via Flask CLI:
    flask --app app run --debug
"""
from app import create_app

application = create_app()   # nome 'application' compatível com WSGI (Gunicorn/uWSGI)

if __name__ == "__main__":
    application.run(debug=False, use_reloader=False)
