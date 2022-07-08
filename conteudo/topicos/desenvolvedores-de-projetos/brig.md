---
title: brig CLI
description: Usando brig CLI
weight: 4
aliases:
  - /brig
  - /topicos/brig.md
  - /topicos/desenvolvedores-de-projetos/brig.md
---

`brig` CLI fornece acesso ao repertório completo de usuários suportados com interações no Brigade, seja entrando no Brigade com `brig login`, inicializando um novo projeto Brigade com `brig init`, criando eventos com `brig event create` -- a lista continua.

Neste documento, veremos como [instalar o brig] e, em seguida, forneceremos uma breve visão geral dos [comandos] que o brig fornece.

[Instalando brig CLI]: #instalando-brig-cli
[comandos]: #comandos

## Instalando brig CLI

Em geral, o `brig` pode ser instalado baixando o binário apropriado da nossa [página de lançamentos](https://github.com/brigadecore/brigade/releases) para um diretório em sua máquina que esteja incluso na variável de sistema `PATH`. Em alguns sistemas, é ainda mais fácil do que isso.

Você também pode compilar o `brig` a partir do fonte; consulte o guia [Desenvolvedores] para mais informações.

[Desenvolvedores]: /topicos/desenvolvedores

**linux**

```shell
curl -Lo /usr/local/bin/brig https://github.com/brigadecore/brigade/releases/download/v2.6.0/brig-linux-amd64
chmod +x /usr/local/bin/brig
```

**macos**

O popular gerenciador de pacotes [Homebrew](https://brew.sh/) fornece o método mais conveniente de instalar o Brigade CLI em um Mac:

```shell
$ brew install brigade-cli
```

Alternativamente, você pode instalar manualmente baixando diretamente um binário pré-construído:

```shell
$ curl -Lo /usr/local/bin/brig https://github.com/brigadecore/brigade/releases/download/v2.6.0/brig-darwin-amd64
$ chmod +x /usr/local/bin/brig
```

**windows**

```powershell
> mkdir -force $env:USERPROFILE\bin
> (New-Object Net.WebClient).DownloadFile("https://github.com/brigadecore/brigade/releases/download/v2.6.0/brig-windows-amd64.exe", "$ENV:USERPROFILE\bin\brig.exe")
> $env:PATH+=";$env:USERPROFILE\bin"
```

O script acima baixa o brig.exe e o adiciona ao seu `PATH` para a sessão atual. Adicione a seguinte linha ao seu [Perfil do PowerShell] para tornar a alteração permanente.

```powershell
> $env:PATH+=";$env:USERPROFILE\bin"
```

[Perfil do PowerShell]: https://www.howtogeek.com/126469/how-to-create-a-powershell-profile/

## Comandos

Para ver todos os comandos que o brig suporta, basta digitar `brig` em seu terminal. Você deve ver os comandos disponíveis em `COMMANDS`. Incluíndo:

* `event`: Criar e gerenciar [Eventos] do Brigade 
* `init`: Iniciar um novo [Projeto] do Brigade 
* `login`: Realizar login no Brigade
* `logout`: Realizar logout do Brigade
* `project`: Criar e gerenciar [Projetos] do Brigade 
* `role`: Conceder, revogar e listar funções do sistema para [usuários] ou [contas de serviço]
* `service-account`: Criar e gerenciar [contas de serviço]
* `users`: Gerenciar [usuários] autenticados

Digite qualquer um desses comandos para obter um menu de ajuda e começar a se aprofundar na seleção completa de funcionalidades que cada um oferece. Por exemplo:

```plain
 $ brig event

NAME:
   Brigade event - Manage events

USAGE:
   Brigade event command [command options] [arguments...]

COMMANDS:
   cancel           Cancel a single event without deleting it
   cancel-many, cm  Cancel multiple events without deleting them
   clone            Clone an existing event
   create           Create a new event
   delete           Delete a single event
   delete-many, dm  Delete multiple events
   get              Retrieve an event
   list, ls         List events
   retry            Retry an event
   log, logs        View worker or job logs
   help, h          Shows a list of commands or help for one command

OPTIONS:
   --help, -h     show help (default: false)
   --version, -v  print the version (default: false)
```

[Eventos]: /topicos/desenvolvedores-de-projetos/eventos
[Projeto]: /topicos/desenvolvedores-de-projetos/projetos
[Projetos]: /topicos/desenvolvedores-de-projetos/projetos
[Projects]: /topicos/desenvolvedores-de-projetos/projetos
[usuários]: /topicos/administradores/autorizacao.md
[contas de serviço]: topicos/administradores/autorizacao.md
