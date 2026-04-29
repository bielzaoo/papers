# Do 0 ao Top 1 Global em CTFs - Parte 3 (Máquina 1 - User Flag - HackTheBox)

## Tentenado entender o cenário...
Neste ponto aqui começou minha caminhada rumo ao *initial access*. Eu tinha um objetivo claro em mente, e fui em busca de vetores que me possibilitariam isso. Gosto de falar da importância de levar em consideração o cenário e agir orientado a objetivos, isso te poupa tempo e energia para gastar com o que realmente importa.

O dashboard era grande. Não conhecia muito bem o Flowise. Muitas funções, muitos opções e era um ambiente focando em IA. Como não conhecia muito, fui pesquisar. Com a ajuda do ChatGPT fui entendendo e ligando pontos. Meu objetivo não era enumerar informações no Flowise, estou em um ambiente de CTF, e sabia que precisava chegar no servidor.

Obviamente, em um ambiente real, eu enumeraria todo o Flowise em busca de informações sensíveis sobre o alvo, inclusive cheguei a fazer isso e já vou explicar o porquê. 
### A "IA" pode fazer algo por mim...

Descobri que poderia abusar de fluxos e conseguir executar coisas no servidor. Foi então que começou minha jornada com esse foco: preciso abusar de um fluxo ou mesmo montar um para conseguir executar código no servidor e conseguir shell.

Tentei de todas as formas criar fluxos para atingir esse objetivo. Vasculhei o Flowise para isso, mas nada. E o meu erro foi, ao invés de tentar entender melhor o serviço, ficar fazendo "fuzzing manual" nas coisas. Se não conheço ou entendo o serviço, buscar aprender um pouco mais sobre ele é primordial e esse fui justamente o meu erro.

Até que de tanto caçar um ponto que me permitisse executar coisas no servidor, eu achei: uma feature que me permitia criar fluxos de agentes. Mas é fato, se eu tivesse buscado entender o que seria cada feature dos menus, eu teria poupando MUITO ESFORÇO.

Mas, ao chegar neste ponto, mais uma vez, sem entender muito, agi movido pelo meu objetivo, eu precisava executar, e achei: uma feature que me permitia executar código JS. Porém, aqui começou outra dor. Bati numa parede chamada: Sandbox.

O ambiente estava "sanboxed", ou seja, eu consegua executar JS, mas nem tudo. Fui recorrer mais uma vez ao ChatGPT em busca de insights. E tentar enumerar o que eu tinha permissão de executar, tentei de diversas formas e então fui atrás de técnicas de "Sandbox JS scape", achei algumas, mas nada.

Mais uma vez travado, o fato é que, isso é culpa achar que por estarmos em CTFs, há um caminho para ser seguido a fim de se resolver a máquina e que é bem provável que o autor só tenha projetado ele, portanto, quando achamos um **forte** candidato a ser esse caminho, "apostamos todas as nossas fichas" ali e quando não dá certo, vem a frustração e a vontade de olhar o write-up, mas não era possível.

Foi então que decidi investir numa feature que eu vi la trás, quando eu ainda estava "batendo" no Flowise para tentar encontrar algo. No meio desse mar de features tinha uma "tool" que poderia ser executada "via IA" (falando a grosso modo), que me permitia **ler arquivos do disco.**

Eu já insistido nela lá trás, porém, **por não entender como funcionava o Flowise** eu não consegui executá-la e passei a pensar em tentar executar JS no servidor. Mas, como eu finalmente consegui uma de executar ela decidi retomar meu plano de tentar ler algo do servidor.

Pus em minha mente: Ler arquivos do servidor? Excelente, posso ler `/etc/passwd`, ver quais são os usuários presentes na máquina e tentar pegar uma chave SSH. E adivinha! Pois é, consegui lê, mas o único usuário que tinha não tinha chave SSH. Tentei ler o `/etc/shadow` atrás de alguma coisa, consegui ler, mas não tinha nada. Pelo contrário, descobri que o usuário não tinha um login válido e o login com root estava desabilitado:

```
root:*::0:::::  
usuario:!:20285:0:99999:7:::
```

Mais uma vez travado.E agora? Decidi ver se eu estava num container, geralmente faço isso tentando ler o arquivo `/.dockerenv`, ele existia. Isso é bom, mas e aí, o que fazer com isso? Até agora:
- Não temos usuários com logins válidos.
- Sem chave SSH.
- Não tinha como ter uma visualização dos arquivos presentes (via `ls`) e nem saber em qual diretórios estávamos.
- Dentro de um container.
Cenário ideal para travar.

Tentei ler o arquivo `/etc/os-release` para ter um pouco mais de visão sobre qual era o SO que estávamos lidando. E era um `Alpine`. 

Foi então que decidi recorrer ao ChatGPT em busca de insights mais uma vez. Expliquei a situação e ele me indicou tentar ler um arquivo que, até então, nunca tinha ouvido falar e nem sabia que poderia conter informações boas lá: `/proc/self/environ`.

Basicamente, esse arquivo contém segredos do **runtime** do container. Interessante! E ali foi ouro pra mim! Haviam duas credencias, um possível usuário e mais umas outras informações. Foi então que decidir tentar um **Password Reuse** via SSH e como já deve estar imaginando, foi! Consegui acesso não ao container, mas ao **host principal**. "Tão simples", foi só pegar a **user flag** e outra batalha nos esperava: pegar root.
## Lições aprendidas...
O fato é se eu tivesse focado em entender mais o cenário ao invés de ficar fazendo "fuzzing manual" a fim de tentar conseguir atingir meu objetivo, certamente eu teria evitado "dor de cabeça". No fim de tudo, hacking sempre vai ser majoritariamente tentar entender conceitos, cenários, para só então pensar em como proceder.

Isso é a chave. Eu me vi em um cenário da qual não tinha tanto conhecimento, Flowise é do ramo de IA, criação de fluxos e afins, um ambiente da qual eu não entendia tão bem. Mas o meu erro foi: **ao invés de tentar entender, antes de pensar em atacar, busquei a primeira brecha que julguei ser o real caminho e gastei todas as minhas fichas ali.**

O fato é que esses insights só conseguimos com vivência. Mas é claro, se pudermos absorver de alguém, certamente nos pouparemos de gastos de energia desnecessários, nem sempre precisamos errar para aprender.

---

Por hoje é isso. Te aguardo nas próximas partes desta série.
Fique à vontade para entrar em contato comigo, caso queira. Até breve!
Deus te abençoe!
