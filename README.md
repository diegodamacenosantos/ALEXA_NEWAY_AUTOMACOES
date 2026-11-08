# alexa-neway-automacoes

Nós para o Node-RED que controlam os Echos de uma conta Amazon: abrir uma skill, falar um texto, enviar um comando de texto e ajustar o volume.

Versão atual: **1.6.0**. Neway Automações.

Aviso: a Amazon não tem API oficial para isso. O login e os comandos usam endereços internos da Amazon, que podem mudar sem aviso. Se algo parar de funcionar depois de um tempo, o pacote precisa de atualização.

Base: biblioteca `alexa-remote2`, na versão 8.0.6, já embutida no pacote (a instalação não baixa nada).

## Instalar

1. Baixe o arquivo `alexa-neway-automacoes-1.6.0.tgz` deste repositório.
2. No Node-RED, abra o menu, **Gerenciar paleta**, aba **Instalar**, e envie o arquivo `.tgz`. Se o seu Node-RED não aceitar o envio, copie o arquivo para o servidor e rode, na pasta de dados do Node-RED (a que tem o `package.json` do usuário):

```bash
npm install /caminho/alexa-neway-automacoes-1.6.0.tgz
```

3. Reinicie o Node-RED. Na paleta aparece o grupo **alexa-neway-automacoes** com 5 nós: `set skills`, `set text`, `set volume`, `set comando` e `set status`.

Se você tinha a versão antiga `node-red-contrib-alexa-plex`, remova-a antes (os tipos de nó são iguais e conflitam).

## Conta (configuração, feita uma vez)

Todos os nós usam uma **conta Alexa**. Ao adicionar o primeiro nó, clique no lápis do campo Conta e preencha:

| Campo | O que é |
|---|---|
| Nome | opcional |
| Região | preenche sozinha o Service Host, a Page e o Idioma (EUA, Brasil, Reino Unido, Alemanha) |
| Este IP | o IP ou nome do servidor do Node-RED, o mesmo que você usa no navegador |
| Porta | porta do login (padrão 3456). Se o Node-RED roda em Docker, essa porta precisa estar publicada |
| Arquivo do token | **obrigatório.** Caminho completo onde o login fica salvo, para os próximos serem automáticos (ex.: `/data/chave`). Precisa ficar em disco que sobrevive ao reinício (no Docker, um volume) |
| Renovar (dias) | de quanto em quanto tempo renova o login (padrão 3) |
| User Agent | opcional |

### Primeiro login

1. Faça o **deploy**.
2. O status dos nós mostra `login: http://IP:PORTA`.
3. Abra esse endereço no navegador **em http (não https)**, entre na sua conta Amazon e espere o status ficar **pronto** (verde).

O login fica salvo no arquivo do token. Depois disso ele renova sozinho.

### Como a conta se mantém de pé

- Renova o token sem derrubar a sessão: o cookie atual segue em uso até o novo estar confirmado.
- Se a internet estiver fora (por exemplo, o container sobe antes da rede), o estado fica `OFFLINE` e ela tenta de novo sozinha, com espera crescente (10 s até 5 min), sem limite.
- Só pede login (`WAIT_PROXY`) quando a Amazon recusa o token três vezes seguidas, ou se não existir token.
- Grava o token com segurança (arquivo temporário e troca) e guarda a cópia anterior em `<arquivo>.bak`, que é usada se o principal estragar.
- Confere a sessão a cada 5 minutos e renova ao detectar que caiu.
- Os comandos esperam até 30 s a conta voltar, e repetem uma vez se a Amazon responder erro de autenticação.
- Estados: `INIT`, `READY`, `OFFLINE`, `ERROR`, `WAIT_PROXY`.
- O log detalhado da biblioteca aparece no log do Node-RED quando o nível é *trace*.

## Os nós

Todos têm o campo **Nome** (opcional). Escolha a **Conta** e clique em **Atualizar lista** para carregar os Echos. O dispositivo é escolhido numa lista.

Regra comum, menos no `set skills`: se o campo do nó estiver **vazio**, o valor vem do `msg.payload`.

Todos devolvem em `msg.payload` um objeto com `ok: true` e os dados do que foi feito.

### set skills

Abre uma skill da Alexa no Echo.

- **Skill:** escolhida numa lista. A lista traz as skills da conta, inclusive as em desenvolvimento (marcadas "(dev)").
- **Dispositivo:** o Echo.
- **Entrada opcional:** `msg.skill` (id `amzn1...` ou o nome da skill) e `msg.device` (serial do Echo) sobrescrevem o que está no nó.
- **Saída:** `{ ok, skill, device }`.

### set text

Faz a Alexa **falar** um texto.

- **Texto:** o que ela fala. Se estiver vazio, fala o `msg.payload` (texto, número ou booleano).
- **Entrada opcional:** `msg.text` sobrescreve tudo; `msg.device` sobrescreve o Echo.
- **Saída:** `{ ok, device, text }`.

### set volume

Ajusta o volume do Echo, de 1 a 100.

- **Volume:** o valor do nó. Se estiver vazio, usa o `msg.payload` (número ou texto numérico, arredondado).
- **Entrada opcional:** `msg.volume` sobrescreve tudo; `msg.device` sobrescreve o Echo.
- **Erro:** valor fora de 1 a 100 (inclusive 0) ou que não seja número.
- **Saída:** `{ ok, device, volume }`.

### set comando

Envia um **comando de texto** à Alexa, como se você falasse com ela ("tocar música brasileira", "que horas são", "parar").

- **Comando:** o texto. Se estiver vazio, usa o `msg.payload`.
- **Saída:** `{ ok, device, command }`.

### set status

Avisa quando a conta muda de estado e informa o diagnóstico.

- **Saída automática:** a cada mudança de estado, `msg.payload` = `{ account, code, text, needsLogin }`. `needsLogin` fica `true` em `WAIT_PROXY`; ligue a saída a Telegram, WhatsApp ou e-mail para saber na hora que o login é necessário.
- **Entrada:** qualquer mensagem devolve o diagnóstico (estado, idade do token, última renovação, último erro, tentativas, próxima tentativa).

## Exemplos

- Um inject com o texto `parar` ligado a um `set comando` vazio: manda a Alexa parar.
- Um inject numérico `40` ligado a um `set volume` vazio: volume 40.
- Uma função que monta `msg.payload = "A porta abriu"` ligada a um `set text` vazio: a Alexa fala a frase.

## Erros comuns

| Mensagem | Causa |
|---|---|
| `conta nao pronta: WAIT_PROXY` | falta fazer o login (veja acima) |
| `conta nao pronta: OFFLINE` | sem internet; a conta tenta de novo sozinha |
| `dispositivo nao definido` | escolha o Echo no nó |
| `comando vazio` / `texto vazio` | preencha o nó ou envie `msg.payload` |
| `volume de 1 a 100` | valor inválido |
| `sem conta` | o nó não tem conta escolhida |

## Mudanças

- **1.6.0:** conta reescrita por dentro para ser estável (novas tentativas sem limite, renovação sem derrubar a sessão, token gravado com cópia, verificação de saúde, comandos que esperam e repetem); arquivo do token obrigatório; novo nó `set status`; biblioteca embutida e travada.
- **1.5.4:** o pacote e o grupo da paleta passam a se chamar `alexa-neway-automacoes`.
- **1.5.3:** nomes dos nós viram `set skills`, `set text`, `set volume` e `set comando`; o `set volume` passa a aceitar `msg.payload`.
- **1.5.2:** o `text command` passa a usar `msg.payload` quando o comando do nó está vazio.
