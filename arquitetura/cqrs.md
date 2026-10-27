# CQRS (Command Query Responsibility Segregation)

O *CQRS* (Segregação de Responsabilidade entre Comando e Consulta) é um padrão arquitetural que separa as operações de leitura (consultas) das operações de escrita (comandos) de um sistema.

Em arquiteturas tradicionais (CRUD), o mesmo modelo de dados é utilizado tanto para atualizar quanto para consultar informações. O CQRS quebra essa premissa ao reconhecer que os requisitos de desempenho, consistência e modelagem para ler dados são fundamentalmente diferentes dos requisitos para alterá-los.

O padrão foi introduzido por Bertrand Meyer no contexto do princípio CQS (*Command Query Separation*) e posteriormente expandido para o nível arquitetural por Greg Young.

| __Conceito__ | __Tipo__          | __Objetivo Principal__         | __Característica do Modelo__                                        |
| ------------ | ----------------- | ------------------------------ | ------------------------------------------------------------------- |
| **Command**  | Escrita (*Write*) | Alterar o estado do sistema    | Mantém regras de negócio complexas, validações e transações ácidas  |
| **Query**    | Leitura (*Read*)  | Obter dados sem alterar estado | Modelo simplificado, otimizado para performance e consultas rápidas |

### O Mapa Mental do CQRS

Aqui está a essência do padrão, focada no seu impacto arquitetural:

- **Separação de Modelos (Command Model vs Query Model)**: O modelo de escrita foca na integridade do negócio e validações; o modelo de leitura foca na apresentação rápida dos dados.<br>`OBS: No modelo de consulta, evite junções (JOINs) complexas no banco de dados. Muitas vezes, os dados de leitura são armazenados já desnormalizados e prontos para a tela.`

- **Sincronização por Eventos**: Quando um Comando altera o estado no modelo de escrita, ele emite um evento (ex: `PedidoCriado`) que atualiza a base de leitura.<br>`OBS: Essa atualização assíncrona introduz a Consistência Eventual. O sistema de leitura pode levar alguns milissegundos para refletir a alteração feita no lado de escrita.`

- **Escalabilidade Assimétrica**: Você pode escalar a infraestrutura de leitura e de escrita de forma totalmente independente.<br>`OBS: Em sistemas reais, a proporção de leituras costuma ser de 10:1 ou 100:1 em relação às escritas. O CQRS permite colocar réplicas ou bancos ultra-rápidos (ex: Redis, ElasticSearch) apenas no lado das Queries.`

- **Sinergia com Event Sourcing**: Embora o CQRS funcione bem com bancos tradicionais, ele combina perfeitamente com *Event Sourcing*, onde a escrita guarda o histórico imutável de eventos e a leitura reconstrói a visão atual.<br>`OBS: CQRS e Event Sourcing são padrões distintos, mas costumam ser usados juntos em sistemas distribuídos complexos.`

### Exemplos

> O Código "Sem CQRS" (Modelo CRUD Tradicional)

Aqui, a mesma classe e banco de dados lidam tanto com regras complexas de atualização quanto com a montagem de relatórios/consultas extensas.

~~~c#
public class OrderService
{
    private readonly AppDbContext _context;

    public OrderService(AppDbContext context) { _context = context; }

    // Método de Escrita (Comando)
    public void CreateOrder(CreateOrderDto dto)
    {
        var order = new Order { CustomerId = dto.CustomerId, Total = dto.Total };
        _context.Orders.Add(order);
        _context.SaveChanges();
    }

    // Método de Leitura (Consulta) - Forçado a usar o mesmo modelo relacional complexo
    public OrderReportDto GetOrderSummaryForDashboard(int orderId)
    {
        // JOINs pesados e mapeamento complexo para atender à UI
        return _context.Orders
            .Include(o => o.Customer)
            .Include(o => o.Items)
            .ThenInclude(i => i.Product)
            .Where(o => o.Id == orderId)
            .Select(o => new OrderReportDto { /* ... */ })
            .FirstOrDefault();
    }
}
~~~

Por que este código apresenta limitações em escala?

- Acoplamento de Responsabilidades: Mudanças no modelo de dados de escrita impactam diretamente as consultas e a interface do utilizador.

- Gargalo de Performance: O banco de dados precisa atender a transações rigorosas e a consultas pesadas simultaneamente no mesmo esquema relacional.

- Dificuldade de Otimização: Não é possível otimizar a estrutura de dados separadamente para gravar rápido e ler rápido.

> O Código "Com CQRS" (Refatorado com MediatR)

Aqui, separamos completamente os fluxos em **Comandos** e **Consultas**, permitindo modelos e estratégias de persistência distintas.

~~~c#
// --- LADO DE ESCRITA (COMMAND) ---
public record CreateOrderCommand(int CustomerId, decimal Total) : IRequest<int>;

public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int>
{
    private readonly IOrderWriteRepository _repository;
    private readonly IEventBus _eventBus;

    public CreateOrderCommandHandler(IOrderWriteRepository repository, IEventBus eventBus)
    {
        _repository = repository;
        _eventBus = eventBus;
    }

    public async Task<int> Handle(CreateOrderCommand request, CancellationToken cancellationToken)
    {
        var order = new Order(request.CustomerId, request.Total);
        await _repository.SaveAsync(order);

        // Dispara evento para atualizar a base de leitura assincronamente
        await _eventBus.PublishAsync(new OrderCreatedEvent(order.Id, order.CustomerId, order.Total));
        return order.Id;
    }
}

// --- LADO DE LEITURA (QUERY) ---
public record GetOrderSummaryQuery(int OrderId) : IRequest<OrderReportDto>;

public class GetOrderSummaryQueryHandler : IRequestHandler<GetOrderSummaryQuery, OrderReportDto>
{
    private readonly IReadDatabase _readDb; // Pode ser NoSQL ou tabela desnormalizada

    public GetOrderSummaryQueryHandler(IReadDatabase readDb) { _readDb = readDb; }

    public async Task<OrderReportDto> Handle(GetOrderSummaryQuery request, CancellationToken cancellationToken)
    {
        // Consulta direta e ultra-rápida no modelo de leitura pré-calculado
        return await _readDb.FindByIdAsync<OrderReportDto>(request.OrderId);
    }
}
~~~

Por que este código é excelente?

- Alta Performance e Escalabilidade: A consulta não faz JOINs nem regras complexas; lê diretamente de uma visão otimizada para leitura.

- Baixo Acoplamento: Regras de negócio de escrita não afetam o contrato ou a estrutura de consulta da interface.

- Evolução Independente: Permite usar bancos relacionais no lado da escrita (para garantir transações) e bancos NoSQL/Cache no lado da leitura (para velocidade extrema).

`NOTA: O CQRS adiciona complexidade ao sistema. Só deve ser adotado quando o domínio de negócio for complexo ou quando houver uma diferença significativa entre o volume e os requisitos de leitura e escrita.`

### Material Complementar

- [CQRS Pattern - Microsoft Architecture Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [CQRS - Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [O que é CQRS e quando utilizá-lo na sua arquitetura](https://medium.com/@devpira/o-que-%C3%A9-cqrs-e-quando-usar-57c2a71f02f0)
