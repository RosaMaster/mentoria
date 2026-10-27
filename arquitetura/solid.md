# SOLID

O *SOLID* é a base fundamental para qualquer arquiteto ou desenvolvedor que almeja construir sistemas de alta qualidade, legíveis e sustentáveis.

Não é apenas um conjunto de regras, mas uma filosofia de design focada na Orientação a Objetos que visa tornar o código fácil de entender, manter, testar e estender.

O nome SOLID é um acrônimo cunhado por Michael Feathers no início dos anos 2000, baseado nos princípios compilados e popularizados por Robert C. Martin (conhecido como "Uncle Bob") em seu livro *Agile Software Development, Principles, Patterns, and Practices*.

| __Letra__ | __Sigla__ | __Princípio (em inglês)__       | __Tradução__                         | __Foco Principal__                               |
| --------- | --------- | ------------------------------- | ------------------------------------ | ------------------------------------------------ |
| **S**     | SRP       | Single Responsibility Principle | Princípio da Responsabilidade Única  | Coesão e isolamento de razões para mudança       |
| **O**     | OCP       | Open/Closed Principle           | Princípio Aberto/Fechado             | Extensibilidade sem alteração de código fonte    |
| **L**     | LSP       | Liskov Substitution Principle   | Princípio da Substituição de Liskov  | Substitutibilidade e integridade do polimorfismo |
| **I**     | ISP       | Interface Segregation Principle | Princípio da Segregação de Interface | Interfaces coesas e específicas por cliente      |
| **D**     | DIP       | Dependency Inversion Principle  | Princípio da Inversão de Dependência | Desacoplamento através de abstrações             |

### O Mapa Mental do SOLID

Aqui está a essência de cada letra, focada no impacto arquitetural:

- **S (Single Responsibility Principle - SRP)**: Uma classe ou módulo deve ter apenas uma razão para mudar.<br>`OBS: O SRP é o maior inimigo do "código espaguete". Se uma classe faz o cálculo de um imposto, a validação de um dado e salva no banco de dados, ela está fadada a quebrar quando qualquer uma dessas três regras mudar.`

- **O (Open/Closed Principle - OCP)**: Você deve ser capaz de estender o comportamento de uma classe sem precisar modificar o código-fonte dela.<br>`OBS: A chave aqui é o uso de interfaces ou classes abstratas. Se você vive editando uma classe para adicionar novos comportamentos (if/else ou switch intermináveis), você violou o OCP.`

- **L (Liskov Substitution Principle - LSP)**: Classes derivadas devem ser substituíveis por suas classes base sem quebrar o comportamento do sistema.<br>`OBS: Se você precisa checar o tipo de um objeto (if (x is Y)) ou lançar NotImplementedException em métodos herdados, você violou o LSP. O polimorfismo deve ser transparente.`

- **I (Interface Segregation Principle - ISP)**: É melhor ter várias interfaces específicas do que uma única interface "gigante" (Fat Interface).<br>`OBS: Clientes não devem ser forçados a depender de métodos que eles não utilizam.`

- **D (Dependency Inversion Principle - DIP)**: Dependa de abstrações, não de implementações concretas.<br>`OBS: Este é o pilar da desacoplagem. Módulos de alto nível não devem depender de módulos de baixo nível; ambos devem depender de abstrações.`

### Exemplos

> O Código "Sem SOLID" (Violando múltiplos princípios)

Aqui, a classe `OrderProcessor` faz tudo (viola SRP e OCP) e a interface `IMultiWorker` força a implementação de métodos desnecessários (viola ISP e LSP).

~~~c#
// Interface "Gorda" (Violação do ISP)
public interface IOrderService
{
    void ProcessOrder(Order order);
    void GenerateInvoicePdf(Order order); // Nem todo processador gera PDF
}

public class OrderProcessor : IOrderService
{
    public void ProcessOrder(Order order)
    {
        // 1. Validação (SRP violado)
        if (order.Amount <= 0) throw new Exception("Pedido inválido");

        // 2. Persistência relacional direta (SRP e DIP violados)
        Console.WriteLine("Salvando pedido no SQL Server...");

        // 3. Notificação por E-mail direta (SRP e OCP violados)
        Console.WriteLine("Enviando e-mail de confirmação...");
    }

    public void GenerateInvoicePdf(Order order)
    {
        // Violação de LSP e ISP: Forçado a implementar algo que não suporta
        throw new NotImplementedException("Este processador não gera PDFs.");
    }
}
~~~

Por que este código apresenta limitações?

- Viola SRP e DIP: A classe `OrderProcessor` possui múltiplas responsabilidades e depende diretamente de detalhes de infraestrutura (SQL, E-mail).

- Viola OCP: Adicionar um novo canal de notificação (ex: SMS) exige alterar a classe original.

- Viola ISP e LSP: A interface obriga a classe a implementar `GenerateInvoicePdf`, lançando uma exceção de não implementação que pode quebrar chamadas polimórficas.

> O Código "Com SOLID" (Refatorado)

Aqui, aplicamos SRP, OCP, LSP, ISP e DIP. Cada classe tem uma única responsabilidade, as interfaces são segregadas e dependemos de abstrações injetadas.

~~~c#
// 1. Interfaces Segregadas e Específicas (ISP)
public interface IOrderRepository 
{ 
    Task SaveAsync(Order order); 
}

public interface INotificationService 
{ 
    Task SendAsync(string message); 
}

// 2. Abstração e Extensibilidade (OCP e DIP)
public class OrderProcessor
{
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notifier;

    // Injeção de Dependência via construtor (DIP)
    public OrderProcessor(IOrderRepository repository, INotificationService notifier)
    {
        _repository = repository;
        _notifier = notifier;
    }

    public async Task ProcessOrderAsync(Order order)
    {
        // Validação encarregada ao próprio modelo ou validador
        if (order.Amount <= 0) throw new ArgumentException("Pedido inválido");

        await _repository.SaveAsync(order);
        await _notifier.SendAsync("Pedido processado com sucesso!");
    }
}

// 3. Implementações concretas substituíveis sem quebrar o código (LSP)
public class SqlOrderRepository : IOrderRepository
{
    public Task SaveAsync(Order order) => Task.CompletedTask; // Salva no SQL
}

public class EmailNotificationService : INotificationService
{
    public Task SendAsync(string message) => Task.CompletedTask; // Envia E-mail
}
~~~

Por que este código é excelente?

- Respeita SRP e DIP: `OrderProcessor` apenas coordena o fluxo. Ele não sabe como o pedido é salvo nem como a notificação é enviada.

- Respeita OCP e LSP: Para trocar o repositório por MongoDB ou enviar SMS em vez de E-mail, basta criar novas classes que implementam as interfaces e injetá-las sem alterar `OrderProcessor` nem lançar exceções inesperadas.

- Alta Testabilidade: É trivial criar "Mocks" (simulações) de `IOrderRepository` e `INotificationService` para testar a lógica de negócio unitariamente sem tocar em banco de dados ou rede.

`NOTA: A aplicação do SOLID deve ser equilibrada. Aplicar todos os princípios em um código simples/CRUD pode gerar complexidade desnecessária (Overengineering). Aplique-os à medida que o domínio e a complexidade do software crescerem.`

### Material Complementar

- [Os princípios SOLID da Programação Orientada a Objetos explicados em bom português](https://www.freecodecamp.org/portuguese/news/os-principios-solid-da-programacao-orientada-a-objetos-explicados-em-bom-portugues/)
- [Princípios SOLID: criando e mantendo softwares simples e eficientes](https://medium.com/itautech/princ%C3%ADpios-solid-criando-e-mantendo-softwares-simples-e-eficientes-2614ca3417b2)
- [The Clean Architecture - Uncle Bob](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
