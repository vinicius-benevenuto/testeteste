"""
IPAM - Reserva Automática de Redes
===================================
Objetivo: Reservar redes no phpIPAM via API REST.
URL de Acesso: http://10.113.144.242/administration/users/
"""

import os
import logging
import requests
from requests.exceptions import RequestException

# ---------------------------------------------------------------------------
# Configuração de logging
# ---------------------------------------------------------------------------
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)

# ---------------------------------------------------------------------------
# Configurações gerais (use variáveis de ambiente em produção)
# ---------------------------------------------------------------------------
IPAM_USER = os.getenv("IPAM_USER", "40418843")
IPAM_PASS = os.getenv("IPAM_PASS", "230581Bs.@")

BASE_URL         = "http://10.113.144.242"
URL_AUTH         = f"{BASE_URL}/api/ctp/user/"
URL_SUBNETS_CN   = f"{BASE_URL}/api/ctp/sections/186/subnets/"

# Mapeamento máscara → nome do POOL
MASK_POOL_MAP: dict[str, str] = {
    "24": "POOL 1",
    "28": "POOL 3",
    "29": "POOL 4",
}
DEFAULT_POOL = "POOL 5"

# ---------------------------------------------------------------------------
# Cliente IPAM
# ---------------------------------------------------------------------------
class IPAMClient:
    """Encapsula autenticação e chamadas à API do phpIPAM."""

    def __init__(self, user: str, password: str) -> None:
        self.user     = user
        self.password = password
        self.token: str | None = None
        self.session  = requests.Session()

    # ------------------------------------------------------------------
    # Autenticação
    # ------------------------------------------------------------------
    def autenticar(self) -> None:
        """
        Realiza login na API e armazena o token de acesso.
        O token expira periodicamente; chame este método novamente
        quando necessário.
        """
        logger.info("Autenticando na API phpIPAM...")
        try:
            resposta = self.session.post(URL_AUTH, auth=(self.user, self.password))
            resposta.raise_for_status()
        except RequestException as exc:
            raise RuntimeError(f"Falha na autenticação: {exc}") from exc

        dados = resposta.json()
        self.token = dados["data"]["token"]
        self.session.headers.update({"token": self.token})
        logger.info("Autenticação bem-sucedida.")

    # ------------------------------------------------------------------
    # Requisições genéricas
    # ------------------------------------------------------------------
    def _get(self, url: str) -> dict:
        """GET com tratamento de erros centralizado."""
        try:
            resposta = self.session.get(url)
            resposta.raise_for_status()
        except RequestException as exc:
            raise RuntimeError(f"Erro na requisição GET [{url}]: {exc}") from exc
        return resposta.json()

    def _post(self, url: str, payload: dict) -> requests.Response:
        """POST com tratamento de erros centralizado."""
        try:
            resposta = self.session.post(url, json=payload)
        except RequestException as exc:
            raise RuntimeError(f"Erro na requisição POST [{url}]: {exc}") from exc
        return resposta

    # ------------------------------------------------------------------
    # Lógica de negócio
    # ------------------------------------------------------------------
    def buscar_id_cn(self, nome_cn: str) -> int:
        """Retorna o ID da pasta do CN dentro da seção 186."""
        logger.info("Buscando CN '%s'...", nome_cn)
        dados = self._get(URL_SUBNETS_CN)

        for item in dados.get("data", []):
            if item.get("description") == nome_cn:
                logger.info("CN '%s' encontrado → ID: %s", nome_cn, item["id"])
                return int(item["id"])

        raise ValueError(f"CN '{nome_cn}' não encontrado na seção 186.")

    def buscar_id_pool(self, id_cn: int, nome_pool: str) -> int:
        """Retorna o ID da pasta POOL dentro do CN informado."""
        logger.info("Buscando pool '%s' no CN ID %d...", nome_pool, id_cn)
        url  = f"{BASE_URL}/api/ctp/subnets/{id_cn}/slaves/"
        dados = self._get(url)

        for item in dados.get("data", []):
            descricao = item.get("description", "").upper()
            if nome_pool.upper() in descricao:
                logger.info("Pool '%s' encontrado → ID: %s", nome_pool, item["id"])
                return int(item["id"])

        raise ValueError(
            f"Pool '{nome_pool}' não encontrado dentro do CN ID {id_cn}."
        )

    def buscar_rede_pai(self, id_cn: int, id_pool: int, mascara: str) -> int | None:
        """
        Tenta localizar a rede pai com a máscara especificada.

        Retorna o ID se encontrada, ou None caso contrário.
        Essa etapa é apenas uma validação informativa — não bloqueia a reserva,
        pois o endpoint de reserva opera diretamente sobre o POOL.
        """
        logger.info(
            "Verificando rede pai com máscara /%s no pool ID %d...", mascara, id_pool
        )
        url   = f"{BASE_URL}/api/ctp/subnets/{id_cn}/slaves/{id_pool}"

        try:
            dados = self._get(url)
        except RuntimeError as exc:
            logger.warning("Não foi possível verificar rede pai: %s", exc)
            return None

        itens = dados.get("data", [])
        logger.debug("Itens retornados pelo endpoint de rede pai: %s", itens)

        for item in itens:
            if item.get("mask") == mascara:
                logger.info("Rede pai (/%s) encontrada → ID: %s", mascara, item["id"])
                return int(item["id"])

        # Aviso apenas — a reserva ainda será tentada
        logger.warning(
            "Rede pai com máscara /%s não localizada no pool ID %d. "
            "Prosseguindo com a reserva mesmo assim.",
            mascara, id_pool,
        )
        return None

    def reservar_rede(
        self,
        id_pool: int,
        mascara: str,
        descricao: str,
    ) -> dict:
        """
        Reserva a primeira sub-rede disponível com a máscara informada
        dentro do pool especificado.

        Retorna os dados da rede criada.
        """
        url     = f"{BASE_URL}/api/ctp/subnets/{id_pool}/first_subnet/{mascara}/"
        payload = {"description": descricao}

        logger.info(
            "Reservando rede /%s no pool ID %d com descrição '%s'...",
            mascara, id_pool, descricao,
        )
        resposta = self._post(url, payload)

        if resposta.status_code in (200, 201):
            dados = resposta.json()
            nova_rede = dados.get("data")
            logger.info("Rede /%s reservada com sucesso: %s", mascara, nova_rede)
            return nova_rede

        raise RuntimeError(
            f"Falha ao reservar subnet (HTTP {resposta.status_code}): {resposta.text}"
        )


# ---------------------------------------------------------------------------
# Ponto de entrada
# ---------------------------------------------------------------------------
def main() -> None:
    # --- Parâmetros da reserva -------------------------------------------
    mask_alvo  = "28"
    cn_usuario = "CN 01"
    descricao_rede = "982 - VIVO2"
    # -----------------------------------------------------------------------

    nome_pool = MASK_POOL_MAP.get(mask_alvo, DEFAULT_POOL)
    logger.info(
        "Iniciando reserva | CN: %s | Máscara: /%s | Pool: %s",
        cn_usuario, mask_alvo, nome_pool,
    )

    cliente = IPAMClient(IPAM_USER, IPAM_PASS)

    try:
        # 1. Autenticação
        cliente.autenticar()

        # 2. Localiza o CN
        id_cn = cliente.buscar_id_cn(cn_usuario)

        # 3. Localiza o POOL correto para a máscara
        id_pool = cliente.buscar_id_pool(id_cn, nome_pool)

        # 4. Confirma existência da rede pai (validação)
        cliente.buscar_rede_pai(id_cn, id_pool, mask_alvo)

        # 5. Efetua a reserva
        nova_rede = cliente.reservar_rede(id_pool, mask_alvo, descricao_rede)

        print("\n✅ Reserva concluída com sucesso!")
        print(f"   Rede criada: {nova_rede}")

    except (ValueError, RuntimeError) as exc:
        logger.error("❌ %s", exc)


if __name__ == "__main__":
    main()

