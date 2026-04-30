# Do 0 ao Top 1 Global em CTFs - Parte 3 (Máquina 1 - Root Flag - HackTheBox)

Bem, com a **user flag** em mãos, agora uma nova batalha se inicia: **pegar root.** E eu confesso que estou acostumado com máquinas do TryHackMe, onde a forma de pegar root quase sempre se repetia, mudava uma coisa ou outra, mas a metodologia era a mesma. Mas sabia que no HackTheBox seria diferente, por conta de já ter tido contado com outras máquinas de lá, antes dessa.
### Em busca do vetor...

Para não deixar passar, testei o famoso `sudo -l`, não custa nada. Nesse ponto aqui, a enumeração é a chave, como sempre. Eu particularmente gosto de fazer de forma manual, mas há quem goste de usar tools para isso, como o `linpeas.sh`. Quanto menos dependermos de ferramentas, melhor será.

Por conhecer as máquinas do HackTheBox, sei que geralmente se formos em `/opt` acharemos alguma coisa interessante, e advinha? Pois é, lá estava. Basicamente, tinha um serviço, chamado  `gogs`, até então nunca tinha ouvido falar. Indo no ChatGPT, vi que é como se fosse um "Github self-hosting".

Interessante. Nesse ponto, nossa cabeça já deve começar a apitar: Certo, "Github self-hosting"? Repositórios! E dentro desses repositórios podem contem senhas hardcoded, ou outros indicativos de serviços vulneráveis.

Pegando insights com ChatGPT, descobri que o arquivo de configuração deste serviço poderia ser uma mina de ouro, e realmente foi. Ainda mantendo o pensamento acesso em minha cabeça, fiz um *SSH Port Fowarding* para ter acesso ao serviço e, de todas as formas tentei acessar repositórios internos.

Nesse ponto, eu cometi o mesmo erro de antes: achei um **possível vetor** mas o tratei como único caminho para algo útil. Mais um vez é claro, isso me levou ao esgotamento. Foi então que decidi tentar outros serviços internos, mas nada consegui também.

Resultado? Mais uma vez travado. Tudo isso por estar agindo com "muita sede ao pote", isso é bem ruim em alguns casos pois pode gerar um 'boa" frustração. Um erro que cometi também foi agir com premissas sobre o HackTheBox: "Ah, eles não colocariam algo assim" ou "Não vai ser tão óbvio". Isso é ruim, devemos levar o cenário em consideração, mas também devemos considerar todas as possibilidades. Em um cenário real, tudo pode acontecer.

Foi então que me veio a luz: olhar a versão do serviço que tinha achado no inicio. E advinha? Sim, a versão do serviço era o **vetor inicial para escalar privilégios.** A partir deli fui buscar se estava vulnerável e sim, uma baita falha, por sinal.  Porém, **não encontrei exploits**. Nesse momento aqui, se eu fosse só um "executador de tool" eu teria parado e talvez deixado a máquina de lado.

Mas decidi entender melhor a falha, achei um "artigo" explicando o **attack chain** para explorar a falha. Tentei reproduzir, mas não *triggava* o payload. O bug estava ali, mas não era acionado. E agora? ChatGPT! Pedi que o ChatGPT me guiasse exatamente nos passos necessários da chain, mas mesmo assim, nada.

Foi então que recebi a sugestão dele (que até então vinha ignorando, mas uma vez por conta de pressupostos), e adivinha? Funcionou. Não vou revelar mais detalhes da falha, apesar de já ter fornecido o nome do serviço, para evitar problemas, visto que até o presente momento a máquina ainda se encontra ativa.

Peguei root, colocando o uma **pub key SSH** no `.ssh` do root através da falha,  de modo a conseguir um SSH como root e então conseguir pegar a **flag root**.
## Lições aprendidas...

O fato é, o que mais me atrapalhou aqui foi agir com pressupostos errados em minha cabeça, se eu tivesse abraçado, ainda que por maior, **todas as possibilidades** teria evitado essa "dor de cabeça" toda para pegar root. Narrando aqui parece que foi pouca coisa, mas investi bastante tempo em frentes que não me levaram a nada.

Tudo isso por conta de **pressupostos**, **superestimar** o HackTheBox. Principalmente se tratando de um ambiente de CTF, tudo é possível. Mas não duvido nada que em cenários reais também seja, já vi coisas bizarras. Portanto, a principal lição que fica é essa: **considere toda as possibilidades e nunca superestime o cenário em que você está lidando**. as vezes, pode ser mais óbvio do que parece.

---

Por hoje é isso. Te aguardo nas próximas partes desta série.
Fique à vontade para entrar em contato comigo, caso queira. Até breve!
Deus te abençoe!
