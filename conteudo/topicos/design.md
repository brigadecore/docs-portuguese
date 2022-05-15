---
title: Design
description: Arquitetura e Design do Brigade
section: Tópicos
weight: 1
aliases:
  - /design.md
  - /topicos/design.md
---

Esse documento descreve a funcionalidade de cada um dos principais componente do Brigade e 
como eles se integram entre si para prover uma plataforma de scripting baseada em 
eventos. Aqui você também encontra contexto histórico de como os mantenedores do Brigade chegaram a essa arquitetura atual.

## Terminologia

Vamos começar com uma descrição neutra da implementação dos conceitos mais
importantes do Brigade. Esta é uma introdução inicial dos tópicos que são explicados
em mais detalhes em outras partes desta documentação.

### Eventos

__Eventos__ se originam em sistemas externos e são enviados para o Brigade
[event bus](#o-event-bus) através dos [gateways](#gateways-de-eventos) e da
[API do Brigade](#o-servidor-de-api).

![Eventos](/img/design-events.png)

### Projetos

__Projetos__ são definidos pelo usuário. Eles fazem o link entre subscrever a eventos com configurações e scripts que vão processar esses eventos.

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

### O ambiente de execução dos Workloads

O slogan do Brigade vem sendo por muito tempo "scripting baseado em eventos para Kubernetes,"
mas desde a versão v2, Kubernetes é efetivamente um detalhe na implementação.

Com a exceção de casos mais avançados onde um script do Brigade pode modificar
o cluster Kubernetes aonde ele esta rodando, e por este motivo talvez necessite
configurações especificas no cluster, usuários finais do Brigade não interagem 
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

Outros componentes do Brigade interagem com os sistemas de back-end como o
[Banco de Dados](#o-banco-de-dados) ou [O ambiente de Execução dos Workloads](#o-ambiente-de-execução-dos-workloads)
são normalmente abstraídos pela API. Entretanto, a maioria dos componentes depende
do Servidor de API.

É justo considerar o Servidor de API como o "cérebro" do Brigade.

### O Banco de Dados

O [Servidor de API](#o-servidor-de-api) utiliza o [MongoDB](https://www.mongodb.com/)
para manter os registros. Isso inclui, mas não é limitado a isso, armazenamento dos
projetos definidos pelos usuários, eventos, e até logs gerados pelos scripts dos usuários.

### O Event Bus

O [Servidot de API](#o-servidor-de-api) utiliza o
[Apache ActiveMQ Artemis](https://activemq.apache.org/components/artemis/),
o qual implementa AMQP 1.0 (Advanced Message Queueing Protocol), como seu event bus
interno. A utilização de uma fila por projeto do Brigade garante que os eventos para cada
projeto sejam processados de uma forma FIFO (first in, first out), ou seja primeiro a ser ]
recebido, será o primeiro a ser processado.

### Gateways de Eventos

__Gateways de Eventos__ são componentes periféricos instalados separadamente do
Brigade. Sua função é de receber eventos de sistemas externos, transformar estes eventos
em eventos do _Brigade_, e utilizar a API do Brigade para adicioná-los na fila do
[event bus](#o-event-bus).

### O Agendador

O componente __Agendador__ escuta a fila de cada projeto na [event bus](#o-event-bus)
e também utiliza a API do Brigade para monitorar a capacidade que 
[O Ambiente de Execução dos Workloads](#o-ambiente-de-execução-dos-workloads) tem disponível.
O agendador aloca a capacidade disponível, através de chamadas a API do Brigade, para executar
[workers](#workers) que manipulam o processamento de cada evento.

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

A componente __Observador__ tem a função direta de monitorar os workloads que 
[o Ambiente de Execução dos Workloads](#o-ambiente-de-execução-dos-workloads) esta
executando e depois informar o seu estado através de uma chamada a API do Brigade.

Quando o observador identifica que o workload foi processado, ele é também responsável
por finalizar a sua execução após um curto período de carência(grace period). Isto não
é feito diretamente, mas novamente utilizando uma chamada ao servidor de API. O propósito
dessa carrência é melhorar a semelhança que agentes de log capturam _all_ logs produzidos
pelos scripts dos usuários _antes_ que os [workers](#workers) que executam os scripts sejam
deletados para sempre.

Da mesma forma que o [agendador](#o-agendador), o componente do observador não pode ser
escalado horizontalmente. Separando essa operação do Servidor de API e fazendo sua
implementação como um micro serviço independente garante que a implantação do Brigade
possa restringir-se a uma única instância do observador e ao mesmo tempo permita que o
Servidor de API possa escalar horizontalmente.

### Workers

Um __worker__ é um ambiente de execução de scripts containerizado que
[O Ambiente de Execução dos Workloads](#o-ambiente-de-execução-dos-workloads) inicia
para manipular o evento. Pelo fato de o ambiente de execução ser o Kubernetes, o worker
é implementado como um [Kubernetes pod](https://kubernetes.io/docs/concepts/workloads/pods/).

O imagem docker padrão do worker pode executar scripts escritos em
[JavaScript](https://en.wikipedia.org/wiki/JavaScript) ou
[TypeScript](https://www.typescriptlang.org/) usando
[Node.js](https://nodejs.org/). Podendo também resolver qualquer dependência dos
scripts usando
[npm](https://nodejs.org/en/knowledge/getting-started/npm/what-is-npm/) ou
[yarn](https://yarnpkg.com/).

Através da configuração do projeto, é possível usar workers baseados numa imagem docker
alternativa. Estas images poderiam prover funcionalidade capaz de manipular scripts denifidos
usando linguagens de alternativas or até usando uma syntax declarativa.

### Agentes de Log

Brigade utiliza uma instância do [Fluentd](https://www.fluentd.org/) em cada node do
cluster Kubernetes para recolher logs dos containers utilizados pelo [worker](#workers) 
e encaminhá-los para o [Banco de Dados](#o-banco-de-dados). Isto garante que logs produzidos 
pelos scripts do usuário são persistidos, e podem ser utilizados mesmo com os containers dos 
workers tendo uma vida útil muito curta após sua execução.

## História do Brigade

Da mesma forma que o seu projeto irmão [Helm](https://helm.sh/), Brigade nasceu na Deis
antes de sua eventual aquisição pela Microsoft. Ambos projetos foram resultado
do mesmo processo e visualizam Kubernetes como um tipo de "sistema operacional
do cluster". O time da Deis procurou identificar funcionalidades dos sistemas
operacionais tradicionais que são muitas vezes consideradas padrão ou básicas,
mas visivelmente não fazem parte do Kubernetes. Kubernetes não possui
um gerenciador de pacotes e isso foi o que nos levou a criar o Helm, para cobrir
este vazio no mercado. Da mesma forma, Kubernetes não possui um ambiente de
scripting -- e este é o vazio que o Brigade foi desenhado para cobrir no mercado.

Brigade alcançou sua versão estável em Março de 2019 com o lançamento da v1, tendo sido doado a
[CNCF](https://www.cncf.io/) nesta época, se tornando um
[sandbox project](https://www.cncf.io/sandbox-projects/). Quatro outros lançamentos menores
aconteceram após isso.

No começo de 2020, Brigade se provou razoavelmenre popular entre entrevistados da
[pesquisa da CNCF](https://www.cncf.io/wp-content/uploads/2020/08/CNCF_Survey_Report.pdf).
Tendo sido popular, mantenedores tabém acumularam suficiente feedback da comunidade para
reconhecer que algumas correcões seriam necessárias para que o projeto tivesse ainda mais
sucesso. A maioria dos insights coletados mostraram a necessidade de encontrar uma melhor
forma de abstrair os usuários do Brigade da complexidade do seu ambiente de execução, no
caso o Kubernetes. Uma proposta formal para o Brigade V2 foi elaborada pelos mantenedores
e ratificada pela comunidade. Nos últimos dois anos, Brigade foi re-escrito do zero. Brigade
v2 foi lançado em 1 de December de 2021.