---
linkTitle: Início Rápido
title: Início Rápido do Brigade
description: Início Rápido do Brigade.
section: Introdução
weight: 3
aliases:
  - /iniciorapido.md
  - /introducao/iniciorapido.md
---

Esse início rápido introduz o Brigade de uma forma abrangente. Você irá
instalar Brigade com as configurações padrão em seu cluster local
de desenvolvimento, vai criar um projeto e um evento, observar Brigade
processando o evento criado e no final você vai apagar todo ambiente criado.

Se você preferir aprender assistindo um vídeo, confira este
[guia adaptado para vídeo](https://www.youtube.com/watch?v=VFyvYOjm6zc) no
nosso canal do YouTube.

- [Pré-requisitos](#pré-requisitos)
- [Como Instalar](#como-instalar)
- [Experimentando](#experimentando)
  - [Port Forwarding](#port-forwarding)
  - [Instalar a linha de comando CLI do Brigade](#instalar-a-linha-de-comando-cli-do-brigade)
  - [Logando no Brigade](#logando-no-brigade)
  - [Criando um Projecto do Brigade](#criando-um-projecto-do-brigade)
  - [Criando um Evento](#criando-um-evento)
- [Limpeza do Ambiente](#limpeza-do-ambiente)
- [Próximos passos](#próximos-passos)
- [Resolução de Problemas](#resolução-de-problemas)
  - [Instalação não foi realizada com sucesso](#instalação-não-foi-realizada-com-sucesso)
  - [Comando de Logar fica travado](#comando-de-logar-fica-travado)

## Pré-requisitos

> ⚠️ A configuração padrão utilizada neste guia é apropriada apenas para
> avaliar o Brigade em seu cluster local de desenvolvimento, não devendo ser
> utilizada para ambientes compartilhados, _especialmente para ambiente de produção_. 
> Olhe nosso [Guida de Desenvolvimento](/topics/operators/deploy/) para instruções que sejam 
> aplicavéis para ambientes compartilhados ou clusters de produção. Nós testamos essas instruções em 
> um cluster [KinD](https://kind.sigs.k8s.io/) local.

* Cluster Kubernetes local de _desenvolvimento_ utilizando versão v1.16.0 ou superior
* [Helm v3.7.0+](https://helm.sh/docs/intro/install/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* Espaço em disco livre. Para ser bem sucedida, a instalação requer espaço em disco livre suficiente.

## Como Instalar

Essa seção especificamente cobre a instalação dos componentes do servidor
do Brigade num cluster Kubernetes local de desenvolvimento. Nos iremos instalar
a linha de comando CLI do Brigade mais tarde quando formos testar o Brigade na prática.

1. Abilitar a flag experimental do Helm para dar suporte ao OCI (Open Container Initiative):

    **POSIX**
    ```shell
    $ export HELM_EXPERIMENTAL_OCI=1
    ```

    **PowerShell**
    ```powershell
    > $env:HELM_EXPERIMENTAL_OCI=1
    ```

2. Execute os seguintes comandos para instalar o Brigade com configurações padrão:

    ```shell
    $ helm install brigade \
        oci://ghcr.io/brigadecore/brigade \
        --version v2.4.0 \
        --create-namespace \
        --namespace brigade \
        --wait \
        --timeout 300s
    ```

    > ⚠️ A Instalação e processo de inicialização pode demorar alguns minutos para finalizar.

    Se a instalação falhar, leia instruções na seção de [Resolução de Problemas](#resolução-de-problemas).

## Experimentando

### Port Forwarding

Como você esta executando Brigade localmente, deve utilizar port forwarding para fazer com que 
a API do Brigade fique disponível na sua network local:

**POSIX**
```shell
$ kubectl --namespace brigade port-forward service/brigade-apiserver 8443:443 &>/dev/null &
```

**PowerShell**
```powershell
> kubectl --namespace brigade port-forward service/brigade-apiserver 8443:443 *> $null  
```

### Instalar a linha de comando CLI do Brigade

Normalmente, a CLI do Brigade, `brig`, pode ser instalada fazendo o download do
binário apropriado disponível na   
[página de release](https://github.com/brigadecore/brigade/releases) para um diretório
local na sua máquina que esteja incluído na sua variável de ambiente `PATH`. Em alguns
sistemas operacionais o processo será ainda mais simples. Siga as instruções abaixo 
para os sistemas operacionais mais comuns:

**Linux**

```shell
$ curl -Lo /usr/local/bin/brig https://github.com/brigadecore/brigade/releases/download/v2.4.0/brig-linux-amd64
$ chmod +x /usr/local/bin/brig
```

**macOS**

O pupular [Homebrew](https://brew.sh/) gerenciador de pacotes permite uma forma mais 
conveniente de instalar a linha de comando CLI do Brigade no Mac:

```shell
$ brew install brigade-cli
```

Alternativamente, você pode instalar o Brigade manualmente fazendo o download do binário:

```shell
$ curl -Lo /usr/local/bin/brig https://github.com/brigadecore/brigade/releases/download/v2.4.0/brig-darwin-amd64
$ chmod +x /usr/local/bin/brig
```

**Windows**

```powershell
> mkdir -force $env:USERPROFILE\bin
> (New-Object Net.WebClient).DownloadFile("https://github.com/brigadecore/brigade/releases/download/v2.4.0/brig-windows-amd64.exe", "$ENV:USERPROFILE\bin\brig.exe")
> $env:PATH+=";$env:USERPROFILE\bin"
```

O script acima faz o download do `brig.exe` e adiciona ele no sua variável de ambiente local `PATH` na seccão atual
da sua sheel. Adicione a seguinte linha para o seu
[PowerShell Profile](https://www.howtogeek.com/126469/how-to-create-a-powershell-profile/)
para que esta configuração se torne permanente:

```powershell
> $env:PATH+=";$env:USERPROFILE\bin"
```

### Logando no Brigade

> ⚠️ Nesta seção iremos logar no Brigade como usuário root. Essa opção deveria estar disabilitada em ambientes de produção. Leia mais sobre
> autenticação de usuário [aqui](/topics/administrators/authentication/).

Para autenticar no Brigade como usuário root, você primeiro precisa adquirir a senha gerada automaticamente pelo sistema durante instalação:

**POSIX**
```shell
$ export APISERVER_ROOT_PASSWORD=$(kubectl get secret --namespace brigade brigade-apiserver --output jsonpath='{.data.root-user-password}' | base64 --decode)
```

**PowerShell**
```powershell
> $env:APISERVER_ROOT_PASSWORD=$(kubectl get secret --namespace brigade brigade-apiserver --output jsonpath='{.data.root-user-password}' | base64 --decode)
```

Após adquirir a senha, faça o login:

**POSIX**
```shell
$ brig login --insecure --server https://localhost:8443 --root --password "${APISERVER_ROOT_PASSWORD}"
```

**PowerShell**
```powershell
> brig login --insecure --server https://localhost:8443 --root --password "$env:APISERVER_ROOT_PASSWORD"
```

A flag `--insecure` informa ao comando `brig login` para ignorar o certificado auto-assinado
usado pela instalação local do Brigade.

Se o comando `brig login` travar or falhar, você deve checar se a port-forwarding
criado para acessar o serviço do `brigade-apiserver` foi bem sucessida na seção anterior.

### Criando um Projecto do Brigade

Um [projeto](/topics/project-developers/projects) no brigade faz o link entre eventos subscritos e a configuração 
de seus manipuladores(event handler).

1. Ao invés de criar uma definição do projeto do zero iremos acelerar o processo
   usando o comando `brig init`:

    ```shell
    $ mkdir first-project
    $ cd first-project
    $ brig init --id first-project
    ```

    Esse comando irá criar um definição de projeto em `.brigade/project.yaml`
    similar com a descrita abaixo. Esta definição subscreve para o evento `exec`
    emitido pela fonte chamada `brigade.sh/cli`. (Esse tipo de evento é criado usando
    a linha de comando CLI do brigade, e adequado para efeito de demonstração.) Quando
    este tipo de evento é recebido pelo brigade, o script em typescript é executado.
    Dependendo da fonte e do tipo de evento o script poderá executar ações diferentes.
    Para um evento `exec` vindo da fonte chamado `brigade.sh/cli`, esse script irá executar
    um simples "Hello World!" job. Para qualquer outro tipo de evento, esse script não fará nada.

    ```yaml
    apiVersion: brigade.sh/v2
    kind: Project
    metadata:
      id: first-project
    description: My new Brigade project
    spec:
      eventSubscriptions:
        - source: brigade.sh/cli
          types:
            - exec
    workerTemplate:
      logLevel: DEBUG
      defaultConfigFiles:
      brigade.ts: |
        import { events, Job } from "@brigadecore/brigadier"
        
        // Use events.on() to define how your script responds to different events. 
        // The example below depicts handling of "exec" events originating from
        // the Brigade CLI.
        
        events.on("brigade.sh/cli", "exec", async event => {
            let job = new Job("hello", "debian:latest", event)
            job.primaryContainer.command = ["echo"]
            job.primaryContainer.arguments = ["Hello, World!"]
            await job.run()
        })

        events.process()
    ```

2. O comando anterior apenas gera a definição de projeto baseado num template. Nos 
   precisamos fazer o upload dessa definição no brigade para completar a criação do projeto:

    ```shell
    $ brig project create --file .brigade/project.yaml
    ```

3. para confirmar a criação do projeto no Brigade, usamos este comando `brig project list`:

    ```shell
    $ brig project list

    ID           	DESCRIPTION                         	AGE
    first-project	My new Brigade project               	1m
    ```

### Criando um Evento

Com nosso projeto criado, podemos agora manualmente criar um evento e acompanhar o processamento pelo Brigade:

```shell
$ brig event create --project first-project --follow
```

Abaixo temos um exemplo de execução do comando:

```plain
Created event "2cb85062-f964-454d-ac5c-526cdbdd2679".

Waiting for event's worker to be RUNNING...
2021-08-10T16:52:01.699Z INFO: brigade-worker version: v2.4.0
2021-08-10T16:52:01.701Z DEBUG: writing default brigade.ts to /var/vcs/.brigade/brigade.ts
2021-08-10T16:52:01.702Z DEBUG: using npm as the package manager
2021-08-10T16:52:01.702Z DEBUG: path /var/vcs/.brigade/node_modules/@brigadecore does not exist; creating it
2021-08-10T16:52:01.702Z DEBUG: polyfilling @brigadecore/brigadier with /var/brigade-worker/brigadier-polyfill
2021-08-10T16:52:01.703Z DEBUG: compiling brigade.ts with flags --target ES6 --module commonjs --esModuleInterop
2021-08-10T16:52:04.210Z DEBUG: running node brigade.js
2021-08-10T16:52:04.360Z [job: hello] INFO: Creating job hello
2021-08-10T16:52:06.921Z [job: hello] DEBUG: Current job phase is SUCCEEDED
```

> ⚠️ Como Padrão, o componente de agendamento de processos do Brigade procura por novos projetos a cada 30
> segundos. Se o brigade for devagar para processar o seu primeiro evento, este é o motivo.

## Limpeza do Ambiente

Se você deseja manter a sua instalação do Brigade, execute o seguinte comando para
remover o projeto exemplo criado por este início rápido:

```shell
$ brig project delete --id first-project
```

para remover por completo o Brigade do seu cluster, você pode apagar todos os recursos criados por este 
Início rápido usando este comando:

```shell
$ helm delete brigade -n brigade
```

## Próximos passos

Você agora sabe como instalar o Brigade no seu ambiente local de desenvolvimento, criar um projeto,
e manualmente criar um evento. Como próximo passo, [leia em seguida](/intro/readnext)
sobre tópicos mais avançados a serem explorados.

## Resolução de Problemas

* [Instalação não foi realizada com sucesso](#instalação-não-foi-realizada-com-sucesso)
* [Comando de Logar fica travado](#comando-de-logar-fica-travado)

### Instalação não foi realizada com sucesso

Uma falha comum na instalação do Brigade é falta de espaço de disco aonde o node
do cluster foi criado. Em um cluster de desenvolvimento local rodando no macOS ou Windows, isto
poderia ter sido causado por espaço insuficiente no disco alocado para o Docker Desktop,
ou espaço alocado esta quase totalmente utilizado. Se este é o seu caso, isto será fácil de
descobrir analisando os logs do Banco de dados MongoDB do Brigade ou os pods do ActiveMQ Artemis.
Se os logs incluem mensagens como "No space left on device" or "Disk Full!", então você deve
liberar espaço no disco e tentar instalar novamente. Executando `docker system prune` é
uma forma de recuperar espaço em disco.

Depois de você liberar espaço em disco, remova a instação que falhou, e tente novamente
usando o seguinte comando:

```shell
$ helm uninstall brigade -n brigade
$ helm install brigade \
    oci://ghcr.io/brigadecore/brigade \
    --version v2.4.0 \
    --namespace brigade \
    --wait \
    --timeout 300s
```

### Comando de Logar fica travado

Se o comando `brig login` travar, confirme se você incluiu a flag `--insecure` (ou
`-k`). Essa flag é requirida pela configuração padrão utilizada por este Início rápido,
visto que este utiliza um certificado de ssl auto-assinado e que não pode ser verificado.