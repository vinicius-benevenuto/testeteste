"""
config.py — Todas as constantes e configurações da aplicação VIVOHUB.
Nenhuma importação de Flask aqui; este módulo é importado por todos os outros.
"""
import os
from typing import Final

# =============================================================================
# APLICAÇÃO
# =============================================================================
MAX_TABLE_ROWS: Final[int] = 10
DEFAULT_ADMIN_CODE: Final[str] = "v28B112004"
DEFAULT_DIAGRAM_IMAGE: Final[str] = (
    r"C:\Users\40418843\Desktop\VIVOHUB\templates\20251007_140942_0000.png"
)
DEFAULT_SBC_DATA_DIR: Final[str] = r"C:/Users/40418843/Desktop/dados-sbcs"

# =============================================================================
# CAMPOS DO FORMULÁRIO
# =============================================================================
BOOLEAN_FIELDS: Final[tuple] = (
    "csp", "servicos_especiais", "cng",
    "scm", "av",
    "sbc_ativo", "ip_reservado", "vivo_reserva",
    "operadora_ciente", "lcr_nacional", "white_list",
    "prefixos_liberados_abr", "premissas_ok",
)

TEXT_FIELDS: Final[tuple] = (
    "nome_operadora", "rn1", "atendimento", "redes", "qual", "tmr",
    "responsavel_operadora", "responsavel_vivo", "asn",
    "responsavel_infra", "aprovado_por",
    "status", "escopo_text",
    "responsavel_atacado", "responsavel_engenharia",
)

JSON_FIELDS: Final[tuple] = (
    "escopo_flags_json", "dados_vivo_json",
    "dados_operadora_json", "engenharia_params_json",
)

ALLOWED_SCOPE_FLAGS: Final[frozenset] = frozenset({
    "LC", "LD15 + CNG", "LDS/CSP + CNG", "Transporte", "VC1", "Concentração"
})

# =============================================================================
# SBC
# =============================================================================
SBC_CACHE_TTL_SECONDS: Final[int] = 300

SBC_STATUS_TO_HEALTH: Final[dict[str, str]] = {
    "normal":   "disponivel",
    "atenção":  "moderado",
    "atencao":  "moderado",
    "crítico":  "critico",
    "critico":  "critico",
}

SBC_STATUS_BASE_SCORE: Final[dict[str, int]] = {
    "disponivel": 70,
    "moderado":   45,
    "critico":    15,
}

# =============================================================================
# IPAM
# =============================================================================
IPAM_BASE_URL: str = os.getenv("IPAM_BASE_URL", "http://10.113.144.242")
IPAM_USER:     str = os.getenv("IPAM_USER", "40418843")
IPAM_PASS:     str = os.getenv("IPAM_PASS", "230581Bs.@@")   # produção: só env var
IPAM_SECTION_ID: Final[int] = 186

IPAM_MASK_POOL_MAP: Final[dict[str, str]] = {
    "24": "POOL 1",
    "28": "POOL 3",
    "29": "POOL 4",
}
IPAM_DEFAULT_POOL: Final[str] = "POOL 5"

# =============================================================================
# CNs — SEED E METADATA
# =============================================================================
_CN_SEED_RAW: Final[str] = """
68 82 97 92 96 77 75 74 73 71 88 85 61 27 28 64 62 61 98 99
34 37 31 35 32 38 33 67 66 65 91 94 93 83 81 87 86 89
43 44 45 46 41 24 22 21 84 69 95 51 53 54 55 47 48 49 79
18 14 15 16 13 19 17 11 12 63
"""

CN_METADATA: Final[dict[str, tuple[str, str]]] = {
    "11": ("São Paulo", "SP"),          "12": ("São José dos Campos", "SP"),
    "13": ("Santos", "SP"),             "14": ("Bauru", "SP"),
    "15": ("Sorocaba", "SP"),           "16": ("Ribeirão Preto", "SP"),
    "17": ("São José do Rio Preto", "SP"), "18": ("Presidente Prudente", "SP"),
    "19": ("Campinas", "SP"),
    "21": ("Rio de Janeiro", "RJ"),     "22": ("Campos dos Goytacazes", "RJ"),
    "24": ("Volta Redonda", "RJ"),      "27": ("Vitória", "ES"),
    "28": ("Cachoeiro de Itapemirim", "ES"),
    "31": ("Belo Horizonte", "MG"),     "32": ("Juiz de Fora", "MG"),
    "33": ("Governador Valadares", "MG"), "34": ("Uberlândia", "MG"),
    "35": ("Poços de Caldas", "MG"),    "37": ("Divinópolis", "MG"),
    "38": ("Montes Claros", "MG"),
    "41": ("Curitiba", "PR"),           "42": ("Ponta Grossa", "PR"),
    "43": ("Londrina", "PR"),           "44": ("Maringá", "PR"),
    "45": ("Foz do Iguaçu", "PR"),      "46": ("Francisco Beltrão", "PR"),
    "47": ("Joinville", "SC"),          "48": ("Florianópolis", "SC"),
    "49": ("Chapecó", "SC"),
    "51": ("Porto Alegre", "RS"),       "53": ("Pelotas", "RS"),
    "54": ("Caxias do Sul", "RS"),      "55": ("Santa Maria", "RS"),
    "61": ("Brasília", "DF"),           "62": ("Goiânia", "GO"),
    "63": ("Palmas", "TO"),             "64": ("Rio Verde", "GO"),
    "65": ("Cuiabá", "MT"),             "66": ("Rondonópolis", "MT"),
    "67": ("Campo Grande", "MS"),
    "71": ("Salvador", "BA"),           "73": ("Ilhéus", "BA"),
    "74": ("Juazeiro", "BA"),           "75": ("Feira de Santana", "BA"),
    "77": ("Vitória da Conquista", "BA"),
    "79": ("Aracaju", "SE"),            "81": ("Recife", "PE"),
    "82": ("Maceió", "AL"),             "83": ("João Pessoa", "PB"),
    "84": ("Natal", "RN"),              "85": ("Fortaleza", "CE"),
    "86": ("Teresina", "PI"),           "87": ("Petrolina", "PE"),
    "88": ("Juazeiro do Norte", "CE"),  "89": ("Picos", "PI"),
    "91": ("Belém", "PA"),              "92": ("Manaus", "AM"),
    "93": ("Santarém", "PA"),           "94": ("Marabá", "PA"),
    "95": ("Boa Vista", "RR"),          "96": ("Macapá", "AP"),
    "97": ("Coari", "AM"),
    "98": ("São Luís", "MA"),           "99": ("Imperatriz", "MA"),
    "68": ("Rio Branco", "AC"),         "69": ("Porto Velho", "RO"),
}

# =============================================================================
# UF → REGIONAL E VIZINHOS
# =============================================================================
UF_TO_REGIONAL: Final[dict[str, str]] = {
    "AC": "NORTE",  "AM": "NORTE",  "AP": "NORTE",  "PA": "NORTE",
    "RO": "NORTE",  "RR": "NORTE",  "TO": "NORTE",
    "AL": "NORDESTE", "BA": "NORDESTE", "CE": "NORDESTE",
    "MA": "NORDESTE", "PB": "NORDESTE", "PE": "NORDESTE",
    "PI": "NORDESTE", "RN": "NORDESTE", "SE": "NORDESTE",
    "DF": "CENTRO-OESTE", "GO": "CENTRO-OESTE",
    "MS": "CENTRO-OESTE", "MT": "CENTRO-OESTE",
    "ES": "SUDESTE", "MG": "SUDESTE", "RJ": "SUDESTE", "SP": "SUDESTE",
    "PR": "SUL",    "RS": "SUL",    "SC": "SUL",
}

UF_NEIGHBORS: Final[dict[str, list[str]]] = {
    "AC": ["RO", "AM"],
    "AL": ["PE", "SE", "BA"],
    "AM": ["PA", "RR", "AC", "RO", "MT"],
    "AP": ["PA"],
    "BA": ["SE", "AL", "PE", "PI", "MG", "GO", "TO", "MA"],
    "CE": ["RN", "PB", "PE", "PI"],
    "DF": ["GO", "MG"],
    "ES": ["MG", "RJ", "BA"],
    "GO": ["DF", "MG", "MS", "MT", "TO", "BA"],
    "MA": ["PI", "TO", "PA"],
    "MG": ["SP", "RJ", "ES", "BA", "GO", "DF", "MS"],
    "MS": ["PR", "SP", "MG", "GO", "MT"],
    "MT": ["MS", "GO", "TO", "PA", "AM", "RO"],
    "PA": ["MA", "TO", "MT", "AM", "AP", "RR"],
    "PB": ["PE", "RN", "CE"],
    "PE": ["PB", "AL", "BA", "CE", "PI"],
    "PI": ["MA", "CE", "PE", "BA", "TO"],
    "PR": ["SP", "SC", "MS"],
    "RJ": ["SP", "MG", "ES"],
    "RN": ["PB", "CE"],
    "RO": ["MT", "AM", "AC"],
    "RR": ["AM", "PA"],
    "RS": ["SC"],
    "SC": ["PR", "RS"],
    "SE": ["AL", "BA"],
    "SP": ["RJ", "MG", "PR", "MS"],
    "TO": ["MA", "PI", "BA", "GO", "MT", "PA"],
}

