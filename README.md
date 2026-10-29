# alexa-neway-automacoes

Nós para o Node-RED que controlam os Echos de uma conta Amazon: abrir uma skill, falar um texto, enviar um comando de texto e ajustar o volume.

Versão atual: **1.5.4**. Neway Automações.

Aviso: a Amazon não tem API oficial para isso. O login e os comandos usam endereços internos da Amazon, que podem mudar sem aviso. Se algo parar de funcionar depois de um tempo, o pacote precisa de atualização.

Base: biblioteca `alexa-remote2` (única dependência).

## Instalar

1. Baixe o arquivo `alexa-neway-automacoes-1.5.4.tgz` deste repositório.
2. No Node-RED, abra o menu, **Gerenciar paleta**, aba **Instalar**, e envie o arquivo `.tgz`. Se o seu Node-RED não aceitar o envio, copie o arquivo para o servidor e rode, na pasta de dados do Node-RED (a que tem o `package.json` do usuário):

```bash
npm install /caminho/alexa-neway-automacoes-1.5.4.tgz
```

3. Reinicie o Node-RED. Na paleta aparece o grupo **alexa-neway-automacoes** com 4 nós: `set skills`, `set text`, `set volume` e `set comando`.

Se você tinha a versão antiga `node-red-contrib-alexa-plex`, remova-a antes (os tipos de nó são iguais e conflitam).

## Conta (configuração, feita uma vez)

Todos os nós usam uma **conta Alexa**. Ao adicionar o primeiro nó, clique no lápis do campo Conta e preencha:

| Campo | O que é |
|---|---|
| Nome | opcional |
| Região | preenche sozinha o Service Host, a Page e o Idioma (EUA, Brasil, Reino Unido, Alemanha) |
| Este IP | o IP ou nome do servidor do Node-RED, o mesmo que você usa no navegador |
| Porta | porta do login (padrão 3456). Se o Node-RED roda em Docker, essa porta precisa estar publicada |
| Arquivo do token | caminho completo onde o login fica salvo, para os próximos serem automáticos. Vazio = pasta do Node-RED |
| Renovar (dias) | de quanto em quanto tempo renova o login (padrão 3) |
| User Agent | opcional |

### Primeiro login

1. Faça o **deploy**.
2. O status dos nós mostra `login: http://IP:PORTA`.
3. Abra esse endereço no navegador **em http (não https)**, entre na sua conta Amazon e espere o status ficar **pronto** (verde).

O login fica salvo no arquivo do token. Depois disso ele renova sozinho.

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

## Exemplos

- Um inject com o texto `parar` ligado a um `set comando` vazio: manda a Alexa parar.
- Um inject numérico `40` ligado a um `set volume` vazio: volume 40.
- Uma função que monta `msg.payload = "A porta abriu"` ligada a um `set text` vazio: a Alexa fala a frase.

## Erros comuns

| Mensagem | Causa |
|---|---|
| `conta nao pronta: WAIT_PROXY` | falta fazer o primeiro login (veja acima) |
| `dispositivo nao definido` | escolha o Echo no nó |
| `comando vazio` / `texto vazio` | preencha o nó ou envie `msg.payload` |
| `volume de 1 a 100` | valor inválido |
| `sem conta` | o nó não tem conta escolhida |

## Mudanças

- **1.5.4:** o pacote e o grupo da paleta passam a se chamar `alexa-neway-automacoes`.
- **1.5.3:** nomes dos nós viram `set skills`, `set text`, `set volume` e `set comando`; o `set volume` passa a aceitar `msg.payload`.
- **1.5.2:** o `text command` passa a usar `msg.payload` quando o comando do nó está vazio.
