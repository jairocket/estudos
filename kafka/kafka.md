Kafka é uma plataforma distribuída de streaming de *eventos* voltada para sistemas de processamento de grandes volumes de dados em tempo real de forma escalável. 

### Eventos vs Comandos

Em uma arquitetura de microsserviços, um evento é um acontecimento que interessa um ou mais serviços que integram o sistema. Pode ser o cliente finalizando uma compra em um e-commerce. Nesse cenário, o evento `order-created` pode ser publicado em um tópico kafka e os serviços interessados se tornam responsáveis por reagir a esse evento (separar estoques, emitir nota fiscal, preparar entrega, etc).
Os comandos, por outro lado, são requisições para que um serviço execute uma ação específica. Por exemplo, separe o item x no estoque, emita nota fiscal de venda do item x, prepare a entrega do item x, etc.

Requisições http proveem um protocolo de comunicação síncrona em que serviços podem enviar comandos entre si. Aqui seguem alguns potenciais problemas que podem emergir:

#### Acoplamento

- Se o frontend fizer requests para cada um dos sistemas separadamente, várias dependências são adicionadas, o que dificulta a testabilidade. 
- O frontend precisa conhecer o contrato de todas as apis e a condicionando a implementação. Se um novo serviço for adicionado - como um sistema especializado em detectar fraudes, por exemplo - uma nova chamada precisa ser adicionada.

#### Resiliência

Falhas em requests http tem o potencial de interromper o fluxo ou de pular etapas importantes do processo, gerando a necessidade de desenvolver estratégias de fallback. Se a requisição para cadastro de nota fiscal falhar, pode ser necessário de fazê-lo manualmente, sob pena de repercuções legais que nosso e-commerce deseja evitar.

### Kafka

De forma simplificada, o serviço `produtor` publica um log de um evento relevante em um tópico. kafka persiste este registro para que os serviços `consumidores` possam lê-los e reagir conforme necessário. Os tópicos são repositórios desses logs de evento que são persistidos em servidores chamados de brokers, os quais se organizam em aglomerados chamados de kafka clusters. 
Em sua essência, o kafka implementa o padrão publisher/subscriber. Isso permite que quando um cliente finalizar uma compra no e-commerce, no lugar de enviar comandos separados para reservar o produto no estoque e emitir nota fiscal, um evento com todos os detalhes relacionados à compra e os serviços downstream consumam essa mensagem de forma paralela. Agora, o frontend não depende mais dos serviços interessados no evento de compra.
 
### Producers

### Brokers

