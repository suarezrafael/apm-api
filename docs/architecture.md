# Arquitetura — apm-api

## Visão Geral

A **apm-api** segue os princípios de **Clean Architecture** com camadas numeradas que expressam a direção das dependências: camadas internas nunca dependem de camadas externas.

```
┌─────────────────────────────────────────────────┐
│  01 - Presentation (Controllers, Middleware)     │
│  ↓ depende de                                    │
│  02 - Application (Services, DTOs, Validators)   │
│  ↓ depende de                                    │
│  03 - Domain (Entities, Enums, Attributes)       │
│  ↑ implementado por                              │
│  04 - Infra / 4.2-Data (Repos, DbContext)        │
│                                                  │
│  05 - Tests (xUnit + Moq)                        │
└─────────────────────────────────────────────────┘
```

> **Regra de ouro:** `03-Domain` não importa nenhuma das outras camadas.

---

## Camadas em Detalhe

### 01 - Presentation
**Namespace:** `Redacao.Controllers`
**Responsabilidade:** Receber requisições HTTP e delegar para a camada de Application.

| Componente | Descrição |
|---|---|
| `Controllers/` | Herdam de `ControllerBase`, retornam `IResult` |
| `Middleware/` | Cross-cutting concerns: logging, correlationId, claims |
| `Swagger/` | Configuração do Swashbuckle, filtros de documento |

**Padrão de Controller:**
```csharp
[Authorize]
[ApiController]
[Route("v1/[controller]")]
[Tags("NN. NomeDoGrupo")]
public class RecursoController(ILogger<RecursoController> logger, IRecursoService service) : ControllerBase
{
	[HttpGet("{id}", Name = nameof(GetByIdAsync))]
	[SwaggerOperation(Summary = "...", OperationId = "GetByIdAsync")]
	[SwaggerResponse(200, "Sucesso")]
	[SwaggerResponse(404, "Não encontrado")]
	public async Task<IResult> GetByIdAsync(
		[Required][FromHeader(Name = "X-User-Profile")] string profile,
		[Required][FromHeader(Name = "X-SchoolId")] int schoolId,
		Guid id, CancellationToken cancellationToken)
	{
		return Results.Ok(await service.GetByIdAsync(id, cancellationToken));
	}
}
```

---

### 02 - Application
**Namespaces:** `Redacao.Application.Services`, `Redacao.Application.Dtos.*`, `Redacao.Application.Validators`
**Responsabilidade:** Orquestrar regras de negócio, transformar dados, validar entradas.

**Estrutura interna:**
```
02-Application/
├── Dtos/
│   ├── Requests/
│   │   └── <Recurso>/
│   │       ├── <Recurso>CreateRequest.cs
│   │       ├── <Recurso>UpdateRequest.cs
│   │       └── <Recurso>FilterRequest.cs
│   └── Responses/
│       └── <Recurso>/
│           └── <Recurso>Response.cs
├── Services/
│   ├── I<Recurso>Service.cs
│   └── <Recurso>Service.cs
└── Validators/
	├── <Recurso>CreateRequestValidator.cs
	└── <Recurso>UpdateRequestValidator.cs
```

**DTOs como records (imutáveis):**
```csharp
public record HomeworkTopicCreateRequest(
	string Title,
	string? Description,
	int SchoolId,
	DateTime DueDate,
	int StatusId
);
```

**Services com primary constructor (C# 12):**
```csharp
public class HomeworkTopicService(
	IHomeworkTopicRepository repository,
	ILogger<HomeworkTopicService> logger) : IHomeworkTopicService { }
```

---

### 03 - Domain
**Namespaces:** `Redacao.Domain.Entities`, `Redacao.Domain.Enum`, `Redacao.Domain.Attributes`
**Responsabilidade:** Modelagem do domínio. Não depende de nenhuma outra camada.

**Hierarquia de entidades:**
```
BaseEntity                  → DeleteDate (soft delete), CreateDate, UpdateDate
  └── BaseDomainEntity      → Guid Id = Guid.NewGuid()
		└── <Entidade>      → propriedades de negócio
```

**Exemplo de entidade:**
```csharp
namespace Redacao.Domain.Entities
{
	public class HomeworkTopic : BaseDomainEntity
	{
		public string Title { get; set; } = default!;
		public string? Description { get; set; }
		public int SchoolId { get; set; }
		public DateTime DueDate { get; set; }
		public int StatusId { get; set; }
		public virtual HomeworkTopicStatus Status { get; set; } = default!;
		public virtual ICollection<StudentHomework> StudentHomeworks { get; set; } = [];
	}
}
```

**Soft Delete:** Nunca deletar registros do banco. Usar `SoftDeleteAsync()` do repositório base que preenche `DeleteDate`. O `DbContext` aplica filtro global automático excluindo registros com `DeleteDate != null`.

---

### 04 - Infra / 4.2-Data
**Namespaces:** `Redacao.Dtos.Repositories` (implementações), `Redacao.Data.Interfaces` (contratos)
**Responsabilidade:** Persistência de dados, cache, integrações externas.

**Estrutura interna:**
```
04-Infra/4.2-Data/
├── Interfaces/
│   ├── IRepository<TEntity, TKey>.cs   → contrato genérico base
│   └── I<Recurso>Repository.cs         → contrato específico
├── Repositories/
│   ├── BaseRepository<TEntity, TKey>.cs → implementação genérica
│   └── <Recurso>Repository.cs          → implementação específica
├── Mappers/                             → configurações EF Core (Fluent API)
├── DbContext/ProjectContext.cs
└── DatabaseInitializer.cs
```

**Padrão de repositório:**
```csharp
public class HomeworkTopicRepository(ProjectContext projectContext)
	: BaseRepository<HomeworkTopic, Guid>(projectContext), IHomeworkTopicRepository
{
	protected override DbSet<HomeworkTopic> DbSet => projectContext.HomeworkTopics;

	public override IQueryable<HomeworkTopic> Query()
		=> DbSet.AsNoTracking().Include(h => h.Status);
}
```

**Regras de consulta:**
- Leitura: sempre `AsNoTracking()`
- Incluir soft-deleted: `IgnoreQueryFilters()`
- Operações em lote: `BulkInsertAsync()` via EFCore.BulkExtensions
- Paginação: `FindAsync<TResponse, TOrderingKey>(...)` do BaseRepository

---

### 05 - Tests
**Framework:** xUnit + Moq
**Namespace:** `Redacao.Tests`

**Estrutura:**
```
05-Tests/
└── <Recurso>/
	└── <Recurso>ServiceTests.cs
```

**Convenções:**
- Nomenclatura: `NomeDoMetodo_Cenario_ResultadoEsperado`
- Mocks via `Mock<IInterface>()` do Moq
- Um arquivo de teste por serviço
- Testar: happy path, not found, dados inválidos, edge cases

---

## Decisões Arquiteturais (ADRs)

### ADR-001: Clean Architecture com camadas numeradas
**Contexto:** Projetos grandes tendem a misturar responsabilidades.
**Decisão:** Prefixo numérico obrigatório nas pastas de camada (`01-`, `02-`, etc.) para expressar visualmente a direção de dependência e facilitar a navegação.
**Consequência:** Estrutura autodocumentada; regras de dependência evidentes pelo número.

### ADR-002: Soft Delete global via EF Core Query Filters
**Contexto:** Dados históricos precisam ser preservados para auditoria.
**Decisão:** `BaseEntity` contém `DeleteDate?`. O `DbContext` registra um filtro global `e => e.DeleteDate == null` para todas as entidades. Nunca usar `Remove()` diretamente.
**Consequência:** Toda query automaticamente exclui registros deletados. Usar `IgnoreQueryFilters()` quando necessário.

### ADR-003: IResult nos Controllers (não ActionResult)
**Contexto:** ASP.NET Core 6+ introduziu `Results.*` como API mais expressiva para Minimal APIs.
**Decisão:** Controllers retornam `IResult` usando `Results.Ok()`, `Results.Created()`, `Results.NoContent()`, `Results.NotFound()`.
**Consequência:** Sintaxe mais limpa e consistente entre controllers.

### ADR-004: Primary Constructors (C# 12)
**Contexto:** Injeção de dependência gera boilerplate desnecessário.
**Decisão:** Usar primary constructors em Services, Repositories e Controllers onde possível.
**Consequência:** Código mais conciso; parâmetros imutáveis por convenção.

### ADR-005: Redis para cache de dados frequentes
**Contexto:** Dados como listas de status, tipos e configurações de escola mudam raramente.
**Decisão:** Injetar `ICacheRepository` nos services que precisam de cache. Nunca acessar Redis diretamente nos controllers.
**Consequência:** Cache centralizado e testável.

### ADR-006: FluentValidation para validação de entrada
**Contexto:** Validação inline nos controllers gera acoplamento e dificulta testes.
**Decisão:** Um `AbstractValidator<TRequest>` por DTO de request. Registrado via DI. Mensagens em português.
**Consequência:** Validações isoladas, testáveis e centralizadas.

---

## Fluxo de uma Requisição

```
HTTP Request
	│
	▼
[Middleware] CorrelationId, UserProfile, Logging
	│
	▼
[Controller] Valida headers X-User-Profile e X-SchoolId
	│         FluentValidation do request body (automático via DI)
	▼
[Service]   Orquestra: valida regras de negócio, chama repositório
	│         Lança NotFoundException, BadRequestException se necessário
	▼
[Repository] Executa query no EF Core (AsNoTracking em leituras)
	│          Usa cache Redis quando disponível
	▼
[DbContext]  Aplica global query filter (soft delete)
	│
	▼
[Response]  DTO mapeado → Results.Ok(dto)
```

---

## Convenções de Nomenclatura

| Artefato | Padrão | Exemplo |
|---|---|---|
| Entidade | PascalCase singular | `HomeworkTopic` |
| Controller | `<Entidade>Controller` | `HomeworkTopicController` |
| Service Interface | `I<Entidade>Service` | `IHomeworkTopicService` |
| Service Impl | `<Entidade>Service` | `HomeworkTopicService` |
| Repository Interface | `I<Entidade>Repository` | `IHomeworkTopicRepository` |
| Repository Impl | `<Entidade>Repository` | `HomeworkTopicRepository` |
| Create DTO | `<Entidade>CreateRequest` | `HomeworkTopicCreateRequest` |
| Update DTO | `<Entidade>UpdateRequest` | `HomeworkTopicUpdateRequest` |
| Filter DTO | `<Entidade>FilterRequest` | `HomeworkTopicFilterRequest` |
| Response DTO | `<Entidade>Response` | `HomeworkTopicResponse` |
| Validator | `<DTO>Validator` | `HomeworkTopicCreateRequestValidator` |
| Teste | `<Service>Tests` | `HomeworkTopicServiceTests` |
