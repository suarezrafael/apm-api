# GitHub Copilot Instructions — apm-api

## Visão Geral do Projeto

Este repositório implementa uma **Web API RESTful em C# (.NET 8)** seguindo os princípios de **Clean Architecture** com camadas numeradas.

---

## Stack Tecnológica

| Camada | Tecnologia |
|---|---|
| Framework | ASP.NET Core 8 (Minimal Results + ControllerBase) |
| ORM | Entity Framework Core + EFCore.BulkExtensions |
| Cache | Redis (via `ICacheRepository`) |
| Validação | FluentValidation |
| Documentação | Swagger/Swashbuckle com annotations |
| Testes | xUnit + Moq |
| Autenticação | JWT Bearer (`[Authorize]`) |

---

## Estrutura de Camadas (prefixo numérico obrigatório)

```
src/
├── 01-Presentation/    → Controllers, Middleware, Swagger
├── 02-Application/     → Services, DTOs (Requests/Responses), Validators
├── 03-Domain/          → Entities, Enums, Attributes
├── 04-Infra/
│   └── 4.2-Data/       → Repositories, Interfaces, Mappers, DbContext
└── 05-Tests/           → Testes unitários (xUnit + Moq)
```

---

## Convenções Obrigatórias

### Namespaces
- Presentation: `Redacao.Controllers`
- Application Services: `Redacao.Application.Services`
- Application DTOs: `Redacao.Application.Dtos.Requests.<Recurso>` e `Redacao.Application.Dtos.Responses.<Recurso>`
- Domain: `Redacao.Domain.Entities` / `Redacao.Domain.Enum`
- Infra Repositories: `Redacao.Dtos.Repositories`
- Infra Interfaces: `Redacao.Data.Interfaces`

### Entidades
- Herdar sempre de `BaseDomainEntity` (contém `Guid Id = Guid.NewGuid()`)
- Usar `default!` para propriedades não-anuláveis obrigatórias
- Soft delete: propriedade `DeleteDate` herdada de `BaseEntity`
- Coleções: inicializar com `[]` (C# 12)

### Controllers
- Herdar de `ControllerBase` com `[ApiController]` e `[Authorize]`
- Rota: `[Route("v1/[controller]")]`
- Retornar `IResult` (usar `Results.Ok()`, `Results.NotFound()`, `Results.NoContent()`, etc.)
- Headers obrigatórios de contexto: `X-User-Profile` e `X-SchoolId`
- Documentar com `[SwaggerOperation]` e `[SwaggerResponse]`
- Agrupar com `[Tags("NN. NomeDoGrupo")]`

### Services (Application)
- Sempre `async/await`
- Injetar `ILogger<T>` como dependência
- Lançar exceções do namespace `Redacao.Shared.Exceptions`
- Interfaces na mesma pasta com prefixo `I`

### Repositórios
- Implementar `IRepository<TEntity, TKey>` via `BaseRepository<TEntity, TKey>`
- Primary constructor: `public class XxxRepository(ProjectContext projectContext) : BaseRepository<Xxx, Guid>(projectContext), IXxxRepository`
- Sobrescrever `DbSet<TEntity> DbSet` da base
- Usar `AsNoTracking()` em queries de leitura
- Usar `IgnoreQueryFilters()` quando precisar incluir soft-deleted

### Validators (FluentValidation)
- Um validator por request DTO
- Herdar de `AbstractValidator<TRequest>`
- Mensagens de erro em português

### Testes
- Projeto: `05-Tests`
- Nomenclatura de método: `NomeDoMetodo_Cenario_ResultadoEsperado`
- Usar `Moq` para mocks, `xUnit` como framework
- Um arquivo por serviço/use case

---

## Referências
- Arquitetura detalhada: [`docs/architecture.md`](../docs/architecture.md)
- Skills do agente: [`SKILL.md`](../SKILL.md)
- Como usar o APM: [`docs/HOW-TO-USE-APM.md`](../docs/HOW-TO-USE-APM.md)
