# Event-Driven Architecture (EDA)

A *Event-Driven Architecture* (Arquitetura Orientada a Eventos) é um padrão de design de software em que a comunicação entre os componentes do sistema é impulsionada pela emissão, deteção e consumo de **eventos**.

Nesta arquitetura, os componentes reagem a mudanças relevantes de estado em tempo real de forma assíncrona. Em vez de chamadas diretas e síncronas entre serviços, os produtores apenas publicam o que aconteceu e os consumidores reagem de forma independente.

O padrão é amplamente adotado em ecossistemas de microsserviços distribuídos e em plataformas de alto alto desempenho e grandes volumes de dados, como o Apache Kafka.

| __Componente__                | __Papel Principal__            | __Característica__                                             | __Exemplos / Ferramentas__                    |
| ----------------------------- | ------------------------------ | -------------------------------------------------------------- | --------------------------------------------- |
| **Produtor**                  | Emite a notificação do fato    | Não sabe quem irá consumir nem o que será feito com o evento   | Aplicações web, microsserviços, sensores      |
| **Canal de Eventos / Broker** | Intermediário e roteador       | Recebe, armazena, organiza em partições e distribui os eventos | Apache Kafka, AWS EventBridge, RabbitMQ       |
| **Consumidor**                | Reage aos eventos de interesse | Processa as mensagens de forma assíncrona e independente       | Serviços de notificação, motores de analytics |

### O Mapa Mental da Arquitetura Orientada a Eventos

Aqui está a essência da arquitetura, focada no seu impacto em sistemas distribuídos:

- **Desacoplamento e Comunicação Assíncrona**: Produtores e consumidores não dependem diretamente uns dos outros. O produtor publica o evento no canal e continua a sua execução sem esperar pela resposta do consumidor.

- **Consistência Eventual**: Como o processamento ocorre em segundo plano, as atualizações de estado podem levar um pequeno intervalo de tempo para se refletirem em todos os serviços do sistema.

- **Idempotência no Consumo**: Devido a potenciais falhas de rede ou reenvios pelo broker, o mesmo evento pode ser entregue mais do que uma vez. O consumidor deve ser projetado para reprocessar a mensagem sem gerar efeitos colaterais duplicados (ex: não debitar uma conta duas vezes).

- **Contratos e Governação de Schemas**: Para garantir que produtores e consumidores troquem dados sem erros de compatibilidade, utiliza-se a gestão de schemas (ex: Apache Avro com Schema Registry) para versionar e validar a estrutura dos eventos.

### Classificação de Mensagens em EDA

Ao projetar tópicos num barramento como o Kafka, as mensagens costumam ser classificadas segundo a sua intenção:

- **Evento Corporativo**: Representa um facto do passado (ex: `PedidoCriado`). É publicado via Publish/Subscribe e pode ser consumido por múltiplos subdomínios.
- **Comando**: Representa uma ordem para executar uma ação específica. É direcionado a um consumidor específico (ex: `AverbarRefinanciamento`).
- **Data Transfer**: Utilizado para mover dados de um lugar para outro entre componentes do mesmo subdomínio (ex: integração Mainframe para AWS).

### Exemplos

> O Código "Sem EDA" (Comunicação Síncrona e Acoplada)

Aqui, o serviço de pedidos chama diretamente múltiplos serviços externos via HTTP REST. Se um dos serviços falhar ou estiver lento, todo o fluxo de criação do pedido é bloqueado.

~~~c#
public class OrderService
{
    private readonly HttpClient _httpClient;

    public OrderService(HttpClient httpClient) { _httpClient = httpClient; }

    public async Task CreateOrderAsync(Order order)
    {
        // Save no banco local
        SaveOrder(order);

        // Chamadas síncronas em cadeia (Alto Acoplamento / Ponto Único de Falha)
        await _httpClient.PostAsJsonAsync("https://inventory-service/reserve", order);
        await _httpClient.PostAsJsonAsync("https://payment-service/process", order);
        await _httpClient.PostAsJsonAsync("https://notification-service/email", order);
    }
}
~~~

Por que este código apresenta limitações em escala?

- Alto Acoplamento: O `OrderService` precisa de conhecer os endereços e interfaces de todos os outros serviços.

- Baixa Resiliência: Se o serviço de e-mail estiver temporariamente indisponível, a criação do pedido falhará ou ficará travada.

- Desempenho Comprometido: A latência total é a soma de todas as chamadas HTTP síncronas.

> O Código "Com EDA" (Publicação Assíncrona de Eventos)

Aqui, o `OrderService` apenas persiste o pedido e publica um evento imutável no barramento de mensagens. Os demais serviços reagem de forma 100% assíncrona.

~~~c#
// Definição do Evento Corporativo
public record OrderCreatedEvent(Guid OrderId, string CustomerId, decimal Amount, DateTime CreatedAt);

public class OrderService
{
    private readonly IEventBus _eventBus; // Interface para o Kafka / Broker

    public OrderService(IEventBus eventBus) { _eventBus = eventBus; }

    public async Task CreateOrderAsync(Order order)
    {
        // 1. Salva o estado local do pedido
        SaveOrder(order);

        // 2. Emite o evento e finaliza a requisição imediatamente (Baixa Latência)
        var @event = new OrderCreatedEvent(order.Id, order.CustomerId, order.Amount, DateTime.UtcNow);
        await _eventBus.PublishAsync("pedidos-evento-pedido-criado", @event);
    }
}

// Consumidor Independente em outro Microsserviço
public class NotificationConsumer
{
    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        // Reage ao evento no seu próprio tempo, garantindo idempotência
        await SendEmailConfirmationAsync(@event.CustomerId, @event.OrderId);
    }
}
~~~

Por que este código é excelente?

- Baixo Acoplamento e Extensibilidade: Novos serviços (ex: Análise de Fraude) podem passar a escutar o tópico `pedidos-evento-pedido-criado` sem alterar uma única linha do `OrderService`.

- Alta Resiliência: Se o consumidor de notificações cair, o barramento retém as mensagens e o processamento é retomado assim que o serviço voltar a estar online.

- Desempenho e Escalabilidade: A resposta ao utilizador é imediata, e os consumidores podem ser escalados horizontalmente de acordo com a carga.

`NOTA: A adoção de EDA exige ferramentas adequadas de observabilidade, tratamento de erros (com Dead Letter Queues) e monitorização do fluxo de mensagens entre sistemas distribuídos.`

### Material Complementar

- [Event-Driven Architecture Style - Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
- [Apache Kafka Documentation & Core Concepts](https://kafka.apache.org/documentation/)
- [What is Event-Driven Architecture? - AWS Architecture Center](https://aws.amazon.com/event-driven-architecture/)
