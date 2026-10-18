# Dual Write Problem

A consistência dos dados em um software é requisito fundamental. A criticidade da tempestividade dessa consistência pode variar de acordo com o problema que a aplicação pretende resolver. Todavia, é algo que não deveria ser negligenciado na fase de planejamento. Dentre os desafios que podemos enfrentar para garantir essa consistência, destacamos o Dual Write Problem. Para compreender esse conceito, é preciso conhecer o contexto em que ele se manifesta. Para tanto, o caminho que me parece mais simples é entender a diferença entre sistemas monolíticos e microsserviços.

## Sistemas Monolíticos

Imagine uma aplicação que gerencia a venda de planos de internet que possui interface do usuário, um solução de autenticação, o sistema de cobranças, o backend enforça as regras de negócio e persiste as informações no banco de dados. Esses subsistemas interagem entre si para garantir o pleno funcionamento do nosso sistema imaginário. Em um sistema monolítico, todos esses subsistemas vivem na mesma aplicação e compartilham o mesmo banco de dados.

![alt text](image.png)

Esse tipo de arquitetura tende a ser menos complexa de ser construída, o que permite entrega de valor mais rápida. Contudo, tende a deixar a desejar quando o quesito é resiliência. Se houver um problema em qualquer dos módulos,toda a aplicação deixa de funcionar.

## Microsserviços

Se o mesmo sistema tivesse sido construído com uma arquitetura baseada em microsserviços, cada um dos módulos constituiria uma aplicação especializada, com seu próprio banco de dados dedicado. Isso permite que essas aplicações se comuniquem assincronamente, contribuindo para melhorar a resiliencia do sistema como um todo.

### Comunicação Síncrona

Na comunicação síncrona, um serviço A envia uma requisição diretamente a um serviço B, que processa o dado e retorna uma resposta ao serviço A. Por exemplo, no nosso exemplo, quando o usuário for se autenticar no sistema, a interface poderia enviar as credenciais do usuário via requisição http para o sistema responsável pela autenticação. Este última faz todas as validações e devolve um http response reportando o sucesso ou a falha na autenticação.
Perceba que o serviço que requisita precisa esperar a resposta do serviço requisitado. Se o serviço requisitado estiver fora do ar, a requisição não será processada. 

### Comunicação Assíncrona

Em contraste, na comunicação assíncrona, o serviço requisitante não precisa conhecer os serviços que processam os dados. Utilizando o padrão publisher/subscriber, um serviço A publica uma mensagem em um [message broker](https://devskin.com/post/message-broker-comparativo-tcnico-entre-kafka-rabbitmq-e-sqs) (como kafka, aws sqs/sns, RabbitMQ, etc) e os serviços B, C - ou o próprio serviço A - consomem a mensagem e executa o processamento necessário.
Nesse cenário, se uma mensagem for publicada no broker enquanto algum serviço consumidor estiver fora do ar, o processamento poderá ocorrer quando o serviço voltar a funcionar. Isso traz maior resiliência ao sistema.
Em nosso sistema, o serviço de compras pode estar configurado para validar se a compra é válida, salvar a compra no banco de dados e publicar uma mensagem em um tópico kafka `internet-plan-sold-topic`. A aplicação responsável pelas ordens de serviço consome esse tópico e cria uma nova requisição pendente para o time operacional realizar a instalação da internet do cliente que efetuou a compra.
A comunicação assíncrona traz a complexidade implementar, implementar e manter o message broker.

-- Imagem --

## Dual Write Problem

Mas o que acontece quando o serviço de compras salva no seu banco de dados a requisição mas a publicação no message broker falha? Exatamente, os dados ficam inconsistentes. No nosso exemplo, a compra será registrada mas o time responsável por instalar a internet para o cliente, gerando atritos indesejáveis, podendo gerar crise de reputação. 
O problema de dual write ocorre porque não é possível implementar um fluxo que salve a mesma informação em dois sistemas diferentes de forma atômica: não tem como apagar o registro da compra porque a publicação do tópico falhou. Felizmente, existem algumas formas de mitigar esse problema.

### Outbox Pattern

O padrão outbox consiste em o serviço salvar a mensagem que precisa ser publicada em uma tabela separada (outbox). Em paralelo, o serviço salva os dados processados em seu própria tabela. Um processo separado detecta novos registros na tabela outbox, recupera os dados da mensagem e a publica no message broker. Isso garante que a consistência dos dados mesmo que o broker esteja fora do ar: Assim que voltar a funcionar, as mensagens voltarão a ser publicadas.
Um ponto importante é que esse padrão garante a publicação da mensagem pelo menos uma vez. Por isso é preciso garantir a idempotencia do lado dos serviços que consomem as mensagens.

### Event Sourcing

Consiste em persistir os eventos e não o estado atual de determinada entidade. Confuso? Imagine que no nosso exemplo, a aplicação que gera a ordem de serviço é responsável também por seu ciclo de vida:
* o consumo do tópico `internet-plan-sold-topic` gera uma ordem de serviço com status `PENDING`
* Quando atribuída a um colaborador, o status passa a ser `EXECUTING`
* Quando a internet é disponibilizada para o cliente, o status passa a ser `FINISHED`.

Tradicionalmente, persistimos no banco de dados a ordem com status `PENDING` e apenas atualizamos o status do registro quando necessário. Em Event Sourcing, salvamos todos os eventos e ao final do ciclo de vida da ordem de serviço teremos três registros no banco, um para cada status. Um outro processo publica as mensagens posteriormente.

Isso permite que guardar detalhes de cada evento. Bancos são um bom exemplo de caso de uso para event sourcing.

### Listen to Yourself

Uma outra possibilidade é o serviço publicar a mensagem no broker, consumir a mensagem e salvar no seu banco de dados. 

#### Qual escolher?

A solução depende da situação concreta que que se pretende resolver. O padrão outbox traz a complexidade de manter duas tabelas. Event Sourcing te obriga a calcular o estado atual a partir do último evento. 
Algo importante a se considerar é o conceito de conceito de consistência eventual. Essas estratégias garantem a consistência dos dados. Problemas de rede ou lag no message broker pode levar a um atraso na sincronização entre  microsserviços. Quando o problema for solucionado, a consistência dos dados será reestabelecida.