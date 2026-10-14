# AGENTS.md

Guidelines for implementing features in **BookRatings**, a .NET 10 Blazor Web App (interactive server rendering, single project, nullable enabled).

The architecture has five parts that work together:

1. **Vertical slices with feature folders.** Code is grouped by entity and use case.
2. **CQRS.** Every use case is either a command (write) or a query (read), each with its own model and data path.
3. **Custom mediator.** All commands and queries are sent through our own `IMediator`, which runs cross-cutting pipeline behaviors (validation, logging) before the handler.
4. **Dependency injection.** Everything depends on interfaces and is wired up in the DI container.
5. **Minimal APIs.** Each use case exposes an HTTP endpoint, defined in its own slice.

Handlers hold the use case logic. Minimal API endpoints and Blazor pages are thin adapters in front of them: both live in the slice, both send requests through `IMediator`, and neither contains business logic. Blazor pages use the mediator in-process, not over HTTP.

```
Endpoint / Blazor page
   → IMediator.SendAsync / QueryAsync
      → LoggingBehavior → ValidationBehavior → (more behaviors)
         → ICommandHandler / IQueryHandler
            → repositories, entities
```

## 1. Vertical Slices with Feature Folders

Code is organized **by feature, not by technical layer**. There are no top-level `Services/`, `Models/`, `Repositories/` or `Dtos/` folders. Everything a use case needs lives together in one folder.

- One folder per **entity** under `Features/` (e.g. `Features/Books/`, `Features/Ratings/`).
- Inside it, one folder per **use case** (e.g. `CreateBook/`, `GetBookById/`, `ListBooks/`).
- Slices **do not reference each other**. If two or more slices in the same entity need the same thing, move it to that entity's `Shared/` folder. If several entities need it, move it to the top-level `Shared/` folder. Don't share code before a second consumer exists.

### Folder layout

```
Features/
  Books/
    Shared/                          # Used by several Books slices
      Book.cs                        # Entity (write model)
      IBookRepository.cs             # Write-side persistence abstraction
      BookRepository.cs              # Write-side persistence implementation
      IBookReadRepository.cs         # Read-side persistence abstraction
      BookReadRepository.cs          # Read-side persistence implementation
      BookErrors.cs                  # Reusable Error definitions for this entity
    CreateBook/                      # Command slice
      CreateBookCommand.cs
      CreateBookCommandHandler.cs
      CreateBookCommandValidator.cs
      CreateBookResponse.cs
      CreateBookEndpoint.cs          # Minimal API endpoint
      CreateBookFormModel.cs         # Mutable model for EditForm binding (UI slices only)
      CreateBookPage.razor
    UpdateBook/                      # Command slice with route + body input
      UpdateBookCommand.cs
      UpdateBookCommandHandler.cs
      UpdateBookCommandValidator.cs
      UpdateBookRequest.cs           # HTTP body; combined with the route id into the command
      UpdateBookEndpoint.cs
    GetBookById/                     # Query slice
      GetBookByIdQuery.cs
      GetBookByIdQueryHandler.cs
      GetBookByIdResponse.cs
      GetBookByIdEndpoint.cs
      BookDetailsPage.razor
    ListBooks/                       # Query slice
      ListBooksQuery.cs
      ListBooksQueryHandler.cs
      ListBooksResponse.cs
      BookListItemDto.cs
      ListBooksEndpoint.cs
      BooksPage.razor
    BooksFeatureRegistration.cs      # DI registration for every Books slice
    BooksEndpoints.cs                # Maps the /api/books route group and every Books endpoint
  Ratings/
    ...
Shared/                              # Cross-cutting building blocks only
  Messaging/
    ICommand.cs
    ICommandHandler.cs
    IQuery.cs
    IQueryHandler.cs
    IMediator.cs
    Mediator.cs
    IPipelineBehavior.cs
    RequestHandlerDelegate.cs
    Internal/                        # Generic wrappers the mediator uses to call typed handlers
      CommandHandlerWrapper.cs
      CommandHandlerWrapperImpl.cs
      QueryHandlerWrapper.cs
      QueryHandlerWrapperImpl.cs
    Behaviors/
      LoggingBehavior.cs
      ValidationBehavior.cs
  Results/
    Result.cs
    ResultOfT.cs
    Error.cs
    ErrorType.cs
  Validation/
    IValidator.cs
    ValidationResult.cs
  Endpoints/
    IEndpoint.cs
    ResultHttpExtensions.cs          # Maps Result/Error to HTTP results (ProblemDetails)
  SharedRegistration.cs              # DI registration for the building blocks
Components/                          # App shell only: App, Routes, Layout, Error/NotFound pages
```

Not every slice needs every file. Add a validator only if there is input to validate, a form model only if there is a form, a page only if there is UI.

### One type per file

- Every class, record, interface, enum and struct goes in **its own file**, named exactly after the type (`CreateBookCommand.cs` contains `CreateBookCommand`).
- Namespaces mirror folders: `BookRatings.Features.Books.CreateBook`. Use file-scoped namespaces.
- No nested public types; extract them into their own file instead.

## 2. CQRS

Every use case is **exactly one** of these:

| | Command | Query |
| --- | --- | --- |
| Purpose | Changes state | Reads state |
| Side effects | Yes | **None**, ever |
| Model used | Entities (write model) via `IXxxRepository` | DTOs/responses (read model) via `IXxxReadRepository` |
| Returns | The minimum the caller needs (usually the id) | Data shaped for the screen that asked for it |
| Validation | Validator (run by the pipeline) + business rules in entity/handler | Parameter checks only (paging, ids) |
| Sent with | `mediator.SendAsync(command)` | `mediator.QueryAsync(query)` |

### Rules

- **No combined operations.** A "create and return the full details" use case is a command followed by a query.
- **Commands go through the domain.** Load the entity, call a behavior method on it (`book.Rename(...)`), save it. Business rules live in the entity, not in the handler and never in the page.
- **Queries skip the domain.** They read straight into response DTOs through the read repository, without loading or tracking entities. Each query has its own response type, shaped for its consumer, even if two look alike now.
- **Read and write repositories are separate interfaces.** `IBookRepository` (write) works with `Book` entities and has methods like `GetByIdAsync`, `AddAsync`, `UpdateAsync`. `IBookReadRepository` (read) returns DTOs and is only used by query handlers. Command handlers must not use read repositories, and query handlers must not use write repositories.
- Both sides share one data store for now. The split is in the code, not in the infrastructure, so we can introduce a separate read store later without touching handlers.

### Request and handler contracts

Defined once in `Shared/Messaging/`:

```csharp
// Shared/Messaging/ICommand.cs
public interface ICommand<TResponse>;

// Shared/Messaging/ICommandHandler.cs
public interface ICommandHandler<in TCommand, TResponse>
    where TCommand : ICommand<TResponse>
{
    Task<Result<TResponse>> HandleAsync(TCommand command, CancellationToken cancellationToken);
}

// Shared/Messaging/IQuery.cs
public interface IQuery<TResponse>;

// Shared/Messaging/IQueryHandler.cs
public interface IQueryHandler<in TQuery, TResponse>
    where TQuery : IQuery<TResponse>
{
    Task<Result<TResponse>> HandleAsync(TQuery query, CancellationToken cancellationToken);
}
```

Every request has **exactly one** handler. Commands that have nothing meaningful to return use a `Unit`-like type or return the created/updated id; don't invent a second handler interface.

### Results and errors

`Result` / `Result<T>` (in `Shared/Results/`) carry either a value or an `Error` (with an `ErrorType` such as `Validation`, `NotFound`, `Conflict`). Use them for **expected** failures (not found, invalid input, business rule violated). Throw exceptions only for **unexpected** failures (bugs, infrastructure down).

## 3. Custom Mediator

We use our own small mediator instead of MediatR (which now requires a commercial license) or other mediator packages. Don't add MediatR, FluentValidation, AutoMapper or similar packages without asking first.

### Contracts

```csharp
// Shared/Messaging/IMediator.cs
public interface IMediator
{
    Task<Result<TResponse>> SendAsync<TResponse>(ICommand<TResponse> command, CancellationToken cancellationToken = default);

    Task<Result<TResponse>> QueryAsync<TResponse>(IQuery<TResponse> query, CancellationToken cancellationToken = default);
}

// Shared/Messaging/RequestHandlerDelegate.cs
public delegate Task<Result<TResponse>> RequestHandlerDelegate<TResponse>();

// Shared/Messaging/IPipelineBehavior.cs
public interface IPipelineBehavior<in TRequest, TResponse>
    where TRequest : notnull
{
    Task<Result<TResponse>> HandleAsync(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken);
}
```

`SendAsync` only accepts commands and `QueryAsync` only accepts queries, so the CQRS split is visible at every call site.

### How `Mediator` works

- `Mediator` (sealed, scoped) takes `IServiceProvider` in its constructor. It's the **only** class allowed to resolve services from `IServiceProvider` directly.
- Calling a generic handler when the request type is only known at runtime needs a little reflection. Keep it in one place: for each concrete request type, create a `CommandHandlerWrapperImpl<TCommand, TResponse>` (or the query equivalent) once with `MakeGenericType`, cache it in a `static ConcurrentDictionary<Type, ...>`, and from then on call it without reflection.
- The wrapper resolves the `ICommandHandler<TCommand, TResponse>` and every `IPipelineBehavior<TCommand, TResponse>` from the service provider, chains the behaviors around the handler, and runs the chain.
- Behaviors run in **registration order**: the first one registered is the outermost.
- If no handler is registered for a request, throw an `InvalidOperationException` that names the request type. That's a wiring bug, not an expected failure.

### Pipeline behaviors

Cross-cutting concerns live in behaviors, not in handlers. Behaviors are open generics in `Shared/Messaging/Behaviors/` and apply to every command and query.

```csharp
namespace BookRatings.Shared.Messaging.Behaviors;

public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<Result<TResponse>> HandleAsync(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        var failures = validators
            .Select(validator => validator.Validate(request))
            .Where(result => !result.IsValid)
            .ToList();

        if (failures.Count > 0)
        {
            return ValidationResult.Combine(failures).ToError();
        }

        return await next();
    }
}
```

Current behaviors, in order:
1. `LoggingBehavior`: logs the request name, duration and whether it failed (never logs full request contents, which may hold personal data).
2. `ValidationBehavior`: runs every `IValidator<TRequest>` and short-circuits with a validation `Error` before the handler runs.

Add a behavior only for concerns that apply to many requests (e.g. a transaction behavior for commands once a database exists). One-off logic belongs in the handler. To limit a behavior to commands only, constrain it with `where TRequest : ICommand<TResponse>`.

### Rules

- Endpoints and pages **always** go through `IMediator`. Never inject a handler directly, or the pipeline (validation, logging) is skipped.
- Handlers never call `IMediator`. A handler that sends another request hides control flow; extract the shared logic into the entity or `Shared/` instead.
- Handlers don't validate input themselves; `ValidationBehavior` does that before they run.
- No notifications/domain events yet. Add an `INotification`/`INotificationHandler<T>` pair to the mediator only when a real use case needs it, and ask first.

## 4. Dependency Injection

### Rules

- **Constructor injection only**, using primary constructors. No `new` for services, no static service classes, no service locator (`IServiceProvider` is only used inside `Mediator` and its wrappers).
- **Depend on interfaces, not implementations.** Every handler, validator, repository and service is registered and consumed by its interface (`IBookRepository`, `IValidator<T>`, `ICommandHandler<,>`). Framework abstractions count too: `ILogger<T>`, `TimeProvider`, `IOptions<T>`.
- **Don't use `DateTime.Now`/`UtcNow` directly.** Inject `TimeProvider` so time can be faked in tests.
- **Small interfaces.** Split an interface when consumers use different parts of it (that's why read and write repositories are separate).
- Keep constructors free of logic: only store dependencies. A constructor with more than ~4 dependencies is a sign the class does too much.

### Lifetimes

| Type | Lifetime | Why |
| --- | --- | --- |
| Command/query handlers | Scoped | Depend on scoped data access |
| `IMediator` | Scoped | Resolves scoped handlers and behaviors |
| Pipeline behaviors | Scoped (open generic) | Same scope as the handler they wrap |
| Validators | Scoped (or singleton if stateless and dependency-free) | |
| Repositories / DbContext | Scoped | One unit of work per scope |
| `TimeProvider` | Singleton (`TimeProvider.System`) | |
| In-memory placeholder stores | Singleton | Must survive across scopes |

Never inject a scoped service into a singleton. In Blazor Server a scope lives as long as the user's circuit, so if a component needs a fresh unit of work per operation, inherit from `OwningComponentBase` or create a scope with `IServiceScopeFactory`.

### Registration

Each entity has one registration file that registers **every** slice of that entity. Register new slices there as you add them.

```csharp
namespace BookRatings.Features.Books;

public static class BooksFeatureRegistration
{
    public static IServiceCollection AddBooksFeature(this IServiceCollection services)
    {
        // Persistence
        services.AddScoped<IBookRepository, BookRepository>();
        services.AddScoped<IBookReadRepository, BookReadRepository>();

        // Commands
        services.AddScoped<ICommandHandler<CreateBookCommand, CreateBookResponse>, CreateBookCommandHandler>();
        services.AddScoped<IValidator<CreateBookCommand>, CreateBookCommandValidator>();

        // Queries
        services.AddScoped<IQueryHandler<GetBookByIdQuery, GetBookByIdResponse>, GetBookByIdQueryHandler>();
        services.AddScoped<IQueryHandler<ListBooksQuery, ListBooksResponse>, ListBooksQueryHandler>();

        return services;
    }
}
```

`Shared/SharedRegistration.cs` registers the mediator, the pipeline behaviors (in order), `TimeProvider` and `ProblemDetails`:

```csharp
services.AddScoped<IMediator, Mediator>();
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddSingleton(TimeProvider.System);
services.AddProblemDetails();
```

`Program.cs` only calls the extension methods:

```csharp
builder.Services
    .AddShared()
    .AddBooksFeature()
    .AddRatingsFeature();

// ...

var api = app.MapGroup("/api");
api.MapBooksEndpoints();
api.MapRatingsEndpoints();
```

## 5. Minimal APIs

Every use case that should be reachable over HTTP gets an endpoint class **in its own slice**. No controllers, and no endpoint definitions directly in `Program.cs`.

### Endpoint building blocks

```csharp
// Shared/Endpoints/IEndpoint.cs
public interface IEndpoint
{
    static abstract void Map(IEndpointRouteBuilder routes);
}
```

`Shared/Endpoints/ResultHttpExtensions.cs` converts a failed `Result` into a `ProblemDetails` response, so status codes are decided in one place:

| `ErrorType` | HTTP status |
| --- | --- |
| `Validation` | 400 (`ValidationProblem` with per-field errors) |
| `NotFound` | 404 |
| `Conflict` | 409 |
| anything else | 500 |

### Endpoint

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed class CreateBookEndpoint : IEndpoint
{
    public static void Map(IEndpointRouteBuilder routes) =>
        routes.MapPost("/", HandleAsync)
            .WithName("CreateBook")
            .WithSummary("Creates a new book");

    private static async Task<Results<Created<CreateBookResponse>, ProblemHttpResult>> HandleAsync(
        CreateBookCommand command,
        IMediator mediator,
        CancellationToken cancellationToken)
    {
        var result = await mediator.SendAsync(command, cancellationToken);

        return result.IsSuccess
            ? TypedResults.Created($"/api/books/{result.Value.Id}", result.Value)
            : result.ToProblem();
    }
}
```

Endpoint rules:
- `IMediator` is injected as a **parameter of the endpoint delegate**. Minimal APIs resolve it from DI automatically. Never inject a handler directly; that would skip the pipeline.
- Endpoints only bind input, send it through the mediator and translate the `Result` to HTTP. No business logic, no data access.
- Return `TypedResults` with an explicit `Results<...>` return type, so the OpenAPI metadata is accurate and the compiler checks every outcome.
- Always accept a `CancellationToken` and pass it on.
- Bind the command or query record directly when the HTTP input has the same shape. When input comes from several places (e.g. id from the route, rest from the body), add a `{UseCase}Request` record for the body and build the command in the endpoint.
- Never return entities. Responses are the slice's response records.
- HTTP verbs follow CQRS: queries use `GET`; commands use `POST`, `PUT`, `PATCH` or `DELETE`. A `GET` never runs a command.
- Status codes: `201 Created` with a location for creates, `204 No Content` for updates and deletes without a body, `200 OK` for queries.

### Route group per entity

```csharp
namespace BookRatings.Features.Books;

public static class BooksEndpoints
{
    public static IEndpointRouteBuilder MapBooksEndpoints(this IEndpointRouteBuilder routes)
    {
        var books = routes.MapGroup("/books").WithTags("Books");

        CreateBookEndpoint.Map(books);
        UpdateBookEndpoint.Map(books);
        GetBookByIdEndpoint.Map(books);
        ListBooksEndpoint.Map(books);

        return routes;
    }
}
```

Group-wide settings (tags, authorization, rate limiting) go on the group, not on each endpoint. Add each new endpoint to its entity's `{Entity}Endpoints.cs`.

Routes are plural, lowercase nouns: `/api/books`, `/api/books/{id:guid}`, `/api/books/{bookId:guid}/ratings`. Use route constraints (`:guid`, `:int`).

Don't add packages such as `Microsoft.AspNetCore.OpenApi` or Scalar without asking first. Once OpenAPI is added, every endpoint must have a `WithName` and `WithSummary`.

## Implementing a Use Case

### Command or query: immutable record

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed record CreateBookCommand(string Title, string Author, int PublishedYear)
    : ICommand<CreateBookResponse>;
```

### Response: immutable record, never an entity

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed record CreateBookResponse(Guid Id);
```

Entities never leave a command handler. Map to a response explicitly in the handler (or a small static `ToResponse()` method in the slice); don't use reflection-based mappers.

### Validator

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed class CreateBookCommandValidator : IValidator<CreateBookCommand>
{
    public ValidationResult Validate(CreateBookCommand command)
    {
        // Check required fields, lengths, ranges. No database access here.
    }
}
```

Validators check input shape only. Rules that need data (e.g. "title must be unique") belong in the handler. Registering a validator is enough: `ValidationBehavior` picks it up and runs it before the handler.

### Command handler

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed class CreateBookCommandHandler(
    IBookRepository bookRepository,
    TimeProvider timeProvider,
    ILogger<CreateBookCommandHandler> logger)
    : ICommandHandler<CreateBookCommand, CreateBookResponse>
{
    public async Task<Result<CreateBookResponse>> HandleAsync(
        CreateBookCommand command,
        CancellationToken cancellationToken)
    {
        // Input was already validated by ValidationBehavior.
        var book = Book.Create(command.Title, command.Author, command.PublishedYear, timeProvider.GetUtcNow());
        await bookRepository.AddAsync(book, cancellationToken);

        logger.LogInformation("Created book {BookId}", book.Id);
        return new CreateBookResponse(book.Id);
    }
}
```

### Query handler

```csharp
namespace BookRatings.Features.Books.GetBookById;

public sealed class GetBookByIdQueryHandler(IBookReadRepository bookReadRepository)
    : IQueryHandler<GetBookByIdQuery, GetBookByIdResponse>
{
    public async Task<Result<GetBookByIdResponse>> HandleAsync(
        GetBookByIdQuery query,
        CancellationToken cancellationToken)
    {
        var book = await bookReadRepository.GetDetailsAsync(query.Id, cancellationToken);
        return book is null ? BookErrors.NotFound(query.Id) : book;
    }
}
```

Handler rules:
- `sealed`, uses a primary constructor, depends on interfaces only.
- All I/O is `async` and accepts and passes on a `CancellationToken`.
- A handler never calls another handler or `IMediator`. If logic needs to be shared, extract it into the entity or into `Shared/`.

### Entity: protects its own invariants

```csharp
namespace BookRatings.Features.Books.Shared;

public sealed class Book
{
    private Book() { } // For the persistence framework

    public Guid Id { get; private set; }
    public string Title { get; private set; } = string.Empty;
    // ...

    public static Book Create(string title, string author, int publishedYear, DateTimeOffset createdAt)
    {
        // Enforce invariants, then return a valid instance.
    }
}
```

Use private setters and factory/behavior methods (`Create`, `Rename`, `AddRating`) instead of public setters, so an entity can't be put into an invalid state.

### UI (Blazor page in the slice)

- The page lives in the slice folder and has its `@page` route there. Routing finds it automatically.
- Pages inject **`IMediator`**, never handlers or repositories:
  ```razor
  @inject IMediator Mediator

  @code {
      private async Task SubmitAsync()
      {
          var result = await Mediator.SendAsync(form.ToCommand());
          // Show errors or navigate based on the result.
      }
  }
  ```
- `EditForm` needs a mutable model: bind to `CreateBookFormModel`, then map it to the immutable command on submit.
- Pages contain only presentation logic: binding, sending through the mediator, showing the `Result`. No business rules and no direct data access in `.razor` files.
- Add new feature namespaces to `Components/_Imports.razor` (or a `_Imports.razor` in the feature folder) instead of repeating `@using` in each page.
- Add a link in `Components/Layout/NavMenu.razor` when the page is meant to be reached from the navigation menu.

## Naming Conventions

| Kind | Pattern | Example |
| --- | --- | --- |
| Use case folder | `VerbNoun` | `CreateBook`, `RateBook`, `ListBooks` |
| Command | `{UseCase}Command` | `CreateBookCommand` |
| Query | `{UseCase}Query` | `GetBookByIdQuery` |
| Handler | `{Command or Query}Handler` | `CreateBookCommandHandler` |
| Validator | `{Command or Query}Validator` | `CreateBookCommandValidator` |
| Response | `{UseCase}Response` | `CreateBookResponse` |
| Nested DTO | `{Thing}Dto` | `BookListItemDto` |
| Form model | `{UseCase}FormModel` | `CreateBookFormModel` |
| HTTP request body | `{UseCase}Request` | `UpdateBookRequest` |
| Endpoint | `{UseCase}Endpoint` | `CreateBookEndpoint` |
| Endpoint group | `{Entity}Endpoints` | `BooksEndpoints` |
| Pipeline behavior | `{Concern}Behavior` | `ValidationBehavior` |
| Write repository | `I{Entity}Repository` | `IBookRepository` |
| Read repository | `I{Entity}ReadRepository` | `IBookReadRepository` |
| DI registration | `{Entity}FeatureRegistration` | `BooksFeatureRegistration` |
| Page | `{Descriptive}Page.razor` | `CreateBookPage.razor` |

Commands are named as imperatives (`CreateBook`, `RateBook`), queries as questions (`GetBookById`, `ListBooks`).

## General Best Practices

- **SOLID**, especially single responsibility (one use case = one handler) and dependency inversion (depend on interfaces, register them in DI).
- Prefer `record` for commands, queries, responses and DTOs; `sealed class` for handlers, validators and implementations.
- Keep nullable reference types enabled and warning-free. Don't silence warnings with `!` unless there's a reason, and explain it in a comment.
- Use guard clauses and early returns instead of deep nesting.
- Use `ILogger<T>` with structured logging (`logger.LogInformation("Created book {BookId}", book.Id)`), not string interpolation.
- Keep configuration out of code: bind settings with the Options pattern (`IOptions<T>`), with one options class per file inside the feature that owns it.
- Don't add unused abstractions, base classes or "generic repositories". Add structure when a real need appears.
- Comments explain *why*, not *what*.

## Persistence

No persistence technology has been chosen yet. Until then:
- Put each entity's persistence behind its write and read repository interfaces in its `Shared/` folder.
- An in-memory store registered as a singleton, wrapped by scoped repository implementations, is fine as a placeholder.
- When a database is introduced (likely EF Core), keep entity configuration inside the feature (`Features/Books/Shared/BookConfiguration.cs`). Read repositories use `AsNoTracking()` and project straight into DTOs with `Select`. Handlers keep depending on the interfaces and don't change.

## Testing

When a test project is added, its folder structure mirrors the feature folders (`BookRatings.Tests/Features/Books/CreateBook/CreateBookCommandHandlerTests.cs`). Test handlers, validators and pipeline behaviors directly, replacing their dependencies with fakes or mocks of the interfaces they depend on. Use `FakeTimeProvider` for time. Also add a test that builds the real service collection and checks that every `ICommand<>`/`IQuery<>` type in the assembly has a registered handler, so missing registrations fail in tests instead of at runtime.

## Checklist for a New Use Case

1. Decide: command (changes state) or query (reads state)? Never both.
2. Create `Features/{Entity}/{UseCase}/`.
3. Add the command or query record, the response record and any DTOs, each in its own file.
4. Add a validator if there is input (the pipeline runs it; the handler doesn't).
5. Implement the handler against interfaces only, returning `Result<T>`. Commands use the write repository, queries use the read repository.
6. Register the handler (and validator) in `{Entity}FeatureRegistration.cs`.
7. Add the `{UseCase}Endpoint` (plus a `{UseCase}Request` if needed), sending through `IMediator`, and map it in `{Entity}Endpoints.cs`.
8. Add the page and form model if the use case has UI. The page uses `IMediator`. Update `NavMenu.razor` if needed.
9. Build with `dotnet build` and make sure there are no warnings or errors.
