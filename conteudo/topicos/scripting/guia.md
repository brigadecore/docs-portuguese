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
os quais podem ser escritos em uma destas linguagens: JavaScript (`brigade.js`) 
ou TypeScript (`brigade.ts`).

## Scripts, Projetos e Repositórios no Brigade

Na sua essência, o script do Brigade é simplesmente um arquivo JavaScript (ou TypeScript)
definido pelo projeto, o qual Brigade executa no contexto de uma worker para manipular
eventos que o projeto subscreveu.

Os scripts do Brigade podem ser armazenados na definição do projeto ou git repository
que o projeto referencia. Nos vemos estes scripts como um tipo de shell scripts orchestrador:
O script apenas conecta vários outros programas.

Este documento irá lhe guiar através de [exemplos de projetos], aonde os scripts 
vão gradualmente aumentar de complexidade.

## Preparação

Para continuar, você irá precisar ter acesso ao servidor de API do Brigade e logar nele
usando a linha de comando CLI [brig]. Veja o [Início Rápido] se você ainda não fez isso.

Então, cada exemplo de projeto pode ser criado desta forma:

```shell
$ brig project create -f examples/<project>/project.yaml
```

Vamos começar!

[Início Rápido]: /topicos/intro/quickstart
[Projeto]: /topicos/project-developers/projects
[`package.json`]: /topicos/scripting/dependencias
[Segredos]: /topicos/project-developers/secrets
[Definição do Projeto]: /topicos/project-developers/projects#project-definition-files
[exemplos de projetos]: https://github.com/brigadecore/brigade/tree/main/examples
[brig]: /topicos/project-developers/brig

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

O projeto subscreve para um evento, o `exec` evento gerado pelo comando
`brig event create`. Os eventos que um projeto subscreve são configurados
na secão `eventSubscriptions` do arquivo de configuração(e.g. `project.yaml`).

Próximo passo, vamos disparar a execução do script denifido do projeto através
da criação de um evento deste tipo:

```plain
$ brig event create --project hello-world --follow

Created event "261229dc-1140-4f6a-bf91-bd2a69f31721".

Waiting for event's worker to be RUNNING...
2021-09-20T22:15:12.047Z INFO: brigade-worker version: a398ba8-dirty
Hello, World!
```

Esse exemplo _incondicionalmente_ loga "Hello, World!" como resposta para _qualquer_
evento que o projeto subscreveu(embora, como mencionado, o projeto apenas subescreveu
para um evento). É mais comum incorporar _manipuladores de eventos_ para manipular
eventos específicos de formas diferentes.

## Eventos do Brigade e Manipuladores de Eventos

Exemplos de eventos poderia incluir pushes para o GitHub ou repositórios do DockerHub.
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

- Ele importa o object `events` da biblioteca `@brigadecore/brigadier`. Quase todos
  scripts do Brigade fazem isso.
- Ele declara um único manipulador de evento que fala "Quando um evento `exec` acontecer, rode
  esta função".
- Não estamos utlizando o objeto `event` neste script, mas nos iremos utilizá-lo
  em um exemplo futuro.
- O manipulador de evento retorna uma string, a qual é opcional[sumário do evento].
  Isto pode ser utilizado pelos gateways interessados em resultados ou outro contexto
  relacionados com o manipulador de eventos. O sumário apenas tem significado para o
  script e possivelmente o gateway -- caso contrário, isto é opaco para o próprio Brigade.
- Scripts com manipuladores de eventos definidos, como este aqui, precisa chamar
  `events.process()` _depois_ de registrar todos manipuladores para que todos eventos sejam
  despachados.

De forma similar ao nosso primeiro script, esta função de manipulador de evento exibe uma
,mensagem para o log, prodizindo a seguinte saída:

```plain
$ brig event create --project first-event --follow

Created event "5b0bd00a-4f31-40da-ad01-0d2f62f4d70e".

Waiting for event's worker to be RUNNING...
2021-09-20T22:24:57.655Z INFO: brigade-worker version: a398ba8-dirty
Hello, World!
```

> Note: o sucesso/fracasso de um evento é determinado pelo código de saída do
> Worker executando o script. Se o script falhar ou If the script fails or
> apresentar um erro, o Worker irá suspender(exit) execução com um erro que
> não seja zero e seja considerado como uma falha.

Como os projetos podem subscrever para vários tipos de eventos(oriundos de várias
fontea de código também), definindo vários manipuladores de eventos nos seus scripts
permite tipos de eventos diferentes serem manipulados de forma diferente.

Por exemplo, Nos podemos expandir o exemplo acima para também prover um manipulador
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

Agora, se nos executarmos novamente nosso comando `brig`, veremos a mesma saída que vimos anteriormente:

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

Para mais informações veja a documentação dos [Eventos] e entenda como eventos são estruturados
no Brigade.

[Eventos]: /topicos/project-developers/eventos
[sumário do evento]: /topicos/project-developers/events#summary

## Jobs e Containers

A imagem docker do worker é derivada da imagem do Node.js e os containers baseados
nesta imagem podem executar JavaScript e TypeScript apenas. Se outros runtimes, tools,
etc forem necessários para responder ao evento e não estiverem presentes na imagem do worker,
a criação de um _Job_ permite partes da manipulação do evento serem delegadas para um
container mais apropriado e com uma imagem docker diferente do worker e definida no script.

O blibioteca [Brigadier] expõe a interface par definir e executar Jobs.

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
importar o object `events`, nos também estamos importando `Job`. `Job` é a class
usada para criar todos os jobs no Brigade.

Em Seguida, dentro do nosso manipulador de evento `exec`, nos criamos um novo `Job`
e passamos para ele 3 parâmetros:

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
(`ps` and `grep`). Brigade faz a mesma coisa, com excessão que ao invés de executar
_programas_, ele executa _containers_. Cada container é interpretado como um Job,
que é um wrapper que entende como executar containers.

No nosso caso acima, criamos um Job chamado "do-nothing" que roda um Debian Linux 
container e (como o nome sugere) não faz nada.

Jobs são criados e executados em passos diferentes. Desta forma podemos fazer algumas
configurações no nosso job (como vamos ver em seguida) antes de executá-lo.

### Job.run()

Para rodar o job usamos o método `run()` da classe Job. Durante a execução deste comando,
Brigade cria um novo `debian` container (carregando a imagem docker se for necessário),
inicia o container e monitora. Quando o container termina sua execução, o comando run é
finalizado.

É importante mencionar que o comando `run()` é _assíncrono_ e na prática é apenas uma
_chamada_ para o servidor de API do Brigade pedindo para agendar a execução do job.
Essa execução depende das restrições do Agendador, como por exemplo configurações do 
número máximo de Jobs concorrentes permitidos, a capacidade ambiente de execução(cluster kubernetes),
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

Para que nossos Jobs possam fazer algo a mais, podemos adicionar um _comando_ nele.
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

A command can be anything that is supported by the job's image. In the example
above, our command invokes the `echo` binary with the arguments supplied.

For multiple commands, a common approach is for the `command` array to consist
of one element such as `["/bin/sh"]` or `["/bin/bash"]` and then the first
element of the `arguments` array be `"-c"`, followed by entries comprising the
shell commands needed.

However, as construction of such commands may soon prove unwieldy due to
increased complexity, we recommend writing a shell script and making this
available to the script to run (e.g. placing it in the git repository
associated with the project or adding it to a [custom Worker image]).

Let's run the example:

```plain
$ brig event create --project first-job --follow

Created event "046c09cd-76cb-49ea-b40c-d3e0e557de62".

Waiting for event's worker to be RUNNING...
2021-09-22T20:11:10.453Z INFO: brigade-worker version: 927850b-dirty
2021-09-22T20:11:10.909Z [job: my-first-job] INFO: Creating job my-first-job
```

Now, to see the logs from `my-first-job`, we issue the following brig command
utilizing the generated event ID.

```plain
$ brig event logs --id 046c09cd-76cb-49ea-b40c-d3e0e557de62 --job my-first-job

My first job!
```

> Note: `job.run()` will throw an exception if the job fails. Additionally, job
> success/failure is determined by the exit code of the job's
> `primaryContainer`.

[custom Worker image]: /topics/scripting/workers

### Combining jobs into a pipeline

Now we can take things one more step and create _two jobs_ that each do something.

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

In this example we create two jobs (`my-first-job` and `my-second-job`). Each
starts a Debian container and prints a message, then exits. On account of the
use of `await`, `job2` won't run until `job1` completes, so these jobs
_implicitly_ run sequentially.

Let's run the example and then view each job's logs:

```
$ brig event create --project simple-pipeline

Created event "58e7d3cf-b7d2-4ab7-98ad-326a99f10a25".

$ brig event logs --id 58e7d3cf-b7d2-4ab7-98ad-326a99f10a25 --job my-first-job

My first job!

$ brig event logs --id 58e7d3cf-b7d2-4ab7-98ad-326a99f10a25 --job my-second-job

My second job!
```

## Serial and Concurrent job groups

Now that we've seen an example project that runs multiple jobs, let's look at
the methods we have for specifiying _how_ the jobs run, i.e. sequentially or
concurrently -- or, as we're about to see, a combination thereof.

For example, we can run two sequences of jobs concurrently:

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

There are two things to notice in the example above:

1. Both the `concurrent` and `sequence` methods exist on the `Job` object.
2. The return type for both methods is a generic "runnable", i.e. an object
  that `run()` can then be called on, just like a standalone job instance.

Here's a breakdown of each method:

- `Job.sequence()` takes an array of runnables (e.g. a job or a group of
  jobs) and runs them _in sequence_. A new runnable is started only when the
  previous one completes. The sequence completes when the last runnable has
  completed (or when any runnable fails). A sequential group is itself
  considered a success or failure on the basis of all its jobs completing
  successfully.
- `Job.concurrent()` takes an array of runnables and runs them all
  _concurrently_. When run, all runnables are started simultaneously (subject
  to scheduling constraints). The concurrent group completes when all Runnables
  have completed. A concurrent group is itself considered a success or failure
  on the basis of all its jobs completing successfully.

As both of these methods return a runnable, they can be chained. In the example
above, `job1` and `job2` run _in sequence_, as do `jobA` and `jobB`, but both
sequences are run _concurrently_.

This is the way script writers can control the order in which groups of jobs
are run.

For example, if using Brigade to implement CI, you might wish to divide checks
into those that are resource intensive (e.g. longer-running builds, integration
tests) and those that are less so (e.g. linting, unit tests). Both groups can
run the jobs within concurrently, but the groups themselves might be run in
sequence, such that no resource intensive checks are executed unless/until all
of the less resource intensive checks have passed.

### Running a script from a Git Repository

Earlier we talked about how a project may have an associated Git repository.
Let's look at one such project now.

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

Notice that there is no embedded script in this project definition. Rather, the
project specifies where the Brigade config file directory
(`configFilesDirectory`) should be located in the repository configured under
the `git` section. (If not supplied, the default is to look for the `.brigade`
directory at the git repository's root).

The config file directory is where the Brigade script is placed. In this
example, the `brigade.js` script is simply a `console.log()` statement.

```javascript
console.log("Hello, Git!");
```

Let's run the example:

```plain
$ brig event create --project git --follow

Created event "5ff386ed-060e-49fa-8292-0bade75f8840".

Waiting for event's worker to be RUNNING...
2021-09-22T21:38:10.667Z INFO: brigade-worker version: 927850b-dirty
Hello, Git!
```

Here's what is happening behind the scenes when we create an event for this
project: Because the project has a Git repository associated with it, Brigade
is automatically fetching a clone of that repository and attaching it to the
Worker in charge of running the script.

By default, the repository contents are not automatically mounted to jobs in
the project's Brigade script. However, mounting the contents to a job is easily
accomplished via the `sourceMountPath` configuration on a job's
`primaryContainer`.

The following example shows how a job can be configured to access the repo in
order to run a test target. It also configures `workingDirectory` with the same
value as `sourceMountPath` so that the job needn't worry about changing into
the appropriate directory before running commands:

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

Being able to associated a Git repository to a project is a convenient way to
provide version-controlled data to our Brigade scripts. For instance, instead
of embedding a project's script and other configuration inside the project
definition, these files can be fetched from source control.

Additionally, this functionality makes Brigade a great tool for executing CI
pipelines, deployments, packaging tasks, end-to-end tests, and other DevOps
tasks for a given repository.

## Working with Event and Project Data

As we've seen, the event object is always passed into an event handler. This
object also includes project data. Let's look at the data we have access to.

### The Brigade Event

From the event, we can find out what triggered the event, what data was sent
with the event, the details of the worker in charge of running the event and
more.

Here are some notable fields on the event object:

- `id` is a unique, per-event ID. Every time a new event is triggered, a new ID
  will  be generated.
- `project` is the project that registered the handler for this event. We'll
  look at the fields accessible on this object below.
- `source` is the event source. The `brig event create` command, for example,
  would set source to `brigade.sh/cli`.
- `type` is the event type. A GitHub Pull Request event, for example, would set
  type to `pull_request`.
- `payload` contains any information that the external service sent when
  triggering the event. For example, a GitHub push request generates
  [a rather large payload][GitHub push event]. Payload is an unparsed string.
- `worker` is the Brigade worker assigned to handle the event. Among other
  things, git details such as the commit (revision ID) and clone URL can be
  found on this object.

For a full overview of the event object supplied by the brigadier library, see
the [Brigadier] documentation.

[GitHub push event]: https://developer.github.com/v3/activity/events/types/#pushevent

### The Project

The project object (`event.project`) gives us the following fields:

- `id` is the project ID.
- `secrets` is the key/value map of secrets defined on the project. These are
  set via `brig secret set` (see the [Secrets Guide] for more info).

[Secrets Guide]: /topics/project-developers/secrets

### Using Event and Project Objects

Let's look at some examples that utilizes event and project data.

The first example extracts the payload from the event object and logs it to the
console:

```javascript
const { events } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("Hello, " + event.payload + "!");
});

events.process();
```
[07-first-payload](https://github.com/brigadecore/brigade/tree/main/examples/07-first-payload)

When we create an event with a payload for the script above, we'll see output
like this:

```plain
$ brig event create --project first-payload --payload "Brigade" --follow

Created event "05e31d97-945b-4727-b710-7d983d137d40".

Waiting for event's worker to be RUNNING...
2021-09-23T16:25:29.761Z INFO: brigade-worker version: 927850b-dirty
Hello, Brigade!
```

We can update the example to print the project ID as well.

```javascript
const { events } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("Project: " + event.project.id);
  console.log("Hello, " + event.payload + "!");
});

events.process();
```

The following output is generated:

```plain
$ brig event create --project first-payload --payload "Brigade" --follow

Created event "b45720c4-115c-4c9b-b668-a872479f2210".

Waiting for event's worker to be RUNNING...
2021-09-23T16:30:29.267Z INFO: brigade-worker version: 927850b-dirty
Project: first-payload
Hello, Brigade!
```

Note that some event and project data should be treated with care. Things like
`event.project.secrets` or `event.worker.git.cloneURL` might not be the sorts
of information you want accidentally displayed. Check out the [Secrets] guide
for examples on how to safely handle project secrets in your scripts.

[Secrets]: /topics/project-developers/secrets

## Worker storage and shared workspace

Brigade offers the ability to set up storage for the worker that can then be
shared amongst jobs. This functionality isn't enabled by default and needs to
be configured on the `workerTemplate` section of the project definiton as well
as on each job in the script that requires access to the workspace.

### An example demonstrating use of a shared workspace

First, the `useWorkspace` field on the `workerTemplate` of the project
definition must be set to `true`:

```yaml
workerTemplate:
  useWorkspace: true
```

Next, for each job to use the shared workspace, provide a value for the
`workspaceMountPath` field on the job's `primaryContainer`:

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

> Note: Shared storage is dependent on the underlying Kubernetes cluster and
> the availability of the correct type of storage class. See the [Storage] doc
> for more information on cluster requirements.

[Storage]: /topics/operators/storage

## Sidecar containers

Jobs can optionally be configured with one or more sidecar containers, which
run alongside the job's primary container. All sidecar containers will be
terminated a short time after the job's primary container completes. A few
additional notes:

- Regardless of whether sidecar containers are present, job success or failure
  is still determined solely upon the exit code of its primary container.

- All the containers are networked together such that processes listening for
  network connections in any one of them can be addressed by processes running
  in the others using the local network interface.

- Brigade does not understand the relationship(s) between your containers and
  therefore cannot coordinate startup. If, for instance, a process in the
  primary container should be delayed until some supplementary process in some
  sidecar container is up and running and listening for connections, then the
  script author needs to account for that themselves.

As an example, consider an event handler that needs to run tests but also needs
to provision a backing database required by the tests. The backing database
could run as a sidecar container while the job's primary container is concerned
with running the tests.

Another use case is a job that needs the Docker daemon. The safest way to do
this is to run Docker-in-Docker, i.e. starting up the daemon within a container
that can then be used as needed. A sidecar container is a perfect application
for starting up the daemon whilst the job's primary container handles the main
tasks at hand.

Let's take a look at this latter example and introduce further configuration
needed for this scenario.

### Docker-in-Docker

Docker-in-Docker (DinD) containers must run as privileged in order to function.
This also needs to be allowed at the project level in order for jobs to run as
privileged. The default is for this feature to be disallowed.

Here is an example project configuration allowing containers to run in
privileged mode:

```yaml
workerTemplate:
  jobPolicies:
    allowPrivileged: true
```

Each job container needing to run in this mode must also add explicit
configuration. In the example below, the `docker` sidecar container has
`privileged` set to true, while the primary container does not:

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

Here's the output from creating an event and then looking at the job logs:

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

Note we could also take a look at the sidecar container logs on the job like so:

```plain
$ brig event logs --id 94d0fcd5-61dc-49be-bb81-3e5784e66a4a --job dind --container docker

time="2021-09-23T19:00:11.230907700Z" level=info msg="Starting up"
...
```

### Accessing the host Docker socket

For security reasons, it is recommended that you use Docker-in-Docker (DinD)
instead of using a host's Docker socket directly.

However, each job also has the option to mount in a docker socket. When
enabled, the docker socket from the Kubernetes Node running the job Pod is
mounted to `/var/run/docker.sock` in the job's container. This is typically
required only for "Docker-out-of-Docker" ("DooD") scenarios where the
container needs to use the host's Docker daemon. This is strongly discouraged
for almost all use cases.

In order for the socket to be mounted, the Brigade project must have the
`allowDockerSocket` field on the `jobPolicies` section of its worker spec set
to `true`. The default is `false`, disallowing use of the host docker socket.

Example project configuration enabling this feature:

```yaml
workerTemplate:
  jobPolicies:
    allowDockerSocketMount: true
```

Additionally, a job must declare that it needs a docker socket by setting
`useHostDockerSocket` on its `primaryContainer` to true:

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

Here's the output when we create an event for the script above:

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

> Note: Not all cluster providers use Docker as their container runtime. For
> example, [KinD] uses [containerd] and so the usual Docker socket is not
> available. Here we mention again that DinD is the preferred route when a
> Docker socket is necessary.

[KinD]: https://kind.sigs.k8s.io/
[containerd]: https://containerd.io/

## Conclusion

This guide covers the basics of writing Brigade scripts. Here are some links
for further reading:

* If you'd like more details about the Brigadier JS/TS API, take a look at the
  [Brigadier] docs
* For a more advanced script examples and techniques, see the
  [Advanced Scripting Guide]
* Peruse the other example projects/scripts in the [Examples] directory. There
  are example projects using npm, yarn, TypeScript and more.

Happy Scripting!

[Advanced Scripting Guide]: /topics/scripting/advanced
[Examples]: https://github.com/brigadecore/brigade/tree/main/examples