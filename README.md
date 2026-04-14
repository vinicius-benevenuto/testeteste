"""routes/api.py — API REST: SBC, CNs e SIP Router SP."""
import logging
from dataclasses import asdict

from flask import Blueprint, current_app, jsonify, request

from auth import login_required, role_required
from db import get_db
from sbc.analyzer import SBCAnalyzer

logger = logging.getLogger(__name__)
bp = Blueprint("api", __name__, url_prefix="/api")


# ---------------------------------------------------------------------------
# SBC
# ---------------------------------------------------------------------------
@bp.get("/sbc/suggest")
@login_required
@role_required("engenharia")
def sbc_suggest():
    cn  = request.args.get("cn", "").strip()
    uf  = request.args.get("uf", "").strip().upper()
    analyzer: SBCAnalyzer = current_app.config["SBC_ANALYZER"]

    if cn:
        result = analyzer.suggest_for_cn(cn)
    elif uf:
        result = analyzer.suggest_for_uf(uf)
    else:
        return jsonify({"error": "Parâmetro 'cn' ou 'uf' é obrigatório."}), 400

    return jsonify(asdict(result))


@bp.get("/sbc/overview")
@login_required
@role_required("engenharia")
def sbc_overview():
    analyzer: SBCAnalyzer = current_app.config["SBC_ANALYZER"]
    return jsonify(analyzer.get_overview())


@bp.get("/sbc/health")
@login_required
@role_required("engenharia")
def sbc_health():
    analyzer: SBCAnalyzer = current_app.config["SBC_ANALYZER"]
    return jsonify(analyzer.health_check())


@bp.get("/sbc/reload")
@login_required
@role_required("engenharia")
def sbc_reload():
    analyzer: SBCAnalyzer = current_app.config["SBC_ANALYZER"]
    measurements, aggregated = analyzer.processor.get_data(force_reload=True)
    return jsonify({
        "status":          "reloaded",
        "measurements":    len(measurements),
        "aggregated_rows": len(aggregated),
        "file_info":       analyzer.processor.get_file_info(),
    })


# ---------------------------------------------------------------------------
# CNs
# ---------------------------------------------------------------------------
@bp.get("/cns")
@login_required
def api_cns():
    db   = get_db()
    rows = db.execute(
        "SELECT codigo, COALESCE(nome,'') AS nome, COALESCE(uf,'') AS uf "
        "FROM cns WHERE ativo=1 ORDER BY codigo ASC"
    ).fetchall()
    return jsonify([
        {"codigo": r["codigo"], "nome": r["nome"], "uf": r["uf"]}
        for r in rows
    ])


# ---------------------------------------------------------------------------
# SIPROUTER SP
# ---------------------------------------------------------------------------
@bp.get("/siprouter/query")
@login_required
def siprouter_query():
    """
    Consulta a tabela siprouter_sp.

    Parâmetros obrigatórios:
        cn  — Código Nacional (inteiro, ex.: 11)
        rn1 — RN1 da operadora (ex.: 55131)

    Parâmetros opcionais:
        scm — 1 para filtrar por SCM, 0 (padrão) para ignorar
        av  — 1 para filtrar por AV,  0 (padrão) para ignorar

    Retorno JSON:
        { found, cn, rn1_consultado, resultados, dados_vivo, mensagem }
    """
    from siprouter_sp import query_siprouter_sp

    cn_raw  = request.args.get("cn",  "").strip()
    rn1     = request.args.get("rn1", "").strip()
    scm     = request.args.get("scm", "0").strip() == "1"
    av      = request.args.get("av",  "0").strip() == "1"

    if not cn_raw or not rn1:
        return jsonify({
            "error": "Parâmetros 'cn' e 'rn1' são obrigatórios."
        }), 400

    try:
        cn = int(cn_raw)
    except ValueError:
        return jsonify({"error": f"'cn' deve ser um número inteiro. Recebido: {cn_raw!r}"}), 400

    db     = get_db()
    result = query_siprouter_sp(db, cn=cn, rn1=rn1, scm=scm, av=av)
    return jsonify(result)
