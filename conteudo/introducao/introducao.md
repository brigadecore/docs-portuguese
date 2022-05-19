---
linkTitle: Visão Geral
title: Visão geral do Brigade
description: Visão geral do Brigade.
section: Introdução
weight: 1
aliases: 
  - /introducao.md
  - /introducao/introducao.md
---

Brigade é uma ferramenta de script baseado em eventos que roda em cima do Kubernetes. O que isso siginifica:

- Brigade foi pensado para executar scripts de tarefas automatizadas na Cloud.
- Brigade não requer que você gerencie servidores.
- Brigade é particularmente adequado para CI e CD workloads como:
  - Teste de Automação
  - Integração com GitHub hooks
  - Criar artefatos e releases de software
  - Gerenciar implantações
- Brigade vem com o seu próprio API server
  - Podemos interagir com o Brigade diretamente, via a linha de comando CLI `brig` ou usando o SDK(Software development Kit) disponível.
  - A operação e o gerenciamento do Brigade utiliza a API do K8s para investigar os recursos do Brigade.
  - O Servidor do Brigade é instalado e gerenciado usando o Helm, permitindo a sua implantação em qualquer cluster do Kubernetes, desde Azure até Minikube.
- Brigade manipula questões de autenticação e autorização - authn/authz
  - Brigade permite soluções de terceiros para autenticação, incluindo GitHub OAuth2 identity provider e OIDC-compliant solutions tipo Azure Active Directory e Google Identity Platform.
  - Administradores do Brigade tem total controle sobre autorização, incluindo Conta de Serviço, Função/Papel e gerenciamento de usuário.
- Brigade utiliza tecnologias populares para manipular os eventos de forma simples:
  - Os desenvolvedores de Projetos no Brigade escrevem scripts em JavaScript ou TypeScript (sem necessidade de aprender Node.js)
  - Utiliza imagens no formato OCI/Docker para manipular eventos recebidos
  - Configuração do Projeto é escrita em YAML e facilmente inserida em seu VCS(Sistema de gerenciamento de código) preferido, como o Git.

Brigade foi pensado para ser usado por times. Brigade não é um SaaS (Software as a Service), não requer um grupo enorme de especialistas para ser instalado e configurado. Em vez disso, Brigade é simples de instalar, configurar e manter.