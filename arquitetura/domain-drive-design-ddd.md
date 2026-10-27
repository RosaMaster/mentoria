# Domain-Driven Design (DDD)

O *Domain-Driven Design* (Design Guiado pelo Domínio) é uma abordagem de desenvolvimento de software focada em modelar sistemas complexos com base nas regras e dinâmicas reais do negócio.

Não se trata de uma arquitetura fixa ou framework, mas de um conjunto de princípios e padrões que alinham o código da aplicação ao modelo mental dos especialistas do negócio (*Domain Experts*).

O conceito foi introduzido em 2003 por Eric Evans em seu livro *Domain-Driven Design: Tackling Complexity in the Heart of Software* (conhecido como o "Livro Azul"), tornando-se referência obrigatória para o desenvolvimento de sistemas corporativos.

| __Dimensão__    | __Foco Principal__       | __Ferramentas e Conceitos__                                      | __Objetivo__                                                                 |
| --------------- | ------------------------ | ---------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| **Estratégica** | Organização e Fronteiras | Bounded Contexts, Ubiquitous Language, Context Mapping           | Delimitar responsabilidades e unificar a linguagem entre negócio e TI        |
| **Tática**      | Implementação de Código  | Aggregates, Entities, Value Objects, Repositories, Domain Events | Estruturar o modelo de domínio no código mantendo as regras de negócio puras |

### O Mapa Mental do DDD

Aqui está a essência dos pilares do DDD, focada no seu impacto arquitetural:

- **Linguagem Ubíqua (Ubiquitous Language)**: Linguagem única e rigorosa compartilhada por desenvolvedores e especialistas do negócio.<br>`OBS: Se o especialista de negócio fala "Cliente Cancelado" e no código a variável se chama status = 3 ou is_inactive = true, a Linguagem Ubíqua foi violada. O código deve falar a língua do negócio.`

- **Contexto Delimitado (Bounded Context)**: Fronteira explícita onde um modelo de domínio se aplica e mantém seu significado consistente.<br>`OBS: A palavra "Produto" pode significar coisas diferentes na Venda (preço, nome), no Estoque (dimensões, peso) e no Fiscal (NCM, impostos). Cada área possui seu próprio Bounded Context com seus próprios modelos.`

- **Entidades vs. Objetos de Valor (Value Objects)**: Entidades possuem uma identidade única e mutável ao longo do tempo (ex: `Cliente` com ID). Value Objects são imutáveis e definidos apenas por seus atributos (ex: `Endereço`, `CPF`, `Moeda`).<br>`OBS: Dê preferência a Value Objects sempre que possível. Eles simplificam o código, garantem validações automáticas no construtor e evitam efeitos colaterais por serem imutáveis.`

- **Agregados (Aggregates) e Raiz do Agregado (Aggregate Root)**: Agrupamento de Entidades e Value Objects tratados como uma única unidade de consistência transacional.<br>`OBS: Alterações dentro do Agregado só podem ser feitas através da sua Raiz (Aggregate Root). O banco de dados salva ou atualiza o Agregado inteiro para garantir as regras de integridade.`

### Exemplos

> O Código "Sem DDD" (Modelo Anêmico / Data-Driven)

Aqui, a classe `Order` é apenas uma estrutura de dados com *getters* e *setters* sem validação. Toda a regra de negócio fica espalhada em um serviço genérico, tornando o código vulnerável a estados inválidos.

~~~c#
// Modelo Anêmico: Apenas dados, sem comportamento
public class Order
{
    public int Id { get; set; }
    public string Status { get; set; }
    public decimal TotalAmount { get; set; }
}

public class OrderService
{
    public void CancelOrder(Order order)
    {
        // Regra de negócio isolada do modelo e propensa a duplicação
        if (order.Status == "SHIPPED")
        {
            throw new Exception("Não é possível cancelar um pedido já enviado");
        }

        order.Status = "CANCELLED"; // Modificação direta do estado sem validação interna
    }
}
~~~

Por que este código apresenta limitações em DDD?

- Modelo Anêmico: O objeto do domínio não protege suas próprias regras de negócio. Qualquer classe pode alterar `order.Status` para um estado inválido.

- Violação da Linguagem Ubíqua: Utilização de strings mágicas (`"SHIPPED"`, `"CANCELLED"`) em vez de conceitos explícitos do domínio.

- Falta de Encapsulamento: As validações ficam espalhadas na camada de aplicação/serviço em vez de estarem centralizadas dentro da própria entidade.

> O Código "Com DDD" (Modelo Rico e Encapsulado)

Aqui, a classe `Order` é uma Raiz de Agregado (*Aggregate Root*) com comportamento rico, garantindo suas próprias invariantes e emitindo eventos de domínio.

~~~c#
// Value Object Imutável
public record Money(decimal Amount, string Currency);

// Enum ou Value Object expressando o Domínio
public enum OrderStatus { Draft, Paid, Shipped, Cancelled }

// Aggregate Root com Modelo Rico
public class Order
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }

    public Order(Money total)
    {
        Id = Guid.NewGuid();
        Total = total;
        Status = OrderStatus.Draft;
    }

    // Comportamento focado na regra de negócio (Linguagem Ubíqua)
    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new DomainException("Pedidos já enviados não podem ser cancelados.");

        Status = OrderStatus.Cancelled;
    }
}
~~~

Por que este código é excelente?

- Encapsulamento e Integridade: O estado `Status` possui *private setter*. A única forma de cancelar o pedido é através do método explícito `Cancel()`, que valida a regra de negócio.

- Expressividade: O código traduz exatamente a regra do especialista de negócio. Não há vazamento de lógica para camadas externas.

- Segurança do Modelo: É impossível colocar o objeto `Order` em um estado inválido a partir do mundo externo.

`NOTA: O DDD é especialmente indicado para domínios complexos. Aplicá-lo em sistemas simples ou CRUDs puros pode gerar complexidade desnecessária.`

### Material Complementar

- [Domain-Driven Design Quickly - InfoQ (E-book gratuito)](https://www.infoq.com/minibooks/domain-driven-design-quickly/)
- [DDD Reference por Eric Evans](https://domainlanguage.com/ddd/reference/)
- [Guias da Microsoft sobre Design Orientado ao Domínio no .NET](https://learn.microsoft.com/pt-br/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/)
