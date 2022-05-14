---
title: Design
description: Arquitetura e Design do Brigade
section: topicos
weight: 1
aliases:
  - /design.md
  - /topicos/design.md
---

Esse documento descreve a funcionalidade de cada principal componente do Brigade e 
como eles se integram entre si para prover uma plataforma de scripting baseada em 
eventos. Aqui você também encontra contexto histórico de como os mantenedores do Brigade chegaram a essa arquitetura atual.

## Terminologia

Vamos começar com uma descrição neutra da implementação dos conceitos mais
importantes do Brigade. Esta é uma introdução inicial dos tópicos que são explicados
em mais detalhes em outras partes desta documentação.

### Eventos

__Eventos__ se originam em sistemas externos e são enviados para o Brigade
[event bus](#o-event-bus) através dos [gateways](#event-gateways) e da
[API do Brigade](#o-servidor-de-api).

![Events](/img/design-events.png)

### Projetos

__Projetos__ são definidos pelo usuário. Eles fazem o link entre subscrever a eventos com configurações e scripts que iram processar esses eventos.

![Projetos](/img/design-projects.png)

### O Ciclo de vida de um Evento

Quando um novo [evento](#eventos) é recebido no [API server](#o-servidor-de-api)
do Brigade, uma cópia única do evento será adicionada na fila do [event bus](#o-event-bus)
para cada [projeto](#projetos) que subscreveu ao evento.

Se existir capacidade disponivel no Cluster, e seguindo a configuração do projeto, um [worker](#workers)
será executado para processar cada evento, utilizando o script definido no projeto.

## Arquitetura Básica

Os itens a seguir retratam os principais componentes do Brigade:

![Arquitetura](/img/design-architecture.png)

As seções a seguir descrevem cada um dos componentes em mais detalhes. Eles
são apresentados numa ordem que visa acelerar o entendimento da arquitetura
básica por usuários recém-chegados ao Brigade.

### A execução dos Workloads

O slogan do Brigade vem sendo por muito tempo "scripting baseado em eventos para Kubernetes,"
mas desde a versão v2, Kubernetes é efetivamente um detalhe na implementação.

Com a exceção de casos mais avançados onde um script do Brigade pode modificar
o cluster Kubernetes aonde ele esta rodando, e por este motivo talvez necessite
configurações especificas no cluster, usuários finais do Brigade não interage 
diretamente com o cluster Kubernetes. Em vez disso, usuários interagem com o Brigade
utilizando a própria API do Brigade. (Normalmente utlizando CLI ou outra ferramenta)

O [Servidor da API](#o-servidor-de-api) trata Kubernetes como um dos seus vários
componentes de back-end. Especificamente, ele utiliza Kubernetes para executar
seus "workloads". Em outras palavras, Kubernetes é usado para hospedar os ambientes
containerizados na qual ele executa os scripts dos usuários.

Apesar de os mantenedores do projeto do Brigade não ter planos imediatos para dar
suporte a formas alternativas de implantar o Brigade, fora o Kubernetes, não valeria
em nada se o Brigade tivesse sido arquitetado de uma forma que impedisse isso de
acontecer no futuro.

### O Servidor de API

Todas as mais importantes funções do Brigade são abstraídas por uma API RESTful.
O __Servidor de API__ implementa e hospeda essa API.

Outros componentes do Brigade interagem com os sistemas de  back-end como o
[Banco de Dados](#o-banco-de-dados) ou [Execução dos Workloads](#a-execução-dos-workloads)
são normalmente abstraídos pela API. Entretanto, a maioria dos componentes depende
do Servidor de API.

É justo considerar o Servidor de API como o "cérebro" do Brigade.

### O Banco de Dados

O [Servidor de API](#o-servidor-de-api) utiliza o [MongoDB](https://www.mongodb.com/)
para manter os registros. Isso inclui, mas não é limitado a isso, armazenando
projetos definidos pelos usuários, eventos, e até logs gerados pelos scripts dos usuários.

### O Event Bus

O [Servidot de API](#o-servidor-de-api) utiliza o
[Apache ActiveMQ Artemis](https://activemq.apache.org/components/artemis/),a-execução-dos-workloads
o qual implementa AMQP 1.0 (Advanced Message Queueing Protocol), como seu event bus
interno. Uma fila por projeto do Brigade garante que os eventos para cada projeto sejam
processados de uma forma FIFO (first in, first out), ou seja primeiro a ser recebido, será
o primeiro a ser processado.

### Gateways de Eventos

__Gateways de Eventos__ são componentes periféricos instaldos separadamente do
Brigade. Sua função é de receber eventos de sistemas externos, transformar este eventos
em eventos do _Brigade_, e utilizar a API do Brigade para adicioná-los na fila do
[event bus](#o-event-bus) do Brigade.

### O Agendador

O componente __Agendador__ escuta a fila de cada projeto na [event bus](#o-event-bus)
e também utiliza a API do Brigade para monitorar a capacidade de
[Execução dos Workloads](#a-execução-dos-workloads). O agendador aloca a capacidade
disponível, através de chamados a API do Brigade, para executar [workers](#workers)
que manipulam o processamento de cada evento.

Quando a demanda para executar os workloads supera a demanda disponível no cluster,
o agendador começa a alocar randomicamente quando a capacidade se tornar disponível
no cluster. Isso proporciona ao agendador do Brigade uma forma justa de agendamento
dos eventos e impede projetos com maior número de eventos de monopolizar a capacidade
de execução de workloads.

Observe que a operação do gerenciador não pode escalar horizontalmente. Separando
essa operação do Servidor de API e fazendo sua implementação como um micro serviço
independente garante que a implantação do Brigade possa restringir-se a uma única
instância do gerenciador e ao mesmo tempo permita que o Servidor de API possa
escalar horizontalmente.

### O Observador

A componente __Observador__ tem a função de diretamente monitorar workloads durante a
[Execução dos Workloads](#a-execução-dos-workloads) e depois informar o seu estado
através de uma chamado a API do Brigade.

Quando o observador identifica que o workload foi processado, ele é também responsável
por finalizar a sua execução após um cruto periodo de carência(grace period). Isto não
é feito diretamente, mas novamente utilizando uma chamado ao servidor de API. O propósito
dessa carrência é melhorar a semelhança que agents de log capturam _all_ logs produzidos
pelos scripts dos usuários _antes_ que os [workers](#workers) que  execute them are deleted forever.

As with the [scheduler](#the-scheduler), the observer function cannot be scaled
horizontally. Decoupling this function from the API server and implementing it
as its own microservice ensures that deployments of Brigade can constrain
themselves to a single instance of the observer component whilst still
permitting the API server to scale horizontally.

### Workers

A __worker__ is a containerized script execution environment that is launched
on the [Execução dos Workloads](#a-execução-dos-workloads) to handle an event.
Because that underlying substrate is Kubernetes, a worker is implemented as a
[Kubernetes pod](https://kubernetes.io/docs/concepts/workloads/pods/).

The default worker Docker image can execute scripts written in
[JavaScript](https://en.wikipedia.org/wiki/JavaScript) or
[TypeScript](https://www.typescriptlang.org/) using
[Node.js](https://nodejs.org/). It can also resolve any of a script's
dependencies using either
[npm](https://nodejs.org/en/knowledge/getting-started/npm/what-is-npm/) or
[yarn](https://yarnpkg.com/).

It is also possible, through project configuration, to use workers based on an
alternative Docker image. Such images could provide support for handlers that
are defined using alternative scripting languages or even a declarative syntax.

### Log Agents

Brigade utilizes an instance of [Fluentd](https://www.fluentd.org/) per
Kubernetes node to scrape logs from [worker](#workers) containers and forward
them to the [Banco de Dados](#o-banco-de-dados). This ensures that logs produced by users'
scripts are persisted beyond the short lifetime of the workers that run them.

## History of Brigade

Like its sister project, [Helm](https://helm.sh/), Brigade was born at Deis
prior to the company's eventual acquisition by Microsoft. Both projects were the
product of the same process. Viewing Kubernetes as a sort of "cluster operating
system," the team at Deis sought to identify features of a traditional operating
system that are often taken for granted, but were conspicuously absent from
Kubernetes. Kubernetes lacked a package manager and that directly led to the
inception of Helm to close that gap. Similarly, Kubernetes lacked a scripting
environment -- and that is the gap that Brigade was designed to close.

Brigade reached a stable, 1.0 release in March 2019 and was donated to
[CNCF](https://www.cncf.io/) at that time, becoming a
[sandbox project](https://www.cncf.io/sandbox-projects/). Four minor releases
followed.

By early 2020, Brigade had proven reasonably popular among among
[CNCF survey](https://www.cncf.io/wp-content/uploads/2020/08/CNCF_Survey_Report.pdf)
respondents. Popular though it may have been, maintainers had also accumulated
enough feedback from the community to recognize some course corrections were
required to position the project for greater success. Most insights led back to
a fundamental need to better abstract Brigade users from the underlying
complexities of Kubernetes. A formal proposal for Brigade v2 was drafted by
maintainers and ratified by the community. Over the following two years, Brigade
was re-written from the ground up. Brigade v2 was released December 1, 2021.