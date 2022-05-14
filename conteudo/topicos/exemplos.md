---
title: Exemplos
description: Exemplos do Brigade
section: Tópicos
weight: 7
aliases:
  - /exemplos.md
  - /topicos/examplos.md
---

Está procurando exemplos de projetos do Brigade prontos para usar e/ou aprender?
Confira o diretório de [Exemplos] na raiz do repositório git do Brigade.

[Exemplos]: https://github.com/brigadecore/brigade/tree/main/examples
## Projetos

Estes são alguns dos tipos de projetos que você irá encontrar neste diretório:

  * [Primeira Tarefa][first-job]: Um projeto com script embutido para gerenciamento de
    eventos capaz de criar e executar uma tarefa(Job).
  * [Grupos][groups]: Um projeto com script embutido para orquestração de múltiplas
    tarefas(Jobs) executadas em sequência e concorrentemente.
  * [Git][git]: Um projeto com script carregado de um repositório Git associado.
  * [Área de Trabalho Compartilhada][shared-workspace]: Um projeto com script embutido para
    demonstrar o uso de Área de trabalho(workspace) compartilhada entre o Worker e suas tarefas(Jobs)
  * [Primeiro Payload][first-payload]: Um projeto com script embutido para demonstrar
    como usar o código de manipulação de eventos para acessar os dados do payload do evento.

[first-job]: https://github.com/brigadecore/brigade/tree/main/examples/03-first-job
[groups]: https://github.com/brigadecore/brigade/tree/main/examples/05-groups
[git]: https://github.com/brigadecore/brigade/tree/main/examples/06-git
[shared-workspace]: https://github.com/brigadecore/brigade/tree/main/examples/08-shared-workspace
[first-payload]: https://github.com/brigadecore/brigade/tree/main/examples/07-first-payload

## Gateways

O diretório [gateways] dentro do diretório de exemplos tem um [gateway exemplo] escrito usando o GoLang SDK do Brigade.

Para mais contexto e uma descrição desse exemplo de gateway, veja a documentação dos [Gateways].

[gateways]: https://github.com/brigadecore/brigade/tree/main/examples/gateways
[gateway exemplo]: https://github.com/brigadecore/brigade/tree/main/examples/gateways/example-gateway
[Gateways]: /topics/operators/gateways
