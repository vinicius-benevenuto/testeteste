'''URL de Acesso ao IPAN: http://10.113.144.242/administration/users/
   LOGIN: 40418843 SENHA DE REDE: 230581Bs.@'''
 
# Objetivo Principal: Reservar Redes no IPAN
 
# Importação de Bibliotecas
import requests
import json
import pandas as pd
 
# URL Base da API + Endpoint de Autenticação
link_autenticacao = "http://10.113.144.242/api/ctp/user/"
 
'''O IPAN exige autenticação via usuário e senha para gerar o token de
acesso. Este token deve ser incluído no cabeçalho de todas as
requisições subsequentes para identificação e autorização. Devido ao seu
tempo de expiração, o processo de login deve ser repetido
periodicamente, conforme a necessidade de uso da API.'''
 
resposta_json = requests.post(link_autenticacao, auth=("40418843","230581Bs.@"))
 
resposta = resposta_json.json()
 
'''Após converter o JSON em um dicionário utilizando o método .json(), acesso o valor do token pela
chave e o armazeno na variável token_de_acesso.'''
token_de_acesso = resposta['data']['token']

cabecalho = {
    "token":token_de_acesso
}

mask_alvo = "28"
cn_usuario = "CN 01"

# 1. Mapeamento de qual POOL corresponde a qual máscara (conforme sua explicação)

if mask_alvo == "24":
    nome_pool_alvo = "POOL 1"
elif mask_alvo == "28":
    nome_pool_alvo = "POOL 3"
elif mask_alvo == "29":
    nome_pool_alvo = "POOL 4"
else:
    nome_pool_alvo = "POOL 5"

# 2. Busca o CN

url_subnets = "http://10.113.144.242/api/ctp/sections/186/subnets/"
res = requests.get(url_subnets, headers=cabecalho).json()
id_pasta_cn = next(item['id'] for item in res['data'] if item['description'] == cn_usuario)

print(id_pasta_cn)

# 3. Busca a pasta POOL específica para aquela máscara

url_pool = f"http://10.113.144.242/api/ctp/subnets/{id_pasta_cn}/slaves/"
res_pool = requests.get(url_pool, headers=cabecalho).json()

# Agora ele busca o POOL certo (ex: POOL 4 se a máscara for 29)

id_pasta_pool = next(
    (item['id'] for item in res_pool.get('data', []) 
     if nome_pool_alvo in item.get('description', '').upper()), 
    None
)

print(id_pasta_pool)

if not id_pasta_pool:
    print(f"Erro: Pasta {nome_pool_alvo} não encontrada para o CN {cn_usuario}.")
else:
    # 4. Entra na pasta POOL correta e busca a rede pai (master)
    url_masks = f"http://10.113.144.242/api/ctp/subnets/{id_pasta_cn}/slaves/{id_pasta_pool}"
    res_masks = requests.get(url_masks, headers=cabecalho).json()

    id_rede_pai = next(
        (item['id'] for item in res_masks.get('data', []) 
         if item.get('mask') == mask_alvo),
        None
    )

    if id_rede_pai:
        print(f"Sucesso! Entrou na {nome_pool_alvo} e achou a Rede Pai ID: {id_rede_pai}")
    else:
        print(f"Erro: Não achou rede /{mask_alvo} dentro da {nome_pool_alvo}.")

url_reserva = f"http://10.113.144.242/api/ctp/subnets/{id_pasta_pool}/first_subnet/{mask_alvo}/"


dados_nova_rede = {
    "description": f"982 - VIVO2"
}

resposta_reserva = requests.post(url_reserva, headers=cabecalho, json=dados_nova_rede)

if resposta_reserva.status_code in [200, 201]:
    res_final = resposta_reserva.json()
    nova_rede_ip = res_final.get('data') 
    print(f"Sucesso! Nova rede /{mask_para_criar} criada: {nova_rede_ip}")
else:
    print(f"Erro ao criar subnet: {resposta_reserva.status_code}")
    print(f"Mensagem: {resposta_reserva.text}")
