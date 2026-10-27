# Programação Orientada a Objetos (POO)

A *Programação Orientada a Objetos* (POO) é um paradigma de programação baseado no conceito de "objetos", que contêm dados na forma de **atributos** e código na forma de **métodos**.

Em vez de estruturar o software como uma sequência de comandos e funções isoladas (programação procedural), a POO organiza o sistema como um conjunto de entidades autônomas que interagem entre si, aproximando a modelagem do código do mundo real.

O paradigma começou a ser desenhado na década de 1960 com a linguagem Simula, foi consolidado com o Smalltalk por Alan Kay e tornou-se a base de linguagens amplamente utilizadas no mercado, como C#, Java, C++ e Python.

| __Pilar__          | __Conceito__            | __Foco Principal__                                            | __Objetivo__                                                      |
| ------------------ | ----------------------- | ------------------------------------------------------------- | ----------------------------------------------------------------- |
| **Encapsulamento** | Proteção e Ocultação    | Esconder detalhes internos de implementação                   | Proteger o estado do objeto e expor apenas interfaces seguras     |
| **Abstração**      | Simplificação do Modelo | Modelar apenas as características essenciais do domínio       | Isolar a complexidade e focar no "que faz" e não no "como faz"    |
| **Herança**        | Reuso e Hierarquia      | Derivar novas classes a partir de classes existentes          | Compartilhar código e definir relações de especialização ("é um") |
| **Polimorfismo**   | Múltiplas Formas        | Executar comportamentos diferentes através da mesma interface | Permitir flexibilidade e desacoplamento via invocação dinâmica    |

### O Mapa Mental da POO

Aqui está a essência dos quatro pilares, focada no seu impacto no design de software:

- **Encapsulamento**: Os dados de um objeto não devem ser alterados diretamente por componentes externos. O acesso deve ser mediado por métodos e propriedades públicas.<br>`OBS: Evite criar getters e setters para tudo de forma automática. O encapsulamento real envolve expor métodos de negócio expressivos que mantêm as invariantes do objeto válidas.`

- **Abstração**: Permite representar um conceito do mundo real reduzindo sua complexidade ao que realmente importa para o contexto do sistema.<br>`OBS: Interfaces e classes abstratas são ferramentas primordiais de abstração. Elas definem contratos que os módulos consumidores usam sem precisar conhecer a implementação concreta.`

- **Herança**: Permite criar uma hierarquia de classes reutilizando atributos e métodos da classe pai (*Base Class*).<br>`OBS: Use herança com cautela. A herança excessiva gera acoplamento rígido. Sempre favoreça a composição em detrimento da herança ("Favor composition over inheritance").`

- **Polimorfismo**: Capacidade de objetos de diferentes classes responderem à mesma mensagem (ou chamada de método) de maneiras específicas.<br>`OBS: É o coração do desacoplamento em POO. Combinado com o Princípio da Substituição de Liskov (LSP), permite trocar implementações sem alterar o código cliente.`

### Exemplos

> O Código "Sem POO / Procedural" (Violando os pilares)

Aqui, os dados estão expostos em uma estrutura simples e a lógica de negócio está espalhada em funções globais ou procedurais sem proteção de estado.

~~~c#
// Estrutura anêmica sem proteção de dados (Violando Encapsulamento)
public class BankAccountData
{
    public string AccountNumber;
    public decimal Balance;
    public string AccountType;
}

public class BankProcessor
{
    public void ProcessWithdrawal(BankAccountData account, decimal amount)
    {
        // Lógica procedural externa controlando o estado do objeto
        if (account.AccountType == "Savings" && account.Balance - amount < 0)
        {
            Console.WriteLine("Conta Poupança não pode ter saldo negativo.");
            return;
        }

        // Alteração direta e insegura da variável
        account.Balance -= amount;
    }
}
~~~

Por que este código apresenta limitações em POO?

- Violação do Encapsulamento: Qualquer parte do sistema pode alterar `account.Balance` diretamente para valores inválidos sem passar por validações.

- Falta de Abstração e Polimorfismo: Tipos de conta são tratados via condicionais (`account.AccountType == "Savings"`), o que torna a adição de novos tipos trabalhosa e propensa a erros.

- Alto Acoplamento Procedural: A classe `BankProcessor` precisa conhecer regras internas de todos os tipos de conta.

> O Código "Com POO" (Refatorado com Encapsulamento e Polimorfismo)

Aqui, aplicamos os pilares da POO: o saldo é encapsulado, os tipos de conta abstraem o comportamento de saque e o polimorfismo é utilizado para executar a regra correta.

~~~c#
// Abstração e Contrato (Abstração)
public abstract class BankAccount
{
    public string AccountNumber { get; }
    public decimal Balance { get; protected set; } // Encapsulamento: alteração restrita

    protected BankAccount(string accountNumber, decimal initialBalance)
    {
        AccountNumber = accountNumber;
        Balance = initialBalance;
    }

    // Método abstrato a ser implementado com polimorfismo
    public abstract void Withdraw(decimal amount);

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("O valor do depósito deve ser positivo.");
        Balance += amount;
    }
}

// Especialização e Polimorfismo (Herança + Polimorfismo)
public class SavingsAccount : BankAccount
{
    public SavingsAccount(string accountNumber, decimal initialBalance) 
        : base(accountNumber, initialBalance) { }

    public override void Withdraw(decimal amount)
    {
        if (Balance - amount < 0)
            throw new InvalidOperationException("Conta Poupança não pode ter saldo negativo.");

        Balance -= amount;
    }
}

public class CheckingAccount : BankAccount
{
    public decimal OverdraftLimit { get; }

    public CheckingAccount(string accountNumber, decimal initialBalance, decimal overdraftLimit) 
        : base(accountNumber, initialBalance)
    {
        OverdraftLimit = overdraftLimit;
    }

    public override void Withdraw(decimal amount)
    {
        if (Balance - amount < -OverdraftLimit)
            throw new InvalidOperationException("Limite de cheque especial excedido.");

        Balance -= amount;
    }
}
~~~

Por que este código é excelente?

- Respeita o Encapsulamento: O atributo `Balance` só pode ser modificado pelos métodos protegidos e públicos da própria classe, garantindo a integridade dos dados.

- Polimorfismo Nativo: O sistema pode chamar `account.Withdraw(100)` em qualquer instância derivada de `BankAccount` sem precisar saber se é Poupança ou Conta Corrente.

- Extensibilidade: Para adicionar um novo tipo de conta (ex: `InvestmentAccount`), basta herdar de `BankAccount` e sobrescrever o método `Withdraw()`, sem alterar as contas existentes.

`NOTA: A Programação Orientada a Objetos é o alicerce para a maioria dos Padrões de Projeto (Design Patterns) e dos princípios SOLID. Dominar seus pilares é essencial para escrever software extensível e sustentável.`

### Material Complementar

- [Object-Oriented Programming Concepts - Microsoft C# Guide](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/tutorials/oop)
- [POO e seus pilares explicados de forma simples](https://www.freecodecamp.org/portuguese/news/os-quatro-pilares-da-programacao-orientada-a-objetos-com-exemplos-em-java/)
- [Composition over Inheritance - Refactoring.Guru](https://refactoring.guru/pt-br/design-patterns/composition-over-inheritance)
