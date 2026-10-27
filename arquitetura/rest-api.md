# REST API

A *REST API* (Interface de Programação de Aplicações baseada em Representational State Transfer) é um estilo arquitetural para o desenvolvimento de serviços web que utilizam o protocolo HTTP para permitir a comunicação e troca de dados entre sistemas distribuídos.

Em vez de expor métodos e procedimentos remotos (como no modelo RPC ou SOAP), uma API RESTful expõe **recursos** (ex: utilizadores, pedidos, produtos) identificados por URIs únicas e manipulados através dos métodos padronizados do protocolo HTTP.

O conceito foi definido em 2000 por Roy Fielding na sua tese de doutoramento na UC Irvine, tornando-se o padrão dominante para integração de aplicações web, móveis e microsserviços.

| __Método HTTP__ | __Ação CRUD__         | __Propriedades HTTP__        | __Objetivo Principal__                                           |
| --------------- | --------------------- | ---------------------------- | ---------------------------------------------------------------- |
| **GET**         | Read (Leitura)        | Seguro e Idempotente         | Obter a representação de um recurso ou lista de recursos         |
| **POST**        | Create (Criação)      | Nem Seguro, Nem Idempotente  | Criar um novo recurso no servidor                                |
| **PUT**         | Update (Substituição) | Idempotente                  | Substituir integralmente a representação de um recurso existente |
| **PATCH**       | Update (Parcial)      | Não Idempotente (geralmente) | Modificar parcialmente os atributos de um recurso existente      |
| **DELETE**      | Delete (Remoção)      | Idempotente                  | Remover um recurso específico do servidor                        |

### O Mapa Mental das APIs REST

Aqui está a essência das restrições e boas práticas de design RESTful:

- **Modelo Cliente-Servidor e Stateless (Sem Estado)**: A interface do cliente e o armazenamento de dados do servidor são independentes. Cada requisição do cliente deve conter todas as informações necessárias para ser processada, sem dependência de sessões salvas no servidor.<br>`OBS: Servidores stateless facilitam imensamente a escalabilidade horizontal. Se o servidor 1 cair, a próxima requisição pode ser processada pelo servidor 2 sem perda de contexto.`

- **Design Centrado em Recursos (URIs Substantivas)**: As rotas devem representar substantivos no plural que identificam entidades do sistema, e nunca verbos de ação.<br>`OBS: Prefira URIs limpas como GET /api/v1/orders/123 em vez de rotas orientadas a ação como GET /api/v1/getOrderById?id=123 ou POST /api/v1/cancelOrder.`

- **Uso Correto de Códigos de Estado HTTP**: As respostas da API devem utilizar os códigos de estado HTTP adequados para transmitir o resultado da operação (2xx para sucesso, 4xx para erros do cliente e 5xx para erros do servidor).<br>`OBS: Evite o anti-padrão de retornar HTTP 200 OK acompanhado de uma mensagem interna { "status": "error", "message": "Item não encontrado" }. Se o recurso não existe, retorne HTTP 404 Not Found.`

- **Níveis de Maturidade de Richardson**: Modelo que avalia a conformidade do serviço com o REST (Nível 0: URI única/POX; Nível 1: Recursos; Nível 2: Verbos HTTP; Nível 3: HATEOAS - Hypermedia as the Engine of Application State).<br>`OBS: A maioria dos serviços modernos no mercado atinge o Nível 2. O Nível 3 inclui hiperlinks na resposta que guiam o cliente sobre as ações possíveis a partir do estado atual.`

### Exemplos

> O Código "Sem boas práticas REST" (Design estilo RPC sobre HTTP)

Aqui, a API utiliza incorretamente os métodos HTTP, ignora códigos de estado adequados e expõe verbos nas rotas, tornando a interface confusa e inconsistente.

~~~c#
[ApiController]
[Route("api/[controller]")]
public class LegacyOrderController : ControllerBase
{
    private readonly IOrderRepository _repository;

    public LegacyOrderController(IOrderRepository repository) { _repository = repository; }

    // Violação: Usa POST para leitura e inclui o verbo "get" na rota
    [HttpPost("getOrdersByCustomer")]
    public IActionResult FetchOrders([FromBody] int customerId)
    {
        var orders = _repository.GetByCustomer(customerId);
        
        // Violação: Retorna HTTP 200 OK mesmo quando o cliente enviou um ID inválido
        if (customerId <= 0)
            return Ok(new { success = false, message = "ID de cliente inválido." });

        return Ok(orders);
    }

    // Violação: Usa GET para alterar dados e não especifica o recurso na URI
    [HttpGet("deleteOrder")]
    public IActionResult RemoveOrder(int orderId)
    {
        _repository.Delete(orderId);
        return Ok(new { status = "deletado com sucesso" });
    }
}
~~~

Por que este código apresenta limitações?

- Rotas Baseadas em Ação (RPC-style): Mistura métodos HTTP e nomes de ações na URI (`getOrdersByCustomer`, `deleteOrder`), quebrando a padronização REST.

- Uso Incorreto de Métodos HTTP: Utiliza `POST` para consultas e `GET` para operações de remoção com efeitos colaterais, violando a idempotência e segurança dos métodos.

- Má Gestão de Respostas HTTP: Retorna sempre `200 OK` com objetos customizados de erro, dificultando o tratamento automatizado por clientes e gateways de API.

> O Código "Com boas práticas REST" (Controller RESTful Refatorado)

Aqui, a controller utiliza URIs orientadas a recursos, aplica os verbos HTTP adequados e retorna códigos de estado semânticos e padronizados.

~~~c#
[ApiController]
[Route("api/v1/customers/{customerId:int}/orders")]
public class OrdersController : ControllerBase
{
    private readonly IOrderRepository _repository;

    public OrdersController(IOrderRepository repository) { _repository = repository; }

    // GET /api/v1/customers/10/orders -> Busca coleção de recursos
    [HttpGet]
    public async Task<IActionResult> GetOrdersByCustomer(int customerId)
    {
        if (customerId <= 0)
            return BadRequest(new { detail = "O ID do cliente deve ser maior que zero." }); // HTTP 400

        var orders = await _repository.GetByCustomerAsync(customerId);
        return Ok(orders); // HTTP 200 OK
    }

    // GET /api/v1/customers/10/orders/5 -> Busca recurso específico
    [HttpGet("{orderId:guid}", Name = "GetOrderById")]
    public async Task<IActionResult> GetOrderById(int customerId, Guid orderId)
    {
        var order = await _repository.GetByIdAsync(orderId);
        
        if (order == null)
            return NotFound(); // HTTP 404 Not Found

        return Ok(order); // HTTP 200 OK
    }

    // POST /api/v1/customers/10/orders -> Cria um novo recurso
    [HttpPost]
    public async Task<IActionResult> CreateOrder(int customerId, [FromBody] CreateOrderDto dto)
    {
        var newOrder = await _repository.CreateAsync(customerId, dto);

        // HTTP 201 Created com o cabeçalho 'Location' apontando para a URI do novo recurso
        return CreatedAtRoute("GetOrderById", new { customerId, orderId = newOrder.Id }, newOrder);
    }

    // DELETE /api/v1/customers/10/orders/5 -> Remove o recurso
    [HttpDelete("{orderId:guid}")]
    public async Task<IActionResult> DeleteOrder(Guid orderId)
    {
        var exists = await _repository.ExistsAsync(orderId);
        if (!exists)
            return NotFound(); // HTTP 404 Not Found

        await _repository.DeleteAsync(orderId);
        return NoContent(); // HTTP 204 No Content
    }
}
~~~

Por que este código é excelente?

- Orientado a Recursos: As rotas utilizam hierarquias claras de recursos (`/customers/{customerId}/orders/{orderId}`) sem exposição de verbos no caminho.

- Semântica HTTP Rigorosa: Retorna `201 Created` com a rota do novo recurso ao criar, `204 No Content` ao remover com sucesso, `400 BadRequest` para erros de validação e `404 NotFound` para recursos inexistentes.

- Previsibilidade e Padronização: Qualquer cliente HTTP (navegadores, aplicações móveis ou outros microsserviços) entende o comportamento da API sem necessidade de documentação customizada para tratar erros.

`NOTA: Projetar APIs RESTful exige atenção à consistência da interface, versionamento adequado de rotas (ex: /v1/, /v2/) e uso correto de cabeçalhos HTTP como Content-Type e Authorization.`

### Material Complementar

- [RESTful Web API Design Best Practices - Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [Richardson Maturity Model - Martin Fowler](https://martinfowler.com/articles/richardsonMaturityModel.html)
- [RFC 9110: HTTP Semantics and Architecture](https://www.rfc-editor.org/rfc/rfc9110.html)
