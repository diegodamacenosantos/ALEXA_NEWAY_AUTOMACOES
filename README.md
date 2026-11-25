# alexa-neway-automacoes

Nó para o Node-RED que controla os Echos de uma conta Amazon: abrir uma skill, falar um texto, enviar um comando de texto e ajustar o volume. O login é feito uma vez no navegador e depois se renova sozinho.

Neway Automações.

Aviso: a Amazon não tem API oficial para isso. O login e os comandos usam endereços internos da Amazon, que podem mudar sem aviso. Se algo parar de funcionar depois de um tempo, o pacote precisa de atualização.

Base: biblioteca `alexa-remote2`, já embutida no pacote (a instalação não baixa nada).

## O que vem no pacote

Na paleta, no grupo **alexa-neway-automacoes**, aparece um só nó, **Alexa**. O primeiro campo é o **Tipo**, que define o que ele faz; só os campos do tipo escolhido aparecem. Tipos:

- **Skill**: abre uma skill no Echo.
- **Falar**: faz a Alexa falar um texto.
- **Volume**: ajusta o volume do Echo.
- **Comando**: envia um comando de texto, como se você falasse com ela.

A conta Alexa (login e conexão com a Amazon) não aparece na paleta, porque é um nó de configuração. Ela é criada de dentro do nó Alexa (campo **Conta**, botão do lápis) e fica listada no menu, em **Configuration nodes**. Todo nó Alexa usa uma conta.

## Instalar

A instalação é feita dentro do próprio Node-RED.

1. Baixe o arquivo `.tgz` deste repositório.
2. No Node-RED, abra o menu (três barras), **Gerenciar paleta**, aba **Instalar**, e envie o arquivo `.tgz` (botão de enviar/upload).
3. Confirme a instalação. Na paleta aparece o grupo **alexa-neway-automacoes**. Se não aparecer, reinicie o Node-RED.

Se você tinha a versão antiga `node-red-contrib-alexa-plex`, desinstale-a antes (veja Desinstalar).

## Conta (feita uma vez)

Ao adicionar o primeiro nó, clique no lápis do campo **Conta** e preencha:

| Campo | O que é |
|---|---|
| Nome | opcional |
| Região | EUA, Brasil, Reino Unido ou Alemanha; ao escolher, preenche sozinha os três campos abaixo |
| Service Host | endereço do servidor da Alexa da região (preenchido pela Região) |
| Page | página de login da Amazon da região (preenchida pela Região) |
| Idioma | idioma das respostas da Alexa (preenchido pela Região) |
| Este IP | o IP ou nome do servidor do Node-RED, o mesmo que você usa no navegador |
| Porta | porta do login (padrão 3456). Se o Node-RED roda em Docker, essa porta precisa estar publicada |
| Arquivo do token | **obrigatório.** Caminho completo onde o login fica salvo, para os próximos serem automáticos (ex.: `/data/chave`). Precisa ficar em disco que sobrevive ao reinício (no Docker, um volume) |
| Renovar (dias) | de quanto em quanto tempo renova o login (padrão 3) |
| User Agent | opcional |

### Primeiro login

1. Faça o **Deploy**.
2. O status do nó mostra `login: http://IP:PORTA`.
3. Abra esse endereço no navegador **em http (não https)**, entre na sua conta Amazon e espere o status ficar **pronto** (verde).

O login fica salvo no arquivo do token e se renova sozinho.

### Como a conta se mantém de pé

- Renova o token sem derrubar a sessão: o cookie atual segue em uso até o novo estar confirmado.
- Se a internet estiver fora (por exemplo, o container sobe antes da rede), o estado fica `OFFLINE` e ela tenta de novo sozinha, com espera crescente (10 s até 5 min), sem limite.
- Só pede login (`WAIT_PROXY`) quando a Amazon recusa o token três vezes seguidas, ou se não existir token.
- Grava o token com segurança e guarda a cópia anterior em `<arquivo>.bak`, usada se o principal estragar.
- Confere a sessão a cada 5 minutos e renova ao detectar que caiu.
- Os comandos esperam até 30 s a conta voltar, e repetem uma vez se a Amazon responder erro de autenticação.
- Estados: `INIT`, `READY`, `OFFLINE`, `ERROR`, `WAIT_PROXY`.

## Usar o nó

Os campos aparecem nesta ordem; **Dispositivo** é sempre o último.

| Campo | O que faz |
|---|---|
| Tipo | Skill, Falar, Volume ou Comando; define os demais campos |
| Nome | só um rótulo para você reconhecer o nó no fluxo (opcional) |
| Conta | a conta Alexa usada pelo nó. O lápis cria ou edita uma conta |
| Atualizar listas | as listas de Echos (e de skills, no tipo Skill) são carregadas ao abrir o nó; o botão carrega de novo. A conta precisa estar implantada e pronta |
| Skill, Texto, Volume ou Comando | o campo próprio do tipo escolhido (veja abaixo) |
| Dispositivo | o Echo que vai executar a ação, escolhido numa lista |

### Texto, Volume e Comando

Cada um desses campos tem um seletor de tipo de valor:

| Seletor | O que o campo vale |
|---|---|
| caracteres (a z) | o texto digitado |
| números (0 9) | o número digitado |
| msg. | o nome de uma propriedade da mensagem, como `payload` ou `payload.texto`; o valor vem só dela e, se ela não existir, o nó dá erro |

Campo **vazio** = o valor vem do `msg.payload`. Campo preenchido com caracteres ou números = valor fixo, e a mensagem só serve para disparar.

Em todos os tipos, `msg.device` (serial ou nome do Echo) sobrescreve o **Dispositivo** do nó. A skill, o texto, o volume e o comando vêm sempre do que está configurado no nó (com o campo vazio, do `msg.payload`).

Todos os tipos devolvem em `msg.payload` um objeto com `ok: true` e os dados do que foi feito. Se algo falhar, o nó mostra o erro na etiqueta e dispara o erro do fluxo (dá para capturar com um nó *catch*).

### Tipo Skill

Abre uma skill da Alexa no Echo.

| Campo | O que faz |
|---|---|
| Skill | a skill a abrir, escolhida numa lista com as skills da conta (as em desenvolvimento aparecem com "(dev)") |
| Dispositivo | o Echo onde a skill abre |

- **Saída:** `{ ok, skill, device }`.

### Tipo Falar

Faz a Alexa **falar** um texto.

| Campo | O que faz |
|---|---|
| Texto | a frase que ela fala. **Vazio** = fala o `msg.payload` (texto, número ou booleano) |
| Dispositivo | o Echo que vai falar |

- **Saída:** `{ ok, device, text }`.

### Tipo Volume

Ajusta o volume do Echo, de 1 a 100.

| Campo | O que faz |
|---|---|
| Volume | número de 1 a 100. **Vazio** = usa o `msg.payload` (número ou texto numérico, arredondado) |
| Dispositivo | o Echo que muda de volume |

- **Erro:** valor fora de 1 a 100 (inclusive 0) ou que não seja número.
- **Saída:** `{ ok, device, volume }`.

### Tipo Comando

Envia um **comando de texto** à Alexa, como se você falasse com ela ("tocar música brasileira", "que horas são", "parar").

| Campo | O que faz |
|---|---|
| Comando | o texto do comando. **Vazio** = usa o `msg.payload` |
| Dispositivo | o Echo que recebe o comando |

- **Saída:** `{ ok, device, command }`.

## Exemplos

- **Parar a Alexa:** um *inject* com o texto `parar` ligado a um nó Alexa do tipo Comando com o campo vazio.
- **Ajustar o volume:** um *inject* numérico `40` ligado a um nó Alexa do tipo Volume com o campo vazio.
- **Avisar por voz:** uma *function* com `msg.payload = "A porta abriu"` ligada a um nó Alexa do tipo Falar com o campo vazio.
- **Abrir uma skill:** um nó Alexa do tipo Skill com a skill e o Echo escolhidos nas listas, ligado a um *inject*.

## Problemas comuns

| Mensagem | Causa |
|---|---|
| `conta nao pronta: WAIT_PROXY` | falta fazer o login (veja Primeiro login) |
| `conta nao pronta: OFFLINE` | sem internet; a conta tenta de novo sozinha |
| `dispositivo nao definido` | escolha o Echo no nó |
| `comando vazio` / `texto vazio` | preencha o nó ou envie `msg.payload` |
| `volume de 1 a 100` | valor inválido |
| `sem conta` | o nó não tem conta escolhida |
| Editor mostra o nó como era antes da atualização | reiniciar o Node-RED |

## Atualizar

Baixe o `.tgz` atual e instale do mesmo jeito (Gerenciar paleta, Instalar, enviar o arquivo). Reinicie o Node-RED depois. O login da conta é mantido no arquivo do token.

**Quem usa a versão antiga, com vários nós na paleta** (set skills, set text, set volume, set comando, set status): o nó único não reconhece os nós antigos. Antes de atualizar, faça uma cópia dos fluxos, apague os nós antigos e, depois de atualizar, recrie-os com o nó **Alexa** escolhendo o Tipo. O antigo "set status" não existe mais.

**Atalhos de mensagem retirados:** `msg.text`, `msg.volume` e `msg.skill` não são mais lidos. Para texto ou volume vindos da mensagem, use o seletor **msg.** no campo, ou deixe o campo vazio e envie em `msg.payload`.

## Desinstalar

Antes de remover o pacote, apague tudo que o usa:

1. Apague **todos os nós** Alexa de todos os fluxos.
2. Apague a **conta Alexa salva**: menu, **Configuration nodes**, escolha a conta e exclua.
3. Faça o **Deploy**.
4. Menu, **Gerenciar paleta**, aba **Nós**, ache `alexa-neway-automacoes` e clique em **Remover**.

## Segurança

O login (token) fica no arquivo que você indicou em **Arquivo do token**. Quem tem acesso a essa pasta ou ao editor do Node-RED consegue controlar seus Echos. Proteja o administrador do Node-RED com senha se ele estiver exposto na rede, e não publique esse arquivo.
