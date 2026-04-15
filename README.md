"""
siprouter_sp.py — Base de dados do siprouter de São Paulo.

Responsabilidades:
  - Criar a tabela siprouter_sp no banco SQLite da aplicação.
  - Inserir os 180 registros oficiais SE a tabela estiver vazia.
  - Expor query_siprouter_sp() como única interface de consulta.

Restrições absolutas:
  - NÃO sugere SBCs.
  - NÃO utiliza localidade para busca.
  - NÃO consulta fontes externas.
  - NÃO infere dados.
  - NÃO altera registros após a carga inicial.
"""
import sqlite3
import logging
from typing import Optional

logger = logging.getLogger(__name__)

# =============================================================================
# BASE OFICIAL — 180 registros exatos (CDSIP_SPO_PL: 90 + CDSIP_SPO_JG: 90)
# (elemento, bloco_ip, vlan, vrf, descricao, cn, rn1)
# =============================================================================
_SEED_DATA: tuple[tuple, ...] = (
    # ── CDSIP_SPO_PL — CN 11 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.11.130.0/28',   402, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 ESPELHINHOS', 11, 'Diversos'),
    ('CDSIP_SPO_PL', '10.11.130.16/28',  403, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 OI',         11, '55131/55114'),
    ('CDSIP_SPO_PL', '10.11.130.32/28',  404, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 TIM',        11, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.11.130.48/28',  405, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 CLARO',      11, '55121/55321'),
    ('CDSIP_SPO_PL', '10.11.130.64/28',  406, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 ALGAR',      11, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.11.130.80/28',  407, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 DATORA',     11, '55181'),
    ('CDSIP_SPO_PL', '10.11.130.96/28',  408, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 AV',         11, 'Diversos'),
    ('CDSIP_SPO_PL', '10.11.130.112/28', 409, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 SCM',        11, 'Diversos'),
    ('CDSIP_SPO_PL', '10.11.130.128/28', 410, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 RESERVA',    11, 'Diversos'),
    ('CDSIP_SPO_PL', '10.11.130.144/28', 411, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 RESERVA',    11, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 12 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.12.130.0/28',   412, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 ESPELHINHOS', 12, 'Diversos'),
    ('CDSIP_SPO_PL', '10.12.130.16/28',  413, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 OI',          12, '55131/55114'),
    ('CDSIP_SPO_PL', '10.12.130.32/28',  414, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 TIM',         12, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.12.130.48/28',  415, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 CLARO',       12, '55121/55321'),
    ('CDSIP_SPO_PL', '10.12.130.64/28',  416, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 ALGAR',       12, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.12.130.80/28',  417, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 DATORA',      12, '55181'),
    ('CDSIP_SPO_PL', '10.12.130.96/28',  418, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 AV',          12, 'Diversos'),
    ('CDSIP_SPO_PL', '10.12.130.112/28', 419, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 SCM',         12, 'Diversos'),
    ('CDSIP_SPO_PL', '10.12.130.128/28', 420, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 RESERVA',     12, 'Diversos'),
    ('CDSIP_SPO_PL', '10.12.130.144/28', 421, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 RESERVA',     12, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 13 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.13.130.0/28',   422, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 ESPELHINHOS', 13, 'Diversos'),
    ('CDSIP_SPO_PL', '10.13.130.16/28',  423, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 OI',          13, '55131/55114'),
    ('CDSIP_SPO_PL', '10.13.130.32/28',  424, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 TIM',         13, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.13.130.48/28',  425, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 CLARO',       13, '55121/55321'),
    ('CDSIP_SPO_PL', '10.13.130.64/28',  426, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 ALGAR',       13, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.13.130.80/28',  427, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 DATORA',      13, '55181'),
    ('CDSIP_SPO_PL', '10.13.130.96/28',  428, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 AV',          13, 'Diversos'),
    ('CDSIP_SPO_PL', '10.13.130.112/28', 429, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 SCM',         13, 'Diversos'),
    ('CDSIP_SPO_PL', '10.13.130.128/28', 430, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 RESERVA',     13, 'Diversos'),
    ('CDSIP_SPO_PL', '10.13.130.144/28', 431, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 RESERVA',     13, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 14 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.14.130.0/28',   432, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 ESPELHINHOS', 14, 'Diversos'),
    ('CDSIP_SPO_PL', '10.14.130.16/28',  433, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 OI',          14, '55131/55114'),
    ('CDSIP_SPO_PL', '10.14.130.32/28',  434, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 TIM',         14, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.14.130.48/28',  435, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 CLARO',       14, '55121/55321'),
    ('CDSIP_SPO_PL', '10.14.130.64/28',  436, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 ALGAR',       14, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.14.130.80/28',  437, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 DATORA',      14, '55181'),
    ('CDSIP_SPO_PL', '10.14.130.96/28',  438, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 AV',          14, 'Diversos'),
    ('CDSIP_SPO_PL', '10.14.130.112/28', 439, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 SCM',         14, 'Diversos'),
    ('CDSIP_SPO_PL', '10.14.130.128/28', 440, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 RESERVA',     14, 'Diversos'),
    ('CDSIP_SPO_PL', '10.14.130.144/28', 441, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 RESERVA',     14, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 15 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.15.130.0/28',   442, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 ESPELHINHOS', 15, 'Diversos'),
    ('CDSIP_SPO_PL', '10.15.130.16/28',  443, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 OI',          15, '55131/55114'),
    ('CDSIP_SPO_PL', '10.15.130.32/28',  444, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 TIM',         15, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.15.130.48/28',  445, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 CLARO',       15, '55121/55321'),
    ('CDSIP_SPO_PL', '10.15.130.64/28',  446, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 ALGAR',       15, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.15.130.80/28',  447, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 DATORA',      15, '55181'),
    ('CDSIP_SPO_PL', '10.15.130.96/28',  448, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 AV',          15, 'Diversos'),
    ('CDSIP_SPO_PL', '10.15.130.112/28', 449, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 SCM',         15, 'Diversos'),
    ('CDSIP_SPO_PL', '10.15.130.128/28', 450, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 RESERVA',     15, 'Diversos'),
    ('CDSIP_SPO_PL', '10.15.130.144/28', 451, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 RESERVA',     15, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 16 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.16.130.0/28',   452, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 16 ESPELHINHOS', 16, 'Diversos'),
    ('CDSIP_SPO_PL', '10.16.130.16/28',  453, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 OI',          16, '55131/55114'),
    ('CDSIP_SPO_PL', '10.16.130.32/28',  454, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 TIM',         16, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.16.130.48/28',  455, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 CLARO',       16, '55121/55321'),
    ('CDSIP_SPO_PL', '10.16.130.64/28',  456, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 ALGAR',       16, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.16.130.80/28',  457, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 DATORA',      16, '55181'),
    ('CDSIP_SPO_PL', '10.16.130.96/28',  458, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 AV',          16, 'Diversos'),
    ('CDSIP_SPO_PL', '10.16.130.112/28', 459, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 SCM',         16, 'Diversos'),
    ('CDSIP_SPO_PL', '10.16.130.128/28', 460, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 RESERVA',     16, 'Diversos'),
    ('CDSIP_SPO_PL', '10.16.130.144/28', 461, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 RESERVA',     16, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 17 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.17.130.0/28',   462, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 ESPELHINHOS', 17, 'Diversos'),
    ('CDSIP_SPO_PL', '10.17.130.16/28',  463, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 OI',          17, '55131/55114'),
    ('CDSIP_SPO_PL', '10.17.130.32/28',  464, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 TIM',         17, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.17.130.48/28',  465, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 CLARO',       17, '55121/55321'),
    ('CDSIP_SPO_PL', '10.17.130.64/28',  466, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 ALGAR',       17, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.17.130.80/28',  467, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 DATORA',      17, '55181'),
    ('CDSIP_SPO_PL', '10.17.130.96/28',  468, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 AV',          17, 'Diversos'),
    ('CDSIP_SPO_PL', '10.17.130.112/28', 469, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 SCM',         17, 'Diversos'),
    ('CDSIP_SPO_PL', '10.17.130.128/28', 470, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 RESERVA',     17, 'Diversos'),
    ('CDSIP_SPO_PL', '10.17.130.144/28', 471, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 RESERVA',     17, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 18 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.18.130.0/28',   472, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 ESPELHINHOS', 18, 'Diversos'),
    ('CDSIP_SPO_PL', '10.18.130.16/28',  473, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 OI',          18, '55131/55114'),
    ('CDSIP_SPO_PL', '10.18.130.32/28',  474, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 TIM',         18, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.18.130.48/28',  475, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 CLARO',       18, '55121/55321'),
    ('CDSIP_SPO_PL', '10.18.130.64/28',  476, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 ALGAR',       18, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.18.130.80/28',  477, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 DATORA',      18, '55181'),
    ('CDSIP_SPO_PL', '10.18.130.96/28',  478, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 AV',          18, 'Diversos'),
    ('CDSIP_SPO_PL', '10.18.130.112/28', 479, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 SCM',         18, 'Diversos'),
    ('CDSIP_SPO_PL', '10.18.130.128/28', 480, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 RESERVA',     18, 'Diversos'),
    ('CDSIP_SPO_PL', '10.18.130.144/28', 481, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 RESERVA',     18, 'Diversos'),
    # ── CDSIP_SPO_PL — CN 19 ──────────────────────────────────────────────
    ('CDSIP_SPO_PL', '10.19.130.0/28',   482, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 ESPELHINHOS', 19, 'Diversos'),
    ('CDSIP_SPO_PL', '10.19.130.16/28',  483, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 OI',          19, '55131/55114'),
    ('CDSIP_SPO_PL', '10.19.130.32/28',  484, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 TIM',         19, '55123/55141/55341'),
    ('CDSIP_SPO_PL', '10.19.130.48/28',  485, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 CLARO',       19, '55121/55321'),
    ('CDSIP_SPO_PL', '10.19.130.64/28',  486, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 ALGAR',       19, '55112/55224/55312'),
    ('CDSIP_SPO_PL', '10.19.130.80/28',  487, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 DATORA',      19, '55181'),
    ('CDSIP_SPO_PL', '10.19.130.96/28',  488, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 AV',          19, 'Diversos'),
    ('CDSIP_SPO_PL', '10.19.130.112/28', 489, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 SCM',         19, 'Diversos'),
    ('CDSIP_SPO_PL', '10.19.130.128/28', 490, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 RESERVA',     19, 'Diversos'),
    ('CDSIP_SPO_PL', '10.19.130.144/28', 491, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 RESERVA',     19, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 11 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.11.131.0/28',   402, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 ESPELHINHOS', 11, 'Diversos'),
    ('CDSIP_SPO_JG', '10.11.131.16/28',  403, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 OI',          11, '55131/55114'),
    ('CDSIP_SPO_JG', '10.11.131.32/28',  404, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 TIM',         11, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.11.131.48/28',  405, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 CLARO',       11, '55121/55321'),
    ('CDSIP_SPO_JG', '10.11.131.64/28',  406, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 ALGAR',       11, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.11.131.80/28',  407, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 DATORA',      11, '55181'),
    ('CDSIP_SPO_JG', '10.11.131.96/28',  408, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 AV',          11, 'Diversos'),
    ('CDSIP_SPO_JG', '10.11.131.112/28', 409, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 SCM',         11, 'Diversos'),
    ('CDSIP_SPO_JG', '10.11.131.128/28', 410, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 RESERVA',     11, 'Diversos'),
    ('CDSIP_SPO_JG', '10.11.131.144/28', 411, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 11 RESERVA',     11, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 12 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.12.131.0/28',   412, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 ESPELHINHOS', 12, 'Diversos'),
    ('CDSIP_SPO_JG', '10.12.131.16/28',  413, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 OI',          12, '55131/55114'),
    ('CDSIP_SPO_JG', '10.12.131.32/28',  414, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 TIM',         12, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.12.131.48/28',  415, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 CLARO',       12, '55121/55321'),
    ('CDSIP_SPO_JG', '10.12.131.64/28',  416, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 ALGAR',       12, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.12.131.80/28',  417, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 DATORA',      12, '55181'),
    ('CDSIP_SPO_JG', '10.12.131.96/28',  418, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 AV',          12, 'Diversos'),
    ('CDSIP_SPO_JG', '10.12.131.112/28', 419, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 SCM',         12, 'Diversos'),
    ('CDSIP_SPO_JG', '10.12.131.128/28', 420, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 RESERVA',     12, 'Diversos'),
    ('CDSIP_SPO_JG', '10.12.131.144/28', 421, 'V75055:Interconexao_SIP_12_New_H', 'ROTAS CN 12 RESERVA',     12, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 13 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.13.131.0/28',   422, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 ESPELHINHOS', 13, 'Diversos'),
    ('CDSIP_SPO_JG', '10.13.131.16/28',  423, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 OI',          13, '55131/55114'),
    ('CDSIP_SPO_JG', '10.13.131.32/28',  424, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 TIM',         13, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.13.131.48/28',  425, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 CLARO',       13, '55121/55321'),
    ('CDSIP_SPO_JG', '10.13.131.64/28',  426, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 ALGAR',       13, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.13.131.80/28',  427, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 DATORA',      13, '55181'),
    ('CDSIP_SPO_JG', '10.13.131.96/28',  428, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 AV',          13, 'Diversos'),
    ('CDSIP_SPO_JG', '10.13.131.112/28', 429, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 SCM',         13, 'Diversos'),
    ('CDSIP_SPO_JG', '10.13.131.128/28', 430, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 RESERVA',     13, 'Diversos'),
    ('CDSIP_SPO_JG', '10.13.131.144/28', 431, 'V75057:Interconexao_SIP_13_New_H', 'ROTAS CN 13 RESERVA',     13, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 14 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.14.131.0/28',   432, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 ESPELHINHOS', 14, 'Diversos'),
    ('CDSIP_SPO_JG', '10.14.131.16/28',  433, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 OI',          14, '55131/55114'),
    ('CDSIP_SPO_JG', '10.14.131.32/28',  434, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 TIM',         14, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.14.131.48/28',  435, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 CLARO',       14, '55121/55321'),
    ('CDSIP_SPO_JG', '10.14.131.64/28',  436, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 ALGAR',       14, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.14.131.80/28',  437, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 DATORA',      14, '55181'),
    ('CDSIP_SPO_JG', '10.14.131.96/28',  438, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 AV',          14, 'Diversos'),
    ('CDSIP_SPO_JG', '10.14.131.112/28', 439, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 SCM',         14, 'Diversos'),
    ('CDSIP_SPO_JG', '10.14.131.128/28', 440, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 RESERVA',     14, 'Diversos'),
    ('CDSIP_SPO_JG', '10.14.131.144/28', 441, 'V80898:Interconexao_SIP_14_New_H', 'ROTAS CN 14 RESERVA',     14, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 15 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.15.131.0/28',   442, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 ESPELHINHOS', 15, 'Diversos'),
    ('CDSIP_SPO_JG', '10.15.131.16/28',  443, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 OI',          15, '55131/55114'),
    ('CDSIP_SPO_JG', '10.15.131.32/28',  444, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 TIM',         15, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.15.131.48/28',  445, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 CLARO',       15, '55121/55321'),
    ('CDSIP_SPO_JG', '10.15.131.64/28',  446, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 ALGAR',       15, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.15.131.80/28',  447, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 DATORA',      15, '55181'),
    ('CDSIP_SPO_JG', '10.15.131.96/28',  448, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 AV',          15, 'Diversos'),
    ('CDSIP_SPO_JG', '10.15.131.112/28', 449, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 SCM',         15, 'Diversos'),
    ('CDSIP_SPO_JG', '10.15.131.128/28', 450, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 RESERVA',     15, 'Diversos'),
    ('CDSIP_SPO_JG', '10.15.131.144/28', 451, 'V79865:Interconexao_SIP_15_New_H', 'ROTAS CN 15 RESERVA',     15, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 16 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.16.131.0/28',   452, 'V75054:Interconexao_SIP_11_New_H', 'ROTAS CN 16 ESPELHINHOS', 16, 'Diversos'),
    ('CDSIP_SPO_JG', '10.16.131.16/28',  453, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 OI',          16, '55131/55114'),
    ('CDSIP_SPO_JG', '10.16.131.32/28',  454, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 TIM',         16, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.16.131.48/28',  455, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 CLARO',       16, '55121/55321'),
    ('CDSIP_SPO_JG', '10.16.131.64/28',  456, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 ALGAR',       16, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.16.131.80/28',  457, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 DATORA',      16, '55181'),
    ('CDSIP_SPO_JG', '10.16.131.96/28',  458, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 AV',          16, 'Diversos'),
    ('CDSIP_SPO_JG', '10.16.131.112/28', 459, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 SCM',         16, 'Diversos'),
    ('CDSIP_SPO_JG', '10.16.131.128/28', 460, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 RESERVA',     16, 'Diversos'),
    ('CDSIP_SPO_JG', '10.16.131.144/28', 461, 'V79216:Interconexao_SIP_16_New_H', 'ROTAS CN 16 RESERVA',     16, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 17 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.17.131.0/28',   462, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 ESPELHINHOS', 17, 'Diversos'),
    ('CDSIP_SPO_JG', '10.17.131.16/28',  463, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 OI',          17, '55131/55114'),
    ('CDSIP_SPO_JG', '10.17.131.32/28',  464, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 TIM',         17, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.17.131.48/28',  465, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 CLARO',       17, '55121/55321'),
    ('CDSIP_SPO_JG', '10.17.131.64/28',  466, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 ALGAR',       17, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.17.131.80/28',  467, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 DATORA',      17, '55181'),
    ('CDSIP_SPO_JG', '10.17.131.96/28',  468, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 AV',          17, 'Diversos'),
    ('CDSIP_SPO_JG', '10.17.131.112/28', 469, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 SCM',         17, 'Diversos'),
    ('CDSIP_SPO_JG', '10.17.131.128/28', 470, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 RESERVA',     17, 'Diversos'),
    ('CDSIP_SPO_JG', '10.17.131.144/28', 471, 'V79886:Interconexao_SIP_17_New_H', 'ROTAS CN 17 RESERVA',     17, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 18 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.18.131.0/28',   472, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 ESPELHINHOS', 18, 'Diversos'),
    ('CDSIP_SPO_JG', '10.18.131.16/28',  473, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 OI',          18, '55131/55114'),
    ('CDSIP_SPO_JG', '10.18.131.32/28',  474, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 TIM',         18, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.18.131.48/28',  475, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 CLARO',       18, '55121/55321'),
    ('CDSIP_SPO_JG', '10.18.131.64/28',  476, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 ALGAR',       18, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.18.131.80/28',  477, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 DATORA',      18, '55181'),
    ('CDSIP_SPO_JG', '10.18.131.96/28',  478, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 AV',          18, 'Diversos'),
    ('CDSIP_SPO_JG', '10.18.131.112/28', 479, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 SCM',         18, 'Diversos'),
    ('CDSIP_SPO_JG', '10.18.131.128/28', 480, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 RESERVA',     18, 'Diversos'),
    ('CDSIP_SPO_JG', '10.18.131.144/28', 481, 'V79894:Interconexao_SIP_18_New_H', 'ROTAS CN 18 RESERVA',     18, 'Diversos'),
    # ── CDSIP_SPO_JG — CN 19 ──────────────────────────────────────────────
    ('CDSIP_SPO_JG', '10.19.131.0/28',   482, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 ESPELHINHOS', 19, 'Diversos'),
    ('CDSIP_SPO_JG', '10.19.131.16/28',  483, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 OI',          19, '55131/55114'),
    ('CDSIP_SPO_JG', '10.19.131.32/28',  484, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 TIM',         19, '55123/55141/55341'),
    ('CDSIP_SPO_JG', '10.19.131.48/28',  485, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 CLARO',       19, '55121/55321'),
    ('CDSIP_SPO_JG', '10.19.131.64/28',  486, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 ALGAR',       19, '55112/55224/55312'),
    ('CDSIP_SPO_JG', '10.19.131.80/28',  487, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 DATORA',      19, '55181'),
    ('CDSIP_SPO_JG', '10.19.131.96/28',  488, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 AV',          19, 'Diversos'),
    ('CDSIP_SPO_JG', '10.19.131.112/28', 489, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 SCM',         19, 'Diversos'),
    ('CDSIP_SPO_JG', '10.19.131.128/28', 490, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 RESERVA',     19, 'Diversos'),
    ('CDSIP_SPO_JG', '10.19.131.144/28', 491, 'V79903:Interconexao_SIP_19_New_H', 'ROTAS CN 19 RESERVA',     19, 'Diversos'),
)


# =============================================================================
# INICIALIZAÇÃO — cria tabela e carrega seed se vazia
# =============================================================================
def init_siprouter_sp(db: sqlite3.Connection) -> None:
    """
    Garante que a tabela siprouter_sp existe e está preenchida.
    Chamado automaticamente por _init_db() a cada request.
    """
    db.execute("""
        CREATE TABLE IF NOT EXISTS siprouter_sp (
            id        INTEGER PRIMARY KEY AUTOINCREMENT,
            elemento  TEXT    NOT NULL,
            bloco_ip  TEXT    NOT NULL,
            vlan      INTEGER NOT NULL,
            vrf       TEXT    NOT NULL,
            descricao TEXT    NOT NULL,
            cn        INTEGER NOT NULL,
            rn1       TEXT    NOT NULL
        )
    """)

    count = db.execute("SELECT COUNT(*) FROM siprouter_sp").fetchone()[0]
    if count == 0:
        db.executemany(
            "INSERT INTO siprouter_sp "
            "(elemento, bloco_ip, vlan, vrf, descricao, cn, rn1) "
            "VALUES (?, ?, ?, ?, ?, ?, ?)",
            _SEED_DATA,
        )
        db.commit()
        logger.info("siprouter_sp: %d registros inseridos.", len(_SEED_DATA))
    else:
        logger.debug("siprouter_sp: tabela já preenchida (%d registros).", count)


# =============================================================================
# HELPERS INTERNOS
# =============================================================================
def _rn1_matches(table_rn1: str, user_rn1: str) -> bool:
    """
    Retorna True se o RN1 do usuário está contido no campo rn1 da tabela.

    Regras:
      - "Diversos" sempre corresponde a qualquer RN1.
      - Para valores como "55131/55114", o RN1 do usuário deve ser um dos
        elementos da lista separada por "/".
    """
    if table_rn1 == "Diversos":
        return True
    return user_rn1.strip() in [part.strip() for part in table_rn1.split("/")]


def _desc_is_espelho(descricao: str) -> bool:
    return "ESPELH" in descricao.upper()


def _desc_is_reserva(descricao: str) -> bool:
    return "RESERVA" in descricao.upper()


def _desc_is_scm(descricao: str) -> bool:
    return "SCM" in descricao.upper()


def _desc_is_av(descricao: str) -> bool:
    return "AV" in descricao.upper()


def _desc_is_operadora(descricao: str) -> bool:
    """Registro de operadora: não é ESPELHO, RESERVA, SCM nem AV."""
    d = descricao.upper()
    return not any(kw in d for kw in ("ESPELH", "RESERVA", "SCM", "AV"))


def _pair_pl_jg(rows: list, descricao: str) -> dict | None:
    """
    Retorna o par PL/JG para uma descrição específica,
    ou None se não houver registros.
    """
    pl = next((r for r in rows if r["elemento"] == "CDSIP_SPO_PL" and r["descricao"] == descricao), None)
    jg = next((r for r in rows if r["elemento"] == "CDSIP_SPO_JG" and r["descricao"] == descricao), None)

    if not pl and not jg:
        return None

    ref = pl or jg
    return {
        "descricao": ref["descricao"],
        "vlan":      ref["vlan"],
        "vrf":       ref["vrf"],
        "rn1":       ref["rn1"],
        "pl_bloco":  pl["bloco_ip"] if pl else None,
        "jg_bloco":  jg["bloco_ip"] if jg else None,
    }


# =============================================================================
# QUERY PÚBLICA — retorna SEMPRE um único bloco (par PL/JG)
# =============================================================================
def query_siprouter_sp(
    db: sqlite3.Connection,
    cn: int,
    rn1: str,
    scm: bool = False,
    av: bool  = False,
) -> dict:
    """
    Consulta a tabela siprouter_sp e retorna EXATAMENTE UM bloco (par PL/JG).

    Prioridade de seleção:
      1. SCM marcado  → bloco SCM do CN informado.
      2. AV marcado   → bloco AV do CN informado.
      3. Nenhum flag  → bloco cuja lista de RN1 contenha o RN1 informado
                        (operadoras específicas).
         Fallback     → se nenhuma operadora casar com o RN1, retorna o
                        bloco ESPELHINHOS do CN informado.

    Retorno:
        {
          "found":          bool,
          "cn":             int,
          "rn1_consultado": str,
          "descricao":      str,
          "vlan":           int | None,
          "vrf":            str | None,
          "pl_bloco":       str | None,
          "jg_bloco":       str | None,
          "mensagem":       str,
        }
    """
    def _not_found(msg: str) -> dict:
        return {
            "found": False, "cn": cn, "rn1_consultado": rn1,
            "descricao": None, "vlan": None, "vrf": None,
            "pl_bloco": None, "jg_bloco": None, "mensagem": msg,
        }

    # Buscar todos os registros do CN
    rows = db.execute(
        "SELECT elemento, bloco_ip, vlan, vrf, descricao, cn, rn1 "
        "FROM siprouter_sp WHERE cn = ? ORDER BY vlan ASC",
        (cn,),
    ).fetchall()

    if not rows:
        return _not_found(f"Não existe BLOCO IP disponível para CN {cn}.")

    rows = [dict(r) for r in rows]

    # ------------------------------------------------------------------
    # 1. SCM marcado → bloco SCM do CN
    # ------------------------------------------------------------------
    if scm:
        scm_rows = [r for r in rows if _desc_is_scm(r["descricao"])]
        if not scm_rows:
            return _not_found(f"Nenhum bloco SCM encontrado para CN {cn}.")
        desc = scm_rows[0]["descricao"]
        par  = _pair_pl_jg(rows, desc)
        if not par:
            return _not_found(f"Par PL/JG não encontrado para bloco SCM — CN {cn}.")
        return {**par, "found": True, "cn": cn, "rn1_consultado": rn1,
                "mensagem": f"Bloco SCM · CN {cn}."}

    # ------------------------------------------------------------------
    # 2. AV marcado → bloco AV do CN
    # ------------------------------------------------------------------
    if av:
        av_rows = [r for r in rows if _desc_is_av(r["descricao"])]
        if not av_rows:
            return _not_found(f"Nenhum bloco AV encontrado para CN {cn}.")
        desc = av_rows[0]["descricao"]
        par  = _pair_pl_jg(rows, desc)
        if not par:
            return _not_found(f"Par PL/JG não encontrado para bloco AV — CN {cn}.")
        return {**par, "found": True, "cn": cn, "rn1_consultado": rn1,
                "mensagem": f"Bloco AV · CN {cn}."}

    # ------------------------------------------------------------------
    # 3. Sem flags → tentar casar RN1 com operadora específica
    # ------------------------------------------------------------------
    operadora_rows = [
        r for r in rows
        if _desc_is_operadora(r["descricao"]) and _rn1_matches(r["rn1"], rn1)
    ]

    if operadora_rows:
        desc = operadora_rows[0]["descricao"]
        par  = _pair_pl_jg(rows, desc)
        if par:
            return {**par, "found": True, "cn": cn, "rn1_consultado": rn1,
                    "mensagem": f"{desc} · CN {cn}."}

    # ------------------------------------------------------------------
    # 4. Fallback → bloco ESPELHINHOS do CN
    # ------------------------------------------------------------------
    espelho_rows = [r for r in rows if _desc_is_espelho(r["descricao"])]
    if not espelho_rows:
        return _not_found(
            f"Nenhum bloco encontrado para CN {cn} / RN1 {rn1}."
        )
    desc = espelho_rows[0]["descricao"]
    par  = _pair_pl_jg(rows, desc)
    if not par:
        return _not_found(f"Par PL/JG não encontrado para ESPELHINHOS — CN {cn}.")
    return {**par, "found": True, "cn": cn, "rn1_consultado": rn1,
            "mensagem": f"{desc} · CN {cn} (fallback ESPELHINHOS)."}
