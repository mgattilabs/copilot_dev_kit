# Codebase Reading Guide

Annotated reading order for orienting in C#/.NET, TypeScript/Angular, and
Python codebases. Use this when entering a project for the first time or
returning after a long gap.

---

## C# / .NET — Reading Order

Read files in this order to build a mental model before planning:

### 1. Entry point

```
read/readFile: src/{ProjectName}.WebApi/Program.cs
```

What to extract:
- DI registrations (which modules are loaded)
- Middleware pipeline order
- Authentication scheme (JWT, cookie, etc.)
- Feature module extension methods called (e.g. `AddOrders()`, `MapOrders()`)

### 2. Shared abstractions

```
read/readFile: Shared/Abstractions/ICommandHandler.cs
read/readFile: Shared/Abstractions/IQueryHandler.cs
read/readFile: Shared/Abstractions/Result.cs
```

Confirms the CQRS pattern in use and the Result type contract.
If `IRequest<>` or `IMediator` appears here, note as MediatR (legacy/tech debt).

### 3. DbContext

```
search/codebase: ": DbContext"
read/readFile: {AppDbContext path}
```

Lists all entities and their DbSet names. Cross-reference with
`search/fileSearch: "*Configuration.cs"` to find EF configuration files.

### 4. Domain entry points (one per domain folder)

```
read/listDirectory: Features/{Domain}/         // VSA
read/listDirectory: Domain/Entities/           // Onion
```

Pick the domain most relevant to the current task. Read the aggregate root
entity and one representative handler to confirm the established pattern.

### 5. Representative handler (read one before writing any)

```
search/codebase: ": ICommandHandler<{SomethingCommand}"
read/readFile: {path to that handler}
```

Confirms: constructor injection style, Result usage, EF query style, validation
approach. This handler is the reference for all new handlers in the same domain.

### 6. Tests (if any)

```
search/fileSearch: "*HandlerTests.cs"
read/readFile: {path to the most complete test file found}
```

Confirms: test naming convention, mock library (NSubstitute assumed), assertion
library (FluentAssertions assumed), in-memory DB or mock DbContext.

---

## TypeScript / Angular — Reading Order

### 1. App module or main entry

```
read/readFile: src/main.ts
read/readFile: src/app/app.config.ts        // Angular 19+ standalone
read/readFile: src/app/app.module.ts        // Legacy NgModule
```

What to extract: providers, router configuration, lazy-loaded modules.

### 2. Routing

```
search/codebase: "Routes\|RouterModule\|loadComponent\|loadChildren"
```

Maps URL paths to components. Reveals feature structure.

### 3. Core services

```
search/fileSearch: "*.service.ts"
```

Read the services most relevant to the task. Note: are they using Signals,
BehaviorSubject, or plain async? This determines the state management pattern.

### 4. Representative component (read one before writing any)

```
search/fileSearch: "*.component.ts"
read/readFile: {path to most complex relevant component}
```

Confirms: standalone or NgModule, Signals or RxJS, reactive or template forms,
input/output style (new `input()` signal API vs `@Input()` decorator).

### 5. Models and types

```
search/fileSearch: "*.model.ts" OR "*.types.ts" OR "*.interface.ts"
```

Establishes the data contract between frontend and backend DTOs.

---

## Python — Reading Order

### 1. Entry point

```
search/fileSearch: "main.py" OR "app.py" OR "__main__.py"
read/readFile: {entry point}
```

### 2. Configuration and dependencies

```
read/readFile: pyproject.toml OR requirements.txt
```

Lists packages and constraints. Note: DSPy version if present (RAG/LLM work).

### 3. Module structure

```
read/listDirectory: src/          // or project root
```

Identifies: flat script, package structure, or agent/pipeline structure.

### 4. Type hints and dataclass patterns

```
search/codebase: "@dataclass\|BaseModel\|TypedDict"
```

Confirms: plain dataclasses, Pydantic models, or TypedDict.
Note: Pydantic only if already in the project — do not introduce it new.

### 5. Import patterns

```
search/codebase: "^from \|^import " --max-results 20
```

Confirms: absolute imports (preferred), relative imports (flag if excessive),
wildcard imports (flag as violation — `from x import *` is prohibited).

---

## Anti-Patterns to Flag During Reading

Document these in the Analysis output if found:

| Anti-pattern                              | Flag as         | Recommendation                              |
|-------------------------------------------|-----------------|---------------------------------------------|
| MediatR `IRequest` / `IMediator` usage    | ⚠️ Tech Debt    | Migrate to manual `ICommandHandler`         |
| Business logic in endpoints               | ⚠️ Violation    | Move to handler                             |
| `await` inside `for` loop (sequential)    | ⚠️ Performance  | Replace with `Task.WhenAll` if parallelisable|
| `.Result` or `.Wait()` on async method    | ⚠️ Critical     | Deadlock risk — must be fixed               |
| `catch (Exception e)` with empty body     | ⚠️ Violation    | Always log or rethrow                       |
| Magic number in handler                   | ⚠️ Style        | Extract to named constant                   |
| Nested function inside another function   | ⚠️ Violation    | Python: extract to module-level function    |
| `from x import *`                         | ⚠️ Violation    | Python: explicit imports only               |
| `any` type in TypeScript handler/service  | ⚠️ Violation    | Replace with proper type or `unknown`       |
| Component with 200+ lines                 | ⚠️ Style        | Flag for Woz review — split responsibilities|
