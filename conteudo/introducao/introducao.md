---
linkTitle: Introdução
title: Introdução do Brigade
description: High-level view of Brigade.
section: intro
weight: 1
aliases: 
  - /introducao.md
  - /introducao/introducao.md
---

Brigade é uma ferramenta de script baseado em eventos que roda em cima do Kubernetes. O que isso siginifica:

- Brigade server para executar scripts de tarefas automatizadas na Cloud.
- Brigade não requer que você gerencie servidores.
- Brigade é particulamente adequado para CI e CD workloads como:
  - Test de Automação
  - Integração com GitHub hooks
  - Criar artefatos e releases de software
  - Gerenciar implantações
- Brigade vem com o seu próprio API server
  - Podemos interagir com o Brigade diretamente, via a linha de comando CLI `brig` ou usando o SDK(Software development Kit) disponível.
  - A operação e o gerenciamento do Brigade utiliza a API do K8s para investigar os recursos do Brigade.
  - O Servidor do Brigade é instalado e gerenciado usando o Helm, permitindo sua implantação em qualquer Kubernetes cluster, desde Azure até Minikube.
- Brigade manipula questões de autenticação e autorização - authn/authz
  - Brigade permite soluções de terceiros para autenticação, incluindo GitHub's OAuth2 identity provider e OIDC-compliant solutions tipo Azure Active Directory e Google Identity Platform.
  - Administratores do Brigade tem total controle sobre autorização, incluindo Service Account, Role e gerenciamento de usuário.
- Brigade utiliza technologias populares para manipular os eventos de forma simples:
  - Project developers escrevem scripts em JavaScript ou TypeScript (sem necessidade de aprender Node.js)
  - Utiliza OCI/Docker images para manipular eventos recebidos
  - Configuração do Projeto é escrita em YAML e facilmente inserida em seu VCS Sistema de genrenciamento de código preferido, como o Git.

Brigade foi desenhado para times. Brigade não é um SaaS (Software as a Service), não requer uma grupo enorme de especialistas para ser instalado e configurado. Em vez disso, Brigade é simples de instalar, configurar e manter.