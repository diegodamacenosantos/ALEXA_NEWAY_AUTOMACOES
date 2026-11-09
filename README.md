# alexa-neway-automacoes

Nós para o Node-RED que controlam os Echos de uma conta Amazon: abrir uma skill, falar um texto, enviar um comando de texto, ajustar o volume e acompanhar o estado da conta. O login é feito uma vez no navegador e depois se renova sozinho.

Neway Automações.

Aviso: a Amazon não tem API oficial para isso. O login e os comandos usam endereços internos da Amazon, que podem mudar sem aviso. Se algo parar de funcionar depois de um tempo, o pacote precisa de atualização.

Base: biblioteca `alexa-remote2`, já embutida no pacote (a instalação não baixa nada).

## O que vem no pacote

Na paleta, no grupo **alexa-neway-automacoes**, aparecem 5 nós:

- **set skills**: abre uma skill no Echo.
- **set text**: faz a Alexa falar um texto.
- **set volume**: ajusta o volume do Echo.
- **set comando**: envia um comando de texto, como se você falasse com ela.
- **set status**: avisa quando a conta muda de estado e informa o diagnóstico.

A conta Alexa (login e conexão com a Amazon) não aparece na paleta, porque é um nó de configuração. Ela é criada de dentro de qualquer um dos nós (campo **Conta**, botão do lápis) e fica listada no menu, em **Configuration nodes**. Todos os nós usam uma conta.

## Instalar

A instalação é feita dentro do próprio Node-RED.

1. Baixe o arquivo `.tgz` deste repositório.
2. No Node-RED, abra o menu (três barras), **Gerenciar paleta**, aba **Instalar**, e envie o arquivo `.tgz` (botão de enviar/upload).
3. Confirme a instalação. Na paleta aparece o grupo **alexa-neway-automacoes**. Se não aparecer, reinicie o Node-RED.

Se o seu Node-RED não aceitar o envio, copie o arquivo para o servidor e rode, na pasta de dados do Node-RED (a que tem o `package.json` do usuário):

```bash
npm install /caminho/alexa-neway-automacoes.tgz
```

Se você tinha a versão antiga `node-red-contrib-alexa-plex`, desinstale-a antes (veja Desinstalar): os tipos de nó são iguais e conflitam.

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
2. O status dos nós mostra `login: http://IP:PORTA`.
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

## Usar os nós

Todos os nós têm estes campos:

| Campo | O que faz |
|---|---|
| Nome | só um rótulo para você reconhecer o nó no fluxo (opcional) |
| Conta | a conta Alexa usada pelo nó. O lápis cria ou edita uma conta |
| Atualizar lista | busca na Amazon os Echos (e as skills, no `set skills`) da conta, para aparecerem nas listas |
| Dispositivo | o Echo que vai executar a ação, escolhido numa lista |

Regra comum, menos no `set skills`: se o campo principal do nó (Texto, Volume ou Comando) estiver **vazio**, o valor vem do `msg.payload`. Se estiver preenchido, o nó usa o valor fixo.

Os quatro nós de ação (`set skills`, `set text`, `set volume` e `set comando`) devolvem em `msg.payload` um objeto com `ok: true` e os dados do que foi feito. O `set status` tem saída própria (abaixo). Se algo falhar, o nó mostra o erro na etiqueta e dispara o erro do fluxo (dá para capturar com um nó *catch*).

### set skills

Abre uma skill da Alexa no Echo.

| Campo | O que faz |
|---|---|
| Skill | a skill a abrir, escolhida numa lista com as skills da conta (as em desenvolvimento aparecem com "(dev)") |
| Dispositivo | o Echo onde a skill abre |

- **Entrada opcional:** `msg.skill` (id `amzn1...` ou o nome exato da skill; o nome só funciona para skills que estão na lista da conta, as outras exigem o id) e `msg.device` (serial do Echo) sobrescrevem o que está no nó.
- **Saída:** `{ ok, skill, device }`.

### set text

Faz a Alexa **falar** um texto.

| Campo | O que faz |
|---|---|
| Dispositivo | o Echo que vai falar |
| Texto | a frase que ela fala. **Vazio** = fala o `msg.payload` (texto, número ou booleano) |

- **Entrada opcional:** `msg.text` sobrescreve tudo; `msg.device` sobrescreve o Echo.
- **Saída:** `{ ok, device, text }`.

### set volume

Ajusta o volume do Echo, de 1 a 100.

| Campo | O que faz |
|---|---|
| Dispositivo | o Echo que muda de volume |
| Volume | número de 1 a 100. **Vazio** = usa o `msg.payload` (número ou texto numérico, arredondado) |

- **Entrada opcional:** `msg.volume` sobrescreve tudo; `msg.device` sobrescreve o Echo.
- **Erro:** valor fora de 1 a 100 (inclusive 0) ou que não seja número.
- **Saída:** `{ ok, device, volume }`.

### set comando

Envia um **comando de texto** à Alexa, como se você falasse com ela ("tocar música brasileira", "que horas são", "parar").

| Campo | O que faz |
|---|---|
| Dispositivo | o Echo que recebe o comando |
| Comando | o texto do comando. **Vazio** = usa o `msg.payload` |

Neste nó o Echo vem só do campo **Dispositivo** (`msg.device` não é lido).

- **Saída:** `{ ok, device, command }`.

### set status

Avisa quando a conta muda de estado e informa o diagnóstico. Só tem os campos **Nome** e **Conta** (não precisa de Dispositivo).

- **Saída automática:** a cada mudança de estado, `msg.payload` = `{ account, code, text, needsLogin }`. `needsLogin` fica `true` em `WAIT_PROXY`; ligue a saída a Telegram, WhatsApp ou e-mail para saber na hora que o login é necessário.
- **Entrada:** qualquer mensagem devolve o diagnóstico (estado, idade do token, última renovação, último erro, tentativas, próxima tentativa).

## Exemplos

- **Parar a Alexa:** um *inject* com o texto `parar` ligado a um `set comando` vazio.
- **Ajustar o volume:** um *inject* numérico `40` ligado a um `set volume` vazio.
- **Avisar por voz:** uma *function* com `msg.payload = "A porta abriu"` ligada a um `set text` vazio.
- **Abrir uma skill:** um `set skills` com a skill e o Echo escolhidos nas listas, ligado a um *inject*.
- **Saber quando o login caiu:** `set status` ligado a um nó de aviso (Telegram, WhatsApp ou e-mail).

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

## Desinstalar

Antes de remover o pacote, apague tudo que o usa:

1. Apague **todos os nós** `set ...` de todos os fluxos.
2. Apague a **conta Alexa salva**: menu, **Configuration nodes**, escolha a conta e exclua.
3. Faça o **Deploy**.
4. Menu, **Gerenciar paleta**, aba **Nós**, ache `alexa-neway-automacoes` e clique em **Remover**.

## Segurança

O login (token) fica no arquivo que você indicou em **Arquivo do token**. Quem tem acesso a essa pasta ou ao editor do Node-RED consegue controlar seus Echos. Proteja o administrador do Node-RED com senha se ele estiver exposto na rede, e não publique esse arquivo.
