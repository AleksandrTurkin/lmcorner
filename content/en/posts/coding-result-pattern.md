+++
date = '2026-06-19T00:41:14+02:00'
title = 'Agentic Coding: Result Pattern'
tags = ["best-practices", "software-design", "dotnet", "jardi-tips"]
author = ["Aleksandr T."]
+++

### Hello there! 🖖

### Overview

* **Ecosystem:** .NET 10
* **Technologies:** ASP.NET Core Web API
* **Improvements:** `Result Pattern`, `GlobalExceptionHandler`

**CRUD before applying `Result Pattern`:**

```csharp
public void Register(IServiceCollection services)
{
    services.AddScoped<ICommandHandler<CreateCategoryCommand, Guid>, CreateCategoryCommandHandler>();
    services.AddScoped<IQueryHandler<GetCategoryByIdQuery, CategoryDto>, GetCategoryByIdQueryHandler>();
    services.AddScoped<IQueryHandler<GetCategoriesQuery, List<CategoryDto>>, GetCategoriesQueryHandler>();
    services.AddScoped<ICommandHandler<UpdateCategoryCommand, bool>, UpdateCategoryCommandHandler>();
    services.AddScoped<ICommandHandler<DeleteCategoryCommand, bool>, DeleteCategoryCommandHandler>();
}
```

**CRUD after applying `Result Pattern`:**

```csharp
public void Register(IServiceCollection services)
{
    services.AddScoped<ICommandHandler<CreateCategoryCommand, Result<Guid>>, CreateCategoryCommandHandler>();
    services.AddScoped<IQueryHandler<GetCategoryByIdQuery, Result<CategoryDto>>, GetCategoryByIdQueryHandler>();
    services.AddScoped<IQueryHandler<GetCategoriesQuery, Result<List<CategoryDto>>>, GetCategoriesQueryHandler>();
    services.AddScoped<ICommandHandler<UpdateCategoryCommand, Result>, UpdateCategoryCommandHandler>();
    services.AddScoped<ICommandHandler<DeleteCategoryCommand, Result>, DeleteCategoryCommandHandler>();
}
```

**`Result Pattern` implementation**

- [Result.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result.cs) - base operation result implementation
- [Result[T].cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result%5BT%5D.cs) - operation result implementation with a return value
- [ResultExtensions.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ResultExtensions.cs) - extension methods for mapping operation results to HTTP responses (`IResult`)

**`GlobalExceptionHandler` implementation**

- [GlobalExceptionHandler.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/ExceptionHandlers/GlobalExceptionHandler.cs) - centralized handler for unexpected exceptions

```csharp
...
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

...

app.UseExceptionHandler();
```

## Step-by-Step Implementation

The `Result Pattern` can be implemented in different ways. You can use existing libraries, such as Ardalis.Result, ErrorOr, or FluentResults. Or you can implement it yourself. In our case, we don't need an extra third-party dependency: the implementation is small and requires only a few classes and an extension method for mapping results to HTTP responses.

To add the `Result Pattern`, we need to:

- Create the base classes `Result` and `Result<T>`, which contain information about whether the operation was successful, the return value if needed, and a description of a possible error.
- Create extension methods for mapping operation results to HTTP responses (`IResult`).
- Refactor CRUD operations to use the `Result Pattern`.
- Update AI skills to work with the new pattern.

### `Result`, `Result<T>` and `ResultExtensions`

- [Result.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result.cs) - the base class that contains information about whether the operation was successful and a possible error.
- [Result[T].cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result%5BT%5D.cs) - a generic class that also contains the result value of type `T`.
- [ResultExtensions.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ResultExtensions.cs) - extension methods for mapping operation results to HTTP responses (`IResult`).

The `Update<Entity>Command` and `Delete<Entity>Command` handlers return `Result`. The `Get<Entity>ByIdQuery`, `Get<Entities>Query`, and `Create<Entity>Command` handlers return `Result<T>`.

**Result.cs**

```csharp
public class Result
{
    public bool IsSuccess { get; }

    public bool IsFailure => !IsSuccess;

    public ErrorDetail? Error { get; }

    protected Result(bool isSuccess, ErrorDetail? error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public static Result Success() => new(true, null);
    public static Result Failure(ErrorDetail error) => new(false, error);

    public static implicit operator Result(ErrorDetail error) => Failure(error);
}
```

**Result[T].cs**

```csharp
public class Result<T> : Result
{
    public T? Value { get; }

    private Result(T value) : base(true, null)
    {
        Value = value;
    }

    private Result(ErrorDetail error) : base(false, error)
    {
        Value = default;
    }

    public static Result<T> Success(T value) => new(value);
    public new static Result<T> Failure(ErrorDetail error) => new(error);

    public static implicit operator Result<T>(T value) => Success(value);
    public static implicit operator Result<T>(ErrorDetail error) => Failure(error);
}
```

**ErrorDetail.cs**

```csharp
public record ErrorDetail(
    string Code,
    string Description,
    ErrorType Type,
    Dictionary<string, string[]>? Extensions = null);
```

**ErrorType.cs**

```csharp
public enum ErrorType
{
    NotFound,
    ValidationError,
    BadRequest,
    EntityAlreadyExists
}
```

**ResultExtensions.cs**

```csharp
public static class ResultExtensions
{
    public static IResult ToHttpResult(this Result result)
    {
        if (result.IsSuccess)
            return TypedResults.NoContent();
        
        return MapErrorResult(result.Error!);
    }

    public static IResult ToHttpResult<T>(this Result<T> result)
    {
        if (result.IsSuccess)
            return TypedResults.Ok(result.Value);
       
        return MapErrorResult(result.Error!);
    }

    private static IResult MapErrorResult(ErrorDetail error)
    {
        return error.Type switch
        {
            ErrorType.NotFound => TypedResults.Problem(
                statusCode: StatusCodes.Status404NotFound,
                title: error.Code,
                detail: error.Description),

            ErrorType.ValidationError => TypedResults.ValidationProblem(
                errors: error.Extensions ?? new Dictionary<string, string[]>
                    { { error.Code, [error.Description] } },
                title: "Validation Error",
                detail: "One or more validation failures occurred."),

            ErrorType.BadRequest => TypedResults.Problem(
                statusCode: StatusCodes.Status400BadRequest,
                title: error.Code,
                detail: error.Description),

            ErrorType.EntityAlreadyExists => TypedResults.Problem(
                statusCode: StatusCodes.Status409Conflict,
                title: error.Code,
                detail: error.Description),

            _ => TypedResults.Problem(
                statusCode: StatusCodes.Status500InternalServerError,
                title: "Server Error",
                detail: error.Description)
        };
    }
}
```

In this implementation, a successful `Result<T>` maps to `200 OK`, while a successful `Result` maps to `204 No Content`. If necessary, a separate mapping to `201 Created` can be added for `Create` scenarios.

The files are located in the project as follows:

```text
Solution/
├── src/
│   ├── MyProject.Domain/
│   │    └── Common/
│   │       ├── Result.cs
│   │       └── Result[T].cs
│   └── MyProject.WebApi/
│       └── Extensions/
│           └── ResultExtensions.cs
```

### AI skills

It is important to update the AI skills to use the `Result Pattern`:

- [command-query-handler-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - skill for creating command and query handlers
- [api-endpoint-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/api-endpoint-creator/SKILL.md) - skill for creating API endpoints

If we skip this step, the AI agent will continue to generate handlers and registrations in the old format. This is exactly the case when an architectural change should be reflected not only in the code, but also in the instructions for the agent.

```markdown
Result Pattern Rules (Mandatory)
- All handler result types must use `JardiTips.Domain.Common.Result` wrappers.
- Create command handler returns `Result<Guid>`.
- Get-by-id query handler returns `Result<<EntityName>Dto>`.
- Get-list query handler returns `Result<List<<EntityName>Dto>>`.
- Update and delete command handlers return `Result`.
- Validation/not-found paths should return domain error details instead of primitive failure flags.

...

Implement `Register` method:
- Register create command handler (`ICommandHandler<..., Result<Guid>>`).
- Register get-by-id query handler (`IQueryHandler<..., Result<Dto>>`).
- Register get-list query handler (`IQueryHandler<..., Result<List<Dto>>>`).
- Register update command handler (`ICommandHandler<..., Result>`).
- Register delete command handler (`ICommandHandler<..., Result>`).

...

Use explicit registration pattern:
- `services.AddScoped<ICommandHandler<TCommand, Result<Guid>>, TCreateHandler>();`
- `services.AddScoped<IQueryHandler<TQueryById, Result<TDto>>, TGetByIdHandler>();`
- `services.AddScoped<IQueryHandler<TListQuery, Result<List<TDto>>>, TGetListHandler>();`
- `services.AddScoped<ICommandHandler<TUpdateCommand, Result>, TUpdateHandler>();`
- `services.AddScoped<ICommandHandler<TDeleteCommand, Result>, TDeleteHandler>();`
```

### GlobalExceptionHandler

Unfortunately, sometimes not everything goes according to plan 😇. The `Result Pattern` covers expected errors in business scenarios: validation, entity not found, conflict, etc. But unexpected exceptions can still occur.

For such cases, we add a `GlobalExceptionHandler`: it logs unhandled exceptions in one place and returns a consistent HTTP response via `ProblemDetails`.

```csharp
internal sealed class GlobalExceptionHandler(
    IProblemDetailsService problemDetailsService,
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception, "Unhandled exception occurred");

        httpContext.Response.StatusCode = exception switch
        {
            ApplicationException => StatusCodes.Status400BadRequest,
            _ => StatusCodes.Status500InternalServerError
        };

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails = new ProblemDetails
            {
                Type = exception.GetType().Name,
                Title = "Server.InternalError",
                Detail = exception.Message
            }
        });
    }
}
```

In a real production API, the `Detail` field can be replaced with a more neutral message to avoid exposing internal exception details.

Don't forget to register it:

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

...

app.UseExceptionHandler();
```

### Useful Resources

* [Stop throwing exceptions in C#](https://www.youtube.com/watch?v=a1ye9eGTB98) - a short video demonstrating how costly it can be to use exceptions for expected errors.
* [Error Handling in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling) - official documentation on error handling in ASP.NET Core.
* [Global Error Handling in ASP.NET Core: From Middleware to Modern Handlers](https://www.milanjovanovic.tech/blog/global-error-handling-in-aspnetcore-from-middleware-to-modern-handlers) - an article on the modern approach to global error handling in ASP.NET Core.

### Stress Testing

I used NBomber for stress testing. The scenario is simple: updating a non-existent record with 10 parallel scenarios and 100 requests in each scenario.

So each request goes through a negative scenario: before, it was handled through an exception; after applying the `Result Pattern`, it is handled through a regular operation result.
In this test, `fail RPS` means how many negative responses are processed per second.

**Scenario 1:** without using `Result Pattern` and `GlobalExceptionHandler`

```text
duration: 00:01:01
fail RPS = 1.59/s
```

**Scenario 2:** without using `Result Pattern`, but with `GlobalExceptionHandler`

```text
duration: 00:00:43
fail RPS = 2.33/s
```

**Scenario 3:** with `Result Pattern`

```text
duration: 00:00:02
fail RPS = 50/s
```

This is a local test for a specific scenario, not a universal benchmark. But the difference clearly shows why exceptions should not be used as the main mechanism for handling expected errors.

## Blabber

And here the pitfalls of [agentic coding](https://lmcorner.net/en/posts/agentic-coding-step1/) begin. On the one hand, we defined the initial architecture, created several skills, split responsibilities, described the AI agent, and explained how it should orchestrate the skills. But at the same time, we missed several critical things: **validation**, **pagination**, **error handling**, **logging**...

If we start actively generating code in vibe-coding mode right now without taking these things into account, the product, to put it mildly, will not take off.

An AI agent is good at automating repetitive actions, but it is also good at scaling architectural mistakes. If the template has no validation, error handling, or logging, the agent will consistently generate code without validation, error handling, or logging.

Yes, you can live without the `Result Pattern`: the functionality itself will not disappear. But if exceptions are used as the main way to handle expected errors, problems with readability, predictability, and, in some scenarios, performance will appear very quickly.

In agentic coding, software design and architecture matter more than ever. And for now, it is still our human responsibility to do it right. 😉

#### Thanks! Keep calm and code on! 🚀