# Do 0 ao Top 1 Global em CTFs - Parte 2 (HackTheBox)

## Recon: A Chave...
Continuando... decidi tentar a abordagem de enumerar a API. Tentei acessar alguns endpoints mesmo sem um **token válido**, tentei achar a **documentação** da API e também não foi.

Foi então que decidi ir em busca de, mais uma vez, tentar achar outros "subdomínios", porém, com uma outra wordlist, mas também não deu em nada.

Decidi tentar também uma **enumeração de diretórios** e nada também, a essa altura já estava se esgotando as possibilidades.

Foi então que decidi olhar novamente a página de login que tínhamos conseguido achar anteriormente. Lembrei! **A feature de reset de senha!** Mas antes de chegar nela de novo, eu tinha tentado caçar *fetchs* feitos sem eu saber pela a aplicação, também não encontrei nada. 

Um erro que cometi foi não observar todo o fluxo via **Burp Suite**,  foi então que decidi fazer isso durante todo o fluxo de reset de senha.

Porém, eu pensei: "Certo, reset de senha, e além de um reset de senha, ainda temos uma possível enumeração de usuários aqui". Acontece que, ao informamos um usuários qualquer, recebíamos a "dica"  da aplicação dizendo "User not found". Porém, para comprovar a enumeração de usuários, precisaríamos de um usuários válido. Mas, eis a questão: onde? como?
## Dedução de usuários...

Ao olharmos na página principal, além de outras informações, **lá estava os nomes de algumas pessoas importantes para a empresa fake**.  Voltando ao formulário de login, **um input do formulário dava a dica de como era composto o e-mail** para fazer login.

Foi então que eu decidi ligar os pontos:
- Padrão de e-mail "exposto" pelo input: user@empresa.com
- Nomes de pessoas importantes sendo exibidos na página principal.

Decidi, pegar os nomes dessas pessoas e tentar, manualmente, criar usuários seguindo o formato acima, e depois de umas duas tentativa mais ou menos: opaaaa, encontramos um!

Basicamente eu fiz:

```
user@empresa.htb
```

Recebemos usuário válido!
## Exposição desenfreada de informações...

Todo esse fluxo estava sendo registrado no **Burp Suite**. Quando eu fui observar o fluxo no Burp, encontrei ouro! Novos endpoints, e um deles em especial, com uma falha grave: **Excessive Data Exposure!**

O endpoint era tipo:

```
/api/v1/account/reset-password
```

Simplesmente o endpoint cuspia **TUDO** do usuário, como resposta em um JSON, "de graça". Ou seja, apenas ao interagir com ele, informando um usuário válido, mesmo sem saber senha, sem ter token, sem nada, apenas um usuário válido e ele expunha tudo!

Até o **HASH** da senha do usuário era exposto ali, que loucura! Para descartar essa possibilidade, peguei essa hash, pus em um `txt` coloquei no `john` para quebrar. Gosto de usar ele pois em VMs funciona bem.

Enquanto isso, fui fazer outros testes. Para resetar a senha era necessário ter em mãos um **Token**, creio que esse token seja enviado para o e-mail do funcionário. Até aí, beleza, até tentei um tipo de token qualquer para **tentar induzir a aplicação a me dizer como devia ser o formato do token.**

Porém, essa abordagem não rolou. Mas, para deixar tudo ainda mais absurdo, a mesma requisição que estava "cuspindo" todas as informações do usuários, por incrível que pareça, cuspia também um "TmpToken", sim, como já imaginou, ela cuspia até o token usado para resetar a senha.
## Acesso ao dashboard

Eu não mencionei, mas uma das informações que a request expunha era que o nosso usuário em questão era **administrador**, o que nos colocou em um bom patamar. Agora, começou uma nova "guerra": Entender o que seria o Flowise e como eu poderia me aproveitar dele para tentar um acesso inicial ao servidor.

A partir daqui, continua nos próximos episódios.
## Lições aprendidas...

Creio eu que se eu tivesse me tocado em olhar todo o fluxo logo de primeira pelo Burp, teria evitado todo esse trabalho que tive e uma possível frustração por um detalhe tão simples que deixei passar. **Observar o fluxo, também é recon.** Tentar simular um cenário padrão, como tentei fazer, **também pode ser um recon**.

Buscar entender a aplicação e todo seu fluxo, ainda que não tenhamos tudo que precisamos para isso, também pode ser um recon. É nessa hora que a criatividade pode entrar em cena. As vezes concentramos muitos esforços seguindo rotina: enumerar subdomínios, diretórios, portas abertas, etc.. Mas esquecemos dessa parte simples, porém essencial que é "tatear" a aplicação.

---

Por hoje é isso. Te aguardo nas próximas partes desta série.
Fique à vontade para entrar em contato comigo, caso queira. Até breve!
Deus te abençoe!
