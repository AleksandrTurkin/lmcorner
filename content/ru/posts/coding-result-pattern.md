+++
date = '2026-06-19T00:41:08+02:00'
title = 'Агентское программирование. Часть 2.0: Result Pattern'
tags = ["best-practices", "software-design", "dotnet", "jardi-tips"]
author = ["Александр Т."]
+++

### Всем привет! 🖖

### Краткий обзор

* **Фреймворк:** .NET 10
* **Технологии:** ASP.NET Core Web API
* **Улучшение:** `Result Pattern`, `GlobalExceptionHandler`

**CRUD до применения `Result Pattern`:**

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

**CRUD после применения `Result Pattern`:**

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

**Реализация `Result Pattern`**

- [Result.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result.cs) - реализация базового результата операции
- [Result[T].cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result%5BT%5D.cs) - реализация результата операции с возвращаемым значением
- [ResultExtensions.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ResultExtensions.cs) - методы расширения для маппинга результатов операции в HTTP-ответы (`IResult`)

**Реализация `GlobalExceptionHandler`**

- [GlobalExceptionHandler.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/ExceptionHandlers/GlobalExceptionHandler.cs) - централизованный обработчик непредвиденных исключений

```csharp
...
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

...

app.UseExceptionHandler();
```

## Реализация в деталях

Добавить `Result Pattern` в проект можно как с использованием готовых библиотек, например Ardalis.Result, ErrorOr или FluentResults, так и самостоятельно. В моём случае дополнительная зависимость на сторонний пакет не нужна: реализация небольшая и сводится к нескольким классам и методу расширения для маппинга результата в HTTP-ответы.

Для добавления `Result Pattern` нам необходимо:

- Создать базовые классы `Result` и `Result<T>`, которые будут содержать информацию об успешности операции, возвращаемое значение при необходимости и описание возможной ошибки.
- Создать методы расширения для маппинга результатов операции в HTTP-ответы (`IResult`).
- Переделать CRUD-операции для использования `Result Pattern`.
- Обновить AI-скиллы для работы с новым паттерном.

### `Result`, `Result<T>` и `ResultExtensions`

- [Result.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result.cs) - представляет собой базовый класс, который содержит информацию об успешности операции и возможной ошибке.
- [Result[T].cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Domain/Common/Result%5BT%5D.cs) - является обобщённым классом, который дополнительно содержит результат операции типа `T`.
- [ResultExtensions.cs](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/ResultExtensions.cs) - содержит методы расширения для маппинга результатов операции в HTTP-ответы (`IResult`).

Команды `Update<Entity>Command` и `Delete<Entity>Command` возвращают `Result`, а запросы `Get<Entity>ByIdQuery`, `Get<Entities>Query`, а также команда `Create<Entity>Command` возвращают `Result<T>`.

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

В этой реализации успешный `Result<T>` маппится в `200 OK`, а успешный `Result` - в `204 No Content`. При необходимости для `Create`- сценариев можно добавить отдельный маппинг в `201 Created`.

Размещение файлов в проекте выглядит следующим образом:

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

### AI-скиллы

Обязательно обновляем AI-скиллы для использования `Result Pattern`:

- [command-query-handler-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/command-query-handler-creator/SKILL.md) - скилл для создания обработчиков команд и запросов
- [api-endpoint-creator](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/.github/skills/api-endpoint-creator/SKILL.md) - скилл для создания API-эндпоинтов

Если этого не сделать, AI-агент продолжит генерировать обработчики и регистрации в старом формате. Это как раз тот случай, когда архитектурное изменение должно быть отражено не только в коде, но и в инструкциях для агента.

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

К сожалению, иногда не всё идёт по плану 😇. `Result Pattern` закрывает ожидаемые ошибки бизнес-сценариев: валидацию, отсутствие сущности, конфликт и т.д. Но непредвиденные исключения всё равно могут возникнуть.

Для таких случаев добавляем `GlobalExceptionHandler`: он централизованно логирует необработанные исключения и возвращает единообразный HTTP-ответ через `ProblemDetails`.

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
В реальном production API поле `Detail` можно заменить на более нейтральное сообщение, чтобы не раскрывать внутренние детали исключения.

Не забываем зарегистрировать:

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

...

app.UseExceptionHandler();
```

### Полезные источники

* [Stop throwing exceptions in C#](https://www.youtube.com/watch?v=a1ye9eGTB98) - небольшое видео с наглядной демонстрацией того, насколько дорого может обходиться использование исключений для ожидаемых ошибок.
* [Error Handling in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling) - официальная документация по обработке ошибок в ASP.NET Core.
* [Global Error Handling in ASP.NET Core: From Middleware to Modern Handlers](https://www.milanjovanovic.tech/blog/global-error-handling-in-aspnetcore-from-middleware-to-modern-handlers) - статья о современном подходе к глобальной обработке ошибок в ASP.NET Core.

### Стресс-тестирование

Я использовал NBomber для проведения стресс-тестирования. Сценарий простой: обновление несуществующей записи при 10 параллельных сценариях и 100 запросах в каждом.

То есть каждый запрос попадает в отрицательный сценарий: раньше он обрабатывался через исключение, после внедрения `Result Pattern` - через обычный результат операции.

**Сценарий 1:** без использования `Result Pattern` и `GlobalExceptionHandler`

```text
duration: 00:01:01
fail RPS = 1.59/s
```

**Сценарий 2:** без использования `Result Pattern`, но с использованием `GlobalExceptionHandler`

```text
duration: 00:00:43
fail RPS = 2.33/s
```

**Сценарий 3:** с использованием `Result Pattern`

```text
duration: 00:00:02
fail RPS = 50/s
```

Это локальный тест конкретного сценария, а не универсальный бенчмарк. Но разница хорошо показывает, почему исключения не стоит использовать как штатный механизм обработки ожидаемых ошибок.

## Болтовня

Вот и начинаются подводные камни в [агентском программировании](https://lmcorner.net/ru/posts/agentic-coding-step1/). С одной стороны, мы задали начальную архитектуру, создали несколько скиллов, распределили зоны ответственности, описали AI-агента и то, как он должен оркестрировать работу скиллов. Но при этом упустили несколько критически важных вещей: **валидацию**, **пагинацию**, **обработку ошибок**, **логирование**...

Если мы прямо сейчас начнём активно генерировать код в режиме вайб-программирования без учёта этих моментов, то продукт, мягко говоря, не взлетит.

AI-агент хорошо автоматизирует повторяемые действия, но так же хорошо масштабирует архитектурные ошибки. Если в шаблоне нет валидации, обработки ошибок и логирования, агент будет стабильно генерировать код без валидации, обработки ошибок и логирования.

Да, можно жить без `Result Pattern`: сама функциональность от этого не исчезнет. Но если использовать исключения как основной способ управления ожидаемыми ошибками, быстро начинаются проблемы с читаемостью, предсказуемостью и, в некоторых сценариях, производительностью.

В агентском программировании как никогда остро стоит вопрос дизайна и архитектуры программного обеспечения. И это пока наша, человеческая, ответственность: сделать всё правильно. 😉

#### Спасибо! Улыбаемся и пашем! 🚀