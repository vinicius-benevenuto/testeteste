{
  "mapping": {
    "REDE": {
      "type": "literal",
      "value": "VIVO-SMP"
    },
    "UF": {
      "type": "reference",
      "ref_col": "CNL",
      "value_col": "UF",
      "target": "UF"
    },
    "CLUSTER": {
      "type": "literal",
      "value": "CLUSTER 3"
    },
    "Tipo de Rota": {
      "type": "coalesce",
      "sources": [
        {"table": "science", "col": "Tipo da Rota"},
        {"table": "portal",  "col": "TIPO_ROTA"}
      ]
    },
    "Central": {
      "type": "coalesce",
      "sources": [
        {"table": "science", "col": "Central Interna"},
        {"table": "portal",  "col": "CENTRAL"}
      ]
    },
    "Rótulos de Linha": {
      "type": "concat",
      "sources": [
        {"table": "portal", "col": "LABEL_E"},
        {"table": "portal", "col": "LABEL_S"}
      ],
      "separator": " | "
    },
    "OPERADORA": {
      "type": "coalesce",
      "sources": [
        {"table": "science", "col": "Operadora Origem"},
        {"table": "science", "col": "Operadora destino"},
        {"table": "portal",  "col": "EMPRESA"}
      ]
    },
    "Denominação": {
      "type": "coalesce",
      "sources": [
        {"table": "science", "col": "Descrição"},
        {"table": "portal",  "col": "DESIGNACAO"}
      ]
    }
  },
  "join_keys": ["Central", "Tipo de Rota"],
  "join_type": "outer",
  "fuzzy_threshold": 90,
  "use_fuzzy": false,
  "_notas": [
    "REDE e CLUSTER são valores fixos — ajuste conforme o contexto.",
    "UF usa tabela de referência por CNL — faça upload via UI.",
    "Tipo de Rota usa Science primeiro; cai para Portal se vazio.",
    "Rótulos de Linha concatena LABEL_E e LABEL_S com ' | '.",
    "OPERADORA: prioridade Operadora Origem > destino > EMPRESA do Portal.",
    "join_type 'outer' garante que nenhuma linha seja perdida."
  ]
}
