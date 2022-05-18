---
title: Dependências
description: Como as dependências funcionam no Brigade
section: scripting
weight: 4
aliases:
  - /dependencias.md
  - /topicos/dependencias.md
  - /topicos/scripting/dependencias.md
---

O worker do Brigade é responsável pela execução do seu script no Brigade. Por
padrão, o Brigade vem com um worker que é de propósito geral e não tem nenhuma
dependência externa que não seja crítica para executar o manipulador de eventos
no Brigade.

Se você deseja ter outras dependências disponíveis em seu ambiente de execução
do worker(e disponível no próprio script), existem múltiplas abordagens:

- Criar uma imagem Docker customizada do worker, que tenha suas dependências. Essa
  abordagem é descrita em detalhes na página de [Workers] desta documetação. Em poucas
  palavras, utilize esta abordagem se você tiver a mesma dependência para vários
  projetos, ou se suas dependências demorarem muito para carregar.

- Usando a imagem Docker padrão do Brigade:
  - Forneça o arquivo [package.json] contendo as dependências específicas para o seu
    projeto do Brigade.
  - Utilizar diretamente as dependências locais disponíveis no seu diretório Git do
    projeto.

Este documento descreve estas últimas duas abordagens usando a imagem Docker padrão.

[Workers]: /topicos/scripting/workers

## Adicionando dependências customizadas usando um arquivo `package.json`

Se você precisa de dependências diferentes para cada projeto do Brigade, isso
pode ser facilmente alcançado usando um arquivo [package.json]. Este arquivo
pode ser salvo junto ao script do Brigade no repositório do projeto ou
embutido na definição do projeto.

Vamos deixar os detalhes sobre este arquivo para a [Documentação oficial][package.json],
mas aqui vamos olhar para casos específicos de listar o nome e as versões das dependêcias.
Isto pode ser adicionado na seção `dependencies`, desta forma:

```json
{
    "dependencies": {
        "is-thirteen": "2.0.0"
    }
}
```

Antes de começarmos a executar o script `brigade.js`, o worker irá instalar as  
dependências usando `npm` (ou `yarn` se um arquivo `yarn.lock` existir), e adicionando
estas depedências para o diretório `node_modules`.

Então, no script do Brigade, a nova dependência pode ser usada da mesma forma que
qualquer outra dependência do NodeJS:

```javascript
const { events } = require("@brigadecore/brigadier");
const is = require("is-thirteen");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("is 13 thirteen? " + is(13).thirteen());
});

events.process();
```

Agora se criarmos um evento para o projeto que use esse script(também definimos
`logLevel` como `DEBUG`), veremos o npm ser usado para instalar as dependências,
bem como o log do console que usa `is-thirteen`:

```
$ brig event create --project dependencies --follow

Created event "7987e2bb-5ca9-4f67-8d32-9f5dd667c0c5".

Waiting for event's worker to be RUNNING...
2021-09-27T22:35:23.234Z INFO: brigade-worker version: 9b52569-dirty
2021-09-27T22:35:23.239Z DEBUG: using npm as the package manager
2021-09-27T22:35:23.239Z DEBUG: found a package.json at /var/vcs/examples/13-dependencies/.brigade/package.json
2021-09-27T22:35:23.240Z DEBUG: installing dependencies using npm

added 2 packages, and audited 3 packages in 1s

found 0 vulnerabilities
2021-09-27T22:35:24.742Z DEBUG: path /var/vcs/examples/13-dependencies/.brigade/node_modules/@brigadecore does not exist; creating it
2021-09-27T22:35:24.743Z DEBUG: polyfilling @brigadecore/brigadier with /var/brigade-worker/brigadier-polyfill
2021-09-27T22:35:24.745Z DEBUG: found nothing to compile
2021-09-27T22:35:24.747Z DEBUG: running node brigade.js
is 13 thirteen? true
```

Notas:

- Todas dependências customizadas declaradas no arquivo `package.json` serão adicionadas
  no processo do node dedicado para o ambiente do próprio script, separado do processo
  e dependências do node do worker.

- Dependências são dinamicamente instaladas em cada execução de script do Brigade -
  isso significa que se as dependências são grande em tamanho, e a frequência do
  evento é alta para um determinado projeto, talvez faça sentido criar uma imagem
  Docker que já contenha essas dependências. Leia a documentação do [Workers] para
  mais detalhes em como fazer isso.

[package.json]: https://docs.npmjs.com/cli/v7/configuring-npm/package-json
[Workers]: /topicos/scripting/workers

## Usando dependências locais do repositório do projeto

Dependências locais são resolvidas usando "Node padrão" [resolução de módulo]. Essa
abordagem funciona muito bem para dependências que não são destinadas para pacotes
externos, e que estejam localizadas no repositório do projeto.

Essas dependências podem ser colocadas no diretório padrão da configuração do Brigade,
`./brigade`, junto com outros arquivos de configuração do projeto, como o (e.g. `brigade.js`)
e `package.json`.

Vamos considerar os seguintes cenários: temos um arquivo JavaScript localizado em
`/.brigade/circle.js`. No nosso script do Brigade, podemos usar qualquer método ou
variável que tenha sido exportada do pacote, simplesmente usando uma declaração
`require` tipo qualquer outro projeto JavaScript.

```javascript
// file /.brigade/circle.js
var PI = 3.14;
exports.area = function (r) {
    return PI * r * r;
};
exports.circumference = function (r) {
    return 2 * PI * r;
};
```

Então, em noso arquivo `brigade.js` podemos importar e usar o arquivo:

```javascript
const { events } = require("@brigadecore/brigadier");
const circle = require("./circle");

events.on("brigade.sh/cli", "exec", async event => {
  console.log("area of a circle with radius 3: " + circle.area(3));
});

events.process();
```

Aqui temos a saída quando criamos um evento, via comando `brig`, para o projeto que
esta usando este script(mais `logLevel: DEBUG`):

```plain
$ brig event create --project dependencies --follow

Created event "8aa3c5dd-a685-493a-a366-a6183a9e2650".

Waiting for event's worker to be RUNNING...
2021-09-28T13:43:49.143Z INFO: brigade-worker version: 9b52569-dirty
2021-09-28T13:43:49.148Z DEBUG: using npm as the package manager
2021-09-28T13:43:49.148Z DEBUG: path /var/vcs/examples/13-dependencies/.brigade/node_modules/@brigadecore does not exist; creating it
2021-09-28T13:43:49.149Z DEBUG: polyfilling @brigadecore/brigadier with /var/brigade-worker/brigadier-polyfill
2021-09-28T13:43:49.149Z DEBUG: found nothing to compile
2021-09-28T13:43:49.150Z DEBUG: running node brigade.js
area of a circle with radius 3: 28.259999999999998
```

[resolução de módulo]: https://nodejs.org/api/modules.html#modules_all_together

## Ambas Abordagens em um único exemplo

Confira o exemplo de projeto [13-dependencies] para ver ambas abordagens incorporadas
em um projeto. Sinta-se livre para criar o projeto, criar eventos para o projeto, etc.,
para sentir como ambos os métodos funcionam.

[13-dependencies]: https://github.com/brigadecore/brigade/tree/main/examples/13-dependencies