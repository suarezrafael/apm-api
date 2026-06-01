# Agent Skills — apm-api

Este arquivo define as **habilidades (skills)** disponíveis para o agente GitHub Copilot neste repositório.
Habilite em: **Ferramentas > Opções > GitHub Copilot > Habilitar Habilidades do Agente dos arquivos SKILL.md**.

---

## skill: create-api-resource

**Descrição:** Cria um recurso RESTful completo para uma entidade de domínio.

**O agente irá gerar:**
1. Entidade em `03-Domain/Entities/` herdando de `BaseDomainEntity`
2. Enum de status em `03-Domain/Enum/` (se aplicável)
3. DTOs de Request em `02-Application/Dtos/Requests/<Recurso>/`
   - `<Recurso>CreateRequest`
   - `<Recurso>UpdateRequest`
   - `<Recurso>FilterRequest`
4. DTOs de Response em `02-Application/Dtos/Responses/<Recurso>/`
   - `<Recurso>Response`
5. Validators em `02-Application/Validators/`
   - `<Recurso>CreateRequestValidator`
   - `<Recurso>UpdateRequestValidator`
6. Interface de Service `I<Recurso>Service` em `02-Application/Services/`
7. Implementação `<Recurso>Service` em `02-Application/Services/`
8. Interface de Repositório `I<Recurso>Repository` em `04-Infra/4.2-Data/Interfaces/`
9. Implementação `<Recurso>Repository` em `04-Infra/4.2-Data/Repositories/`
10. Controller `<Recurso>Controller` em `01-Presentation/Controllers/`
11. Testes unitários em `05-Tests/<Recurso>/`

**Referências obrigatórias:**
- `.github/instructions/csharp-api.instructions.md`
- `docs/architecture.md`

**Exemplo de uso no Copilot Chat:**
```
Usar skill create-api-resource para o recurso HomeworkTopic com campos:
Title (string, obrigatório), Description (string, opcional),
SchoolId (int), DueDate (DateTime), StatusId (int → enum Draft/Published/Closed)
```

---

## skill: create-entity

**Descrição:** Cria apenas a entidade de domínio e seu enum de status.

**O agente irá gerar:**
- `03-Domain/Entities/<Recurso>.cs` herdando de `BaseDomainEntity`
- `03-Domain/Enum/<Recurso>StatusEnum.cs` (se solicitado)

**Exemplo:**
```
Usar skill create-entity para Homework com status: Pending, Submitted, Graded
```

---

## skill: create-repository

**Descrição:** Cria a interface e a implementação do repositório para uma entidade existente.

**O agente irá gerar:**
- `04-Infra/4.2-Data/Interfaces/I<Recurso>Repository.cs`
- `04-Infra/4.2-Data/Repositories/<Recurso>Repository.cs`

**Convenções aplicadas:**
- Primary constructor com `ProjectContext`
- `protected override DbSet<TEntity> DbSet => projectContext.<Recurso>s`
- `AsNoTracking()` em todas as queries de leitura
- `IgnoreQueryFilters()` quando precisar de soft-deleted

**Exemplo:**
```
Usar skill create-repository para a entidade HomeworkTopic
com métodos: GetBySchoolAsync(int schoolId), GetByStatusAsync(int statusId)
```

---

## skill: create-service

**Descrição:** Cria a interface e a implementação do service para um recurso existente.

**O agente irá gerar:**
- `02-Application/Services/I<Recurso>Service.cs`
- `02-Application/Services/<Recurso>Service.cs`

**Convenções aplicadas:**
- Primary constructor com injeção do repositório + `ILogger<T>`
- `async/await` em todos os métodos
- Exceções do namespace `<NomeDoProjeto>.Shared.Exceptions`

---

## skill: create-controller

**Descrição:** Cria o controller para um recurso com service já existente.

**O agente irá gerar:**
- `01-Presentation/Controllers/<Recurso>Controller.cs`

**Convenções aplicadas:**
- `[Authorize]`, `[ApiController]`, `[Route("v1/[controller]")]`
- Headers `X-User-Profile` e `X-SchoolId` em todos os endpoints
- Retorno `IResult` com `Results.Ok/Created/NoContent/NotFound`
- `[SwaggerOperation]` e `[SwaggerResponse]` em cada action
- `[Tags("NN. NomeDoGrupo")]`

---

## skill: generate-tests

**Descrição:** Gera testes unitários para um service existente.

**O agente irá gerar:**
- `05-Tests/<Recurso>/<Recurso>ServiceTests.cs`

**Cobertura mínima:**
- Happy path para cada método público
- Cenário de entidade não encontrada (`NotFoundException`)
- Cenário de dados inválidos (quando aplicável)

**Nomenclatura:** `NomeDoMetodo_Cenario_ResultadoEsperado`

**Exemplo:**
```
Usar skill generate-tests para HomeworkTopicService cobrindo
GetByIdAsync, CreateAsync e DeleteAsync
```

---

## skill: add-migration

**Descrição:** Cria uma migration do EF Core para as novas entidades.

**O agente irá executar:**
```bash
dotnet ef migrations add <NomeDaMigration> --project src/04-Infra/4.2-Data --startup-project src/01-Presentation
```

**Exemplo:**
```
Usar skill add-migration com nome AddHomeworkTopicTable
```

---

## skill: implement-plan

**Descrição:** Executa um plano de implementação completo a partir de um arquivo `.md` de plano.

**O agente irá:**
1. Ler o arquivo de plano em `docs/`
2. Executar cada etapa na ordem definida
3. Seguir todas as convenções de `.github/copilot-instructions.md`
4. Reportar progresso etapa a etapa

**Exemplo:**
```
Usar skill implement-plan com base em docs/implementation-plan-homework-api.md
```
