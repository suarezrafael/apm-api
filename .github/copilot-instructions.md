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
├── 01-Presentation/              → Controllers, Middleware, Swagger, Dockerfile
├── 02-Application/               → Services, DTOs (Requests/Responses), Validators
├── 03-Domain/                    → Entities, Enums, Attributes
├── 04-Infra/
│   ├── 4.1-CrossCutting/
│   │   ├── ControllerHandler/    → Middleware de tratamento de exceções HTTP
│   │   ├── IoC/                  → Injeção de dependência centralizada (DI.cs, ApplicationModule, DataModule, InfrastructureModule)
│   │   └── Shared/<NomeDoProjeto>.Shared → Exceções, DTOs compartilhados, Extensions, PagedResponse
│   └── 4.2-Data/                 → Repositories, Interfaces, Mappers, DbContext
├── 05-Tests/                     → Testes unitários (xUnit + Moq)
├── <NomeDoProjeto>.sln                   → Solution vinculando todos os projetos
├── docker-compose.yml            → Orquestração: API + PostgreSQL + Redis + PgAdmin
└── docker-compose.override.yml   → Overrides para desenvolvimento local
```

---

## Convenções Obrigatórias

### Namespaces
- Presentation: `<NomeDoProjeto>.Controllers`
- Application Services: `<NomeDoProjeto>.Application.Services`
- Application DTOs: `<NomeDoProjeto>.Application.Dtos.Requests.<Recurso>` e `<NomeDoProjeto>.Application.Dtos.Responses.<Recurso>`
- Domain: `<NomeDoProjeto>.Domain.Entities` / `<NomeDoProjeto>.Domain.Enum`
- Infra Repositories: `<NomeDoProjeto>.Dtos.Repositories`
- Infra Interfaces: `<NomeDoProjeto>.Data.Interfaces`

### Entidades
- Herdar sempre de `BaseDomainEntity` (contém `Guid Id = Guid.NewGuid()`)
- Usar `default!` para propriedades não-anuláveis obrigatórias
- Soft delete: propriedade `DeleteDate` herdada de `BaseDomainEntity`
- Coleções: inicializar com `[]` (C# 12)


#### Arquivos base obrigatorios -- criar se nao existirem

Ao iniciar um projeto, os arquivos abaixo **devem ser criados** pois os demais projetos dependem deles para compilar.

**`03-Domain/Entities/BaseDomainEntity.cs`**

`csharp
namespace <NomeDoProjeto>.Domain.Entities
{
    public abstract class BaseDomainEntity
    {
        public Guid Id { get; init; } = Guid.NewGuid();
        public DateTime? DeleteDate { get; set; }
    }
}
`"

**`04-Infra/4.1-CrossCutting/Shared/<NomeDoProjeto>.Shared/Repositories/IRepository.cs`**

`csharp
namespace <NomeDoProjeto>.Shared.Repositories
{
    public interface IRepository<TEntity, TKey> where TEntity : class
    {
        Task<TEntity?> GetByIdAsync(TKey id, CancellationToken ct = default);
        Task<IEnumerable<TEntity>> GetAllAsync(CancellationToken ct = default);
        Task AddAsync(TEntity entity, CancellationToken ct = default);
        Task UpdateAsync(TEntity entity, CancellationToken ct = default);
        Task DeleteAsync(TEntity entity, CancellationToken ct = default);
    }
}
`"

> Interfaces em `<NomeDoProjeto>.Data.Interfaces` devem usar `using <NomeDoProjeto>.Shared.Repositories;`.### Controllers
- Herdar de `ControllerBase` com `[ApiController]` e `[Authorize]`
- Rota: `[Route("v1/[controller]")]`
- Retornar `IResult` (usar `Results.Ok()`, `Results.NotFound()`, `Results.NoContent()`, etc.)
- Headers obrigatórios de contexto: `X-User-Profile` e `X-SchoolId`
- Documentar com `[SwaggerOperation]` e `[SwaggerResponse]`
- Agrupar com `[Tags("NN. NomeDoGrupo")]`

### Services (Application)
- Sempre `async/await`
- Injetar `ILogger<T>` como dependência
- Lançar exceções do namespace `<NomeDoProjeto>.Shared.Exceptions`
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

## Injeção de Dependência — Camada 4.1-CrossCutting/IoC

Todo registro de DI é **centralizado** no projeto `<NomeDoProjeto>.IoC` (pasta `04-Infra/4.1-CrossCutting/IoC/`). Nunca registrar serviços diretamente no `Program.cs`.

### Estrutura obrigatória

| Arquivo | Responsabilidade |
|---|---|
| `DI.cs` | Ponto de entrada único: chama `AddInfrastructureModule`, `AddDataModule`, `AddApplicationModule` |
| `ApplicationModule.cs` | Validators (FluentValidation) + Application Services |
| `DataModule.cs` | DbContext (Npgsql), Redis, Repositórios |
| `InfrastructureModule.cs` | JWT, Swagger, HealthChecks, CORS, demais infra |

### Regra: ao criar um novo recurso, registrar em

- **Novo Service** → adicionar em `ApplicationModule.cs`:
  ```csharp
  services.AddScoped<IHomeworkTopicService, HomeworkTopicService>();
  ```
- **Novo Repository** → adicionar em `DataModule.cs`:
  ```csharp
  services.AddScoped<IHomeworkTopicRepository, HomeworkTopicRepository>();
  ```
- **Novo Validator** → adicionar em `ApplicationModule.cs`:
  ```csharp
  services.AddScoped<IValidator<HomeworkTopicCreateRequest>, HomeworkTopicCreateRequestValidator>();
  ```

### Namespace do IoC
```
<NomeDoProjeto>.IoC.DependencyInjection
```

### Chamada no Program.cs
```csharp
builder.Services.AddServices(builder.Configuration); // via DI.cs
```

---

## Solution — <NomeDoProjeto>.sln

O arquivo `src/<NomeDoProjeto>.sln` deve conter **todos os projetos** da solução. Ao criar um novo projeto `.csproj`, vincular à solution com:

```bash
dotnet sln src/<NomeDoProjeto>.sln add <caminho-relativo-ao-csproj>
```

### Projetos existentes na solution

| Projeto | Caminho |
|---|---|
| `<NomeDoProjeto>.Api` | `01-Presentation/<NomeDoProjeto>.Api.csproj` |
| `<NomeDoProjeto>.Application` | `02-Application/<NomeDoProjeto>.Application.csproj` |
| `<NomeDoProjeto>.Domain` | `03-Domain/<NomeDoProjeto>.Domain.csproj` |
| `<NomeDoProjeto>.ControllerHandler` | `04-Infra/4.1-CrossCutting/ControllerHandler/<NomeDoProjeto>.ControllerHandler.csproj` |
| `<NomeDoProjeto>.IoC` | `04-Infra/4.1-CrossCutting/IoC/<NomeDoProjeto>.IoC.csproj` |
| `<NomeDoProjeto>.Shared` | `04-Infra/4.1-CrossCutting/Shared/<NomeDoProjeto>.Shared/<NomeDoProjeto>.Shared.csproj` |
| `<NomeDoProjeto>.Data` | `04-Infra/4.2-Data/<NomeDoProjeto>.Data.csproj` |
| `<NomeDoProjeto>.Tests` | `05-Tests/<NomeDoProjeto>.Tests.csproj` |

### Pastas de solução (Solution Folders)
Ao adicionar um projeto novo, organizar dentro da pasta de solução correspondente (`01-Presentation`, `02-Application`, etc.) usando:
```bash
dotnet sln src/<NomeDoProjeto>.sln add <csproj> --solution-folder <NomePasta>
```

---

## Docker

### Dockerfile (`01-Presentation/Dockerfile`)

O Dockerfile segue padrão **multi-stage build** (`base → build → publish → final`):

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
# Copiar todos os .csproj antes do restore (cache de camadas)
COPY ["01-Presentation/<NomeDoProjeto>.Api.csproj", "01-Presentation/"]
COPY ["04-Infra/4.1-CrossCutting/IoC/<NomeDoProjeto>.IoC.csproj", "04-Infra/4.1-CrossCutting/IoC/"]
COPY ["02-Application/<NomeDoProjeto>.Application.csproj", "02-Application/"]
COPY ["03-Domain/<NomeDoProjeto>.Domain.csproj", "03-Domain/"]
COPY ["04-Infra/4.2-Data/<NomeDoProjeto>.Data.csproj", "04-Infra/4.2-Data/"]
COPY ["04-Infra/4.1-CrossCutting/ControllerHandler/<NomeDoProjeto>.ControllerHandler.csproj", "04-Infra/4.1-CrossCutting/ControllerHandler/"]
COPY ["04-Infra/4.1-CrossCutting/Shared/<NomeDoProjeto>.Shared/<NomeDoProjeto>.Shared.csproj", "04-Infra/4.1-CrossCutting/Shared/<NomeDoProjeto>.Shared/"]
RUN dotnet restore "./01-Presentation/<NomeDoProjeto>.Api.csproj"
COPY . .
WORKDIR "/src/01-Presentation"
RUN dotnet build "./<NomeDoProjeto>.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./<NomeDoProjeto>.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
USER appuser
ENTRYPOINT ["dotnet", "<NomeDoProjeto>.Api.dll"]
```

**Regras:**
- O contexto do build é sempre `src/` (não `01-Presentation/`)
- Rodar como usuário não-root (`appuser`)
- Ao adicionar um novo projeto `.csproj` como dependência, incluir a linha `COPY` antes do `dotnet restore`

### docker-compose (`src/docker-compose.yml`)

O `docker-compose.yml` orquestra os seguintes serviços:

| Serviço | Imagem | Porta |
|---|---|---|
| `<NomeDoProjeto>.api` | Build local (`01-Presentation/Dockerfile`) | 80/443 |
| `postgres` | `postgres:16.6-alpine` | 5432 |
| `pgadmin` | `dpage/pgadmin4` | 5050 |
| `redis` | `redis:latest` | 6379 |
| `redis-commander` | `rediscommander/redis-commander` | 8081 |

**Regras:**
- Variáveis de ambiente sensíveis (`CONNECTION_STRING`, `REDIS_CONNECTION_STRING`) ficam no `docker-compose.override.yml`
- O serviço da API deve ter `depends_on: postgres, redis`
- Volumes nomeados para persistência: `essay_db_data`, `redis_data`
- Todos os serviços dentro da rede `<NomeDoProjeto>`

---

## Referências
- Arquitetura detalhada: [`docs/architecture.md`](../docs/architecture.md)
- Skills do agente: [`SKILL.md`](../SKILL.md)
- Como usar o APM: [`docs/HOW-TO-USE-APM.md`](../docs/HOW-TO-USE-APM.md)
