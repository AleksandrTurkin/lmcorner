+++
date = '2026-07-07T00:08:57+02:00'
title = 'Агентское программирование. Часть 2.1: Validation'
tags = ["best-practices", "dotnet", "jardi-tips"]
author = ["Александр Т."]
+++

### Всем привет! 🖖

### Краткий обзор

* **Фреймворк:** .NET 10
* **Технологии:** ASP.NET Core Web API
* **Улучшение:** валидация через `DataAnnotations` + `Endpoint Filter`

Пример DTO с `DataAnnotations` - атрибутами для валидации:
```csharp
public record CreateCategoryDto(
    [property: Required, StringLength(250)] string Name,
    [property: Required, StringLength(2000)] string Description,
    [property: Required, EnumDataType(typeof(CategoryType))] CategoryType Type);
```

Реализация фильтра-валидатора: [ValidationEndpointFilter](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ValidationEndpointFilter.cs)

Сокращённый пример подключения фильтра к методам POST и PUT через `AddEndpointFilter<ValidationEndpointFilter>()`:

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

## Реализация в деталях

Валидация данных - довольно очевидная, но очень важная часть любого приложения. Есть несколько вариантов её реализации, о которых я хотел бы поговорить: `FluentValidation`, `Endpoint Filter` для перехвата запросов и `.NET 10 Native Validation`.

Давайте быстренько пройдёмся по каждому:

1. `FluentValidation` - сторонняя библиотека, которая позволяет создавать гибкие правила валидации для моделей. Хорошо подходит для сложных сценариев, где нужно проверить несколько свойств модели или выполнить кастомную логику. Из минусов: дополнительная зависимость и потенциально более высокая стоимость выполнения по сравнению со встроенной валидацией.
2. `Endpoint Filter` - это не библиотека для валидации, а механизм Minimal APIs, который позволяет перехватывать запросы до выполнения обработчика endpoint-а. Его удобно использовать для централизованной проверки входных DTO. Из минусов - дополнительная runtime-стоимость по сравнению с `.NET 10 Native Validation`.
3. `.NET 10 Native Validation` - встроенная поддержка валидации в .NET 10. Она позволяет использовать `DataAnnotations` на DTO и подключается через `AddValidation()`. Это простой и производительный способ валидации, но он может быть менее гибким, чем `FluentValidation`, и не всегда удобно сочетается с generic endpoint helpers.

### .NET 10 Native Validation

`.NET 10 Native Validation` работает быстро за счёт встроенной поддержки Minimal APIs и генерации оптимизированного кода. В отличие от ручной проверки через `Endpoint Filter`, часть работы может быть подготовлена заранее, что снижает runtime overhead.

Пример использования:

```csharp
public record CreateCategoryDto(
    [property: Required, StringLength(250)] string Name,
    [property: Required, StringLength(2000)] string Description,
    [property: Required, EnumDataType(typeof(CategoryType))] CategoryType Type);
```

Регистрация валидации:

```csharp
builder.Services.AddValidation();
```

Для .NET 10 может потребоваться пакет `Microsoft.Extensions.Validation`, так как единые API валидации вынесены в отдельный пакет.

Но, как вы помните, реализация наших [`Endpoints`](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Endpoints/CategoryEndpoint.cs) использует generic-методы. В моей текущей реализации из-за этого встроенная валидация не видит конкретный DTO так, как в обычной фиксированной сигнатуре endpoint-а. В результате атрибуты внутри нашего `CreateCategoryDto` могут быть проигнорированы.

Можно отказаться от generic-методов. В таком случае мы получим вот такую "фиксированную" реализацию POST/PUT:

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

### Валидация через `Endpoint Filter`

Валидация через `Endpoint Filter` позволяет проверять входные данные на границе API, до того как запрос попадёт в обработчик. Да, это работает медленнее, чем `.NET 10 Native Validation`, но удобство поддержки кода пока перевешивает.

Я буду использовать `DataAnnotations`-атрибуты для описания валидации. В результате DTO будет выглядеть точно так же, как и при `.NET 10 Native Validation`. Это позволит с меньшими изменениями переключаться между подходами, если возникнет такая необходимость.

```csharp
public record CreateCategoryDto(
    [property: Required, StringLength(250)] string Name,
    [property: Required, StringLength(2000)] string Description,
    [property: Required, EnumDataType(typeof(CategoryType))] CategoryType Type);
```

Сам фильтр я добавлю к методам POST и PUT через `AddEndpointFilter<ValidationEndpointFilter>()`, чтобы проверять входные DTO перед выполнением команд.

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

И сама реализация фильтра-валидатора [ValidationEndpointFilter](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ValidationEndpointFilter.cs) выглядит так:

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

Фильтр проходит по аргументам endpoint-а, ищет объекты с `DataAnnotations`-атрибутами и валидирует только их. Это важно, потому что в аргументах endpoint-а могут быть не только DTO, но и сервисы, `CancellationToken`, `HttpContext` и другие технические параметры.

Если DTO не проходит валидацию, фильтр возвращает `Result.Failure(...).ToHttpResult()` и запрос не доходит до обработчика команды.

Важно: такая реализация валидирует сам DTO и его свойства с атрибутами, но не является полноценной рекурсивной валидацией сложного object graph. Для текущих DTO этого достаточно, но для вложенных объектов может потребоваться дополнительная логика.

### AI-скиллы

Не забываем про AI-скиллы и валидацию 🤖.

Если не обновить skill, агент продолжит генерировать DTO без validation attributes. А значит, архитектурное изменение останется только в текущем коде и не попадёт в следующие сгенерированные CRUD-сценарии.

- [command-query-handler-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - расширим скилл для создания обработчиков команд и запросов

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

## Болтовня

**Зачем нам валидация?** По идее, она добавляет фиксированную стоимость к каждому запросу. В идеальном мире можно было бы отказаться от этой фиксированной стоимости, но это очень маленькая плата, которая убирает огромную стоимость с невалидных запросов.

**Пример 1 (без валидации) - дорогая ошибка**

Каждый невалидный запрос проходит весь стек в `CreateCategoryCommandHandler`:

```text
1. Выполняется обработчик, Map(...) выделяет CategoryEntity.
2. Отслеживание изменений EF Core + генерация SQL.
3. Сетевое обращение к PostgreSQL - самое затратное.
4. БД применяет ограничение и отклоняет запрос - Npgsql выбрасывает DbUpdateException.
5. Исключение распространяется необработанным - раскрутка стека.
6. GlobalExceptionHandler вызывает logger.LogError(exception, ...) - собирается/сериализуется полная трассировка стека.
7. Записывается ProblemDetails - HTTP 500.
```

**Пример 2 (с валидацией) - дешёвая ошибка**

Невалидные запросы отсекаются на границе API:

```text
- Обработчик не выполняется.
- Нет обращения к БД.
- Нет исключения.
- Нет логирования трассировки стека.
- Только микросекундная стоимость валидации, затем Result.Failure(...).ToHttpResult() - 400.
```

Под нагрузкой стоимость примера без валидации растёт вместе с конкурентностью:

```text
- Давление на пул соединений с БД - плохие запросы удерживают соединения во время обращения.
- Давление на GC - объекты исключений и трассировки стека выделяются при каждой ошибке.
- Пул потоков и задержка - обработка исключений и сериализация логов добавляют хвостовую задержку.
```

Валидация ограничивает всё это дешёвой синхронной проверкой до любого ввода-вывода.

Отказаться от generic-методов и перейти на `.NET 10 Native Validation` - вполне рабочая идея. Особенно в парадигме агентского программирования, где объём шаблонного кода становится второстепенной величиной: он всё равно будет написан или сгенерирован AI-агентами.

Но на данном этапе я оставлю generic-методы. Если в будущем валидация станет узким местом по производительности, мы уже знаем, куда двигаться и как ускорить этот участок 😉.

#### Спасибо! Улыбаемся и пашем! 🚀