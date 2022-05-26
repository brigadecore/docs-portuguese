---
title: A Biblioteca Brigadier
description: Descrevendo as APIs públicas normalmente usadas para escrever scripts no Brigade
section: scripting
weight: 2
aliases:
  - /brigadier.md
  - /topicos/brigadier.md
  - /topicos/scripting/brigadier.md
---

Este documento descreve as APIs públicas normalmente usadas para escrever 
scripts no Brigade em JavaScript ou TypeScript. Não vamos descrever as bibliotecas
internas utilizadas, nem iremos listar os métodos e propriedades não públicos
nestes objetos.

Um script no Brigade é executado dentro de um [Brigade Worker]. O worker padrão
contém um ambiente Node.js para instalar [dependências] e executar o script fornecido.

[Brigade Worker]: /topicos/scripting/worker
[dependências]: /topicos/scripting/dependencias

## Conceitos de alto nível

Um arquivo JS/TS do Brigade esta sempre associado com um [projeto]. Um projeto
define informação contextual, e também dita os parâmetros de segurança no qual 
o script será executado.

Um projeto pode associar o script com um repositório Git. Caso Contrário, o script
do projeto deve ser definido embutido na definição do projeto.

Arquivos do Brigade respondem a [eventos]. Scripts do Brigade são normalmente
compostos de um ou mais _manipuladores de eventos_. Quando o ambiente do Brigade
dispara um evento, o manipulador de evento associado irá ser chamado.

[projeto]: /topicos/desenvolvedor-de-projetos/projetos
[eventos]: /topicos/desenvolvedor-de-projetos/eventos

## A biblioteca `brigadier`

A biblioteca principal do Brigade é chamada `brigadier`. O Worker padrão do Brigade
define acesso para essa biblioteca automaticamente. O código fonte para esta biblioteca
esta disponível [aqui][brigadier] e o pacote npm [aqui][brigadier npm].

Para importar e usar a biblioteca no seu script, adicione o seguinte código no começo do seu script:

```javascript
const brigadier = require('@brigadecore/brigadier')
```

É considerado idiomático desconstruir a biblioteca quando importando:

```javascript
const { events, Job, Group } = require('@brigadecore/brigadier')
```

## Desenvolvimento Local

A biblioteca brigadier é realmente dividida em duas implementações:

* [brigadier] quase não contém implementação de interfaces públicas
* [brigadier-polyfill] contém a lógica para realmente se comunicar com o servidor
  de API do Brigade; Esta é a versão substituída pelo worker durante execução.

Usando esta estratégia, desenvolvedores tem a posibilidade de executar seus scripts
do Brigade localmente, sem nenhuma consequência, dando suporte ao desenvolvimento e a
solução de problemas.

Vamos analisar este exemplo:

Primeiramente, criamos um novo diretório de projeto e colocamos o seguinte conteúdo no
arquivo do script chamado `brigade.js`:

```javascript
const { events, Job } = require("@brigadecore/brigadier");

events.on("brigade.sh/cli", "exec", async event => {
  let job = new Job("my-first-job", "debian:latest", event);
  job.primaryContainer.command = ["echo"];
  job.primaryContainer.arguments = ["My first job!"];
  await job.run();
});

events.process();
```

Então, criamos um arquivo `package.json` com a dependência do brigadier adicioanda:

```json
{
  "dependencies": {
    "@brigadecore/brigadier": "^2.5.0"
  }
}
```

Em seguida, buscamos a dependência do brigadier (e por sua vez, as dependências dele também):

```plain
$ npm install

added 3 packages, and audited 4 packages in 1s

found 0 vulnerabilities
```

Agora estamos prontos para executar nosso script do Brigade em um ambiente de
desenvolvimento, usando apenas a biblioteca principal `brigadier`:

```plain
$ node brigade.js
No dummy event file provided
Generating a dummy event
No dummy event type provided
Using default dummy event with source "brigade.sh/cli" and type "exec"
Processing the following dummy event:
{
  id: '7eafd0d3-39e9-4341-bd33-7a215e481024',
  source: 'brigade.sh/cli',
  type: 'exec',
  project: { id: '82259392-feea-4102-a8a3-080fdd85cfa9', secrets: {} },
  worker: {
    apiAddress: 'https://brigade2.example.com',
    apiToken: '7000152b-cd0d-483f-b21f-5ef20292e72a',
    configFilesDirectory: '.brigade',
    defaultConfigFiles: {}
  }
}
O Worker do Brigade iria executar o job my-first-job aqui.
```

Sucesso!

Digamos que esquecemos de adicionar a chamada para `events.process()`
no final do nosso script. Iriamos identificar imediatamente quando o script
executar por que não teriamos nenhuma saída, sinalizando que o manipulador
de eventos não executou.

### Configuração de evento opcionais

Opcionalmente, os desenvolvedores podem fornecer as seguintes configurações quando estiverem
executando seus scripts localmente:

  * `BRIGADE_EVENT_FILE` - Esse é caminho para um arquivo contendo uma representação
    JSON de um evento dummy. Ver como um evento válido se apresenta, da perspectiva do
    Brigadier, veja o examplo do evento dummy na saída acima ou se refira ao [events.ts]
  * `BRIGADE_EVENT` - Essa é a string do form `<source>:<type>` para especificar o source
    do evento e o tipo que sera manipulado pelo script. No exemplo acima, o evento dummy
    utiliza `brigade.sh/cli:exec`

Para exemplos adicionais de como usar brigadier, por favor leia o [Guia de Scripting]
e/ou examine os [Exemplos].

[brigadier]: https://github.com/brigadecore/brigade/tree/main/v2/brigadier
[brigadier npm]: https://www.npmjs.com/package/@brigadecore/brigadier
[brigadier-polyfill]: https://github.com/brigadecore/brigade/tree/main/v2/brigadier-polyfill
[Guia de Scripting]: /topicos/scripting/guia
[events.ts]: https://github.com/brigadecore/brigade/tree/main/v2/brigadier/src/events.ts
[Exemplos]: /topicos/exemplos

## Documentação da API do Brigadier

A documentação da API do Brigadier é gerada a partir do código fonte diretamente.
Podendo ser apresentada em dois formatos: gerado diretamente do código fonte em
TypeScript e gerado do código fonte JavaScript compilado.

Para documentação em TypeScript, leia https://brigadecore.github.io/brigade/ts

Para documentação em JavaScript, leia https://brigadecore.github.io/brigade/js