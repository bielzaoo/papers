# Do 0 ao Top 1 Global no HackTheBox (Máquina 1)

## Como assim?

Talvez você não tenha entendido esse título, mas sim: aqui vou documentar meu dia a dia no CTF, especialmente no HackTheBox.

Todos os dias vou registrar minha jornada, erros, acertos, insights e por aí vai.

Tudo isso tem um objetivo maior: não, não é "ficar famoso" ou apenas ter a glória de alcançar um bom ranking no HackTheBox. O foco é claramente um: **desenvolver uma mente ofensiva**.

Mente ofensiva? Sim, isso mesmo. Ter a capacidade de olhar para um alvo — seja uma aplicação, uma rede, enfim — e conseguir mapear mentalmente brechas, falhas e caminhos de ataque é algo que certamente diferencia excelentes profissionais de meros executores de ferramentas e cumpridores de checklists.

Meu foco aqui será usar o HackTheBox como ferramenta para desenvolver esse mindset.

E aí, vamos comigo nessa jornada? Talvez você pense que não tem capacidade, ou fique se comparando com os tops globais de lá, mas você também pode. Talvez leve tempo, mas com dedicação e com o **objetivo correto**, você consegue.

Não vou revelar os nomes das máquinas aqui; no momento da escrita, elas ainda estarão ativas, então quero evitar quaisquer problemas.

## Part 1: Trabalhando o Recon

Não há fase mais primordial que essa: **o recon**. E, nesta máquina em específico, decidi focar mentalmente em trabalhar o recon.

Decidi começar com um scan mais agressivo logo de cara, para adiantar o processo. Essa máquina é de *nível easy*, mas quem conhece bem o HackTheBox sabe que o *easy* deles não é tão *easy* assim.

Geralmente, começo com um scan mais simples apenas para identificar todas as portas abertas:

```plaintext
sudo nmap -sS -v -p- --min-rate 5000 <ip> 
```

Para depois progredir para um scan mais profundo nas portas encontradas:

```plaintext
sudo nmap -sCV -p22,80 -v <ip>
```

Geralmente essa abordagem é mais rápida e evita problemas com "perda de porta", que é quando o `nmap` deixa uma (ou várias) portas passarem.

Desta vez, mudei a abordagem para:

```plaintext
sudo nmap -sC -sV -O -T4 -p- -n -Pn <ip> -oA fullscan
```

Não mudou muita coisa no resultado, mas já adianta o processo.

O que realmente mudou foi o meu **mindset**. Desta vez, decidi dar foco total ao recon. Quantas vezes eu falhei, principalmente nas máquinas do HTB, por conta de **pressa** para tentar resolver a máquina e acabei negligenciando um bom recon?

Segui com esse mindset durante toda a máquina. Olhando serviço por serviço, tudo parecia bem simples: apenas uma landing page com algumas informações sobre a empresa fictícia.

Desta vez, pensei em ser mais assertivo, **levando em consideração o cenário**. Após observar a página e navegar pelo que era possível, decidi usar o `ffuf` para tentar encontrar algum "subdomínio". Quem conhece o HTB sabe que boa parte das máquinas utiliza a técnica de **VHOST**, que consiste em manter múltiplas aplicações web no mesmo IP, diferenciadas pelo header `Host:`.

Com o `ffuf`:

```plaintext
ffuf -u alvo.htb -w <seclist> -H "Host: FUZZ.alvo.htb" -fc 301
```

Filtrei pelo **status code** 301 porque, para hosts inválidos, o servidor estava retornando esse código. Havia um `nginx` por trás.

E lá estava um outro "subdomínio": `staging`, simulando um ambiente real.

Num primeiro momento, era bem simples: apenas uma página de login com funcionalidade de **reset de senha**. Isso é interessante, pois dependendo da lógica, **podemos alterar a senha de outros usuários**.

Ainda não havia nenhum usuário válido, mas já tínhamos mapeado uma superfície.

Outra coisa que decidi fazer como parte do **recon** foi observar se havia alguma requisição por trás — talvez um *fetch* escondido ou algo do tipo. E, de fato, havia.

Era um GET para o que parecia ser uma configuração. Não ficou muito claro em um primeiro momento, mas já era mais uma informação: **temos uma API aqui**.

Nesses casos, é sempre bom tentar trocar o método da requisição. Ao fazer isso, recebemos como resposta uma espécie de página padrão informando qual era o serviço por trás.

Era o **Flowise**. Nesse ponto, como eu não conhecia o serviço, utilizei o ChatGPT para entender do que se tratava, e ele me direcionou em alguns passos para testes.

O foco agora passa a ser mais um pouco de recon: entender exatamente com o que estamos lidando, verificar se conseguimos encontrar a **documentação** da API e identificar possíveis configurações equivocadas por parte do servidor — sempre com foco em **recon** e **mapeamento de superfície de ataque**.

## Lições aprendidas

Sempre, sempre busque primeiro **recon** — e somente **recon**.

Não se afobe. Sei que se trata de um CTF, que já existe um caminho definido pelo autor da máquina, mas paciência e foco evitam frustrações como:

"HTB é muito difícil",
"Já travei, vou olhar write-up".

Na vida real, não existe write-up. Não podemos nos esquecer disso.

Eu poderia ter começado com brute force de diretórios e ficado rodando longas wordlists, como fazia antes. Mas decidi mudar a abordagem e tentar economizar tempo antes de partir para caminhos mais custosos.

Observe se há algum fluxo por trás. Observe requisições "escondidas". Quase sempre elas estarão lá, vindas de um `fetch` em algum código JavaScript.

Sempre teste — com cautela — mudando métodos HTTP. Pense:
"E se eu mudar isso?"
"E se eu trocar para POST?"

É pensar no fluxo que o **dev** não considerou. Isso faz total diferença.

---

Por hoje é isso. Te aguardo nas próximas partes desta série.
Fique à vontade para entrar em contato comigo, caso queira. Até breve!
Deus te abençoe!
