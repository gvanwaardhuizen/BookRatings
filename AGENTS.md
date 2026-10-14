# AGENTS.md

Guidelines for implementing features in **BookRatings**, a .NET 10 Blazor Web App (interactive server rendering, single project, nullable enabled).

## Architecture: Vertical Slices with Feature Folders

Code is organized **by feature, not by technical layer**. There are no top-level `Services/`, `Models/`, `Repositories/` or `Dtos/` folders. Everything a use case needs lives together in one folder.

- One folder per **entity** under `Features/` (e.g. `Features/Books/`, `Features/Ratings/`).
- Inside it, one folder per **use case** (e.g. `CreateBook/`, `GetBookById/`, `ListBooks/`).
- Each use case is either a **command** (changes state) or a **query** (reads state, no side effects). Never both.
- Slices **do not reference each other**. If two or more slices in the same entity need the same thing, move it to that entity's `Shared/` folder. If several entities need it, move it to the top-level `Shared/` folder. Don't share code before a second consumer exists.

### Folder layout

```
Features/
  Books/
    Shared/                          # Used by several Books slices
      Book.cs                        # Entity
      IBookRepository.cs             # Persistence abstraction
      BookRepository.cs              # Persistence implementation
      BookErrors.cs                  # Reusable Error definitions for this entity
    CreateBook/
      CreateBookCommand.cs           # Immutable input for the use case
      CreateBookCommandHandler.cs    # Use case logic
      CreateBookCommandValidator.cs  # Input validation
      CreateBookResponse.cs          # Output returned to the caller
      CreateBookFormModel.cs         # Mutable model for EditForm binding (UI slices only)
      CreateBookPage.razor           # UI for this use case (if it has one)
    GetBookById/
      GetBookByIdQuery.cs
      GetBookByIdQueryHandler.cs
      GetBookByIdResponse.cs
      BookDetailsPage.razor
    ListBooks/
      ListBooksQuery.cs
      ListBooksQueryHandler.cs
      ListBooksResponse.cs
      BookListItemDto.cs
      BooksPage.razor
    BooksFeatureRegistration.cs      # DI registration for every Books slice
  Ratings/
    ...
Shared/                              # Cross-cutting building blocks only
  Messaging/
    ICommand.cs
    ICommandHandler.cs
    IQuery.cs
    IQueryHandler.cs
  Results/
    Result.cs
    ResultOfT.cs
    Error.cs
    ErrorType.cs
  Validation/
    IValidator.cs
    ValidationResult.cs
Components/                          # App shell only: App, Routes, Layout, Error/NotFound pages
```

Not every slice needs every file. Add a validator only if there is input to validate, a form model only if there is a form, a page only if there is UI.

## One Type Per File

- Every class, record, interface, enum and struct goes in **its own file**, named exactly after the type (`CreateBookCommand.cs` contains `CreateBookCommand`).
- Namespaces mirror folders: `BookRatings.Features.Books.CreateBook`. Use file-scoped namespaces.
- No nested public types; extract them into their own file instead.

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
| Page | `{Descriptive}Page.razor` | `CreateBookPage.razor` |
| Interface | `I{Name}` | `IBookRepository` |

## Building Blocks

Define these in `Shared/` once and reuse them. Don't add MediatR, FluentValidation, AutoMapper or other third-party packages without asking first (several have licensing changes); these small abstractions are enough.

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

// Shared/Validation/IValidator.cs
public interface IValidator<in T>
{
    ValidationResult Validate(T instance);
}
```

`Result` / `Result<T>` carry either a value or an `Error` (with an `ErrorType` such as `Validation`, `NotFound`, `Conflict`). Use them for **expected** failures (not found, invalid input, business rule violated). Throw exceptions only for **unexpected** failures (bugs, infrastructure down).

Commands that return nothing meaningful use a `Unit`-like type or return the created/updated id; don't invent a second handler interface.

## Implementing a Use Case

### 1. Command or query: immutable record

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed record CreateBookCommand(string Title, string Author, int PublishedYear)
    : ICommand<CreateBookResponse>;
```

### 2. Response: immutable record, never an entity

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed record CreateBookResponse(Guid Id, string Title);
```

Entities never leave the handler. Always map to a response/DTO. Do the mapping in the handler (or a small static `ToResponse()` method in the slice); don't use reflection-based mappers.

### 3. Validator

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

Validators check input shape only. Rules that need data (e.g. "title must be unique") belong in the handler.

### 4. Handler: depends only on interfaces

```csharp
namespace BookRatings.Features.Books.CreateBook;

public sealed class CreateBookCommandHandler(
    IBookRepository bookRepository,
    IValidator<CreateBookCommand> validator,
    TimeProvider timeProvider)
    : ICommandHandler<CreateBookCommand, CreateBookResponse>
{
    public async Task<Result<CreateBookResponse>> HandleAsync(
        CreateBookCommand command,
        CancellationToken cancellationToken)
    {
        var validation = validator.Validate(command);
        if (!validation.IsValid)
        {
            return validation.ToError();
        }

        var book = Book.Create(command.Title, command.Author, command.PublishedYear, timeProvider.GetUtcNow());
        await bookRepository.AddAsync(book, cancellationToken);

        return new CreateBookResponse(book.Id, book.Title);
    }
}
```

Handler rules:
- `sealed`, uses a primary constructor, and depends on **interfaces/abstractions only** (`IBookRepository`, `IValidator<T>`, `TimeProvider`, `ILogger<T>`), never on concrete classes.
- All I/O is `async` and accepts and passes on a `CancellationToken`.
- Don't use `DateTime.Now`/`UtcNow` directly; inject `TimeProvider` so it can be tested.
- A handler never calls another handler. If logic needs to be shared, extract it into the entity or into `Shared/`.
- Queries never modify state.

### 5. Entity: protects its own invariants

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

### 6. Registration: one file per entity

```csharp
namespace BookRatings.Features.Books;

public static class BooksFeatureRegistration
{
    public static IServiceCollection AddBooksFeature(this IServiceCollection services)
    {
        services.AddScoped<IBookRepository, BookRepository>();

        services.AddScoped<ICommandHandler<CreateBookCommand, CreateBookResponse>, CreateBookCommandHandler>();
        services.AddScoped<IValidator<CreateBookCommand>, CreateBookCommandValidator>();

        services.AddScoped<IQueryHandler<GetBookByIdQuery, GetBookByIdResponse>, GetBookByIdQueryHandler>();

        return services;
    }
}
```

`Program.cs` only calls `builder.Services.AddBooksFeature();`. Register every new slice in its entity's registration file. Handlers, validators and repositories are `Scoped` by default.

### 7. UI (Blazor page in the slice)

- The page lives in the slice folder and has its `@page` route there. Routing finds it automatically.
- Inject the handler **interface**, never the concrete class:
  ```razor
  @inject ICommandHandler<CreateBookCommand, CreateBookResponse> CreateBookHandler
  ```
- `EditForm` needs a mutable model: bind to `CreateBookFormModel`, then map it to the immutable command on submit.
- Pages contain only presentation logic: binding, calling the handler, showing the `Result`. No business rules and no direct data access in `.razor` files.
- Add new feature namespaces to `Components/_Imports.razor` (or a `_Imports.razor` in the feature folder) instead of repeating `@using` in each page.
- Add a link in `Components/Layout/NavMenu.razor` when the page is meant to be reached from the navigation menu.

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
- Put each entity's persistence behind an interface in its `Shared/` folder (`IBookRepository`).
- An in-memory implementation registered as a singleton is fine as a placeholder.
- When a database is introduced (likely EF Core), keep entity configuration inside the feature (`Features/Books/Shared/BookConfiguration.cs`). Handlers keep depending on the interface and don't change.

## Testing

When a test project is added, its folder structure mirrors the feature folders (`BookRatings.Tests/Features/Books/CreateBook/CreateBookCommandHandlerTests.cs`). Test handlers and validators through their public interface, replacing dependencies with fakes or mocks of the interfaces they depend on.

## Checklist for a New Use Case

1. Create `Features/{Entity}/{UseCase}/`.
2. Add the command or query record, the response record and any DTOs, each in its own file.
3. Add a validator if there is input.
4. Implement the handler against interfaces only, returning `Result<T>`.
5. Register the handler (and validator) in `{Entity}FeatureRegistration.cs`.
6. Add the page and form model if the use case has UI, and update `NavMenu.razor` if needed.
7. Build with `dotnet build` and make sure there are no warnings or errors.
