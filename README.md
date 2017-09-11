# Dicas de Programação Sapiens

## Dica \#1: RTFM

Leia o manual. Antes de começar a escrever seu primeiro programa, descubra quais são e o que fazem as instruções da máquina, mesmo que superficialmente. Aprenda sobre as diretivas de compilador como `DB`, `DS` e `STR` e como são usadas. A princípio esta dica pode parecer óbvia, mas acredite em mim, ela talvez seja dica mais importante dessa lista. O manual do Sapiens é razoavelmente curto, então começe com o pé direito ao invés de desperdiçar horas tentando solucionar problemas sem conhecer diretio a ferramenta.

## Dica \#2: Comente seu código

Devido ao nível extremamente baixo da linguagem, programação em assembly é um processo minucioso, especialmente quando se trata de uma arquitetura limitada como o Sapiens. Para evitar a confusão, é importante manter uma ótima organização do seu código fonte. Nesse sentido, existem várias estratégias que podem ser abordadas. Um método simples, mas bastante efetivo de escrever um programa de forma organizada envolve separar o código em pequenos blocos logicamente distintos. Cada bloco realiza uma função específica e é acompanhado de um comentário descrevendo sua função. Idealmente, cada bloco deve executar sua função de forma autônoma, independentemente dos outros. Por exemplo:

```
    LDA I       ; ENQUANTO I != 4
    SUB #4
    JZ ENDWHILE

    LDA @ADDR   ; TEMP = *ADDR
    STA TEMP

    LDA #0      ; J = 0
    STA J
```

Assim, fica fácil seguir a semântica e o fluxo lógico do programa sem que seja necessário analisar cada instrução individualmente. Independente da estratégia que você decida adotar no seu próprio código, manter uma boa documentação do código fonte é indispensável.

## Dica \#3: Não se esqueça da diretiva `END`

A diretiva `END` é responsável por instruir ao compilador o endereço de início do seu programa. Seu uso é extremamente importante, mesmo que o compilador não efetivamente enforce sua presença no código. Até o momento, seus programas podem ter funcionado sem problemas aparentes. Por padrão, a execução se inicia a partir do endereço `0`, porém este comportamento nem sempre é garantido. Ao não especificar o endereço inicial do programa, o simulador mantém o endereço inicial já estabelecido, seja este qual for. Isto é, caso algum programa anterior altere o endereço inicial para outro valor, seu código não será executado a partir do endereço inicial esperado. Por isso, lembre-se de sempre utilizar esta diretiva, garantindo que seu programa funcione independente do estado inical do simulador.

Outra observação importante é pertinente ao uso da diretiva `ORG`. Esta diretiva define o endereço físico a partir do qual as instruções e dados que seguem são inseridos em memória. É importante perceber que o uso indevido desta diretiva pode fazer com que dados e instruções sejam sobrepostas na memória, causando comportamento inesperado. No programa abaixo, por exemplo, a execução não sai do loop e portanto nunca termina:

```
ORG 10
X: DB 0

ORG 0
WHILE:
    LDA X       ; ENQUANTO X != 10
    SUB #10
    JZ ENDWHILE

    LDA X       ; X++
    ADD #1
    STA X

    JMP WHILE

ENDWHILE:
    HLT

END 0
```

Nos meus programas, eu tento evitar ao máximo o uso desta diretiva, exceto em alguns casos para fim de teste. Eu prefiro definir o início do meu programa com uma label qualquer, como por exemplo `START` ou `MAIN`, desta forma as posições são definidas pelo compilador e não há risco de sobreposição:

```
X: DB 0

WHILE:
    LDA X       ; ENQUANTO X != 10
    SUB #10
    JZ ENDWHILE

    LDA X       ; X++
    ADD #1
    STA X

    JMP WHILE

ENDWHILE:
    HLT

END WHILE
```

## Dica \#4: Soma e subtração em 16 bits

Quando se trabalha com arquiteturas de 8 bits, é essencial saber como lidar com números em larguras maiores como 16, 32 ou até 64 bits. Isso se torna ainda mais importante no caso do Sapiens, que embora seja limitado a aritmética em 8 bits, também faz uso extenso de dados em 16 bits para endereços de memória. Como a ALU da arquitetura é limitada a cálculos com 8 bits de largura, é necessário que haja uma separação dos cálculos com números grandes em múltiplas etapas de apenas um byte cada. No Sapiens, as instruções instruções `ADC` e `SBC`, que fazem uso da flag de carry, são a maneira mais prática de realizar essa separação. Através dessas instruções fica fácil encadear somas e subtrações com vários bytes, automaticamente levando em consideração os eventuais casos de "vai-um" e "vem-um" através da flag de carry. Desta forma, não é necessário o uso de desvios condicionais, o que torna o código mais conciso e fácil de entender. Exemplo:

```
    LDA X       ; Z = X + Y
    ADD Y
    STA Z
    LDA X+1
    ADC Y+1
    STA Z+1
```

## Dica \#5: Endereços de memória e a pilha

O uso correto da pilha é essencial para o funcionamento de uma sub-rotina. Por isso, é importante estabelecer um padrão que determine como os argunetos devem ser passados entre as subrotinas e quem faz a chamada. O padrão que eu uso para o meu código determina que os argumentos devem ser retirados da pilha na ordem normal pela subrotina. Você pode utilizar qualquer padrão em seu código, o mais importante é manter esse padrão de forma consistente. Exemplo:

```
ARR:  DB 10, 11, 12
ARRP: DW NUM

START:
    LDA ARRP+1
    PUSH
    LDA ARRP
    PUSH
    JSR FOO

[...]

PTR: DS 2
VAL: DS 1

; VOID FOO(CHAR *PTR)
FOO:
    STS SP      ; GUARDA ARGS
    POP
    POP
    POP
    STA PTR
    POP
    STA PTR+1

    LDA @PTR    ; VAL = *PTR
    STA VAL
```

É importante observar que, pela própria natureza da pilha, os dados são inseridos e retirados de forma inversa. Não se esqueça disso, principalmente quando for transferir endereços de memória para suas rotinas! Também é bom lebrar que a instrução `JSR` sempre insere o endereço de retorno na pilha, portanto os primeiros dois bytes não fazem parte da lista de argumentos. Outra observação importante é quanto ao uso do atalho `ENDEREÇO+OFFSET` fornecido pelo compilador, que facilita bastante a transferência de dados grandes. Porém, fique atento ao fato de que este cálculo é realizado em tempo de compilação. Por isso só é útil quando se trata de constantes (como uma label) e não serve para dados fornecidos em tempo de execução. Por exemplo, o seguinte trecho de código pode não fazer o que você espera:

```
    LDA @PTR+2
    STA VAL
```

## Dica \#6: Usos alternativos para o apondator de pilha

**Atenção: o uso dos atalhos abaixo irá corromper o valor atual do apontador de pilha. Por isso, é importante sempre salvar o valor original do apontador e restaurá-lo antes de voltar a usar a pilha normalmente, com instruções como `PUSH` e `RET`.**

O registrador `SP` na arquitetura do Sapiens é utilizado como o apontador de pilha. Isto é, ele guarda um ponteiro que indica o endereço atual do topo da pilha. No total, o Sapiens dispõe de quatro instruções que tratam diretamente com o valor desse registrador: `PUSH`, `POP`, `STS` e `LDS`. As duas primeiras lidam com a inserção e remoção de valores na pilha, já as duas últimas servem de mecanismo para salvar e recuperar o valor do apontador. Até o momento, temos usado este registrador única e exclusivamente para sua função projetada. Isto é, para manipular a pilha e seus valores. Porém, também podemos usar o registrador com propósito um pouco mais geral. Isso é importante porque, ao contrário do acumulador, o apontador de pilha trabalha com valores de 16 bits. Com isso, ele pode nos ajudar em certos casos a lidar com valores de 16 bits com maior facilidade. Por exemplo, na transferência de valores com essa largura:

```
    LDS A       ; B = A
    STS B
```

O trecho de código acima, em apenas dois comandos, copia dois bytes do endereço `A` para o `B`. A melhoria se torna ainda mais evidente quando usamos o modo de endereçamento indireto. Por exemplo, observe o seguinte trecho, que faz uso do registrador de pilha:

```
    LDS @A      ; B = *A
    STS B
```

Agora contraste este trecho com o seguinte, no qual nos limitamos apenas ao uso do acumulador, e perceba como ele se torna bem mais complexo:

```
    LDA @A      ; COPIA BYTE DE *A PARA B
    STA B

    LDA A       ; TEMP = A + 1
    ADD #1
    STA TEMP
    LDA A+1
    ADC #0
    STA TEMP+1

    LDA @TEMP   ; COPIA BYTE DE *TEMP PARA B+1
    STA B+1
```

O apontador de pilha também pode ser útil quando se quer incrementar uma variável de 16 bits de maneira concisa. Neste caso, a instrução `POP` pode ser utilizada:

```
   LDS PTR      ; PTR++
   POP
   STS PTR
```

Por outro lado, a instrução `PUSH` não deve ser utilizada, já que há o risco de corromper uma posição inesperada da memória.
