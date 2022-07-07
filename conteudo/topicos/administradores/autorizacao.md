---
title: Autorização
description: Configuração de autorização no Brigade
section: administradores
weight: 2
aliases:
  - /autorizacao
  - /topicos/autorizacao.md
  - /topicos/administradores/autorizacao.md
---

A autorização no Brigade consiste em funções com escopos particulares, que são concedido a Usuários e Contas de Serviço. Quando os usuários interagem com o Brigade via a Brig CLI ou quando uma Conta de Serviço interage com o Brigade por meio de um SDK, o Brigade previamente verifica se o solicitante está suficientemente autorizado.

Os três principais componentes de autorização no Brigade são:

  * [Usuários](#usuários)
  * [Contas de Serviço](#contas-de-serviço)
  * [Funções](#funções)

Os usuários são gerados no sistema após a autenticação bem sucedida com o provedor de autenticação de terceiros selecionado e a criação de contas de serviço e atribuições de funções é de responsabilidade do Administrador do Brigade.

> Nota: Existe um método para que as atribuições de função sejam concedidas automaticamente para um determinado grupo de usuários em seu primeiro login no Brigade. O papel especificamente concede privilégios de administrador em nível de sistema para cada usuário designado.
Detalhes sobre como configurar esta opção de implantação podem ser vistos na seção [Autenticação] da documentação oficial.

[Autenticação]: /topicos/administradores/autenticacao

## Usuários

Um Usuário no Brigade representa um usuário humano autenticado no sistema por meio do provedor de autenticação de terceiros selecionado durante a implantação do Brigade. Não há mecanismo para criar usuários fora deste sistema de autenticação. Os usuários são
atribuídos a [funções](#funções) concedendo permissões com escopo em torno de suas interações com recursos no Brigade.

Além de visualizar os detalhes de um usuário específico, os administradores podem listar, bloquear, desbloquear e excluir usuários. Todas essas funções de gerenciamento existem no grupo de comandos `brig users`. Para ver o pacote completo, utilize o seguinte comando:

```shell
$ brig users --help
```

## Contas de Serviço

Uma Conta de Serviço no Brigade representa um ator não humano que pode ser atribuído a uma [função](#funções) concedendo permissões com escopo para interagir com recursos no Brigade. Um padrão comum é criar uma conta de serviço para um gateway e atribuir a ele um papel `EVENT_CREATOR` para que ele possa enviar eventos para o Brigade.

Os administradores podem criar, listar, obter, bloquear, desbloquear e excluir contas de serviço. Todas essas funções de gerenciamento existem no grupo de comandos `brig service-accounts`. Para ver o pacote completo, utilize o seguinte comando:

```shell
$ brig service-accounts --help
```

## Funções

Uma Função no Brigade representa um conjunto de permissões com escopo em torno do acesso a recursos dentro do Brigade, que pode ser atribuído a um [usuário](#usuários) ou a uma [Conta de Serviço](#contas-de-serviço). Existem funções no nível de sistema, bem como
funções em nível de projeto.

Os administradores podem conceder, revogar e listar funções, seja no nível de sistema ou a nível de projeto. Todas essas funções de gerenciamento existem no grupo de comandos `brig roles` ou `brig project roles`. Para ver o pacote completo, utilize os seguintes comandos:

```shell
$ brig roles --help
$ brig project roles --help
```

### Funções em nível de sistema

As funções em nível de sistema no Brigade são as seguintes:

* `ADMIN` - Permite o gerenciamento do sistema, incluindo permissões em nível de sistema para outros usuários e contas de serviço.
* `EVENT_CREATOR`- Permite a criação de eventos para todos os projetos. Um evento `source` deve ser fornecido para cada atribuição desta função.
* `PROJECT_CREATOR` - Permite a criação de novos projetos. Quando um usuário com esta atribuição de função cria um novo projeto, eles recebem automaticamente todas as [funções em nível de projeto](#funções-em-nível-de-projeto) para o projeto.
* `READER`- Permite acesso global somente leitura ao Brigade.

Cada função é um subcomando em `brig role grant` ou `brig role revoke`. Por exemplo, para conceder a função `ADMIN` ao usuário `Maria`, 
utilize o seguinte comando:

```shell
$ brig role grant ADMIN --user Maria
```

Qualquer função do sistema também pode ser concedida a uma conta de serviço.

### Funções em nível de projeto

As funções em nível de projeto no Brigade são as seguintes:

* `PROJECT_ADMIN` - Permite o gerenciamento de todos os aspectos do projeto, incluindo as chaves, bem como permissões em nível de projeto para outros usuários e contas de serviço.
* `PROJECT_DEVELOPER` - Permite atualizar a definição do projeto, mas NÃO habilita o gerenciamento de chaves do projeto ou permissões em nível de projeto para outros usuários e contas de serviço.
* `PROJECT_USER` - Permite a criação e gerenciamento de eventos associados ao projeto

Cada função é um subcomando de `brig project role grant` ou `brig project role revoke`. Por exemplo, para conceder o papel `PROJECT_ADMIN` ao
usuário `Maria` para o projeto `Arecibo`, utilize o seguinte comando:

```shell
$ brig project role grant ADMIN --id Arecibo --user Maria
```

Qualquer função do projeto também pode ser concedida a uma conta de serviço.