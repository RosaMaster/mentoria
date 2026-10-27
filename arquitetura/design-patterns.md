# Design Patterns

Os *Design Patterns* (Padrões de Projeto) são soluções consolidadas e reutilizáveis para problemas recorrentes no desenvolvimento de software orientado a objetos.

Não se tratam de códigos prontos ou bibliotecas que podem ser simplesmente copiadas e coladas, mas sim de modelos conceituais (*blueprints*) que orientam a estruturação do código para resolver desafios de arquitetura de forma elegante e eficiente.

O conceito foi popularizado em 1994 pelo livro *Design Patterns: Elements of Reusable Object-Oriented Software*, escrito por Erich Gamma, Richard Helm, Ralph Johnson e John Vlissides — grupo mundialmente conhecido como **Gang of Four (GoF)**.

| __Categoria__       | __Foco Principal__       | __Objetivo__                                                      | __Exemplos Clássicos__             |
| ------------------- | ------------------------ | ----------------------------------------------------------------- | ---------------------------------- |
| **Criacionais**     | Criação de Objetos       | Encapsular e controlar o processo de instanciação de objetos      | Singleton, Factory Method, Builder |
| **Estruturais**     | Composição de Classes    | Organizar relacionamentos e estruturas entre classes e objetos    | Adapter, Decorator, Facade         |
| **Comportamentais** | Comunicação e Algoritmos | Gerenciar a distribuição de responsabilidades e fluxo de execução | Strategy, Observer, Command        |

### O Mapa Mental dos Design Patterns

Aqui está a essência das três categorias do GoF, focada no impacto no design de software:

- **Padrões Criacionais**: Abstraem o processo de instanciação. Eles tornam o sistema independente de como seus objetos são criados, compostos e representados.<br>`OBS: Evite usar o operador new diretamente espalhado pelo código de negócio quando a criação do objeto for complexa. Use padrões criacionais para desacoplar o "como criar" do "como usar".`

- **Padrões Estruturais**: Ocupam-se de como classes e objetos são compostos para formar estruturas maiores mantendo a flexibilidade e eficiência.<br>`OBS: Úteis para integrar bibliotecas legadas ou de terceiros sem alterar a interface original (ex: Adapter) ou para adicionar responsabilidades dinamicamente (ex: Decorator).`

- **Padrões Comportamentais**: Concentram-se nos algoritmos e na atribuição de responsabilidades entre os objetos, definindo não apenas os padrões de objetos ou classes, mas também os padrões de comunicação entre eles.<br>`OBS: Permitem eliminar estruturas condicionais extensas (if/else ou switch) substituindo-as por polimorfismo dinâmico em tempo de execução (ex: Strategy).`

- **Padrões não são Balas de Prata**: Aplicar padrões onde não há necessidade gera complexidade acidental (*overengineering*).<br>`OBS: Entenda o problema primeiro; aplique o padrão apenas quando ele simplificar a manutenção ou permitir a extensibilidade necessária.`

### Exemplos

> O Código "Sem Design Pattern" (Violando boas práticas de design)

Aqui, a classe `OrderProcessor` utiliza estruturas condicionais rígidas (`switch/case`) para calcular o frete e enviar notificações, tornando a adição de novas regras custosa e propensa a erros.

~~~c#
public class OrderProcessor
{
    public decimal CalculateShipping(string shippingType, decimal weight)
    {
        // Violação do OCP: A cada novo tipo de frete, esta classe precisa ser alterada
        if (shippingType == "Sedex")
        {
            return weight * 1.5m + 10;
        }
        else if (shippingType == "Pac")
        {
            return weight * 0.8m + 5;
        }
        else if (shippingType == "Transportadora")
        {
            return weight * 2.0m;
        }
        
        throw new ArgumentException("Tipo de frete inválido");
    }
}
~~~

Por que este código apresenta limitações?

- Alto Acoplamento: A lógica de cálculo de todos os tipos de frete está concentrada em um único método.

- Violação do OCP (Open/Closed Principle): Adicionar uma nova modalidade de entrega exige modificar a classe existente, aumentando o risco de regressões.

- Dificuldade de Testes: Não é possível testar uma regra de cálculo de frete isoladamente das demais.

> O Código "Com Design Pattern" (Refatorado com Strategy Pattern)

Aqui, aplicamos o padrão comportamental **Strategy** para isolar cada algoritmo de cálculo em sua própria classe e permitir a seleção dinâmica da estratégia.

~~~c#
// Abstração da Estratégia
public interface IShippingStrategy
{
    decimal Calculate(decimal weight);
}

// Estratégias Concretas
public class SedexShippingStrategy : IShippingStrategy
{
    public decimal Calculate(decimal weight) => weight * 1.5m + 10;
}

public class PacShippingStrategy : IShippingStrategy
{
    public decimal Calculate(decimal weight) => weight * 0.8m + 5;
}

// Contexto que utiliza a Estratégia
public class OrderProcessor
{
    private readonly IShippingStrategy _shippingStrategy;

    // Injeção da estratégia desejada
    public OrderProcessor(IShippingStrategy shippingStrategy)
    {
        _shippingStrategy = shippingStrategy;
    }

    public decimal ProcessShipping(decimal weight)
    {
        return _shippingStrategy.Calculate(weight);
    }
}
~~~

Por que este código é excelente?

- Respeita o OCP: Para adicionar um novo tipo de frete (ex: `DroneShippingStrategy`), basta criar uma nova classe que implementa `IShippingStrategy` sem alterar `OrderProcessor`.

- Testabilidade e Isolamento: Cada estratégia de cálculo de frete pode ser testada unitariamente de forma independente.

- Flexibilidade em Tempo de Execução: A estratégia pode ser trocada dinamicamente de acordo com a escolha do utilizador ou configuração do sistema.

`NOTA: Os Design Patterns fornecem um vocabulário comum entre desenvolvedores. Dizer "estamos usando um Strategy aqui" transmite instantaneamente a intenção arquitetural do código.`

### Material Complementar

- [Refactoring.Guru - Catálogo Interativo de Design Patterns](https://refactoring.guru/pt-br/design-patterns)
- [Design Patterns no C# e .NET - Guia Prático](https://macoratti.net/Csharp/csharp-dp.htm)
- [SourceMaking - Design Patterns & Software Architecture](https://sourcemaking.com/design_patterns)
