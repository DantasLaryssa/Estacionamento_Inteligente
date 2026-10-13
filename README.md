# Estacionamento Inteligente - Validadores & Scanner

Guilherme Pedigone RA: 2712633  
Laryssa Dantas RA: 2342968  
Gabriel Diogo RA: 2683210

Projeto da Prática 3 de Compiladores (Aula CP 08 - Autômatos Finitos na Prática).

## Cenário

Escolhemos o **estacionamento inteligente**.

Num estacionamento, quase tudo gira em torno de textos curtos que alguém digita ou que uma câmera lê: a placa do carro, o horário de entrada e saída, o código da vaga, o valor pago. Se um desses textos vem errado (uma placa com um dígito a mais, um horário `25:10`), o erro precisa ser barrado na entrada, antes de virar cobrança errada ou carro "perdido" no sistema. Já a cancela é um exemplo clássico de máquina de estados: ela não pode "terminar de fechar" sem ter aberto, nem abrir com o motor travado.

Por isso o projeto fabrica, na mão e em Python puro:

- **2 validadores AFD**: a placa (Mercosul e antiga no mesmo autômato) e o horário `HH:MM`;
- **1 máquina de estados de negócio**: a cancela, com o `sensor_obstaculo` reabrindo quando algo atrapalha o fechamento;
- **1 mini-scanner** da linguagem de comandos do estacionamento, que lê linhas como `ENTRADA BRA2E19 08:15 VAGA A12` e `SAIDA BRA2E19 10:40 PAGO 12.00`.

O notebook é o `Estacionamento_Inteligente.ipynb`. Ele usa só a biblioteca padrão do Python, e o `ipywidgets` só aparece no painel da Célula 10.

## O que tem em cada célula

| Célula | O que tem | Requisito |
|---|---|---|
| 0 | Preparação do ambiente | - |
| 1 | Classe `AFD` (`rodar()`, `tabela()` e `minimizar()`) | R1, R2 |
| 2 | Validador 1: placa Mercosul e antiga | R1, R2 |
| 3 | Validador 2: horário `HH:MM` | R1, R2 |
| 4 | Bateria de testes AFD x RegEx | R3 |
| 5 | Máquina de estados: a cancela | R4 |
| 6, 7, 8 | Scanner: função, tokens e erros | R5 |
| 9 | Bônus: AFD mínimo | Bônus |
| 10 | Laboratório interativo (painel ipywidgets) | R6 |
| 11 | Resumo final com todos os requisitos conferidos | - |

## O que foi feito em cada requisito

**R1 - Dois validadores AFD (Células 1, 2 e 3)**
Escrevemos o motor uma vez só, na classe `AFD`, na forma da definição formal da apostila: `Q` (calculado a partir do delta), `Σ` (as classes usadas), `δ` (um dicionário `{(estado, classe): próximo}`), `q0` e `F`. O que muda de um validador para o outro é só a tabela. Se uma chave não existe no dicionário, não há seta no desenho, e o autômato vai para o estado morto e rejeita.

*Validador 1: placa (10 estados, 3 classes).* A placa Mercosul é Letra Letra Letra Dígito Letra Dígito Dígito (`BRA2E19`) e a antiga é Letra Letra Letra, hífen e 4 dígitos (`ABC-1234`). Em vez de uma seta para cada uma das 26 letras e cada um dos 10 dígitos, usamos as classes `L` (letra maiúscula), `D` (dígito) e `-`. Os 3 primeiros estados são iguais nos dois formatos. Depois da 3ª letra o autômato se divide: um dígito segue o caminho Mercosul, um hífen segue o caminho antigo. Os dois caminhos voltam a se encontrar em `q5`, porque dali em diante falta a mesma coisa (2 dígitos, depois 1).

```
→ (q0) ─L→ (q1) ─L→ (q2) ─L→ (q3) ─D→ (q4) ─L→ (q5) ─D→ (q6) ─D→ ((qF))
                              │                  ▲
                              └─ - ─→ (qH) ─D→ (qA) ─D─┘          (antiga: ABC-1234)
```

```
📋 Tabela de transição — Placa (Mercosul e antiga)
     estado       L     D     -
  →  q0          q1     —     —
     q1          q2     —     —
     q2          q3     —     —
     q3           —    q4    qH
     q4          q5     —     —
     qH           —    qA     —
     q5           —    q6     —
     qA           —    q5     —
     q6           —    qF     —
   * qF           —     —     —
```

*Validador 2: horário `HH:MM` (7 estados, 6 classes).* Aqui a classe "dígito" não basta, então os dígitos viram classes finas (`0-1`, `2`, `3`, `4-5`, `6-9`): a hora só vai até 23 e o minuto só até 59. Se a hora começa com 0 ou 1, o segundo dígito é livre; se começa com 2, só vai até 3.

```
                    0-1,2,3,4-5,6-9
→ (H0) ──0-1──→ (H1) ──────────────┐
    │                              ▼
    └───2────→ (H2) ──0-1,2,3──→ (SEP) ──:──→ (M0) ──0-1,2,3,4-5──→ (M1) ──0-1,2,3,4-5,6-9──→ ((FIM))
```

```
📋 Tabela de transição — Horário HH:MM
     estado     0-1     2     3   4-5   6-9     :
  →  H0          H1    H2     —     —     —     —
     H1         SEP   SEP   SEP   SEP   SEP     —
     H2         SEP   SEP   SEP     —     —     —
     SEP          —     —     —     —     —    M0
     M0          M1    M1    M1    M1     —     —
     M1         FIM   FIM   FIM   FIM   FIM     —
   * FIM          —     —     —     —     —     —
```

**R2 - Rastreamento passo a passo (Células 2 e 3)**
O `rodar()` mostra, a cada passo, o caractere lido, a classe dele, de onde o autômato saiu e para onde foi. Para cada AFD mostramos palavras aceitas e rejeitadas, e as rejeitadas cobrem os dois jeitos de rejeitar:

- **Morreu no meio:** `ABC1234` (placa sem hífen e sem a letra do meio). Em q4 chega um dígito e não existe seta, porque depois de `ABC1` a placa Mercosul pede uma letra. No horário, `25:10` morre em H2, porque a hora 25 não existe.
- **Terminou fora de F:** `ABC1D2`. Todos os caracteres foram aceitos, mas a palavra acabou em q6, que não é de aceitação. Faltou o último dígito. No horário, `10:4` termina em M1 pelo mesmo motivo.

```
🔎 Placa (Mercosul e antiga) lendo 'BRA2E19'
  passo lido   classe  de     para
  1     'B'    L       q0     q1
  2     'R'    L       q1     q2
  3     'A'    L       q2     q3
  4     '2'    D       q3     q4
  5     'E'    L       q4     q5
  6     '1'    D       q5     q6
  7     '9'    D       q6     qF
  ✅ ACEITA: terminou em qF, que é de aceitação

🔎 Placa (Mercosul e antiga) lendo 'ABC-1234'
  passo lido   classe  de     para
  1     'A'    L       q0     q1
  2     'B'    L       q1     q2
  3     'C'    L       q2     q3
  4     '-'    -       q3     qH
  5     '1'    D       qH     qA
  6     '2'    D       qA     q5
  7     '3'    D       q5     q6
  8     '4'    D       q6     qF
  ✅ ACEITA: terminou em qF, que é de aceitação

🔎 Placa (Mercosul e antiga) lendo 'ABC1234'
  passo lido   classe  de     para
  1     'A'    L       q0     q1
  2     'B'    L       q1     q2
  3     'C'    L       q2     q3
  4     '1'    D       q3     q4
  5     '2'    D       q4     ☠ morto
  ❌ REJEITADA: morreu no meio, não há transição de q4 com a classe 'D'

🔎 Placa (Mercosul e antiga) lendo 'ABC1D2'
  passo lido   classe  de     para
  1     'A'    L       q0     q1
  2     'B'    L       q1     q2
  3     'C'    L       q2     q3
  4     '1'    D       q3     q4
  5     'D'    L       q4     q5
  6     '2'    D       q5     q6
  ❌ REJEITADA: terminou em q6, que NÃO é de aceitação (faltou texto)
```

**R3 - Bateria de testes AFD x RegEx (Célula 4)**
Cada AFD tem uma RegEx equivalente, usada com `re.fullmatch` (o `fullmatch` evita aceitar texto com lixo antes ou depois):

| AFD | RegEx |
|---|---|
| Placa | `[A-Z]{3}([0-9][A-Z][0-9]{2}\|-[0-9]{4})` |
| Horário | `([01][0-9]\|2[0-3]):[0-5][0-9]` |

Cada bateria tem 6 palavras aceitas e 9 rejeitadas, incluindo a palavra vazia. Resultado: **0 divergências**. Além disso, fizemos uma checagem exaustiva, comparando o AFD e a RegEx em todas as palavras de um alfabeto reduzido: 87.381 palavras para a placa (até 8 caracteres do alfabeto `A`, `1`, `-`, `a`) e 37.449 para o horário (até 5 caracteres), também com 0 divergências.

```
=== Placa ===
  'ABC1D23'   AFD=True  RegEx=True  esperado=True  ✅
  'ABC-1234'  AFD=True  RegEx=True  esperado=True  ✅
  ''          AFD=False RegEx=False esperado=False ✅
  'ABC1234'   AFD=False RegEx=False esperado=False ✅
  'abc1d23'   AFD=False RegEx=False esperado=False ✅
  'ABC1D2'    AFD=False RegEx=False esperado=False ✅
  ...
  -> 6 aceitas, 9 rejeitadas, divergências: 0

Exaustivo placa:   87381 palavras testadas, 0 divergências
Exaustivo horário: 37449 palavras testadas, 0 divergências
```

**R4 - Máquina de estados de negócio: a cancela (Célula 5)**
A cancela tem **5 estados e 9 transições**. Os símbolos agora são eventos do estacionamento, e ao entrar em alguns estados a máquina mostra a ação correspondente (`abrir(cancela)`, `fechar(cancela)` e um aviso de cancela travada). O estado final é `FECHADA`, ou seja, o fluxo só é válido se a cancela terminar fechada.

```
FECHADA ─carro_detectado→ ABRINDO ─abertura_completa→ ABERTA ─carro_passou→ FECHANDO ─fechamento_completo→ FECHADA

ABERTA   ─tempo_esgotado→ FECHANDO          FECHANDO ─sensor_obstaculo→ ABRINDO   (a cancela reabre)
ABRINDO  ─falha_motor→ TRAVADA              FECHANDO ─falha_motor→ TRAVADA        TRAVADA ─manutencao_ok→ FECHADA
```

Quando um evento não é permitido, a máquina recusa e mostra quais eventos o estado aceita. Demonstramos 2 fluxos válidos (um deles com o `sensor_obstaculo` reabrindo a cancela) e 2 inválidos:

```
▶ Cancela: carro_detectado → abertura_completa → carro_passou → sensor_obstaculo → abertura_completa → carro_passou → fechamento_completo
  ✅ FECHADA   --carro_detectado--> ABRINDO   [abrir(cancela)]
  ✅ ABRINDO   --abertura_completa--> ABERTA
  ✅ ABERTA    --carro_passou--> FECHANDO   [fechar(cancela)]
  ✅ FECHANDO  --sensor_obstaculo--> ABRINDO   [abrir(cancela)]
  ✅ ABRINDO   --abertura_completa--> ABERTA
  ✅ ABERTA    --carro_passou--> FECHANDO   [fechar(cancela)]
  ✅ FECHANDO  --fechamento_completo--> FECHADA
  🏁 fluxo VÁLIDO: terminou em FECHADA

▶ Cancela: carro_detectado → carro_passou
  ✅ FECHADA   --carro_detectado--> ABRINDO   [abrir(cancela)]
  ❌ evento 'carro_passou' RECUSADO em ABRINDO. Permitidos: abertura_completa, falha_motor
  ⛔ fluxo INVÁLIDO

▶ Cancela: carro_detectado → falha_motor → abertura_completa
  ✅ FECHADA   --carro_detectado--> ABRINDO   [abrir(cancela)]
  ✅ ABRINDO   --falha_motor--> TRAVADA   [aviso("cancela travada")]
  ❌ evento 'abertura_completa' RECUSADO em TRAVADA. Permitidos: manutencao_ok
  ⛔ fluxo INVÁLIDO
```

**R5 - Mini-scanner da linguagem do estacionamento (Células 6, 7 e 8)**
O scanner lê os comandos do estacionamento e devolve `(tipo, lexema, linha, coluna)`. Ele ignora espaços e comentários (`# ...`) e, quando encontra um erro, **registra e continua lendo**. Quem decide se um trecho é placa, horário ou código de vaga são os próprios AFDs: o scanner junta o trecho e pergunta ao autômato (o da placa, o do horário e um AFD pequeno para o código da vaga, de uma letra e 1 ou 2 dígitos).

*Tipos de token (14):*

| Tipo | Exemplo | Tipo | Exemplo |
|---|---|---|---|
| `RESERVADA` | `ENTRADA` | `MENOR` / `MENOR_IGUAL` | `<` / `<=` |
| `ID` | `cancela` | `MAIOR` / `MAIOR_IGUAL` | `>` / `>=` |
| `PLACA` | `BRA2E19`, `ABC-1234` | `IGUAL` | `==` |
| `HORARIO` | `08:15` | `DIFERENTE` | `!=` |
| `CODIGO_VAGA` | `A12` | `NUM_INT` | `5` |
| `NUM_REAL` | `12.00` | `STRING` | `"patio quase lotado"` |

*Palavras reservadas (9):* `ENTRADA`, `SAIDA`, `VAGA`, `PAGO`, `ABRIR`, `FECHAR`, `AVISO`, `SE`, `ENTAO`.

*Maximal Munch:* o scanner sempre escolhe o maior lexema possível. Mostramos quatro casos: `ABC-1234` vira uma única `PLACA` (e não `ID`, hífen e número), `A12` vira `CODIGO_VAGA` (e não `ID`), `<` contra `<=` e `>` contra `>=` (com `==` e `!=` completos), e `18` contra `12.00` contra `08:15` (inteiro, real e horário).

```
'ABC-1234 BRA2E19'     -> [('PLACA', 'ABC-1234'), ('PLACA', 'BRA2E19')]
'A12 B7 ABC'           -> [('CODIGO_VAGA', 'A12'), ('CODIGO_VAGA', 'B7'), ('ID', 'ABC')]
'< <= > >= == !='      -> [('MENOR', '<'), ('MENOR_IGUAL', '<='), ('MAIOR', '>'), ('MAIOR_IGUAL', '>='), ('IGUAL', '=='), ('DIFERENTE', '!=')]
'18 12.00 08:15'       -> [('NUM_INT', '18'), ('NUM_REAL', '12.00'), ('HORARIO', '08:15')]
```

*Erros com linha, coluna e dica.* O scanner tem 9 dicas, todas pensadas para o estacionamento (o requisito pedia pelo menos 2):

| Erro | Dica do scanner |
|---|---|
| `ABC1234`, `XYZ-12` | placa inválida: use `ABC1D23` (Mercosul) ou `ABC-1234` (antiga), com letras maiúsculas |
| `25:10` | horário vai de 00:00 a 23:59, com dois dígitos em cada parte |
| `12,50` | use ponto como separador decimal |
| `$` | não escreva a moeda: o valor é só o número, por exemplo `PAGO 12.50` |
| `=` sozinho | para comparar use `==`, por exemplo `SE livres == 0 ENTAO ...` |
| `!` sozinho | para diferente use `!=` |
| texto sem aspas de fechamento | `AVISO "pátio lotado"` |
| `-` solto | hífen só aparece dentro da placa antiga; valores não são negativos |
| `.` solto | o ponto decimal precisa de dígito antes e depois |

```
7 erros léxicos encontrados:

 ❌ Linha 1, coluna 9: trecho inesperado 'ABC1234' (placa inválida: use ABC1D23 (Mercosul) ou ABC-1234 (antiga), com letras maiúsculas)
 ❌ Linha 2, coluna 15: trecho inesperado '25:10' (horário vai de 00:00 a 23:59, com dois dígitos em cada parte)
 ❌ Linha 2, coluna 27: trecho inesperado '$' (não escreva a moeda: o valor é só o número, por exemplo PAGO 12.50)
 ❌ Linha 2, coluna 28: trecho inesperado '12,50' (use ponto como separador decimal: 12.50)
 ❌ Linha 3, coluna 9: trecho inesperado 'XYZ-12' (placa inválida: use ABC1D23 (Mercosul) ou ABC-1234 (antiga), com letras maiúsculas)
 ❌ Linha 4, coluna 11: trecho inesperado '=' (para comparar use ==, por exemplo: SE livres == 0 ENTAO ...)
 ❌ Linha 4, coluna 27: trecho inesperado '"patio lotado' (texto sem aspas de fechamento: AVISO "pátio lotado")

O scanner continuou lendo e ainda produziu 19 tokens válidos.
Último token lido: ('ID', 'cancela', 5, 8)
```

Exemplo com o movimento correto (trecho):

```
TIPO         LEXEMA                 LIN COL
RESERVADA    ENTRADA                  2   1
PLACA        BRA2E19                  2   9
HORARIO      08:15                    2  17
RESERVADA    VAGA                     2  23
CODIGO_VAGA  A12                      2  28
RESERVADA    ENTRADA                  3   1
PLACA        ABC-1234                 3   9
...
RESERVADA    SAIDA                    4   1
PLACA        BRA2E19                  4   7
HORARIO      10:40                    4  15
RESERVADA    PAGO                     4  21
NUM_REAL     12.00                    4  26
```

**R6 - Interface (Célula 10)**
Um painel com `ipywidgets`, como o do laboratório da aula. Dá para escolher entre os dois validadores, a máquina de estados da cancela e o scanner, digitar o texto e clicar em **Rodar**. Cada opção já vem com um exemplo sugerido.

**R7 - README e organização do código**
Este arquivo. O notebook tem uma célula de texto explicando cada parte e uma célula final (Célula 11) que confere todos os requisitos com `assert`.

**Bônus - AFD mínimo (Célula 9)**
Implementamos o algoritmo de partições (Moore, da Aula 07) no método `minimizar()` da classe `AFD`. Nos dois validadores o número de grupos é igual ao número de estados, ou seja, **nenhum par de estados é equivalente** e os dois já são mínimos (10 estados na placa, 7 no horário).

Também mostramos como o algoritmo ajuda de verdade. O primeiro rascunho da placa deixava a placa antiga com estados próprios até o fim (13 estados). O algoritmo mostra que três deles repetem estados do caminho Mercosul e podem ser fundidos, o que dá os 10 estados do AFD final:

```
Placa (Mercosul e antiga): 10 estados -> 10 grupos de equivalência
   ✅ AFD já é mínimo (nenhum par de estados equivalentes)

Horário HH:MM: 7 estados -> 7 grupos de equivalência
   ✅ AFD já é mínimo (nenhum par de estados equivalentes)

Rascunho da placa: 13 estados -> 10 grupos
    estados equivalentes, podem ser fundidos: ['q5', 'a2']
    estados equivalentes, podem ser fundidos: ['q6', 'a3']
    estados equivalentes, podem ser fundidos: ['q7', 'a4']
```

## Como rodar

Abrir o notebook `Estacionamento_Inteligente.ipynb` no Colab e usar "Ambiente de execução > Reiniciar e executar tudo". Para rodar no VS Code, selecionar um kernel Python 3 e, antes da Célula 10, instalar `ipywidgets` e `ipykernel` (`pip install ipywidgets ipykernel`). As outras células não precisam de nada além do Python.
