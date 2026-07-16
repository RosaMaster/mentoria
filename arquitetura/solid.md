# Solid

O *SOLID* é a base fundamental para qualquer arquiteto ou desenvolvedor que almeja construir sistemas de alta qualidade. Não é apenas um conjunto de regras, mas uma filosofia de design que visa tornar o código fácil de entender, manter e estender.
O nome SOLID é, na verdade, um acrônimo. A palavra foi cunhada por Michael Feathers no início dos anos 2000, mas os princípios em si foram compilados e popularizados por Robert C. Martin (conhecido como "Uncle Bob") em seu livro Agile Software Development, Principles, Patterns, and Practices.

| __Letra__ | __Sigla__ | __Princípio (em inglês)__       | __Tradução__                         |
| --------- | --------- | ------------------------------- | ------------------------------------ | 
| S         | SRP       | Single Responsibility Principle | Princípio da Responsabilidade Única  |
| O         | OCP       | Open/Closed Principle           | Princípio Aberto/Fechado             |
| L         | LSP       | Liskov Substitution Principle   | Princípio da Substituição de Liskov  |
| I         | ISP       | Interface Segregation Principle | Princípio da Segregação de Interface |
| D         | DIP       | Dependency Inversion Principle  | Princípio da Inversão de Dependência |

### O Mapa Mental do SOLID

Aqui está a essência de cada letra, focada no impacto arquitetural:

- **S (Single Responsibility Principle - SRP)**: Uma classe ou módulo deve ter apenas uma razão para mudar.<br>`OBS: O SRP é o maior inimigo do "código espaguete". Se uma classe faz o cálculo de um imposto, a validação de um dado e salva no banco de dados, ela está fadada a quebrar quando qualquer uma dessas três regras mudar.`

- **O (Open/Closed Principle - OCP)**: Você deve ser capaz de estender o comportamento de uma classe sem precisar modificar o código-fonte dela.<br>`OBS: A chave aqui é o uso de interfaces ou classes abstratas. Se você vive editando uma classe para adicionar novos comportamentos (if/else ou switch intermináveis), você violou o OCP.`

- **L (Liskov Substitution Principle - LSP)**: Classes derivadas devem ser substituíveis por suas classes base sem quebrar o comportamento do sistema.<br>`OBS: Se você precisa checar o tipo de um objeto (if isinstance(x, Y)) para executar uma tarefa, você violou o LSP. O polimorfismo deve ser transparente.`

- **I (Interface Segregation Principle - ISP)**: É melhor ter várias interfaces específicas do que uma única interface "gigante" (o famoso Fat Interface).<br>`OBS: Clientes não devem ser forçados a depender de métodos que eles não utilizam.`

- **D (Dependency Inversion Principle - DIP)**: Dependa de abstrações, não de implementações concretas.<br>`OBS: Este é o pilar da desacoplagem. Se o seu código de serviço depende diretamente de uma classe concreta de um banco de dados, você está preso àquele banco. Se depender de uma interface de repositório, você pode trocar o banco (ou realizar testes unitários com mocks) sem tocar na lógica de negócio.`

### Exemplos

> O Código "Sem SOLID" (Violando os princípios)

Aqui, a classe OrderProcessor é um "Objeto Deus". Ela faz tudo: valida o pedido, calcula o total, salva no banco e ainda envia um e-mail.

~~~c#
public class OrderProcessor
{
    public void ProcessOrder(Order order)
    {
        // 1. Validação (SRP violado - lógica de validação aqui)
        if (order.Amount <= 0) throw new Exception("Pedido inválido");

        // 2. Persistência (SRP violado - lógica de banco aqui)
        Console.WriteLine("Salvando pedido no banco de dados...");

        // 3. Notificação (SRP violado - lógica de e-mail aqui)
        Console.WriteLine("Enviando e-mail de confirmação...");
    }
}
~~~

Por que este código é ruim?

- Viola SRP: Se você precisar mudar o método de envio de e-mail (ex: trocar para SMS), terá que alterar a classe OrderProcessor, que nem deveria saber que e-mails existem.

- Viola OCP: Se quiser adicionar uma nova etapa (ex: emitir Nota Fiscal), você precisará modificar a classe original.

- Difícil de Testar: Você não consegue testar a lógica de cálculo do pedido sem acionar o banco de dados ou o e-mail.

> O Código "Com SOLID" (Refatorado)

Aqui, aplicamos SRP e DIP (Inversão de Dependência). Cada classe tem uma única responsabilidade e dependemos de interfaces.

~~~c#
// Abstrações (DIP)
public interface IOrderRepository { void Save(Order order); }
public interface INotificationService { void Send(string message); }

public class OrderProcessor
{
    private readonly IOrderRepository _repository;
    private readonly INotificationService _notifier;

    // Injeção de dependência via construtor
    public OrderProcessor(IOrderRepository repository, INotificationService notifier)
    {
        _repository = repository;
        _notifier = notifier;
    }

    public void ProcessOrder(Order order)
    {
        // Validação centralizada (SRP)
        if (order.Amount <= 0) throw new Exception("Pedido inválido");

        _repository.Save(order);
        _notifier.Send("Pedido processado com sucesso!");
    }
}
~~~

Por que este código é excelente?

- Respeita SRP: OrderProcessor agora apenas coordena o fluxo. Ele não sabe como o pedido é salvo (SQL? NoSQL?) nem como o e-mail é enviado.

- Respeita OCP/DIP: Se amanhã você quiser salvar no MongoDB em vez de SQL, basta criar uma nova classe que implementa IOrderRepository e injetá-la. Você não altera nenhuma linha de código dentro de OrderProcessor.

- Testabilidade: Agora é trivial criar um "Mock" (simulação) do IOrderRepository e INotificationService para testar o OrderProcessor sem tocar em um banco de dados real.

`NOTA: Note que no exemplo com SOLID, introduzimos a Injeção de Dependência. Isso permite que o sistema seja desacoplado.`

### Material Complementar

- [Os princípios SOLID da Programação Orientada a Objetos explicados em bom português](https://www.freecodecamp.org/portuguese/news/os-principios-solid-da-programacao-orientada-a-objetos-explicados-em-bom-portugues/)
- [Princípios SOLID: criando e mantendo softwares simples e eficientes](https://medium.com/itautech/princ%C3%ADpios-solid-criando-e-mantendo-softwares-simples-e-eficientes-2614ca3417b2)
- [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
