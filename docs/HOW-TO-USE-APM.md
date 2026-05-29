# Como Usar o APM (Agentic Package Manager) no Visual Studio

Este guia explica como usar o **GitHub Copilot Agent** no Visual Studio 2026 para criar e expandir a **apm-api** de forma assistida, respeitando as convenções de arquitetura definidas neste repositório.

---

## O que é o APM no contexto do Copilot?

O **APM (Agentic Package Manager)** no Visual Studio é o conjunto de recursos do **GitHub Copilot Agent** que permite ao agente atuar de forma autônoma para:

- Gerar código seguindo convenções do repositório
- Executar comandos (migrations, builds, testes)
- Navegar e modificar múltiplos arquivos em uma única instrução
- Aprender o contexto do projeto via arquivos de instruções e skills

### Arquivos que configuram o agente neste repositório

| Arquivo | Função |
|---|---|
| `.github/copilot-instructions.md` | Instruções globais aplicadas a **todas** as interações |
| `.github/instructions/csharp-api.instructions.md` | Instruções aplicadas a arquivos `src/**/*.cs` |
| `SKILL.md` | Define skills (habilidades) que o agente pode executar |
| `docs/architecture.md` | Referência de arquitetura consultada pelo agente |

---

## Configuração Inicial no Visual Studio

### 1. Habilitar recursos do agente

Vá em **Ferramentas > Opções > GitHub Copilot** e ative:

- ✅ **Habilitar instruções personalizadas** de `.github/copilot-instructions.md`
- ✅ **Habilitar carregamento de instruções** de `.github/instructions/*.instructions.md`
- ✅ **Habilitar Habilidades do Agente** dos arquivos `SKILL.md`

### 2. Abrir o Copilot Chat

Use o atalho **Ctrl+], C** ou acesse **Exibir > GitHub Copilot Chat**.

### 3. Selecionar o modo agente

No chat, certifique-se de estar no modo **@workspace** (modo agente), que permite ao Copilot ler e modificar arquivos do repositório.

---

## Fluxo de Trabalho Recomendado

### Criando um novo recurso completo

O fluxo mais eficiente é usar o skill `create-api-resource`:

```
@workspace Usar skill create-api-resource para o recurso HomeworkTopic com os seguintes campos:
- Title: string, obrigatório, máx 200 caracteres
- Description: string, opcional
- SchoolId: int, obrigatório
- DueDate: DateTime, deve ser data futura
- StatusId: int → enum HomeworkTopicStatusEnum (Draft=1, Published=2, Closed=3)
- Relacionamento: um HomeworkTopic tem muitos StudentHomework
```

O agente irá gerar **todos os artefatos** automaticamente seguindo a arquitetura definida.

---

### Adicionando apenas um componente específico

**Nova entidade:**
```
@workspace Usar skill create-entity para a entidade StudentHomework com campos:
StudentId (int), HomeworkTopicId (Guid, FK), SubmittedAt (DateTime?), Grade (decimal?)
```

**Novo repositório para entidade existente:**
```
@workspace Usar skill create-repository para HomeworkTopic
com método adicional: GetOverdueBySchoolAsync(int schoolId, DateTime referenceDate)
```

**Novo controller para service existente:**
```
@workspace Usar skill create-controller para HomeworkTopicController
usando IHomeworkTopicService, tag "05. Homework Topics", rota v1/homeworktopic
```

**Testes para um service:**
```
@workspace Usar skill generate-tests para HomeworkTopicService
cobrindo: GetByIdAsync, CreateAsync, UpdateAsync, DeleteAsync
```

---

### Executando um plano de implementação

Para executar um plano completo definido em um arquivo `.md`:

```
@workspace Usar skill implement-plan com base em docs/implementation-plan-homework-api.md
```

O agente vai ler o plano, executar cada etapa em ordem e reportar progresso.

---

## Comandos Úteis no Chat

### Navegação e exploração
```
@workspace Explique a arquitetura do projeto seguindo docs/architecture.md
@workspace Quais serviços existem em 02-Application/Services?
@workspace Como o repositório ProposalRepository implementa a busca paginada?
```

### Geração de código
```
@workspace Gere um migration para adicionar a tabela HomeworkTopics
@workspace Adicione um endpoint PATCH para atualização parcial no HomeworkTopicController
@workspace Adicione cache Redis no HomeworkTopicService para GetByIdAsync com TTL de 5 minutos
```

### Análise e refatoração
```
@workspace Revise o HomeworkTopicService e sugira melhorias seguindo os padrões do projeto
@workspace Identifique se há violações de Clean Architecture nos arquivos recém-criados
```

### Testes
```
@workspace Execute os testes de HomeworkTopicServiceTests e analise os resultados
@workspace Adicione cenários de teste faltantes em HomeworkTopicServiceTests
```

---

## Referência Rápida de Skills

| Skill | Uso |
|---|---|
| `create-api-resource` | Recurso completo (todas as camadas) |
| `create-entity` | Apenas entidade + enum |
| `create-repository` | Interface + implementação do repositório |
| `create-service` | Interface + implementação do service |
| `create-controller` | Controller com Swagger |
| `generate-tests` | Testes unitários para um service |
| `add-migration` | Migration do EF Core |
| `implement-plan` | Executa plano de um arquivo `.md` |

---

## Dicas para Melhores Resultados

### ✅ Seja específico sobre o contexto
```
// Bom — fornece todos os detalhes necessários
@workspace Criar HomeworkTopicService com injeção de IHomeworkTopicRepository e ICacheRepository,
cache de 10 min para GetByIdAsync, lançar NotFoundException quando não encontrado

// Evitar — muito genérico
@workspace Criar service para homework
```

### ✅ Referencie arquivos existentes como exemplos
```
@workspace Criar StudentHomeworkController seguindo o mesmo padrão de EssayController.cs
```

### ✅ Use os planos de implementação para trabalho estruturado
Para features complexas, sempre inicie com um arquivo de plano em `docs/`:
```
@workspace Com base em docs/implementation-plan-homework-api.md,
execute apenas a Fase 1 (entidades de domínio)
```

### ✅ Valide após geração
Após o agente gerar código, sempre peça uma revisão:
```
@workspace Revise os arquivos gerados e verifique se estão de acordo com
.github/copilot-instructions.md e docs/architecture.md
```

---

## Estrutura de Arquivos do Agente

```
apm-api/
├── .github/
│   ├── copilot-instructions.md          ← lido em TODAS as interações
│   └── instructions/
│       └── csharp-api.instructions.md   ← aplicado a src/**/*.cs
├── SKILL.md                             ← skills disponíveis
├── docs/
│   ├── architecture.md                  ← referência de arquitetura
│   ├── HOW-TO-USE-APM.md               ← este arquivo
│   └── implementation-plan-*.md         ← planos de implementação
└── src/
	├── 01-Presentation/
	├── 02-Application/
	├── 03-Domain/
	├── 04-Infra/
	└── 05-Tests/
```

---

## Troubleshooting

**O agente não está seguindo as convenções do projeto:**
→ Verifique se as opções de instruções personalizadas estão habilitadas em Ferramentas > Opções > GitHub Copilot

**O agente não reconhece os skills:**
→ Confirme que `SKILL.md` está na raiz do repositório e a opção "Habilitar Habilidades do Agente" está ativa

**O agente está gerando código com namespace errado:**
→ Referencie explicitamente o namespace: "use o namespace `Redacao.Application.Services`"

**O agente não está incluindo os headers X-User-Profile e X-SchoolId:**
→ Referencie o arquivo de instruções: "siga `.github/instructions/csharp-api.instructions.md` para o controller"
