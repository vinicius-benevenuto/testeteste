"""app.py — Factory principal da aplicação VIVOHUB."""
import logging
import os
from pathlib import Path

from flask import Flask

from auth import _register_security_headers
from cli import register_cli_commands
from config import DEFAULT_ADMIN_CODE, DEFAULT_SBC_DATA_DIR
from db import _register_db_hooks
from helpers import _register_context_processors, _register_template_filters
from routes import register_blueprints
from sbc.analyzer import SBCAnalyzer
from sbc.processor import SBCDataProcessor

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger("vivohub")


def create_app() -> Flask:
    """Cria e configura a aplicação Flask."""
    app = Flask(__name__, instance_relative_config=True)

    # ------------------------------------------------------------------
    # Configuração
    # ------------------------------------------------------------------
    app.config["SECRET_KEY"]   = os.environ.get("SECRET_KEY", "dev-secret-change-me")
    app.config["DATABASE"]     = os.path.join(app.instance_path, "vivohub.db")
    app.config["ADMIN_CODE"]   = os.environ.get("ADMIN_CODE", DEFAULT_ADMIN_CODE)
    app.config["SBC_DATA_DIR"] = os.environ.get("SBC_DATA_DIR", DEFAULT_SBC_DATA_DIR)

    # Diretórios obrigatórios
    Path(app.instance_path).mkdir(parents=True, exist_ok=True)
    export_dir = os.path.join(app.instance_path, "exports")
    Path(export_dir).mkdir(parents=True, exist_ok=True)
    app.config["EXPORT_DIR"] = export_dir

    # ------------------------------------------------------------------
    # SBC (processador + analisador)
    # ------------------------------------------------------------------
    sbc_processor = SBCDataProcessor(app.config["SBC_DATA_DIR"])
    app.config["SBC_ANALYZER"] = SBCAnalyzer(sbc_processor)

    # ------------------------------------------------------------------
    # Extensões e hooks
    # ------------------------------------------------------------------
    _register_db_hooks(app)
    _register_security_headers(app)
    _register_context_processors(app)
    _register_template_filters(app)

    # ------------------------------------------------------------------
    # Blueprints / rotas
    # ------------------------------------------------------------------
    register_blueprints(app)

    # ------------------------------------------------------------------
    # CLI
    # ------------------------------------------------------------------
    register_cli_commands(app)

    logger.info("VIVOHUB iniciado. Diretório SBC: %s", app.config["SBC_DATA_DIR"])
    return app
