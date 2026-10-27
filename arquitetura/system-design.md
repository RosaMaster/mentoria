# System Design

O *System Design* (Design de Sistemas) é o processo de definição da arquitetura, módulos, interfaces, componentes e dados de um sistema para satisfazer requisitos específicos de negócio e operação.

Em contextos de software moderno, o termo refere-se habitualmente ao projeto de **sistemas distribuídos de grande escala**, onde o principal desafio deixa de ser apenas a escrita de código limpo individual e passa a ser a garantia de alta disponibilidade, escalabilidade, tolerância a falhas e desempenho sob volumes massivos de tráfego.

O estudo de System Design envolve o equilíbrio constante entre diferentes restrições e compromissos (*trade-offs*), fundamentado em teoremas como o **CAP** e o **PACELC**.

| __Conceito__                  | __Foco Principal__      | __Estratégia__                        | __Objetivo__                                                              |
| ----------------------------- | ----------------------- | ------------------------------------- | ------------------------------------------------------------------------- |
| **Escalabilidade Horizontal** | Crescimento da Carga    | Adicionar mais máquinas (*Scale-out*) | Lidar com aumento de tráfego sem criar gargalos numa única máquina        |
| **Alta Disponibilidade**      | Continuidade do Serviço | Redundância e *Failover*              | Garantir que o sistema permaneça operacional mesmo com falhas de hardware |
| **Baixa Latência**            | Desempenho de Resposta  | Caching e CDNs                        | Minimizar o tempo decorrido entre uma requisição e a sua resposta         |
| **Tolerância a Falhas**       | Resiliência             | Particionamento e Réplicas            | Evitar que a queda de um componente derrube o sistema inteiro             |

### O Mapa Mental do System Design

Aqui está a essência dos princípios e componentes fundamentais na arquitetura de sistemas:

- **Teorema CAP (Consistency, Availability, Partition Tolerance)**: Num sistema distribuído com particionamento de rede (P), é necessário escolher entre **Consistência (C)** (todos os nós veem os mesmos dados ao mesmo tempo) ou **Disponibilidade (A)** (todas as requisições recebem uma resposta, mesmo que não seja a mais recente).<br>`OBS: Em redes reais, falhas de comunicação (P) são inevitáveis. Por isso, a escolha real num sistema distribuído é sempre entre CP (prioriza precisão) ou AP (prioriza disponibilidade com consistência eventual).`

- **Balanceadores de Carga (Load Balancers)**: Distribuem o tráfego de entrada entre múltiplos servidores de aplicação.<br>`OBS: Utilizam algoritmos como Round-Robin, Least Connections ou IP Hash. Atuam como a primeira linha de defesa para escalar horizontalmente e evitar sobrecarga num único nó.`

- **Estratégias de Caching**: Camada de memória temporária ultra-rápida (ex: Redis, Memcached) utilizada para guardar dados lidos com frequência.<br>`OBS: O maior desafio do cache é a invalidação. Estratégias como Cache-Aside, Write-Through e Write-Back definem como e quando o cache é atualizado em relação à base de dados principal.`

- **Particionamento e Sharding de Bases de Dados**: Divisão de uma base de dados grande em frações menores (*shards*) distribuídas por múltiplos servidores.<br>`OBS: Permite ultrapassar os limites físicos de armazenamento e IOPS de uma única máquina, mas adiciona complexidade em consultas que envolvem múltiplos shards.`

### Exemplos

> O Código / Arquitetura "Sem System Design" (Monolítico Não Escalável)

Aqui, a aplicação depende de uma única instância de servidor conectada diretamente a uma base de dados relacional centralizada. Qualquer pico de tráfego ou falha no servidor derruba todo o serviço.

~~~c#
public class MonolithicProductService
{
    private readonly AppDbContext _dbContext;

    public MonolithicProductService(AppDbContext dbContext) { _dbContext = dbContext; }

    public async Task<ProductDto> GetProductDetailsAsync(int productId)
    {
        // 1. Consulta pesada realizada DIRETAMENTE na base de dados principal a cada requisição
        var product = await _dbContext.Products
            .Include(p => p.Reviews)
            .Include(p => p.Inventory)
            .FirstOrDefaultAsync(p => p.Id == productId);

        // 2. Sem camada de cache, sem balanceamento e com ponto único de falha (SPOF)
        return MapToDto(product);
    }
}
~~~

Por que este design apresenta limitações em grande escala?

- Ponto Único de Falha (SPOF): Se a máquina da base de dados ou da aplicação falhar, 100% dos utilizadores ficam sem acesso.

- Gargalo de I/O na Base de Dados: Milhares de acessos simultâneos ao mesmo produto (ex: uma promoção de Black Friday) irão sobrecarregar o motor de base de dados relacional.

- Escalabilidade Rígida: Para suportar mais tráfego, a única opção é aumentar o hardware da máquina (*Scale-up*), o que possui limites físicos e custos astronómicos.

> O Código / Arquitetura "Com System Design" (Distribuído com Cache e Read-Replica)

Aqui, introduzimos uma estratégia de **Cache-Aside** com Redis para leituras rápidas e isolamos a base de dados principal, permitindo escala horizontal e resiliência.

~~~c#
public class DistributedProductService
{
    private readonly IDistributedCache _cache; // Camada de Cache (ex: Redis)
    private readonly IProductReadRepository _readRepository; // Consulta direcionada a Read Replica

    public DistributedProductService(IDistributedCache cache, IProductReadRepository readRepository)
    {
        _cache = cache;
        _readRepository = readRepository;
    }

    public async Task<ProductDto> GetProductDetailsAsync(int productId)
    {
        string cacheKey = $"products:{productId}";

        // 1. Tenta obter o dado diretamente do Cache (Sub-milissegundo)
        var cachedProduct = await _cache.GetStringAsync(cacheKey);
        if (!string.IsNullOrEmpty(cachedProduct))
        {
            return JsonSerializer.Deserialize<ProductDto>(cachedProduct)!;
        }

        // 2. Cache Miss: Busca na Réplica de Leitura da base de dados (evita impacto na base principal de escrita)
        var product = await _readRepository.GetByIdAsync(productId);
        
        if (product != null)
        {
            var dto = MapToDto(product);
            
            // 3. Popula o cache com tempo de expiração (TTL) para evitar dados obsoletos
            var cacheOptions = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
            };
            await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(dto), cacheOptions);

            return dto;
        }

        return null;
    }
}
~~~

Por que este design é excelente?

- Alta Performance e Baixa Latência: A maioria das requisições é servida diretamente da memória RAM pelo cache Redis, reduzindo drasticamente o tempo de resposta.

- Escalabilidade Horizontal: A camada de aplicação pode rodar em dezenas de instâncias atrás de um Balanceador de Carga. As leituras na base de dados são distribuídas entre várias réplicas de leitura (*Read Replicas*).

- Resiliência a Picos de Tráfego: A base de dados primária fica protegida contra picos de acessos de leitura, garantindo que as operações de escrita (compras/pedidos) continuem a funcionar com segurança.

`NOTA: Projetar sistemas distribuídos é um exercício constante de negociação de requisitos. Nenhuma arquitetura é perfeita para todos os casos; o objetivo do System Design é escolher o conjunto certo de trade-offs para o seu problema específico.`

### Material Complementar

- [System Design Primer - GitHub Repository](https://github.com/donnemartin/system-design-primer)
- [A Primer on Distributed Systems - AWS Architecture Center](https://aws.amazon.com/architecture/)
- [CAP Theorem & PACELC Theorem Explained - Martin Fowler](https://martinfowler.com/articles/patterns-of-distributed-systems/)
