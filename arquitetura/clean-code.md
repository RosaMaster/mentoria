# Clean Code

O *Clean Code* (Código Limpo) é a prática essencial para qualquer desenvolvedor ou arquiteto que busca criar software legível, de fácil manutenção e sustentável ao longo do tempo.

Não se trata apenas de fazer o código funcionar, mas de garantir que ele seja compreensível para outros seres humanos. Um código limpo expressa claramente sua intenção e reduz drasticamente o débito técnico.

O conceito foi sintetizado e amplamente difundido por Robert C. Martin ("Uncle Bob") em seu aclamado livro *Clean Code: A Handbook of Agile Software Craftsmanship*, tornando-se um pilar da engenharia de software moderna.

| __Pilar__        | __Conceito__           | __Foco Principal__          | __Objetivo__                                              |
| ---------------- | ---------------------- | --------------------------- | --------------------------------------------------------- |s
| Legibilidade     | Código como Literatura | Nomenclatura e Clareza      | Permitir leitura fluida sem necessidade de adivinhações   |
| Simplicidade     | Keep It Simple (KISS)  | Funções Pequenas e Diretas  | Evitar complexidade desnecessária e efeitos colaterais    |
| Testabilidade    | Testes Automatizados   | Cobertura e Feedback Rápido | Garantir refatorações seguras sem quebrar funcionalidades |
| Manutenibilidade | Facilidade de Mudança  | Desacoplamento e Coesão     | Reduzir o custo do ciclo de vida e evolução do software   |

### O Mapa Mental do Clean Code

Aqui está a essência dos principais princípios, focada no impacto no desenvolvimento diário:

- **Nomes Reveladores de Intenção**: Nomes de variáveis, funções e classes devem dizer claramente por que existem, o que fazem e como são usados.<br>`OBS: Evite nomes genéricos ou abreviações obscuras como d, data, temp ou processData(). Se um nome exige um comentário para explicar seu significado, ele violou este princípio.`

- **Funções Pequenas e de Responsabilidade Única**: Funções devem ser pequenas e fazer apenas uma coisa, fazendo-a bem e de forma exclusiva.<br>`OBS: Regra de ouro: uma função deve ter poucas linhas e um único nível de abstração. Se ela possui condicionais aninhadas profundas ou realiza múltiplos passos heterogêneos, deve ser dividida.`

- **Comentários Apenas quando Necessários**: O código deve ser autoexplicativo. Comentários não compensam código ruim.<br>`OBS: Comentários frequentemente ficam desatualizados e mentem. Prefira reescrever o código para torná-lo claro antes de adicionar um comentário explicativo. Use comentários prioritariamente para alertar sobre consequências ou decisões de negócio complexas.`

- **Tratamento de Erros Limpo**: O tratamento de erros é importante, mas se ele obscurece a lógica principal, está errado.<br>`OBS: Prefira lançar exceções a retornar códigos de erro ou null. Retornar null força verificações constantes e espalha NullReferenceException pelo sistema.`

- **Regra do Escoteiro (Boy Scout Rule)**: Deixe a área do código mais limpa do que como você a encontrou.<br>`OBS: Se você precisou mexer em um trecho de código e encontrou um nome ruim ou função longa, refatore um pequeno pedaço antes de entregar sua tarefa. A melhoria contínua previne a degradação do sistema.`

### Exemplos

> O Código "Sem Clean Code" (Violando os princípios)

Aqui, o método possui nomes confusos, múltiplos níveis de abstração, efeitos colaterais e torna a leitura extremamente difícil.

~~~c#
public class Process
{
    // Método com nome vago, parâmetros obscuros e retorno de null/códigos genéricos
    public List<user> p(List<user> l, int t)
    {
        List<user> res = new List<user>();
        foreach (var u in l)
        {
            // O que significa o número 1 ou o parâmetro t? Número mágico!
            if (u.st == 1 && u.age >= t)
            {
                // Efeito colateral oculto dentro de uma busca
                u.lastChecked = DateTime.Now;
                res.Add(u);
            }
        }
        return res;
    }
}
~~~

Por que este código é ruim?

- Nomes Ruins: O que significa `p`, `l`, `t` ou `st`? É necessário ler toda a implementação para tentar adivinhar a intenção do autor.

- Números Mágicos e Efeitos Colaterais: O valor `1` não revela seu significado comercial. Além disso, alterar `u.lastChecked` dentro de um método de filtragem gera um efeito colateral inesperado.

- Difícil de Manter: Alterar qualquer regra requer entender todo o método e arrisca quebrar comportamentos escondidos.

> O Código "Com Clean Code" (Refatorado)

Aqui, aplicamos nomenclaturas expressivas, constantes descritivas, isolamento de responsabilidades e eliminamos efeitos colaterais.

~~~c#
public class UserFilterService
{
    private const int ActiveStatusId = 1;

    public IEnumerable<User> GetActiveUsersOlderThan(IEnumerable<User> users, int minimumAge)
    {
        return users.Where(user => IsActiveAndEligible(user, minimumAge));
    }

    private bool IsActiveAndEligible(User user, int minimumAge)
    {
        return user.StatusId == ActiveStatusId && user.Age >= minimumAge;
    }
}
~~~

Por que este código é excelente?

- Autoexplicativo: O nome do método `GetActiveUsersOlderThan` expressa exatamente o que ele faz. Não há necessidade de comentários.

- Clareza e Legibilidade: O uso do número mágico foi substituído pela constante `ActiveStatusId`, e a condição de validação foi extraída para um método auxiliar privado.

- Sem Efeitos Colaterais: A função é pura no contexto de consulta — ela não altera o estado dos objetos filtrados enquanto percorre a lista.

`NOTA: Note que o código limpo exige mais disciplina ao nomear e estruturar, mas reduz drasticamente o tempo necessário para entender e evoluir o sistema no futuro.`

### Material Complementar

- [Clean Code em C# - Práticas para escrever um código limpo](https://balta.io/blog/clean-code)
- [Regras de Ouro do Clean Code com exemplos práticos](https://medium.com/developer-circus/clean-code-principles-every-developer-should-know-10023a1059f1)
- [Código Limpo: Dicas e boas práticas por Robert C. Martin](https://blog.cleancoder.com/)
