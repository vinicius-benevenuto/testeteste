No arquivo do portal de cadastros, a coluna 'CENTRAL' pode ter estes valores:

[SPO.PL.AS6
SPO.JG.AS6
RJO.BA.AS1
BHE.FU.AS1
SDR.CB.AS1
BLM.PD.AS1
CTA.PG.AS1
PAE.IP.AS1
] 

cada valor deste tem um valor correspondente:

[ASIPASA
ASIJGRA
ASIRJOA
ASIBHEA
ASISDRA
ASIBLMA
ASICTAA
ASIPAEA]

Converter cada valor da coluna 'CENTRAL' no valor correspondente, assim, teremos os primeiros 7 caracteres de 'Rótulos de Linha'.

Com os primeiros 7 valores, o programa deve colocar '_' e juntar com os outros 6 valores que vamos encontrar.

O programa vai encontrar os outros 6 valores ou na coluna 'Label E' ou na coluna 'Label S', como assim? Vou te explicar!

primeiro, ele vai verificar a coluna 'Label E' na linha correspondente, se o valor de 'Label E' tiver mais que dois caracteres, usa este valor, senão ele verifica no 'Label S'.

Desta forma, temos os 7 primeiros caracteres concatenados a partir de um '_' com os outros 6 caracteres.

Após isso, vai acontecer a verificação no arquivo 3 na coluna 'Rótulos de Linha'.

Se localizar em 'Rotulos de Linha', temos acesso a todos os dados que precisamos para preencher as colunas.

Caso não localize, será necessário adicionar está rota.

Regras de adição:

Se vier de um arquivo do Science, a [coluna 1] é 'VIVO-SMP', caso vier do portal de cadastros é 'VIVO-STFC'.

A [coluna 2] são os UFs, e como vamos encontrar estes UFs? vou te explicar! Na coluna [Designação] é informado o 'CN', vou te passar a correlação de 'CNs' e seus respectivos 'UFs' aqui:

[Escrever aqui]

