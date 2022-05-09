---
title: Funções/Papéis do Brigade
description: Uma introdução das Funções/Papéis do Brigade
section: topicos
weight: 2
aliases:
  - /funcoes.md
  - /topicos/funcoes.md
---

Nesta Seção, iremos discutir as 3 principais funções que nos preocupam em um 
ambiente de produção do Brigade. Elas são basicamente divididas de acordo com a
nossa forma de interação com o Brigade:

  * Gerenciamento da implantação do Servidor do Brigade e seus sistemas auxiliares
  * Gerenciamento de usuários/contas do Brigade
  * Gerenciamento e desenvolvimento de projetos do Brigade.

Naturalmente, poderá haver sobreposição entre estas funções acima. Para ambientes de desenvolvimento,
um ou dois usuários poderá cobrir todas estas funções. Entretando, estas funções também servem 
muito bem como categorias de contexto para esta documentação. Com essa informação em mãos, 
vamos mergulhar no conteúdo.

  * [Operadores] - Usuários que instalam e gerenciam a implantação do Brigade e gateways
    - [Implantação](/topicos/operadores/implantacao): Como Implantar e Gerenciar o Brigade
    - [Segurança do Brigade](/topicos/operadores/seguranca): Estabelecendo Segurança no Brigade através do uso de TLS, Ingress e componentes de Autenticação de terceiros
    - [Armazenamento](/topicos/operadores/armazenamento): Opções e configurações de Armazenamento(Storage) para o Brigade
    - [Gateways](/topicos/operadores/gateways): Utilizando e desenvolvendo gateways no Brigade
  * [Administradores] - Usuários que se preocupam em gerenciar autenticação e autorização no Brigade
    - [Autenticação](/topicos/administradores/autenticacao): Estratégias de Autenticacao no Brigade
    - [Autorização](/topicos/administradores/autorizacao): Configuração de Autorização no Brigade
  * [Desenvolvedores de Projetos] - Usuários que criam projetos e escrevem o código dos scripts usados para for manipular os eventos
    - [Projetos](/topicos/desenvolvedor-de-projetos/projetos): Criar e gerenciar projetos do Brigade
    - [Eventos](/topicos/desenvolvedor-de-projetos/eventos): Entendendo e manipulando eventos do Brigade
    - [Usando Segredos](/topicos/desenvolvedor-de-projetos/segredos): Usando segredos nos projetos do Brigade
    - [Usando a CLI do Brigade](/topicos/project-developers/brig): Usando a linha de comand CLI, chamada `brig` para interagir com o Brigade
    - [Brig term](/topicos/desenvolvedor-de-projetos/brigterm): Usando o `brig term` como utilitátio de visualização(painel de controle) no seu terminal

[Operadores]: /topicos/operadores
[Administradores]: /topicos/administradores
[Desenvolvedores de Projetos]: /topicos/desenvolvedor-de-projetos