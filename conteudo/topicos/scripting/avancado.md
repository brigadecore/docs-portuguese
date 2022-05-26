---
title: Guia de Scripting Avançado
description: Este guia fornece algumas dicas e idéias avançadas de scripting
section: scripting
weight: 3
aliases:
  - /avancado.md
  - /topicos/avancado.md
  - /topicos/scripting/avancado.md
---

Este guia fornece algumas dicas e idéias avançadas de scripting. Assumimos
que você tenha familiaridade com o [Guia de scripting](/topicos/scripting/guia) e a [Brigadier API](/topicos/scripting/brigadier).

## Promises e o `async` `await` keywords

O Brigade suporta vários métodos para controlar o fluxo assíncrono de execução disponíveis
na linguagem JavaScript. Isto inclui "ligando promises", bem como utilizando as palavras
chave `async` e `await`.

Aqui temos um exemplo que usa [Promise chain] para organizar a execução de dois jobs:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", exec);

function exec(event) {
    let j1 = new Job("j1", "alpine:3.14", event);
    j1.primaryContainer.command = ["echo"];
    j1.primaryContainer.arguments = ["hello " + event.payload];

    let j2 = new Job("j2", "alpine:3.14", event);
    j2.primaryContainer.command = ["echo"];
    j2.primaryContainer.arguments = ["goodbye " + event.payload];

    j1.run()
    .then(() => {
        return j2.run()
    })
    .then(() => {
        console.log("done");
    });
}

events.process();
```

No exemplo acima, usamos o objeto `Promise` do JavaScript para ligar dois jobs,
então imprimimos `done` depois que os dois jobs executaram. Cada chamada a `Job.run()`
retorna uma `Promise`, e então chamamos o método `then()` da `Promise`.

Veremos em seguida o que acontece quando o script executa:

```plain
$ brig event create --project promises --payload world --follow

Created event "882f832a-c156-4afc-9936-00d3b2d61083".

Waiting for event's worker to be RUNNING...
2021-10-04T22:33:22.078Z INFO: brigade-worker version: 5c94a15-dirty
2021-10-04T22:33:22.502Z [job: j1] INFO: Creating job j1
2021-10-04T22:33:25.052Z [job: j2] INFO: Creating job j2
done

$ brig event logs --id 882f832a-c156-4afc-9936-00d3b2d61083 --job j1

hello world

$ brig event logs --id 882f832a-c156-4afc-9936-00d3b2d61083 --job j2

goodbye world
```

Podemos re-escrever o exemplo acima para usar [await] e atingir o mesmo resultado:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", exec);

async function exec(event) {
    let j1 = new Job("j1", "alpine:3.14", event);
    j1.primaryContainer.command = ["echo"];
    j1.primaryContainer.arguments = ["hello " + event.payload];

    let j2 = new Job("j2", "alpine:3.14", event);
    j2.primaryContainer.command = ["echo"];
    j2.primaryContainer.arguments = ["goodbye " + event.payload];

    await j1.run();
    await j2.run();
    console.log("done");
}

events.process();
```

A primeira coisa para perceber sobre este exemplo é que estamos anotando nossa
função `exec()`com a palavra chave `async`. Isso informa o runtime do JavaScript
que a função é um manipulador assíncrono.

As duas declarações `await` irão causar os jobs a [rodarem Sincronicamente][await].
O primeiro job vai excutar até finalizar, depois disso o segundo job irá executar até
finalizar. Então a função do `console.log` irá executar.

Algumas pessoas acham que usando `async`/`await` faz o código mais legível. Outros
preferem a notação usando `Promise`. Brigade traz suporte para ambas. Os mesmos
padrões mostrados acima podem ser usados com `Job.concurrent()` e `Job.sequence()`,
como seus métodos `run()` retornam um objeto `Promise` também.

## Manipulação de Erro

Note que quando erros acontecem, eles são encarados como exceções. Para manipular este
erro, use os blocos `try`/`catch`. Estes podem ser utilizados com as duas abordagens
"Promise chain" ou usando `async`/`await`.

Usando "Promise chaining":

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", exec);

function exec(event) {
    let j1 = new Job("j1", "alpine:3.14", event);
    j1.primaryContainer.command = ["echo"];
    j1.primaryContainer.arguments = ["hello " + event.payload];

    // j2 is configured to fail
    let j2 = new Job("j2", "alpine:3.14", event);
    j2.primaryContainer.command = ["exit"];
    j2.primaryContainer.arguments = ["1"];

    j1.run()
    .then(() => {
        return j2.run()
    })
    .then(() => {
        console.log("done");
    })
    .catch(e => {
        console.log(`Caught Exception ${e}`);
    });
}

events.process();
```

Usando `async`/`await`:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", exec);

async function exec(event) {
    let j1 = new Job("j1", "alpine:3.14", event);
    j1.primaryContainer.command = ["echo"];
    j1.primaryContainer.arguments = ["hello " + event.payload];

    // j2 is configured to fail
    let j2 = new Job("j2", "alpine:3.14", event);
    j2.primaryContainer.command = ["exit"];
    j2.primaryContainer.arguments = ["1"];

    try {
        await j1.run();
        await j2.run();
        console.log("done");
    } catch (e) {
        console.log(`Caught Exception ${e}`);
    }
}

events.process();
```

Olhando o exemplo `async`/`await`, o segundo job(`j2`) irá executar
`exit 1`, e isso fará com que o container finalize com um erro. Quando
`await j2.run()` é executado, isto irá lançar uma excecão por que `j2` terminou
com um erro. Em nosso bloco `catch`, imprimimos a mensagem de erro que recebemos.

Se executarmos isso, veremos algo deste tipo:

```plain
$ brig event create --project await --payload world --follow

Created event "69b5713f-b612-434f-9b52-9bcd57f044c5".

Waiting for event's worker to be RUNNING...
2021-10-04T22:45:45.808Z INFO: brigade-worker version: 5c94a15-dirty
2021-10-04T22:45:46.235Z [job: j1] INFO: Creating job j1
2021-10-04T22:45:48.826Z [job: j2] INFO: Creating job j2
Caught Exception Error: Job "j2" failed
```

A linha `Caught Exception...` mostra o erro que recebemos.

Note, entretanto, que não configuramos o worker para falhar quando capturamos a exceção
no exemplo acima; Simplesmente logamos isso. Embora o `j2` job tenha falhado,
o worker foi bem sucedido. Podemos ver isso quando olhamos para o evento depois de processado:

```plain
$ brig event get --id 69b5713f-b612-434f-9b52-9bcd57f044c5

ID                                  	PROJECT	SOURCE        	TYPE	AGE	WORKER PHASE
69b5713f-b612-434f-9b52-9bcd57f044c5	await  	brigade.sh/cli	exec	20s	SUCCEEDED

Event "69b5713f-b612-434f-9b52-9bcd57f044c5" jobs:

NAME	STARTED	ENDED	PHASE
j1  	16s    	13s  	SUCCEEDED
j2  	11s    	11s  	FAILED
```

Isso ilustra o seguinte ponto: ao escrever o script, capturando as exceções
dos jobs(ou outros runnables), temos a oportunidade para decidir se o workflow
teve sucesso ou fracasso. Talvez desejamos que o worker falhe imediatamente.
Em outros casos, talvez precisamos executar uma lógica condicional como uma última
análise que iria ditar o sucesso do worker ou não.

[Promise chain]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
[await]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await

## Usando Javascript orientado a objetos para extender o `Job`

JavaScript suporta classes, utilizado por programação orientada a objetos. Um exemplo onde
o Brigade faz uso disso é para prover uma forma útil de trabalhar com a classe `Job`.
A classe `Job` pode ser extendida para pré-configurar jobs similares ou para adicionar
funcionalidade extra ao job.

O seguinte exemplo cria uma classe `MyJob` que expande a classe `Job` e fornece alguns campos
pré-definidos:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

class MyJob extends Job {
  constructor(name, event) {
    super(name, "alpine:3.14", event);
    this.primaryContainer.command = ["echo"];
    this.primaryContainer.arguments = ["hello " + event.payload];
  }
}

events.on("brigade.sh/cli", "exec", async event => {
  const j1 = new MyJob("j1", event);
  const j2 = new MyJob("j2", event);

  await Job.sequence(j1, j2).run();
});

events.process();
```

No exemplo acima, ambos `j1` e `j2` terão a mesma imagem e o mesmo comando.
Ambos herdaram essas configurações da classe `MyJob`. Usando herança desta
forma pode se reduzir código padronizado/repetido.

Os campos podem ser seletivamente substituídos também. Poderíamos, por exemplo,
substituir os argumentos do comando do segundo job sem afetar o primeiro:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

class MyJob extends Job {
  constructor(name, event) {
    super(name, "alpine:3.14", event);
    this.primaryContainer.command = ["echo"];
    this.primaryContainer.arguments = ["hello " + event.payload];
  }
}

events.on("brigade.sh/cli", "exec", async event => {
  const j1 = new MyJob("j1", event);
  const j2 = new MyJob("j2", event);
  j2.primaryContainer.arguments = ["goodbye " + event.payload];

  await Job.sequence(j1, j2).run();
});

events.process();
```

Se formos olhar na saída desses dois jobs, veríamos algo assim:

```plain
$ brig event create --project jobs --payload world --follow

Created event "c4906ec3-fec1-400f-8d8f-89fd6a379475".

Waiting for event's worker to be RUNNING...
2021-10-04T23:02:58.191Z INFO: brigade-worker version: 5c94a15-dirty
2021-10-04T23:02:58.545Z [job: j1] INFO: Creating job j1
2021-10-04T23:03:01.088Z [job: j2] INFO: Creating job j2

$ brig event logs --id c4906ec3-fec1-400f-8d8f-89fd6a379475 --job j1

hello world

$ brig event logs --id c4906ec3-fec1-400f-8d8f-89fd6a379475 --job j2

goodbye world
```

O Job `j2` tem um comando diferente, enquanto o job `j1` herdou os padrões de `MyJob`.