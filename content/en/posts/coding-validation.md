+++
date = '2026-07-07T00:09:04+02:00'
title = 'Agentic Coding. Part 2.1: Validation'
tags = ["best-practices", "dotnet", "jardi-tips"]
author = ["Aleksandr T."]
+++

### Hello there! 🖖

### Overview

* **Ecosystem:** .NET 10
* **Technologies:** ASP.NET Core Web API
* **Improvement:** validation with `DataAnnotations` + `Endpoint Filter`

Example DTO with `DataAnnotations` attributes for validation:

```csharp
public record CreateCategoryDto(
    [property: Required, StringLength(250)] string Name,
    [property: Required, StringLength(2000)] string Description,
    [property: Required, EnumDataType(typeof(CategoryType))] CategoryType Type);
```

Validation filter implementation: [ValidationEndpointFilter](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ValidationEndpointFilter.cs)

Short example of connecting the filter to POST and PUT methods via `AddEndpointFilter<ValidationEndpointFilter>()`:

```csharp
public static RouteHandlerBuilder MapPostCommand<TRequest, TDto>(...)
    where TRequest : class
    => builder.MapPost(...) =>
        {
            var request = create(dto);
            var result = await handler.HandleAsync(request, cancellationToken);
            return result.ToHttpResult();
        }).AddEndpointFilter<ValidationEndpointFilter>();

...

public static RouteHandlerBuilder MapPutCommand<TRequest, TDto, TKey>(...)
    where TRequest : class
    => builder.MapPut(...) =>
        {
            var request = create(id, dto);
            var result = await handler.HandleAsync(request, cancellationToken);
            return result.ToHttpResult();
        }).AddEndpointFilter<ValidationEndpointFilter>();
```

## Step-by-Step Implementation

Data validation is an obvious but critical part of any application. There are several ways to implement it, and I would like to talk about a few of them: `FluentValidation`, `Endpoint Filter` for request interception, and `.NET 10 Native Validation`.

Let's quickly go through each option:

1. `FluentValidation` - a third-party library that allows you to create flexible validation rules for models. It is good for complex validation scenarios where you need to check several model properties or run custom logic. The downside is an extra dependency and potentially higher execution cost compared to built-in validation.

2. `Endpoint Filter` - this is not a validation library, but a Minimal APIs mechanism that allows you to intercept requests before the endpoint handler is executed. It is useful when you want centralized validation for input DTOs. The downside is extra runtime cost compared to `.NET 10 Native Validation`.

3. `.NET 10 Native Validation` - built-in validation support in .NET 10. It allows you to use `DataAnnotations` on DTOs and enable validation with `AddValidation()`. This is a simple and fast validation approach, but it can be less flexible than `FluentValidation`, and it does not always work well with generic endpoint helpers.


### .NET 10 Native Validation

`.NET 10 Native Validation` is fast because of built-in validation support for Minimal APIs and optimized code generation. Compared to manual validation through `Endpoint Filter`, part of the work can be prepared in advance, which reduces runtime overhead.

Usage example:

```csharp
public record CreateCategoryDto(
    [property: Required, StringLength(250)] string Name,
    [property: Required, StringLength(2000)] string Description,
    [property: Required, EnumDataType(typeof(CategoryType))] CategoryType Type);
```

Validation registration:

```csharp
builder.Services.AddValidation();
```

For .NET 10, the `Microsoft.Extensions.Validation` package may be required, because unified validation APIs have moved to a separate package.

But, as you remember, our [`Endpoints`](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Endpoints/CategoryEndpoint.cs) implementation uses generic methods. In my current implementation, because of this, built-in validation does not see the concrete DTO in the same way as it does in a regular fixed endpoint signature. As a result, attributes inside our `CreateCategoryDto` can be ignored.

We can remove generic methods. In this case, we will get this "fixed" POST/PUT implementation:

```csharp
group.MapPost("", async (
    CreateCategoryDto dto,
    ICommandHandler<CreateCategoryCommand, Result<Guid>> handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(new CreateCategoryCommand(dto), ct);
    return result.ToHttpResult();
});

group.MapGetByIdQuery<GetCategoryByIdQuery, CategoryDto, Guid>(
    "{id:guid}",
    id => new GetCategoryByIdQuery(id));

group.MapGetFilterQuery<GetCategoriesQuery, List<CategoryDto>, CategoriesFilterDto>(
    "",
    filters => new GetCategoriesQuery(filters));

group.MapPut("{id:guid}", async (
    Guid id,
    UpdateCategoryDto dto,
    ICommandHandler<UpdateCategoryCommand, Result> handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(new UpdateCategoryCommand(id, dto), ct);
    return result.ToHttpResult();
});

group.MapDeleteCommand<DeleteCategoryCommand, Guid>(
    "{id:guid}",
    id => new DeleteCategoryCommand(id));
```

### Validation through `Endpoint Filter`

Validation through `Endpoint Filter` allows us to check input data at the API boundary, before the request reaches the handler. Yes, it is slower than `.NET 10 Native Validation`, but for now the code maintenance convenience wins.

I will use `DataAnnotations` attributes to describe validation. As a result, the DTO will look exactly the same as with `.NET 10 Native Validation`. This will make it easier to switch between approaches later, if needed.

```csharp
public record CreateCategoryDto(
    [property: Required, StringLength(250)] string Name,
    [property: Required, StringLength(2000)] string Description,
    [property: Required, EnumDataType(typeof(CategoryType))] CategoryType Type);
```

I will add the filter to POST and PUT methods via `AddEndpointFilter<ValidationEndpointFilter>()`, so input DTOs are validated before command execution.

```csharp
public static RouteHandlerBuilder MapPostCommand<TRequest, TDto>(
    this IEndpointRouteBuilder builder,
    string pattern,
    Func<TDto, TRequest> create)
    where TRequest : class
    => builder.MapPost(pattern,
        [AllowAnonymous] async (
            [FromBody] TDto dto,
            [FromServices] ICommandHandler<TRequest, Result<Guid>> handler,
            CancellationToken cancellationToken) =>
        {
            var request = create(dto);
            var result = await handler.HandleAsync(request, cancellationToken);
            return result.ToHttpResult();
        }).AddEndpointFilter<ValidationEndpointFilter>();

...

public static RouteHandlerBuilder MapPutCommand<TRequest, TDto, TKey>(
    this IEndpointRouteBuilder builder,
    string pattern,
    Func<TKey, TDto, TRequest> create)
    where TRequest : class
    => builder.MapPut(pattern,
        [AllowAnonymous] async (
            TKey id,
            [FromBody] TDto dto,
            [FromServices] ICommandHandler<TRequest, Result> handler,
            CancellationToken cancellationToken) =>
        {
            var request = create(id, dto);
            var result = await handler.HandleAsync(request, cancellationToken);
            return result.ToHttpResult();
        }).AddEndpointFilter<ValidationEndpointFilter>();
```

And the validator filter implementation [ValidationEndpointFilter](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ValidationEndpointFilter.cs) looks like this:

```csharp
public sealed class ValidationEndpointFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        foreach (var argument in context.Arguments)
        {
            if (argument is null || IsSystemType(argument.GetType()))
                continue;

            var results = new List<ValidationResult>();
            var validationContext = new ValidationContext(argument);

            if (Validator.TryValidateObject(argument, validationContext, results, validateAllProperties: true))
                continue;

            var errors = new Dictionary<string, string[]>();
            foreach (var result in results)
            {
                var message = result.ErrorMessage ?? "Invalid value.";
                var members = result.MemberNames.Any() ? result.MemberNames : [string.Empty];
                foreach (var member in members)
                    errors[member] = errors.TryGetValue(member, out var existing)
                        ? [.. existing, message]
                        : [message];
            }

            var error = new ErrorDetail(
                "validation-failed",
                "One or more validation failures occurred.",
                ErrorType.ValidationError,
                errors);

            return Result.Failure(error).ToHttpResult();
        }

        return await next(context);
    }

    private static bool IsSystemType(Type type)
    {
        type = Nullable.GetUnderlyingType(type) ?? type;

        if (type.IsValueType)
            return true;

        return type == typeof(string) ||
               type == typeof(HttpContext) ||
               type == typeof(System.Security.Claims.ClaimsPrincipal) ||
               type.Namespace?.StartsWith("System", StringComparison.Ordinal) == true ||
               type.Namespace?.StartsWith("Microsoft.AspNetCore", StringComparison.Ordinal) == true;
    }
}
```

The filter goes through endpoint arguments, skips system and technical types, and validates the remaining objects. This is important because endpoint arguments can contain not only DTOs, but also services, `CancellationToken`, `HttpContext`, and other technical parameters.

If the DTO does not pass validation, the filter returns `Result.Failure(...).ToHttpResult()`, and the request does not reach the command handler.

Important: this implementation validates the DTO itself and its properties with attributes, but it is not full recursive validation of a complex object graph. For the current DTOs, this is enough, but nested objects may require additional logic.

### AI skills

Do not forget about AI skills and validation 🤖.

If we do not update the skill, the agent will continue to generate DTOs without validation attributes. This means the architectural change will stay only in the current code and will not be applied to the next generated CRUD scenarios.

- [command-query-handler-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - we extend the skill for creating command and query handlers

```markdown
## DTO Validation Rules (Mandatory)
Input DTOs must declare validation using `System.ComponentModel.DataAnnotations` so model validation is enforced at the endpoint boundary.

Apply validation to input DTOs:
- `Create<EntityName>Dto`
- `Update<EntityName>Dto`

Attribute target rule (critical for records):
- On positional records, annotate each parameter with the `property` target so attributes bind to the generated properties, not the constructor parameters.
- Correct: `[property: Required, StringLength(250)] string Name`
- Incorrect: `[Required][StringLength(250)] string Name` (attributes land on the constructor parameter and are ignored by validation)

Attribute selection guidance:
- Use `[Required]` for mandatory fields.
- Use `[StringLength(max)]` for bounded text; keep the max length aligned with the entity/EF configuration.
- Use `[EnumDataType(typeof(<EnumType>))]` for enum inputs.
- Use `[Range(...)]`, `[EmailAddress]`, `[Url]`, and similar attributes where the field semantics require them.

Non-input DTOs:
- Output DTOs (`<EntityName>Dto`) do not require input validation attributes.
- Filter DTOs (`<FeatureFolderName>FilterDto`) only need validation attributes when paging/filter inputs require bounds; keep `[property: ...]` targeting when the filter is a positional record.

Reference DTOs:
- `JardiTips.Application/Features/Categories/Models/CreateCategoryDto.cs`
- `JardiTips.Application/Features/Categories/Models/UpdateCategoryDto.cs`
```

## Blabber

**Why do we need validation?** In theory, it adds a fixed cost to every request. In an ideal world, we could avoid this fixed cost, but it is a very small price that removes a huge cost from invalid requests.

**Example 1 (without validation) - expensive failure**

Every invalid request goes through the full stack in `CreateCategoryCommandHandler`:

```text
1. Handler runs, Map(...) allocates CategoryEntity.
2. EF Core change tracking + SQL generation.
3. Network round-trip to PostgreSQL - the most expensive part.
4. DB applies the constraint and rejects the request - Npgsql throws DbUpdateException.
5. Exception propagates unhandled - stack unwinding.
6. GlobalExceptionHandler calls logger.LogError(exception, ...) - full stack trace is collected/serialized.
7. ProblemDetails is written - HTTP 500.
```

**Example 2 (with validation) - cheap failure**

Invalid requests are stopped at the API boundary:

```text
- Handler is not executed.
- No DB round-trip.
- No exception.
- No stack trace logging.
- Only microsecond validation cost, then Result.Failure(...).ToHttpResult() - 400.
```

Under load, the cost of the example without validation grows together with concurrency:

```text
- DB connection pool pressure - bad requests hold connections during the round-trip.
- GC pressure - exception objects and stack traces are allocated on every failure.
- Thread pool and latency - exception handling and log serialization add tail latency.
```

Validation limits all of this to a cheap synchronous check before any I/O.

Moving away from generic methods and switching to `.NET 10 Native Validation` is a valid idea. Especially in the agentic coding paradigm, where the amount of boilerplate code becomes secondary: it will be written or generated by AI agents anyway.

But for now, I will keep the generic methods. If validation becomes a performance bottleneck in the future, we already know where to move and how to speed up this part 😉.

#### Thanks! Keep calm and code on! 🚀