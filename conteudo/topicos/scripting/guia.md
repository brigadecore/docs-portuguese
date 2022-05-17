---
title: Guia de Scripting
description: Como criar e estruturar scripts no Brigade
section: scripting
weight: 1
aliases:
  - /guia.md
  - /topicos/guia.md
  - /topicos/scripting/guia.md
---

Este guia explica os fundamentos de como criar e estruturar scripts no Brigade,
que podem ser escritos em uma destas linguagens: JavaScript (`brigade.js`) ou
TypeScript (`brigade.ts`).

## Scripts, Projetos e Repositórios no Brigade

Na sua essência, o script do Brigade é simplesmente um arquivo JavaScript (ou TypeScript)
definido pelo projeto, que o Brigade executa no contexto de uma worker para manipular
eventos que o projeto subscreveu.

Os scripts do Brigade podem ser embutidos na definição do projeto ou em um git repository
que o projeto referencia. Nos vemos estes scripts como um tipo de orquestrador de shell scripts,
O script apenas conecta vários outros programas.

Este documento irá lhe guiar através de [exemplos de projetos] aonde os scripts vão
gradualmente aumentando de complexidade.

## Preparação

Para continuar, você irá precisar ter acesso ao servidor de API do Brigade e fazer o logon
usando a linha de comando CLI [brig]. Leia o [Início Rápido] se você ainda não fez isso.

Então, cada exemplo de projeto pode ser criado desta forma:

```shell
$ brig project create -f examples/<project>/project.yaml
```

Vamos começar!

[Início Rápido]: /topicos/introducao/iniciorapido
[Projeto]: /topicos/desenvolvedor-de-projetos/projetos
[`package.json`]: /topicos/scripting/dependencias
[Segredos]: /topicos/desenvolvedor-de-projetos/segredos
[Definição do Projeto]: /topicos/desenvolvedor-de-projetos/projetos#arquivos-de-definicao-do-projeto
[exemplos de projetos]: https://github.com/brigadecore/brigade/tree/main/examples
[brig]: /topicos/desenvolvedor-de-projetos/brig

## Um Script básico do Brigade

Este é um script básico `brigade.js` que apenas envia uma mensagem para o log:

```javascript
console.log("Hello, World!");
```
[01-hello-world](https://github.com/brigadecore/brigade/tree/main/examples/01-hello-world)

Primeiramente, vamos criar o projeto exemplo:

```plain
$ brig project create --file examples/01-hello-world/project.yaml

Created project "hello-world".
```

O projeto subscreve para um evento, o evento `exec` gerado pelo comando
`brig event create`. Os eventos que um projeto subscreve são configurados
na seção `eventSubscriptions` do arquivo de configuração(e.g. `project.yaml`).

Próximo passo, vamos disparar a execução do script denifido no projeto através
da criação de um evento deste tipo:

```plain
$ brig event create --project hello-world --follow

Created event "261229dc-1140-4f6a-bf91-bd2a69f31721".

Waiting for event's worker to be RUNNING...
2021-09-20T22:15:12.047Z INFO: brigade-worker version: a398ba8-dirty
Hello, World!
```

Esse exemplo _incondicionalmente_ loga "Hello, World!" como resposta para _qualquer_
evento que o projeto subscreveu(embora, como mencionado, o projeto apenas subscreveu
para um evento). É mais comum incorporar _manipuladores de eventos_ para manipular
eventos específicos de formas diferentes.

## Eventos do Brigade e Manipuladores de Eventos

Exemplos de eventos poderiam incluir: pushes para o GitHub ou para repositórios do DockerHub.
Quando um projeto subscreve para um evento, o worker criado para este projeto irá
ser carregado para executar o script do projeto. _Manipuladores de eventos_ no script
definem uma lógica específica para manipular os eventos correspondentes.

Este próximo script consiste de um manipulador de evento _especificamente_ para o evento
`exec` emitido pela linha de comando CLI `brig`. Se qualquer outro evento for gerado para
este projeto, nenhuma ação irá ocorrer como não existe uma lógica definida para manipular
este outro evento.

```javascript
const { events } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("Hello, World!");

  // Optionally return a string and it will automatically be persisted
  // in the event's summary field.
  return "All done."
});

events.process();
```
[02-first-event](https://github.com/brigadecore/brigade/tree/main/examples/02-first-event)

Existem várias coisas para se entender neste script:

- Importa o objeto `events` da biblioteca `@brigadecore/brigadier`. Quase todos
  scripts do Brigade fazem isso.
- Declara um único manipulador de evento que fala "Quando um evento `exec` acontecer, rode
  esta função".
- Não estamos utlizando o objeto `event` neste script, mas nos iremos utilizá-lo
  em um exemplo futuro.
- O manipulador de evento retorna uma string, a qual é opcional [sumário do evento].
  Isto pode ser utilizado pelos gateways interessados em resultados ou outro contextos
  relacionados com o manipulador de eventos. O sumário apenas tem significado para o
  script e possivelmente o gateway -- caso contrário, isto é opaco para o próprio Brigade.
- Scripts com manipuladores de eventos definidos, como este aqui, precisa chamar
  `events.process()` _depois_ de registrar todos manipuladores para que todos eventos sejam
  processados.

De forma similar ao nosso primeiro script, esta função de manipulador de evento exibe uma
mensagem para o log, prodizindo a seguinte saída:

```plain
$ brig event create --project first-event --follow

Created event "5b0bd00a-4f31-40da-ad01-0d2f62f4d70e".

Waiting for event's worker to be RUNNING...
2021-09-20T22:24:57.655Z INFO: brigade-worker version: a398ba8-dirty
Hello, World!
```

> Nota: o sucesso/fracasso de um evento é determinado pelo código de saída do
> Worker executando o script. Se o script falhar ou apresentar um erro, o Worker
> irá suspender(exit) execução com um erro que não seja zero e seja considerado
> como uma falha.

Como os projetos podem subscrever para vários tipos de eventos(oriundos de várias
fontes de código), definindo vários manipuladores de eventos nos seus scripts
permite tipos de eventos diferentes serem manipulados de forma diferente.

Por exemplo, podemos expandir o exemplo acima para também prover um manipulador
para o evento push do GitHub:


```javascript
const { events } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("==> handling an 'exec' event")
});

events.on("brigade.sh/github", "push", async event => {
  console.log(" **** I'm a GitHub 'push' handler")
});
```

Agora, se executarmos novamente nosso comando `brig`, veremos a mesma saída que vimos anteriormente:

```
$ brig event create --project first-event --follow

Created event "6769b7fd-ddd0-4a42-aa17-723c02a2b61a".

Waiting for event's worker to be RUNNING...
2021-09-20T22:29:28.498Z INFO: brigade-worker version: a398ba8-dirty
Hello, World!
```

Desde que o evento que criamos foi oriundo da `brigade.sh/cli` e do tipo `exec`, apenas
os manipuladores de eventos correpondentes foram executados. O manipulador para eventos
oriundos do `brigade.sh/github` e do tipo `push` não foi executado.

Para mais informações leia a documentação dos [Eventos] e entenda como eventos são estruturados
no Brigade.

[Eventos]: /topicos/desenvolvedor-de-projetos/eventos
[sumário do evento]: /topicos/desenvolvedor-de-projetos/eventos#sumario

## Jobs e Containers

A imagem docker do worker é derivada da imagem do Node.js e os containers baseados
nesta imagem podem executar JavaScript e TypeScript apenas. Se outros runtimes, tools,
etc forem necessários para responder ao evento e não estiverem presentes na imagem do worker,
a criação de um _Job_ permite partes da manipulação do evento serem delegadas para um
container mais apropriado e com uma imagem docker diferente do worker e definida no script.

O blibioteca [Brigadier] expõe a interface para definir e executar Jobs.

Na seção anterior, focamos em manipuladores de eventos que apenas logam mensagens. Nesta
seção iremos atualizar o manipulador de evento para criar vários jobs.

Para começar vamos criar um Job simples que realmente não faz nada:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job = new Job("do-nothing", "debian", event);
  await job.run();
});

events.process();
```

A primeira coisa para perceber aqui é que _nos alteramos a primeira linha_. Além de
importar o objeto `events`, nos também estamos importando `Job`. `Job` é a classe
usada para criar todos os jobs no Brigade.

Em Seguida, dentro do nosso manipulador de evento `exec`, nos criamos um novo `Job`
e passamos 3 parâmetros:

- O _name_: O nome precisa ser único para todos os manipuladores de eventos e precisa
  ser composto de letras, números e dashes (`-`).
- A _image_: Isso pode ser _qualquer imagem que seu cluster possa encontrar_. No caso acima
  nos usamos a imagem chamada `debian`, que é carregada
  [diretamente do DockerHub](https://hub.docker.com/_/debian/).
- O próprio _event_ que este manipulador foi criado para processar

O Job é uma parte crucial do ecossistema do Brigade. Um container é criado utilizando
a imagem docker definida pelo Job, e neste container é aonde realmente executamos e
processamos o evento. No começo nos explicamos que vemos os scripts do Brigade como
"shell scripts para o cluster". Quando você executa um shell script, existe normalmente
algum código que faz a chamada de um ou mais programas externos de uma forma específica.

```shell
#!/bin/bash

ps -ef "hello" | grep chrome
```

O script acima apenas organiza a maneira que nos chamamos dois programas pré existentes
(`ps` e `grep`). Brigade faz a mesma coisa, com excessão que ao invés de executar
_programas_, ele executa _containers_. Cada container é interpretado como um Job,
que é um wrapper que entende como executar containers.

No nosso caso acima, criamos um Job chamado "do-nothing" que roda um container Debian
Linux e, como o nome sugere, não faz nada.

Jobs são criados e executados em passos diferentes. Desta forma podemos fazer algumas
configurações no nosso job(como veremos em seguida) antes de executá-lo.

### Job.run()

Para rodar o job usamos o método `run()` da classe Job. Durante a execução deste comando,
Brigade cria um novo `debian` container (carregando a imagem docker se for necessário),
inicia o container e monitora. Quando o container termina sua execução, o comando "run" é
finalizado.

É importante mencionar que o comando `run()` é _assíncrono_ e na prática é apenas uma
_chamada_ para o servidor de API do Brigade pedindo para agendar a execução do job.
Essa execução depende das restrições do Agendador, como por exemplo configurações do 
número máximo de Jobs concorrentes permitidos, a capacidade do ambiente de execução(cluster kubernetes),
etc.

Além disso, perceba que o comando `run()` retorna imediatamente depois de agendar e a palavra chave `await`
pode(e normalmente deveria) ser usada para bloquear outras execuções de script até que o job tenha terminado
sua execução e esteja finalizado.

Se rodarmos o código abaixo, teremos uma saída parecida com isso:

```plain
$ brig event create --project first-job --follow

Created event "aa8fff14-0b8d-4903-9109-ccadc1d9d3fe".

Waiting for event's worker to be RUNNING...
2021-09-22T18:41:01.787Z INFO: brigade-worker version: 927850b-dirty
2021-09-22T18:41:02.130Z [job: do-nothing] INFO: Creating job do-nothing
```

Basicamente, nossa execução apenas criou um pod Linux Debian vazio que não tem nada para
fazer e simplesmente finalizou imediatamente.

[Brigadier]: /topicos/scripting/brigadier

### Adicionando Comandos nos Jobs

Para que nossos Jobs possam fazer algo a mais, podemos adicionar um _comando_ neles.
O comando pode então ser associado com uma lista de _argumentos_. O comando e sua
lista de argumentos são adicionadas ao _primaryContainer_, que é o container rodando
a imagem docker fornecida para o contrutor do `Job`, por exemplo:  `debian` (Todo job terá
um `primaryContainer` e zero ou mais `sidecarContainers`. Vamos entender melhor sobre
`sidecarContainers` em outro exemplo mais a frente.)

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job = new Job("my-first-job", "debian", event);
  job.primaryContainer.command = ["echo"];
  job.primaryContainer.arguments = ["My first job!"];
  await job.run();
});

events.process();
```
[03-first-job](https://github.com/brigadecore/brigade/tree/main/examples/03-first-job)

Um comando pode ser qualquer coisa que a imagem docker do Job suporte. No exemplo
acima, nosso comando chama o binário `echo` e fornece argumentos.

Para múltiplos comandos, uma abordagem é aonde o array do `command` é composto
de um elemento como `["/bin/sh"]` ou `["/bin/bash"]` e o primeiro elemento do
array de `arguments` é `"-c"`, seguidos por entradas compreendendo outros comandos
shell necessários.

Entretanto, quando a contrução destes comandos se tornarem complicadas e complexas,
recomendamos escrever um shell script e torná-lo disponível para ser executado
(exemplo: colocando este shell script em um repositório git associado com o projeto
ou adicionando o script em uma [imagem docker customizada do Worker]).

Vamos rodar este exemplo:

```plain
$ brig event create --project first-job --follow

Created event "046c09cd-76cb-49ea-b40c-d3e0e557de62".

Waiting for event's worker to be RUNNING...
2021-09-22T20:11:10.453Z INFO: brigade-worker version: 927850b-dirty
2021-09-22T20:11:10.909Z [job: my-first-job] INFO: Creating job my-first-job
```

Agora, para visualizar os logs do `my-first-job` precisamos executar o seguinte comando
utilizando o ID gerado pelo evento.

```plain
$ brig event logs --id 046c09cd-76cb-49ea-b40c-d3e0e557de62 --job my-first-job

My first job!
```

> Nota: `job.run()` irá criar um exceção se o job falhar. Adicionalmente, o
> sucesso/fracasso do job é determinado pelo código de saída do `primaryContainer`
> job.

[imagem docker customizada do Worker]: /topicos/scripting/workers

### Combinando jobs em uma pipeline

Agora podemos ir para o próximo passo e criar _dois jobs_ que individualmente fazem alguma coisa.

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job1 = new Job("my-first-job", "debian", event);
  job1.primaryContainer.command = ["echo"];
  job1.primaryContainer.arguments = ["My first job!"];
  await job1.run();

  let job2 = new Job("my-second-job", "debian", event);
  job2.primaryContainer.command = ["echo"];
  job2.primaryContainer.arguments = ["My second job!"];
  await job2.run();
});

events.process();
```
[04-simple-pipeline](https://github.com/brigadecore/brigade/tree/main/examples/04-simple-pipeline)

Neste exemplo criamos dois jobs (`my-first-job` e `my-second-job`). Cada job
inicia um container Debian, imprime uma mensagem e finaliza. Como `job1` usa
`await`, `job2` não irá executar até que o `job1` finalize, então estes jobs
_implicitamente_ estão rodando sequencialmente.

Vamos rodar o exemplo e então visualizar os logs de cada job:

```
$ brig event create --project simple-pipeline

Created event "58e7d3cf-b7d2-4ab7-98ad-326a99f10a25".

$ brig event logs --id 58e7d3cf-b7d2-4ab7-98ad-326a99f10a25 --job my-first-job

My first job!

$ brig event logs --id 58e7d3cf-b7d2-4ab7-98ad-326a99f10a25 --job my-second-job

My second job!
```

## Grupos de Job Serial e Concurrent

Agora que vimos um exemplo de projeto que roda múltiplos jobs, vamos dar uma
olhada nos métodos que temos para especificar _como_ os jobs rodam,
exemplo: sequencialmente ou simultaneamente -- ou, como veremos em seguida,
uma combinação disso.

Por exemplo, podemos rodar duas sequências de jobs simultaneamente:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job1 = new Job("my-first-job", "debian", event);
  job1.primaryContainer.command = ["echo"];
  job1.primaryContainer.arguments = ["My first job!"];

  let job2 = new Job("my-second-job", "debian", event);
  job2.primaryContainer.command = ["echo"];
  job2.primaryContainer.arguments = ["My second job!"];
  
  let jobA = new Job("my-a-job", "debian", event);
  jobA.primaryContainer.command = ["echo"];
  jobA.primaryContainer.arguments = ["My A job!"];

  let jobB = new Job("my-b-job", "debian", event);
  jobB.primaryContainer.command = ["echo"];
  jobB.primaryContainer.arguments = ["My B job!"];

  await Job.concurrent(
    Job.sequence(job1, job2),
    Job.sequence(jobA, jobB)
  ).run();
});

events.process();
```
[05-groups](https://github.com/brigadecore/brigade/tree/main/examples/05-groups)

Existem duas coisas para se perceber neste exemplo acima:

1. Ambos métodos `concurrent` e `sequence` existem no objeto `Job`.
2. O tipo de retorno para ambos os métodos é genérico "runnable", exemplo: um objeto
   `run()` pode então ser chamado, da mesma forma que uma instância standard do Job.

Em seguida temos o descritivo de cada método:

- `Job.sequence()` utiliza um array de objetos runnables (exemplo: um job ou um grupo
  de jobs) e executa cada um _em sequência_. Um novo objeto runnable é iniciado apenas quando
  o anterior finalizar. A sequência completa quando o último objeto runnable tiver finalizado
  (ou quando qualquer objeto runnable falhar). Um grupo sequencial se considera finalizado com
  sucesso ou com falha se todos os jobs foram bem sucessidos ou não.
- `Job.concurrent()` utiliza um array de objetos runnables e executa cada um
  _simultaneamente_. Quando a execução é iniciada, todos os objetos runnables são iniciados
  simultaneamente (dependendo das restrições do agendador). O grupo simultâneo finaliza quando
  todos os objetos runnables finalizarem. Um grupo simultâne se considera finalizado com sucesso
  ou falha se todos os jobs foram bem sucessidos ou não.

Como ambos os métodos retornam um objeto runnable, eles podem ser amarrados. Neste exemplo acima,
`job1` e `job2` rodam _em sequência_, da mesma forma que `jobA` e `jobB`, mas ambas
sequências rodam _simultaneamente_.

Esta é a maneira como desenvolvedores de script podem controlar a ordem na qual os grupos de jobs
são executadas.

Por exemplo, se você utilizar Brigade para executar uma CI(integração contínua), você poderia desejar
dividir verificações em aquelas que requerem mais recursos para serem processadas(exemplo: compilações demoradas,
testes de integração) e aquelas verificações que requerem menos recrusos para serem processadas como(exemplo: linting, teste unitário). Ambos os grupos podem executar os jobs simultaneamente, mas os grupos rodam de forma
sequencial eles mesmos, de forma que nenhuma verificação que necessita de mais recursos para serem processadas
sejam executadas até todas as verificações que requerem menos recursos tenham sido processadas.

### Rodando um script a partir de um repositório Git

Anteriomente discutimos como um projeto poderia ter um repositório Git associado.
Vamos analisar um destes projetos agora.

```yaml
apiVersion: brigade.sh/v2
kind: Project
metadata:
  id: git
description: A project with whose script is stored in a git repository
spec:
  eventSubscriptions:
  - source: brigade.sh/cli
    types:
    - exec
  workerTemplate:
    # logLevel: DEBUG
    configFilesDirectory: examples/06-git/.brigade
    git:
      cloneURL: https://github.com/brigadecore/brigade.git
      ref: refs/heads/main

```
[06-git](https://github.com/brigadecore/brigade/tree/main/examples/06-git)

Perceba que não existe script embutido nesta definição de projeto. De preferência, o
projeto especifica a localização do diretório do arquivo de configuração do Brigade
(`configFilesDirectory`) dentro do repositório configurado na seção `git`.
(Se o diretório não for fornecido, a localização padrão será o diretório `.brigade`
na raiz do repositório Git).

O diretório de configuração é aonde o script do Brigade é armazenado. Neste exemplo,
o script `brigade.js` é simplesmente uma instrução `console.log()`.

```javascript
console.log("Hello, Git!");
```

Vamos rodar o exemplo:

```plain
$ brig event create --project git --follow

Created event "5ff386ed-060e-49fa-8292-0bade75f8840".

Waiting for event's worker to be RUNNING...
2021-09-22T21:38:10.667Z INFO: brigade-worker version: 927850b-dirty
Hello, Git!
```

Vamos discutir aqui o que acontece nos bastidores do Brigade quando criamos
um evento para este projeto: Como o projeto tem um repositório Git associado,
Brigade automaticamente começa a buscar/clonar o repositório Git, o qual fica
disponível para o worker que esta processando o script.

Por padrão, o conteúdo do repositório não é automaticamente montado nos jobs criados
no script do projeto. Entretanto, você pode montar este conteúdo no job de uma forma muito
simples, usando a configuração `sourceMountPath` do `primaryContainer` do Job.

O próximo exemplo mostra como um job pode ser configurado para acessar o repositório
de forma que possamos executar um teste. Este exemplo também configura `workingDirectory`
com os mesmo valor que `sourceMountPath` para que o job não precise se preocupar em
alterar para o diretório apropriado antes de executar comandos:

```javascript
const { events, Job } = require("@brigadecore/brigadier");
const localPath = "/workspaces/brigade";

events.on("brigade.sh/github", "push", async event => {
  let test = new Job("test", "debian", event);
  test.primaryContainer.sourceMountPath = localPath;
  test.primaryContainer.workingDirectory = localPath;
  test.primaryContainer.command = ["bash"];
  test.primaryContainer.arguments = ["-c", "make test"];
  await test.run();
})

events.process();
```

Poder associar um repositório Git com um projeto é uma maneira conviniente de
prover controle de versão para os scripts do Brigade. Por Exemplo, en vez de
embutir o script e outras configurações na definição do projeto, estes arquivos
podem ser carregados do controle de código(Git).

Adicionalmente, esta funcionalidade faz o Brigade uma ótima ferramenta para
executar CI pipelines, implantações, tarefas de empacotamento, testes de ponta a ponta e
outras tarefas de DevOps para um dado repositório.


## Trabalhando com Evento e Dados do Projeto

Como vimos, o objeto evento é sempre passado para o manipulador de evento. Esse
objeto também inclui dados do projeto. Vamos olhar nos dados que temos acesso.

### O evento do Brigade

A partir do evento podemos descobrir o que disparou o evento, quais dados foram
enviados, os detalhes do worker responsável por executar o evento e muito mais.

Aqui estão alguns campos notáveis do objeto do evento:

- `id` é um ID único por evento. Toda vez que um novo evento é disparado, um novo ID
  é gerado.
- `project` é o projeto que foi registrado para manipular esse evento. Iremos olhar
  nos campos acessíveis neste objeto abaixo.
- `source` é a fonte do evento. O comando `brig event create` por exemplo,
  definiria fonte para `brigade.sh/cli`.
- `type` é o tipo do evento. Um evento de Pull Request do GitHub, por exemplo, definiria
  o tipo para `pull_request`.
- `payload` restrições em qualquer informação que o serviço externo enviou quando
  disparando o evento. Por exemplo, um comando push num repositório GitHub gera
  um [evento push do GitHub]. Payload é uma unparsed string.
- `worker` é o worker do Brigade atribuído para manipular o evento. Entre outras
  coisas, detalhes do git como o commit(ID da revisão) e a URL para clonar podem ser
  encontrados neste objeto.

Para um visão geral do objeto do evento fornecido pela biblioteca brigadier, leia
the [Brigadier] documentation.

[evento push do GitHub]: https://developer.github.com/v3/activity/events/types/#pushevent

### O Projeto

O objeto (`event.project`) nos dá os seguintes campos:

- `id` é o ID do projeto.
- `secrets` é o key/value map dos segredos definidos no projeto. Eles são
  definidos através do comando `brig secret set` (mais informações em [Guia dos Segredos] for more info).

[Guia dos Segredos]: /topicos/desenvolvedor-de-projetos/segredos

### Usando Objetos dos Eventos e Projetos

Vamos olhar para alguns exemplos que utilizam dados do evento e do projeto.

O primeiro exemplo extrai o payload do objeto do evento e imprime no console:

```javascript
const { events } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("Hello, " + event.payload + "!");
});

events.process();
```
[07-first-payload](https://github.com/brigadecore/brigade/tree/main/examples/07-first-payload)

Quando criamos um evento com payload no script acima, veremos uma saída assim:

```plain
$ brig event create --project first-payload --payload "Brigade" --follow

Created event "05e31d97-945b-4727-b710-7d983d137d40".

Waiting for event's worker to be RUNNING...
2021-09-23T16:25:29.761Z INFO: brigade-worker version: 927850b-dirty
Hello, Brigade!
```

Podemos atualizar o exemplo para imprimir o ID do projeto também.

```javascript
const { events } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("Project: " + event.project.id);
  console.log("Hello, " + event.payload + "!");
});

events.process();
```

A seguinte saída é gerada:

```plain
$ brig event create --project first-payload --payload "Brigade" --follow

Created event "b45720c4-115c-4c9b-b668-a872479f2210".

Waiting for event's worker to be RUNNING...
2021-09-23T16:30:29.267Z INFO: brigade-worker version: 927850b-dirty
Project: first-payload
Hello, Brigade!
```

Note que alguns dados dos eventos e projetos deveriam ser tratados com cuidado.
Coisas do tipo  `event.project.secrets` ou `event.worker.git.cloneURL` não podem
ser o tipo de informação que você, por acidente, mostre ou imprima. Por exemplo,
confira o [Guia de Segredos] em como manipular os segredos utilizados em seus scripts
de forma segura.

[Guia de Segredos]: /topicos/desenvolvedor-de-projetos/segredos

## Armazenamento no Worker e Workspace compartilhado

Brigade oferece a abilidade para configurar armazenamento para o worker que pode
então compartilhá-lo com os jobs. Esta funcionalidade não é abilitada como padrão
e precisa ser configurada na seção `workerTemplate` da definição do projeto, bem como
em cada job no qual o script precise ter acesso ao workspace.

### Um exemplo para demonstrar o uso de Workspaces compartilhados

Primeiramente, o campo `useWorkspace` no `workerTemplate` da definição
do projeto deve ser definido para `true`:

```yaml
workerTemplate:
  useWorkspace: true
```

Em seguida, para cada job usar o workspace compartilhado, forneça um valor
para o campo `workspaceMountPath` no `primaryContainer` do job:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job1 = new Job("first-job", "debian", event);
  job1.primaryContainer.workspaceMountPath = "/share";
  job1.primaryContainer.command = ["bash"];
  job1.primaryContainer.arguments = ["-c", "echo 'Hello!' > /share/message"];
  await job1.run();

  let job2 = new Job("second-job", "debian", event);
  job2.primaryContainer.workspaceMountPath = "/share";
  job2.primaryContainer.command = ["cat"];
  job2.primaryContainer.arguments = ["/share/message"];
  await job2.run();
});

events.process();
```
[08-shared-workspace](https://github.com/brigadecore/brigade/tree/main/examples/08-shared-workspace)

```plain
$ brig event create --project shared-workspace

Created event "2eee9044-4469-49bd-a58b-aa659951a502".

$ brig event logs --id 2eee9044-4469-49bd-a58b-aa659951a502 --job second-job

Hello!
```

> Nota: Armazenamento compartilhado depende do seu cluster Kubernetes e a
> disponibilidade de uma tipo correto de "storage class". Leia a página
> [Armazenamento] da documetação para mais informações sobre requerimentos
> do cluster.

[Armazenamento]: /topicos/operadores/armazenamento

## Sidecar containers

Jobs podem opcionalmente serem configurados com um ou mais sidecar containers,
que rodam junto ao container primário do job. Todos os sidecar containers serão
finalizados pouco tempo depois que o container primário do job finalize. Algumas
notas adicionais:

- Independentemente se o sidecar containers exista, o sucesso ou fracasso do job
  continua sendo determinado exclusivamente pelo código de saída de seu container primário.
- Todos os containers rodam na mesma "local network interface", possibilitando os processos escutarem
  conexões de rede em qualquer um deles para fazer seu processamento.
- Brigade não entende do relacionamento entre seus containers e portanto não
  conseguem coordenar o seu inicio. Se, por exemplo, um processo no container
  primário deva aguardar até que algum outro processo suplementar num sidecar
  container esteja iniciado, rodando e escutando por conexões. Então os autores
  de script precisam levar em consideração esses fatores.

Como exemplo, vamos considerar um manipulador de evento que necessite executar
testes, mas também precisa prover um banco de dados necessário para os testes.
O Banco de dados poderia ser executado como um sidecar container enquanto o
container primário do job se preocupa em rodar os testes.

Outro caso de uso é um Job que necessita de uma Docker daemon. A maneira mais segura
de fazer isso é rodando Docker-em-Docker (DinD), por exemplo: iniciando o daemon em um
container que possa então ser usado como necessário. Um sidecar container é a perfeita
aplicação para se iniciar o daemon desejado enquanto o container primário do job manipula
as tarefas em mãos.

Vamos dar uma olhada neste último exemplo e introduzir configurações adicionais
necessárias para este cenário.

### Docker-em-Docker

Containers em Docker-em-Docker (DinD) precisam executar como "privilegiado" para que
funcionem apropriadamente. Eles também necessitam ser autorizados a nível de projeto
para que os jobs rodarem como "privilegiados". O padrão é este comportamento não ser
permitido ou aprovado.

Este é um exemplo de configuração de projeto que permite containers rodarem em modo
privilegiado:

```yaml
workerTemplate:
  jobPolicies:
    allowPrivileged: true
```

Cada container do job precisando executar neste modo precisa também adicionar uma
configuração explícita. No exemplo abaixo, o sidecar container `docker` possui o
campo `privileged` configurado para "true", enquanto o container primário não possui:

```javascript
const { events, Job, Container } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job = new Job("dind", "docker:stable-dind", event);
  job.primaryContainer.environment.DOCKER_HOST = "localhost:2375";
  job.primaryContainer.command = ["sh"];
  job.primaryContainer.arguments = [
    "-c",
    // Wait for the Docker daemon to start up
    // And then pull the image
    "sleep 20 && docker pull busybox"
  ];

  // Run the Docker daemon in a sidecar container
  job.sidecarContainers = {
    "docker": new Container("docker:stable-dind")
  };
  job.sidecarContainers.docker.privileged = true

  await job.run();
});

events.process();
```

Esta é a saída da criação de um evento e depois chamamos o comando de visualizalção dos logs do job:

```plain
$ brig event create --project dind --follow

Created event "94d0fcd5-61dc-49be-bb81-3e5784e66a4a".

Waiting for event's worker to be RUNNING...
2021-09-23T19:00:08.915Z INFO: brigade-worker version: 927850b-dirty
2021-09-23T19:00:09.428Z [job: dind] INFO: Creating job dind

$ brig event logs --id 94d0fcd5-61dc-49be-bb81-3e5784e66a4a --job dind

Using default tag: latest
latest: Pulling from library/busybox
24fb2886d6f6: Pulling fs layer
24fb2886d6f6: Verifying Checksum
24fb2886d6f6: Download complete
24fb2886d6f6: Pull complete
Digest: sha256:52f73a0a43a16cf37cd0720c90887ce972fe60ee06a687ee71fb93a7ca601df7
Status: Downloaded newer image for busybox:latest
docker.io/library/busybox:latest
```

Perceba que também podemos observar os logs do sidecar container do job:

```plain
$ brig event logs --id 94d0fcd5-61dc-49be-bb81-3e5784e66a4a --job dind --container docker

time="2021-09-23T19:00:11.230907700Z" level=info msg="Starting up"
...
```

### Acessando o socket do host do Docker

Por questões de segurança, é recomendando que você utilize Docker-em-Docker (DinD)
ao invés de usar o socket do Docker diretamente.

Entretanto, cada job também tem a opção de montar o socket do docker. Quando
abilitado, o socket do docker presente no Node do Kubernetes que esta rodando o Pod
do Job é montado no diretório do container do job chamado `/var/run/docker.sock`.
Isto é normalmente apenas requerido para "Docker-out-of-Docker" ("DooD") cenários
aonde o container precisa usar o Docker daemon do Host. Esta prática é altamente
desencorajada para quase todos os casos de uso.

Para que o socket seja montado, o projeto do Brigade deve ter o campo `allowDockerSocket`
da seção `jobPolicies` da especicação do worker definida como `true`. O padrão é  `false`,
não permitindo o uso do socket do docker host.

Este exemplo de configuração de projeto abilita este recurso:

```yaml
workerTemplate:
  jobPolicies:
    allowDockerSocketMount: true
```

Adicionalmente, o job deve declarar sua necessidade de um socket docker alterando
a configuração `useHostDockerSocket` no seu `primaryContainer` para "true":

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job = new Job("dood", "docker", event);
  job.primaryContainer.useHostDockerSocket = true;
  job.primaryContainer.command = ["docker"];
  job.primaryContainer.arguments = ["ps"];
  await job.run();
});

events.process();
```

Esta é a saída quando criamos um evento para o script acima:

```plain
$ brig event create --project dood --follow

Created event "283be00c-5481-43ae-8634-bd9bd194488b".

Waiting for event's worker to be RUNNING...
2021-09-23T19:50:27.796Z INFO: brigade-worker version: cfa7e5e-dirty
2021-09-23T19:50:27.994Z [job: dood] INFO: Creating job dood

$ brig event logs --id 283be00c-5481-43ae-8634-bd9bd194488b --job dood

CONTAINER ID   IMAGE                         COMMAND                  CREATED          STATUS                  PORTS     NAMES
68ab8fa3f395   docker                        "docker ps"              1 second ago     Up Less than a second             k8s_dood_283be00c-5481-43ae-8634-bd9bd194488b-dood_brigade-2450aa5d-80be-442b-bdd2-425a4d85c3e9_e2bb2688-8362-4824-91a0-797d0396ba04_0
...
```

> Nota: Nem todos os provedores de cluster Kubernetes usam Docker como seu "container runtime".
> Por exemplo, [KinD] utiliza [containerd] e o socket do Docker não esta disponível. Novamente
> informamos que Docker-em-Docker DinD é a forma preferida quando o socket é necessário.

[KinD]: https://kind.sigs.k8s.io/
[containerd]: https://containerd.io/

## Conclusão

Este guia cobre o básico de como escrever scripts do Brigade. Veja aqui alguns links
adicionais para leitura:

* Se você deseja mais detalhes sobre a interface Brigadier JS/TS API, dê uma olhada na
  documentação do [Brigadier].
* Para exemplos de scripts e técnicas mais avançadas, leia a documentação
  [Guia de Scripting Avançado]
* Examine os outros exemplos de projetos/scripts no diretório de [Exemplos]. Existem
  exemplos de projetos usando npm, yarn, TypeScript e muito mais.

Felicidades Scripting!

[Guia de Scripting Avançado]: /topicos/scripting/avancado
[Exemplos]: https://github.com/brigadecore/brigade/tree/main/examples