# Event Sourcing

O *Event Sourcing* (Origem por Eventos) é um padrão de persistência arquitetural no qual as mudanças no estado de uma aplicação não são armazenadas sobrescrevendo a base de dados, mas sim registadas como uma sequência cronológica e imutável de **eventos**.

Em vez de guardar apenas o "estado atual" do sistema (ex: `saldo = R$ 100`), o Event Sourcing armazena todos os acontecimentos históricos que levaram a esse estado (ex: `+ R$ 150`, `- R$ 50`). O estado atual é reconstruído em qualquer momento reprocessando a sequência desses eventos.

O padrão é amplamente associado a sistemas financeiros, auditorias rigorosas e arquiteturas baseadas em DDD e CQRS.

| __Conceito__        | __Foco Principal__        | __Papel no Padrão__                                            | __Benefício__                                                     |
| ------------------- | ------------------------- | -------------------------------------------------------------- | ----------------------------------------------------------------- |
| **Event Store**     | Banco de Dados de Eventos | Armazém *append-only* (apenas inserção) de eventos imutáveis   | Histórico 100% auditável e livre de perda de dados                |
| **Aggregate State** | Estado do Domínio         | Reconstruído aplicando a sequência de eventos na ordem correta | Permite reproduzir o estado do sistema em qualquer ponto do tempo |
| **Snapshots**       | Otimização de Performance | Registos do estado num determinado momento no tempo            | Evita a leitura de milhares de eventos para reconstruir o estado  |

### O Mapa Mental do Event Sourcing

Aqui está a essência do padrão, focada no seu impacto arquitetural:

- **Imutabilidade e Registo Histórico Completo**: Os dados nunca são alterados (`UPDATE`) ou apagados (`DELETE`). O Event Store é uma tabela/base *append-only*, o que elimina a perda de informação por mutação de estado.

- **Viagem no Tempo e Auditabilidade**: Como cada evento regista um facto do passado com o respetivo *timestamp*, é possível saber exatamente como o sistema estava em qualquer data/hora específica ou reproduzir bugs para depuração.

- **Sinergia Natural com CQRS**: O Event Sourcing atua perfeitamente no lado da **escrita** (*Command*), enquanto as visões de **leitura** (*Query*) são mantidas atualizadas consumindo o fluxo de eventos de forma assíncrona.

- **Desafio de Alteração de Schemas**: Como o histórico de eventos é imutável e permanente, mudanças na estrutura de dados (versionamento de eventos) exigem estratégias cuidadosas de migração e retrocompatibilidade no código.

### Exemplos

> O Código "Sem Event Sourcing" (Persistência por Estado Tradicional - CRUD)

Aqui, quando o saldo ou dados de uma conta mudam, a informação anterior é sobrescrita e o histórico de alterações perde-se caso não exista uma tabela de logs separada.

~~~c#
public class BankAccountService
{
    private readonly AppDbContext _context;

    public BankAccountService(AppDbContext context) { _context = context; }

    public void Withdraw(Guid accountId, decimal amount)
    {
        var account = _context.Accounts.Find(accountId);
        
        if (account.Balance < amount)
            throw new InvalidOperationException("Saldo insuficiente.");

        // Atualização de estado por sobrescrita (Perde-se o histórico direto da operação)
        account.Balance -= amount;
        
        _context.SaveChanges();
    }
}
~~~

Por que este código apresenta limitações?

- Perda de Contexto de Negócio: O banco de dados guarda apenas o valor final do saldo. Não há forma de saber diretamente se o valor resultou de um levantamento, transferência ou pagamento.

- Auditoria Complexa: Para saber a razão de uma alteração de saldo, é necessário criar tabelas paralelas de logs que podem ficar desalinhadas com a transação principal.

> O Código "Com Event Sourcing" (Persistência por Sequência de Eventos)

Aqui, a conta bancária não altera diretamente uma coluna de saldo. Ela gera e aplica **eventos de domínio imutáveis**, que são guardados no *Event Store*.

~~~c#
// Eventos Imutáveis do Domínio
public record MoneyDepositedEvent(Guid AccountId, decimal Amount, DateTime OccurredOn);
public record MoneyWithdrawnEvent(Guid AccountId, decimal Amount, DateTime OccurredOn);

public class BankAccount
{
    public Guid Id { get; private set; }
    public decimal Balance { get; private set; }
    
    private readonly List<object> _uncommittedEvents = new();
    public IReadOnlyCollection<object> UncommittedEvents => _uncommittedEvents.AsReadOnly();

    public BankAccount() { }

    // Reconstrução do estado reprocessando o histórico de eventos
    public static BankAccount LoadFromHistory(IEnumerable<object> history)
    {
        var account = new BankAccount();
        foreach (var @event in history)
        {
            account.Apply(@event);
        }
        return account;
    }

    public void Withdraw(decimal amount)
    {
        if (Balance < amount)
            throw new InvalidOperationException("Saldo insuficiente.");

        // Gera e aplica o evento
        var @event = new MoneyWithdrawnEvent(Id, amount, DateTime.UtcNow);
        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    // Mutador interno de estado baseado em eventos
    private void Apply(object @event)
    {
        switch (@event)
        {
            case MoneyDepositedEvent e:
                Balance += e.Amount;
                break;
            case MoneyWithdrawnEvent e:
                Balance -= e.Amount;
                break;
        }
    }
}
~~~

Por que este código é excelente?

- Auditabilidade Nativa: Cada ação é registada explicitamente como um evento de negócio no *Event Store*.

- Reconstrução Perfeita de Estado: É possível reconstruir a conta em qualquer ponto do passado bastando ler os eventos até determinada data.

- Desacoplamento Total: Outros serviços da empresa podem assinar o fluxo de eventos do *Event Store* para atualizar dashboards ou enviar notificações em tempo real.

`NOTA: O Event Sourcing traz grande flexibilidade e rastreabilidade, mas introduz complexidade na gestão do Event Store e no versionamento dos contratos de eventos ao longo dos anos.`

### Material Complementar

- [Event Sourcing Pattern - Microsoft Architecture Docs](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Event Sourcing - Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [EventStoreDB - The Stream Database built for Event Sourcing](https://www.eventstore.com/)
