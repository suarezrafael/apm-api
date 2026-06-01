# Plano de Implementação — API de Temas de Casa (Homework Topics)

## Visão Geral

Este documento descreve o plano de implementação completo para o módulo de **Temas de Casa**, que permite professores e coordenadores criarem propostas de atividades domiciliares, alunos enviarem suas respostas e professores avaliarem as entregas.

**Recurso principal:** `HomeworkTopic` (Tema de Casa)
**Recursos auxiliares:** `StudentHomework` (Entrega do Aluno), `HomeworkTopicStatus` (Status do Tema)

---

## Contexto de Negócio

### Atores
| Perfil | Permissões |
|---|---|
| `Professor` | Criar, editar, publicar e avaliar temas |
| `Coordenador` | Criar, editar, publicar, arquivar temas |
| `Aluno` | Visualizar temas publicados, enviar entrega |
| `Escola` | Visualizar relatórios de todos os temas |

### Regras de Negócio
- Um tema só pode ser visualizado por alunos quando estiver com status `Published`
- A data de entrega (`DueDate`) deve ser futura no momento da criação
- Um aluno pode enviar apenas **uma** entrega por tema
- Uma entrega não pode ser alterada após a data de entrega (`DueDate`)
- Ao arquivar um tema, todas as entregas pendentes são marcadas como `Expired`
- Soft delete: nenhum registro é removido fisicamente do banco

---

## Modelo de Dados

```
HomeworkTopicStatus          HomeworkTopic
─────────────────────        ─────────────────────────────────────
Id (int, PK)                 Id (Guid, PK)  ← BaseDomainEntity
Name (string)                Title (string)
DeleteDate (DateTime?)        Description (string?)
							 SchoolId (int)
							 DueDate (DateTime)
							 StatusId (int, FK → HomeworkTopicStatus)
							 DeleteDate (DateTime?)  ← BaseEntity

StudentHomeworkStatus        StudentHomework
─────────────────────        ─────────────────────────────────────
Id (int, PK)                 Id (Guid, PK)  ← BaseDomainEntity
Name (string)                HomeworkTopicId (Guid, FK)
DeleteDate (DateTime?)        StudentId (int)
							 SubmittedAt (DateTime?)
							 ContentBlobReference (string?)
							 Grade (decimal?)
							 TeacherComment (string?)
							 StatusId (int, FK → StudentHomeworkStatus)
							 DeleteDate (DateTime?)
```

---

## Fases de Implementação

---

### Fase 1 — Domínio

**Objetivo:** Criar as entidades e enums de domínio.

**Instrução para o agente:**
```
@workspace Criar as seguintes entidades em 03-Domain/Entities/:

1. HomeworkTopicStatus (herda BaseDomainEntity):
   - Name (string, obrigatório)

2. HomeworkTopic (herda BaseDomainEntity, atributo [IsAuditableMetadata("true")]):
   - Title (string, obrigatório)
   - Description (string?, opcional)
   - SchoolId (int)
   - DueDate (DateTime)
   - StatusId (int, FK → HomeworkTopicStatus)
   - Status (virtual HomeworkTopicStatus)
   - StudentHomeworks (virtual ICollection<StudentHomework>, = [])

3. StudentHomeworkStatus (herda BaseDomainEntity):
   - Name (string, obrigatório)

4. StudentHomework (herda BaseDomainEntity):
   - HomeworkTopicId (Guid, FK → HomeworkTopic)
   - HomeworkTopic (virtual HomeworkTopic)
   - StudentId (int)
   - SubmittedAt (DateTime?)
   - ContentBlobReference (string?)
   - Grade (decimal?)
   - TeacherComment (string?)
   - StatusId (int, FK → StudentHomeworkStatus)
   - Status (virtual StudentHomeworkStatus)

Criar também em 03-Domain/Enum/:
- HomeworkTopicStatusEnum: Draft=1, Published=2, Closed=3, Archived=4
- StudentHomeworkStatusEnum: Pending=1, Submitted=2, Graded=3, Expired=4
```

**Arquivos esperados:**
```
src/03-Domain/Entities/HomeworkTopicStatus.cs
src/03-Domain/Entities/HomeworkTopic.cs
src/03-Domain/Entities/StudentHomeworkStatus.cs
src/03-Domain/Entities/StudentHomework.cs
src/03-Domain/Enum/HomeworkTopicStatusEnum.cs
src/03-Domain/Enum/StudentHomeworkStatusEnum.cs
```

**Critérios de aceitação:**
- [ ] Todas as entidades herdam de `BaseDomainEntity`
- [ ] Coleções inicializadas com `[]`
- [ ] Propriedades obrigatórias usam `= default!`
- [ ] Propriedades opcionais usam `?`
- [ ] Enums com valores inteiros explícitos

---

### Fase 2 — Infraestrutura (Repositórios)

**Objetivo:** Criar interfaces e implementações dos repositórios.

**Instrução para o agente:**
```
@workspace Criar repositórios para as entidades da Fase 1:

1. IHomeworkTopicRepository (em 04-Infra/4.2-Data/Interfaces/):
   - GetBySchoolAsync(int schoolId, CancellationToken) → IEnumerable<HomeworkTopic>
   - GetByStatusAsync(HomeworkTopicStatusEnum status, int schoolId, CancellationToken) → IEnumerable<HomeworkTopic>
   - GetOverdueAsync(int schoolId, CancellationToken) → IEnumerable<HomeworkTopic>
	 (retorna temas com DueDate < DateTime.UtcNow e status Published)

2. HomeworkTopicRepository (em 04-Infra/4.2-Data/Repositories/):
   - Primary constructor com ProjectContext
   - DbSet → projectContext.HomeworkTopics
   - Query() override: AsNoTracking + Include(Status) + Include(StudentHomeworks)
   - Implementar os 3 métodos da interface

3. IStudentHomeworkRepository (em 04-Infra/4.2-Data/Interfaces/):
   - GetByTopicAsync(Guid topicId, CancellationToken) → IEnumerable<StudentHomework>
   - GetByStudentAsync(int studentId, CancellationToken) → IEnumerable<StudentHomework>
   - GetByStudentAndTopicAsync(int studentId, Guid topicId, CancellationToken) → StudentHomework?

4. StudentHomeworkRepository (em 04-Infra/4.2-Data/Repositories/):
   - Primary constructor com ProjectContext
   - DbSet → projectContext.StudentHomeworks
   - Query() override: AsNoTracking + Include(Status) + Include(HomeworkTopic)
   - Implementar os 3 métodos da interface

Usar AsNoTracking() em todas as queries de leitura.
Seguir padrão de EssayRepository.cs como referência.
```

**Arquivos esperados:**
```
src/04-Infra/4.2-Data/Interfaces/IHomeworkTopicRepository.cs
src/04-Infra/4.2-Data/Repositories/HomeworkTopicRepository.cs
src/04-Infra/4.2-Data/Interfaces/IStudentHomeworkRepository.cs
src/04-Infra/4.2-Data/Repositories/StudentHomeworkRepository.cs
```

**Critérios de aceitação:**
- [ ] Primary constructor pattern aplicado
- [ ] `DbSet` sobrescrito
- [ ] `AsNoTracking()` em todos os métodos de leitura
- [ ] Includes necessários nas queries

---

### Fase 3 — Application (DTOs + Validators + Services)

**Objetivo:** Criar DTOs, validadores e lógica de negócio.

#### 3.1 — DTOs

**Instrução para o agente:**
```
@workspace Criar os seguintes DTOs como records em 02-Application/Dtos/:

Em Requests/HomeworkTopic/:
- HomeworkTopicCreateRequest: Title(string), Description(string?), SchoolId(int), DueDate(DateTime), StatusId(int)
- HomeworkTopicUpdateRequest: Title(string), Description(string?), DueDate(DateTime), StatusId(int)
- HomeworkTopicFilterRequest: SchoolId(int), StatusId(int?), DueDateFrom(DateTime?), DueDateTo(DateTime?), Page(int=1), PageSize(int=20)

Em Responses/HomeworkTopic/:
- HomeworkTopicResponse: Id(Guid), Title(string), Description(string?), SchoolId(int), DueDate(DateTime), StatusName(string), StudentHomeworkCount(int)
- HomeworkTopicSummaryResponse: Id(Guid), Title(string), DueDate(DateTime), StatusName(string)

Em Requests/StudentHomework/:
- StudentHomeworkSubmitRequest: HomeworkTopicId(Guid), ContentBlobReference(string?)
- StudentHomeworkGradeRequest: Grade(decimal), TeacherComment(string?)

Em Responses/StudentHomework/:
- StudentHomeworkResponse: Id(Guid), HomeworkTopicId(Guid), HomeworkTopicTitle(string), StudentId(int), SubmittedAt(DateTime?), Grade(decimal?), TeacherComment(string?), StatusName(string)
```

#### 3.2 — Validators

**Instrução para o agente:**
```
@workspace Criar validators em 02-Application/Validators/ com mensagens em português:

1. HomeworkTopicCreateRequestValidator:
   - Title: NotEmpty, MaxLength(200)
   - SchoolId: GreaterThan(0)
   - DueDate: GreaterThan(DateTime.UtcNow) com mensagem "Data de entrega deve ser uma data futura."
   - StatusId: GreaterThan(0)

2. HomeworkTopicUpdateRequestValidator:
   - Title: NotEmpty, MaxLength(200)
   - DueDate: GreaterThan(DateTime.UtcNow)
   - StatusId: GreaterThan(0)

3. StudentHomeworkSubmitRequestValidator:
   - HomeworkTopicId: NotEmpty
   - ContentBlobReference: MaxLength(500) quando não for null/empty

4. StudentHomeworkGradeRequestValidator:
   - Grade: InclusiveBetween(0, 10) com mensagem "Nota deve estar entre 0 e 10."
   - TeacherComment: MaxLength(1000) quando não for null/empty
```

#### 3.3 — Services

**Instrução para o agente:**
```
@workspace Criar os seguintes services em 02-Application/Services/:

1. IHomeworkTopicService + HomeworkTopicService:
   Dependências: IHomeworkTopicRepository, ILogger<HomeworkTopicService>

   Métodos:
   - GetByIdAsync(Guid id, CancellationToken) → HomeworkTopicResponse
	 Lança NotFoundException se não encontrado
   - GetAllAsync(HomeworkTopicFilterRequest filter, CancellationToken) → PagedResponse<IEnumerable<HomeworkTopicSummaryResponse>>
   - CreateAsync(HomeworkTopicCreateRequest request, CancellationToken) → HomeworkTopicResponse
	 Log: "Criando tema de casa: {Title} para escola {SchoolId}"
   - UpdateAsync(Guid id, HomeworkTopicUpdateRequest request, CancellationToken) → HomeworkTopicResponse
	 Lança NotFoundException se não encontrado
   - PublishAsync(Guid id, CancellationToken) → HomeworkTopicResponse
	 Regra: StatusId deve ser Draft para publicar; lança BadRequestException caso contrário
   - ArchiveAsync(Guid id, CancellationToken) → void
	 Regra: após arquivar, chama StudentHomeworkService.ExpireByTopicAsync
   - DeleteAsync(Guid id, CancellationToken) → void
	 Usa SoftDeleteAsync do repositório

2. IStudentHomeworkService + StudentHomeworkService:
   Dependências: IStudentHomeworkRepository, IHomeworkTopicRepository, ILogger<StudentHomeworkService>

   Métodos:
   - GetByIdAsync(Guid id, CancellationToken) → StudentHomeworkResponse
   - GetByStudentAsync(int studentId, CancellationToken) → IEnumerable<StudentHomeworkResponse>
   - SubmitAsync(int studentId, StudentHomeworkSubmitRequest request, CancellationToken) → StudentHomeworkResponse
	 Regras:
	   1. Tema deve estar Published; lança BadRequestException caso contrário
	   2. DueDate não pode ter passado; lança BadRequestException com "Prazo de entrega encerrado."
	   3. Aluno não pode enviar duas entregas para o mesmo tema; lança ConflictException
   - GradeAsync(Guid id, StudentHomeworkGradeRequest request, CancellationToken) → StudentHomeworkResponse
	 Regra: entrega deve estar Submitted; lança BadRequestException caso contrário
   - ExpireByTopicAsync(Guid topicId, CancellationToken) → void
	 Marca todas as entregas Pending do tema como Expired
```

**Arquivos esperados:**
```
src/02-Application/Dtos/Requests/HomeworkTopic/HomeworkTopicCreateRequest.cs
src/02-Application/Dtos/Requests/HomeworkTopic/HomeworkTopicUpdateRequest.cs
src/02-Application/Dtos/Requests/HomeworkTopic/HomeworkTopicFilterRequest.cs
src/02-Application/Dtos/Responses/HomeworkTopic/HomeworkTopicResponse.cs
src/02-Application/Dtos/Responses/HomeworkTopic/HomeworkTopicSummaryResponse.cs
src/02-Application/Dtos/Requests/StudentHomework/StudentHomeworkSubmitRequest.cs
src/02-Application/Dtos/Requests/StudentHomework/StudentHomeworkGradeRequest.cs
src/02-Application/Dtos/Responses/StudentHomework/StudentHomeworkResponse.cs
src/02-Application/Validators/HomeworkTopicCreateRequestValidator.cs
src/02-Application/Validators/HomeworkTopicUpdateRequestValidator.cs
src/02-Application/Validators/StudentHomeworkSubmitRequestValidator.cs
src/02-Application/Validators/StudentHomeworkGradeRequestValidator.cs
src/02-Application/Services/IHomeworkTopicService.cs
src/02-Application/Services/HomeworkTopicService.cs
src/02-Application/Services/IStudentHomeworkService.cs
src/02-Application/Services/StudentHomeworkService.cs
```

**Critérios de aceitação:**
- [ ] DTOs como `record`
- [ ] Validators com mensagens em português
- [ ] Services com `async/await` em todos os métodos
- [ ] Logger injetado e usado nos métodos de escrita
- [ ] Exceções do namespace `<NomeDoProjeto>.Shared.Exceptions`

---

### Fase 4 — Presentation (Controllers)

**Objetivo:** Criar os controllers REST.

**Instrução para o agente:**
```
@workspace Criar os seguintes controllers em 01-Presentation/Controllers/:

1. HomeworkTopicController:
   Tag: "05. Homework Topics"
   Rota base: v1/homeworktopic

   Endpoints:
   GET    /v1/homeworktopic/{id}        → GetHomeworkTopicByIdAsync
	 Roles: Aluno, Escola, Coordenador, Professor

   GET    /v1/homeworktopic             → GetAllHomeworkTopicsAsync
	 [FromQuery] HomeworkTopicFilterRequest
	 Roles: Escola, Coordenador, Professor

   POST   /v1/homeworktopic             → CreateHomeworkTopicAsync
	 [FromBody] HomeworkTopicCreateRequest
	 Roles: Escola, Coordenador, Professor
	 Retorna: Results.Created com location header

   PUT    /v1/homeworktopic/{id}        → UpdateHomeworkTopicAsync
	 [FromBody] HomeworkTopicUpdateRequest
	 Roles: Escola, Coordenador, Professor

   PATCH  /v1/homeworktopic/{id}/publish → PublishHomeworkTopicAsync
	 Roles: Escola, Coordenador, Professor

   PATCH  /v1/homeworktopic/{id}/archive → ArchiveHomeworkTopicAsync
	 Roles: Escola, Coordenador

   DELETE /v1/homeworktopic/{id}        → DeleteHomeworkTopicAsync
	 Roles: Escola, Coordenador
	 Retorna: Results.NoContent()

2. StudentHomeworkController:
   Tag: "06. Student Homework"
   Rota base: v1/studenthomework

   Endpoints:
   GET    /v1/studenthomework/{id}            → GetStudentHomeworkByIdAsync
	 Roles: Aluno, Professor, Coordenador

   GET    /v1/studenthomework/student/{studentId} → GetByStudentAsync
	 Roles: Aluno, Professor, Coordenador, Responsavel

   POST   /v1/studenthomework/submit          → SubmitHomeworkAsync
	 [FromBody] StudentHomeworkSubmitRequest
	 Roles: Aluno
	 Retorna: Results.Created

   PATCH  /v1/studenthomework/{id}/grade      → GradeHomeworkAsync
	 [FromBody] StudentHomeworkGradeRequest
	 Roles: Professor, Coordenador

Todos os endpoints devem ter:
- Headers: X-User-Profile e X-SchoolId como [Required][FromHeader]
- SwaggerOperation com Summary e OperationId
- SwaggerResponse para 200/201/204, 400, 401, 403, 404, 500
- CancellationToken no parâmetro
```

**Arquivos esperados:**
```
src/01-Presentation/Controllers/HomeworkTopicController.cs
src/01-Presentation/Controllers/StudentHomeworkController.cs
```

**Critérios de aceitação:**
- [ ] `[Authorize]`, `[ApiController]`, `[Route("v1/[controller]")]`
- [ ] Headers obrigatórios em todos os endpoints
- [ ] Retorno `IResult` com `Results.*`
- [ ] `[Tags]` configurado
- [ ] Swagger documentado

---

### Fase 5 — Testes Unitários

**Objetivo:** Garantir cobertura dos services com cenários críticos.

**Instrução para o agente:**
```
@workspace Criar testes unitários em 05-Tests/ usando xUnit + Moq:

1. HomeworkTopicServiceTests (em 05-Tests/HomeworkTopic/):
   Testar HomeworkTopicService com Mock<IHomeworkTopicRepository>:

   - GetByIdAsync_ExistingId_ReturnsResponse
   - GetByIdAsync_NotFound_ThrowsNotFoundException
   - CreateAsync_ValidRequest_ReturnsCreatedResponse
   - CreateAsync_ValidRequest_CallsAddAsync
   - UpdateAsync_ExistingId_ReturnsUpdatedResponse
   - UpdateAsync_NotFound_ThrowsNotFoundException
   - PublishAsync_DraftTopic_ChangesStatusToPublished
   - PublishAsync_AlreadyPublished_ThrowsBadRequestException
   - DeleteAsync_ExistingId_CallsSoftDeleteAsync
   - DeleteAsync_NotFound_ThrowsNotFoundException

2. StudentHomeworkServiceTests (em 05-Tests/StudentHomework/):
   Testar StudentHomeworkService com mocks dos repositórios:

   - SubmitAsync_ValidRequest_PublishedTopic_ReturnsResponse
   - SubmitAsync_TopicNotPublished_ThrowsBadRequestException
   - SubmitAsync_PastDueDate_ThrowsBadRequestException com mensagem "Prazo de entrega encerrado."
   - SubmitAsync_DuplicateSubmission_ThrowsConflictException
   - GradeAsync_SubmittedHomework_ReturnsGradedResponse
   - GradeAsync_NotSubmitted_ThrowsBadRequestException
   - ExpireByTopicAsync_PendingHomeworks_MarksAsExpired

Nomenclatura obrigatória: NomeDoMetodo_Cenario_ResultadoEsperado
```

**Arquivos esperados:**
```
src/05-Tests/HomeworkTopic/HomeworkTopicServiceTests.cs
src/05-Tests/StudentHomework/StudentHomeworkServiceTests.cs
```

**Critérios de aceitação:**
- [ ] Nomenclatura `NomeDoMetodo_Cenario_ResultadoEsperado`
- [ ] Mocks para todos os repositórios
- [ ] Happy path + cenários de erro
- [ ] Assertions claras com `Assert.Equal`, `Assert.NotNull`, `Assert.ThrowsAsync`

---

### Fase 6 — Migration e Registro de DI

**Objetivo:** Configurar persistência e injeção de dependências.

**Instrução para o agente:**
```
@workspace Executar as seguintes tarefas de infraestrutura:

1. Adicionar DbSets no ProjectContext:
   - public DbSet<HomeworkTopic> HomeworkTopics { get; set; }
   - public DbSet<HomeworkTopicStatus> HomeworkTopicStatuses { get; set; }
   - public DbSet<StudentHomework> StudentHomeworks { get; set; }
   - public DbSet<StudentHomeworkStatus> StudentHomeworkStatuses { get; set; }

2. Criar Mappers EF Core (em 04-Infra/4.2-Data/Mappers/):
   - HomeworkTopicMapper.cs: configurar tabela, índices em SchoolId e StatusId
   - StudentHomeworkMapper.cs: configurar unique constraint (StudentId, HomeworkTopicId)

3. Registrar repositórios e services no DI (em Program.cs ou extensão de ServiceCollection):
   - services.AddScoped<IHomeworkTopicRepository, HomeworkTopicRepository>()
   - services.AddScoped<IStudentHomeworkRepository, StudentHomeworkRepository>()
   - services.AddScoped<IHomeworkTopicService, HomeworkTopicService>()
   - services.AddScoped<IStudentHomeworkService, StudentHomeworkService>()

4. Criar migration:
   dotnet ef migrations add AddHomeworkTopicModule --project src/04-Infra/4.2-Data --startup-project src/01-Presentation
```

---

## Checklist de Conclusão

### Fase 1 — Domínio
- [ ] `HomeworkTopic.cs` criado
- [ ] `HomeworkTopicStatus.cs` criado
- [ ] `StudentHomework.cs` criado
- [ ] `StudentHomeworkStatus.cs` criado
- [ ] `HomeworkTopicStatusEnum.cs` criado
- [ ] `StudentHomeworkStatusEnum.cs` criado

### Fase 2 — Repositórios
- [ ] `IHomeworkTopicRepository.cs` criado
- [ ] `HomeworkTopicRepository.cs` criado
- [ ] `IStudentHomeworkRepository.cs` criado
- [ ] `StudentHomeworkRepository.cs` criado

### Fase 3 — Application
- [ ] Todos os DTOs criados como `record`
- [ ] Todos os validators criados com mensagens em português
- [ ] `IHomeworkTopicService.cs` + `HomeworkTopicService.cs` criados
- [ ] `IStudentHomeworkService.cs` + `StudentHomeworkService.cs` criados

### Fase 4 — Controllers
- [ ] `HomeworkTopicController.cs` criado
- [ ] `StudentHomeworkController.cs` criado
- [ ] Swagger documentado em todos os endpoints

### Fase 5 — Testes
- [ ] `HomeworkTopicServiceTests.cs` criado (min. 10 cenários)
- [ ] `StudentHomeworkServiceTests.cs` criado (min. 7 cenários)
- [ ] Todos os testes passando

### Fase 6 — Infraestrutura
- [ ] `DbSet`s adicionados ao `ProjectContext`
- [ ] Mappers EF Core criados
- [ ] DI configurado
- [ ] Migration criada e aplicada

---

## Estimativa de Esforço

| Fase | Arquivos | Estimativa (agente) |
|---|---|---|
| 1 - Domínio | 6 | ~2 min |
| 2 - Repositórios | 4 | ~3 min |
| 3 - Application | 16 | ~8 min |
| 4 - Controllers | 2 | ~4 min |
| 5 - Testes | 2 | ~5 min |
| 6 - Infraestrutura | 4+ | ~5 min |
| **Total** | **34+** | **~27 min** |
