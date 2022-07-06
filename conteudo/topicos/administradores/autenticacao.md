---
title: Autenticação
description: Configuração de autenticação no Brigade
section: administradores
weight: 1
aliases:
  - /autenticacao
  - /topicos/autenticacao.md
  - /topicos/administradores/autenticacao.md
---

O Brigade utiliza provedores de terceiros para sua estratégia de autenticação de usuários.
Veremos as opções disponíveis e sua configuração neste documento.

A seleção de uma estratégia de autenticação de terceiros viável é um pré-requisito para habilitar comandos de autorização do usuário.
## Motivação
`
A motivação por trás deste design deriva de um dos princípios orientadores no Brigade 2: Fornecer uma abstração entre os usuários e o substrato subjacente (Kubernetes), de forma que os usuários do Brigade não precisem interagir (ou ter qualquer presença) no substrato em si. Portanto, a necessidade de obter a identidade do usuário ser de outra autoridade autenticadora tornou-se visível. Ao invés de implementar tal sistema no próprio Brigade, decidimos utilizar serviços externos bem conhecidos
para suprir essa necessidade.

## Opções de autenticação de terceiros

Atualmente, existem duas opções de autenticação para integração com o Brigade.

  * Um provedor [OpenID Connect]. Os serviços de exemplo incluem [Google Identity Platform] e [Azure Active Directory]
  * [GitHub's OAuth Provider]

Além disso, o servidor Brigade deve estar funcionando com TLS habilitado ao usar autenticação de terceiros. Consulte o documento [Deploy] para obter mais detalhes sobre como deixar o Brigade mais seguro.

[OpenID Connect]: https://openid.net/connect/
[Google Identity Platform]: https://cloud.google.com/identity-platform
[Azure Active Directory]: https://azure.microsoft.com/pt-br/services/active-directory/
[GitHub's OAuth Provider]: https://docs.github.com/pt/developers/apps/building-oauth-apps/authorizing-oauth-apps
[Deploy]: /topicos/operadores/implantacao

### Provedor OpenID Connect

Para usar um provedor OpenID Connect para autenticação no Brigade, você precisará os seguintes valores após escolher e configurar seu provedor preferido:

  * URL do provedor, por exemplo, `https://accounts.google.com` ou
    `https://login.microsoftonline.com/<Azure Tenant ID>/v2.0`
  * Client ID
  * Client Secret

Esses valores podem ser fornecidos no arquivo [values.yaml] para o Brigade Helm Chart. Os valores estarão na seção `apiserver.thirdPartyAuth`. Veja o exemplo abaixo:

```yaml
apiserver:
  ## Opções para autenticação por meio de um provedor de autenticação de terceiros.  thirdPartyAuth:
  thirdPartyAuth:
    ## Aqui configuramos a estratégia OpenID Connect (oidc)
    strategy: oidc
    ## Aqui fornecemos os valores do provedor OIDC
    oidc:
      providerURL: https://accounts.google.com
      clientID: <client ID>
      clientSecret: <client secret>
    ## O TTL da Sessão do Usuário determina o tempo de vida padrão para as sessões do usuário.
    ## As unidades de tempo válidas são "ns", "us" (ou "µs"), "ms", "s", "m", "h".
    ## Por exemplo, "60s", "2h45m", "168h" (1 semana)
    userSessionTTL: 168h
    ## Se houver usuários que devem receber privilégios de administrador no Brigade
    ## a partir do momento em que se autenticam, devem ser listados aqui. Por
    ## instância, o operador que está fazendo a implantação e/ou o usuário que irá
    ## ser responsável por atribuir permissões de autorização para autenticados
    ## usuários.
    ## Caso contrário, as permissões precisarão ser configuradas manualmente para cada
    ## usuário autenticado.
    admins:
      - myemail@example.com
```

Observe que o parâmetro `strategy` deve ser definido como `oidc`. O tempo de vida padrão para as sessões do usuário (definido no exemplo acima) é de 1 semana/168 horas. Por último, o campo `admins` é uma lista opcional de endereços de e-mail para usuários que devem ter privilégios totais de administrador no primeiro login no Brigade.

Nota: como esses privilégios são concedidos no primeiro login, adicionar/revogar essas permissões para usuários que já efetuaram login pela primeira vez devem ser feito via brig CLI diretamente em vez de reimplantar com outra
configuração.

[values.yaml]: https://github.com/brigadecore/brigade/blob/main/charts/brigade/values.yaml

### Provedor GitHub OAuth

Para configurar a autenticação do GitHub com o Brigade, você criará um [GitHub OAuth App] com a URL de retorno da chamada de Autorização apontando para `/v2/session/auth` no endereço do servidor de API do Brigade, por exemplo, o nome do host reservado ao configurar a entrada ou o endereço IP externo se nenhuma entrada for usada. Em seguida, você usará o Client ID do GitHub OAuth Apps e um Client Secret gerado no arquivo [values.yaml] para o Brigade Helm Chart.

  1. Siga as instruções de criação do [GitHub OAuth App], fornecendo as opções de valores para `Nome do aplicativo`, `URL da página inicial` e `Descrição do aplicativo`.
  1. Para o `URL de retorno da chamada de Autorização`, você fornecerá um valor com base no nome do host DNS e no caminho mencionados acima. Por exemplo, se nome do host DNS para o servidor de API do Brigade é `mybrigade.example.com`, o valor será:
      ```
      https://mybrigade.example.com/v2/session/auth
      ```
  1. Clique em 'Registrar aplicativo'
  1. Na página de configurações do aplicativo, haverá uma string de `Client ID`. Você vai usar esse valor em breve.
  1. Em `Client secrets`, clique em `Generate a new client secret`
  1. Salve o valor de `Client Secret`, pois ele não será exibido novamente
  1. Clique em 'Update application'

Agora que você tem valores do Client ID e Client Secret do aplicativo GitHub OAuth, você está pronto para atualizar os valores no arquivo do Brigade Helm Chart com esta configuração de autenticação.

Os valores estarão na seção `apiserver.thirdPartyAuth`, que foi mostrada aqui, com outras seções omitidas:

```yaml
apiserver:
  ## Opções para autenticação por meio de um provedor de autenticação de terceiros.
  thirdPartyAuth:
    ## Aqui configuramos a estratégia GitHub (github)
    strategy: github
    ## Aqui fornecemos os valores do GitHub OAuth App
    github:
      clientID: <client ID>
      clientSecret: <client secret>
      ## Se apenas usuários de organizações específicas do GitHub devem ser permitidos
      ## para autenticar, liste-os aqui. Caso contrário, os usuários de qualquer organização
      ## do GitHub poderá tentar autenticar.
      allowedOrganizations:
    ## O TTL da Sessão do Usuário determina o tempo de vida padrão para as sessões do usuário.
    ## As unidades de tempo válidas são "ns", "us" (ou "µs"), "ms", "s", "m", "h".
    ## Por exemplo, "60s", "2h45m", "168h" (1 semana)
    userSessionTTL: 168h
    ## Se houver usuários que devem receber privilégios de administrador no Brigade
    ## a partir do momento em que se autenticam, devem ser listados aqui. Por
    ## instância, o operador que está fazendo a implantação e/ou o usuário que irá
    ## ser responsável por atribuir permissões de autorização para autenticados
    ## usuários.
    ## Caso contrário, as permissões precisarão ser configuradas manualmente para cada
    ## usuário autenticado.
    admins:
      - <GitHub username>
```


[GitHub OAuth App]: https://docs.github.com/pt/developers/apps/creating-an-oauth-app
[Configuring Github Authentication]: /topicos/operadores/deploy#configuring-github-authentication
