---
applyTo: "src/**/*.cs"
---

# Instruções Específicas — API C# (apm-api)

Estas instruções se aplicam a todos os arquivos `.cs` dentro de `src/` e complementam as regras globais em `.github/copilot-instructions.md`.

---

## 1. Entidades de Domínio (`03-Domain/Entities/`)

```csharp
namespace <NomeDoProjeto>.Domain.Entities
{
	public class HomeworkTopic : BaseDomainEntity
	{
		public string Title { get; set; } = default!;
		public string? Description { get; set; }
		public int SchoolId { get; set; }
		public DateTime DueDate { get; set; }
		public int StatusId { get; set; }
		public virtual HomeworkTopicStatus Status { get; set; } = default!;

		// Coleções inicializadas com [] (C# 12)
		public virtual ICollection<StudentHomework> StudentHomeworks { get; set; } = [];
	}
}
```

**Regras:**
- Herdar de `BaseDomainEntity` (fornece `Guid Id = Guid.NewGuid()`)
- `DeleteDate` é herdado de `BaseEntity` — nunca adicionar manualmente
- Propriedades obrigatórias não-anuláveis: usar `= default!`
- Propriedades opcionais: usar `?`
- FK de inteiro: propriedade `int XxxId` + propriedade de navegação `virtual Xxx Xxx`
- Coleções: `= []` (nunca `new List<>()`)

---

## 2. Enums (`03-Domain/Enum/`)

```csharp
namespace <NomeDoProjeto>.Domain.Enum
{
	public enum HomeworkTopicStatusEnum
	{
		Draft = 1,
		Published = 2,
		Closed = 3,
		Archived = 4
	}
}
```

---

## 3. DTOs de Request (`02-Application/Dtos/Requests/<Recurso>/`)

```csharp
namespace <NomeDoProjeto>.Application.Dtos.Requests.HomeworkTopic
{
	public record HomeworkTopicCreateRequest(
		string Title,
		string? Description,
		int SchoolId,
		DateTime DueDate,
		int StatusId
	);

	public record HomeworkTopicUpdateRequest(
		string Title,
		string? Description,
		DateTime DueDate,
		int StatusId
	);

	public record HomeworkTopicFilterRequest(
		int SchoolId,
		int? StatusId,
		DateTime? DueDateFrom,
		DateTime? DueDateTo,
		int Page = 1,
		int PageSize = 20
	);
}
```

---

## 4. DTOs de Response (`02-Application/Dtos/Responses/<Recurso>/`)

```csharp
namespace <NomeDoProjeto>.Application.Dtos.Responses.HomeworkTopic
{
	public record HomeworkTopicResponse(
		Guid Id,
		string Title,
		string? Description,
		int SchoolId,
		DateTime DueDate,
		string StatusName
	);
}
```

---

## 5. Validators (`02-Application/Validators/`)

```csharp
using FluentValidation;
using <NomeDoProjeto>.Application.Dtos.Requests.HomeworkTopic;

namespace <NomeDoProjeto>.Application.Validators
{
	public class HomeworkTopicCreateRequestValidator : AbstractValidator<HomeworkTopicCreateRequest>
	{
		public HomeworkTopicCreateRequestValidator()
		{
			RuleFor(x => x.Title)
				.NotEmpty().WithMessage("Título é obrigatório.")
				.MaximumLength(200).WithMessage("Título deve ter no máximo 200 caracteres.");

			RuleFor(x => x.SchoolId)
				.GreaterThan(0).WithMessage("SchoolId deve ser maior que 0.");

			RuleFor(x => x.DueDate)
				.GreaterThan(DateTime.UtcNow).WithMessage("Data de entrega deve ser futura.");

			RuleFor(x => x.StatusId)
				.GreaterThan(0).WithMessage("StatusId deve ser maior que 0.");
		}
	}
}
```

---

## 6. Interface de Service (`02-Application/Services/`)

```csharp
namespace <NomeDoProjeto>.Application.Services
{
	public interface IHomeworkTopicService
	{
		Task<HomeworkTopicResponse?> GetByIdAsync(Guid id, CancellationToken cancellationToken);
		Task<PagedResponse<IEnumerable<HomeworkTopicResponse>>> GetAllAsync(HomeworkTopicFilterRequest filter, CancellationToken cancellationToken);
		Task<HomeworkTopicResponse> CreateAsync(HomeworkTopicCreateRequest request, CancellationToken cancellationToken);
		Task<HomeworkTopicResponse> UpdateAsync(Guid id, HomeworkTopicUpdateRequest request, CancellationToken cancellationToken);
		Task DeleteAsync(Guid id, CancellationToken cancellationToken);
	}
}
```

---

## 7. Implementação de Service (`02-Application/Services/`)

```csharp
using Microsoft.Extensions.Logging;
using <NomeDoProjeto>.Data.Interfaces;
using <NomeDoProjeto>.Shared.Exceptions;

namespace <NomeDoProjeto>.Application.Services
{
	public class HomeworkTopicService(
		IHomeworkTopicRepository repository,
		ILogger<HomeworkTopicService> logger) : IHomeworkTopicService
	{
		public async Task<HomeworkTopicResponse?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
		{
			var entity = await repository.GetByIdAsync(id, true, cancellationToken)
				?? throw new NotFoundException($"Tema de casa {id} não encontrado.");
			return MapToResponse(entity);
		}

		public async Task<HomeworkTopicResponse> CreateAsync(HomeworkTopicCreateRequest request, CancellationToken cancellationToken)
		{
			logger.LogInformation("Criando tema de casa: {Title}", request.Title);

			var entity = new HomeworkTopic
			{
				Title = request.Title,
				Description = request.Description,
				SchoolId = request.SchoolId,
				DueDate = request.DueDate,
				StatusId = request.StatusId
			};

			await repository.AddAsync(entity, cancellationToken);
			return MapToResponse(entity);
		}

		private static HomeworkTopicResponse MapToResponse(HomeworkTopic e) =>
			new(e.Id, e.Title, e.Description, e.SchoolId, e.DueDate, e.Status?.Name ?? string.Empty);
	}
}
```

---

## 8. Interface de Repositório (`04-Infra/4.2-Data/Interfaces/`)

```csharp
using <NomeDoProjeto>.Data.Interfaces;
using <NomeDoProjeto>.Domain.Entities;

namespace <NomeDoProjeto>.Data.Interfaces
{
	public interface IHomeworkTopicRepository : IRepository<HomeworkTopic, Guid>
	{
		Task<IEnumerable<HomeworkTopic>> GetBySchoolAsync(int schoolId, CancellationToken cancellationToken);
	}
}
```

---

## 9. Implementação de Repositório (`04-Infra/4.2-Data/Repositories/`)

```csharp
using Microsoft.EntityFrameworkCore;
using <NomeDoProjeto>.Data;
using <NomeDoProjeto>.Data.Interfaces;
using <NomeDoProjeto>.Data.Repositories;
using <NomeDoProjeto>.Domain.Entities;

namespace <NomeDoProjeto>.Dtos.Repositories
{
	public class HomeworkTopicRepository(ProjectContext projectContext)
		: BaseRepository<HomeworkTopic, Guid>(projectContext), IHomeworkTopicRepository
	{
		protected override DbSet<HomeworkTopic> DbSet => projectContext.HomeworkTopics;

		public override IQueryable<HomeworkTopic> Query()
			=> DbSet.AsNoTracking().Include(h => h.Status);

		public async Task<IEnumerable<HomeworkTopic>> GetBySchoolAsync(int schoolId, CancellationToken cancellationToken)
			=> await DbSet.AsNoTracking()
				.Where(h => h.SchoolId == schoolId)
				.ToListAsync(cancellationToken);
	}
}
```

---

## 10. Controller (`01-Presentation/Controllers/`)

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using <NomeDoProjeto>.Application.Dtos.Requests.HomeworkTopic;
using <NomeDoProjeto>.Application.Services;
using Swashbuckle.AspNetCore.Annotations;
using System.ComponentModel.DataAnnotations;

namespace <NomeDoProjeto>.Controllers
{
	[Authorize]
	[ApiController]
	[Route("v1/[controller]")]
	[Tags("05. Homework Topics")]
	public class HomeworkTopicController(
		ILogger<HomeworkTopicController> logger,
		IHomeworkTopicService service) : ControllerBase
	{
		[HttpGet("{id}", Name = nameof(GetHomeworkTopicByIdAsync))]
		[Authorize(Roles = "Aluno, Escola, Coordenador, Professor")]
		[SwaggerOperation(Summary = "Retorna um tema de casa pelo Id", OperationId = "GetHomeworkTopicByIdAsync")]
		[SwaggerResponse(200, "Tema encontrado")]
		[SwaggerResponse(404, "Tema não encontrado")]
		public async Task<IResult> GetHomeworkTopicByIdAsync(
			[Required][FromHeader(Name = "X-User-Profile")] string profile,
			[Required][FromHeader(Name = "X-SchoolId")] int schoolId,
			Guid id,
			CancellationToken cancellationToken)
		{
			var response = await service.GetByIdAsync(id, cancellationToken);
			return Results.Ok(response);
		}

		[HttpPost(Name = nameof(CreateHomeworkTopicAsync))]
		[Authorize(Roles = "Escola, Coordenador, Professor")]
		[SwaggerOperation(Summary = "Cria um novo tema de casa", OperationId = "CreateHomeworkTopicAsync")]
		[SwaggerResponse(201, "Tema criado com sucesso")]
		[SwaggerResponse(400, "Dados inválidos")]
		public async Task<IResult> CreateHomeworkTopicAsync(
			[Required][FromHeader(Name = "X-User-Profile")] string profile,
			[Required][FromHeader(Name = "X-SchoolId")] int schoolId,
			[FromBody] HomeworkTopicCreateRequest request,
			CancellationToken cancellationToken)
		{
			var response = await service.CreateAsync(request, cancellationToken);
			return Results.Created($"/v1/homeworktopic/{response.Id}", response);
		}

		[HttpDelete("{id}", Name = nameof(DeleteHomeworkTopicAsync))]
		[Authorize(Roles = "Escola, Coordenador, Professor")]
		[SwaggerOperation(Summary = "Remove (soft delete) um tema de casa", OperationId = "DeleteHomeworkTopicAsync")]
		[SwaggerResponse(204, "Removido com sucesso")]
		[SwaggerResponse(404, "Tema não encontrado")]
		public async Task<IResult> DeleteHomeworkTopicAsync(
			[Required][FromHeader(Name = "X-User-Profile")] string profile,
			[Required][FromHeader(Name = "X-SchoolId")] int schoolId,
			Guid id,
			CancellationToken cancellationToken)
		{
			await service.DeleteAsync(id, cancellationToken);
			return Results.NoContent();
		}
	}
}
```

---

## 11. Testes (`05-Tests/`)

```csharp
using Moq;
using Xunit;
using <NomeDoProjeto>.Application.Services;
using <NomeDoProjeto>.Data.Interfaces;
using <NomeDoProjeto>.Domain.Entities;
using Microsoft.Extensions.Logging;

namespace <NomeDoProjeto>.Tests.Services
{
	public class HomeworkTopicServiceTests
	{
		private readonly Mock<IHomeworkTopicRepository> _repoMock = new();
		private readonly Mock<ILogger<HomeworkTopicService>> _loggerMock = new();
		private HomeworkTopicService CreateService() =>
			new(_repoMock.Object, _loggerMock.Object);

		[Fact]
		public async Task GetByIdAsync_ExistingId_ReturnsResponse()
		{
			var id = Guid.NewGuid();
			var entity = new HomeworkTopic { Id = id, Title = "Matemática", SchoolId = 1, StatusId = 1 };
			_repoMock.Setup(r => r.GetByIdAsync(id, true, It.IsAny<CancellationToken>()))
					 .ReturnsAsync(entity);

			var result = await CreateService().GetByIdAsync(id, CancellationToken.None);

			Assert.NotNull(result);
			Assert.Equal("Matemática", result!.Title);
		}

		[Fact]
		public async Task GetByIdAsync_NotFound_ThrowsNotFoundException()
		{
			_repoMock.Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), true, It.IsAny<CancellationToken>()))
					 .ReturnsAsync((HomeworkTopic?)null);

			await Assert.ThrowsAsync<NotFoundException>(
				() => CreateService().GetByIdAsync(Guid.NewGuid(), CancellationToken.None));
		}
	}
}
```
