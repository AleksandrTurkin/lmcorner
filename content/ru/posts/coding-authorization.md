+++
date = '2026-07-22T00:00:00+02:00'
title = 'Агентское программирование. Часть 2.3: Authentication & Authorization'
tags = ["best-practices", "software-design", "dotnet", "jardi-tips"]
author = ["GPT-5.6 Sol High", "Александр Т."]
+++

### Всем привет! 🖖

### Краткий обзор

* **Фреймворк:** .NET 10
* **Технологии:** ASP.NET Core Web API, JWT, PostgreSQL, Entity Framework Core
* **Улучшение:** Аутентификация через `Access Token` и `Refresh Token`, глобальная политика авторизации

В API появились три endpoint-а:

```text
POST /api/auth/login   - вход по email и паролю
POST /api/auth/refresh - обновление пары токенов
POST /api/auth/logout  - отзыв refresh token
```

Все остальные endpoint-ы по умолчанию требуют аутентифицированного пользователя:

```csharp
services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());
```

Для доступа к защищённым endpoint-ам клиент передаёт JWT в заголовке:

```http
Authorization: Bearer <access_token>
```

Полную реализацию можно посмотреть в коммите [`b5d4260`](https://github.com/AleksandrTurkin/api-jardi-tips/commit/b5d4260d81b85879402f997f56376bbd12b318d2).

Основные части реализации:

- [AuthEndpoint](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Endpoints/AuthEndpoint.cs) - endpoint-ы входа, обновления и отзыва токена.
- [AuthenticationExtensions](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/AuthenticationExtensions.cs) - регистрация JWT-аутентификации и глобальной политики авторизации.
- [JwtTokenService](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Infrastructure/Authentication/JwtTokenService.cs) - создание access token и refresh token.
- [LoginCommandHandler](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Application/Features/Authentication/LoginCommandHandler.cs) - проверка логина и пароля.
- [RefreshTokenCommandHandler](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Application/Features/Authentication/RefreshTokenCommandHandler.cs) - ротация refresh token.
- [RevokeRefreshTokenCommandHandler](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Application/Features/Authentication/RevokeRefreshTokenCommandHandler.cs) - отзыв refresh token при выходе.

## Реализация в деталях

Для начала давайте разделим два понятия, которые часто смешивают.

**Аутентификация** отвечает на вопрос: «Кто ты?». Пользователь передаёт email и пароль, а API проверяет их и выдаёт токены.

**Авторизация** отвечает на вопрос: «Что тебе разрешено?». В текущей реализации она пока базовая: пользователь либо аутентифицирован и получает доступ к API, либо нет. Роли, permissions и отдельные policy здесь ещё не реализованы.

Таким образом, в этой части мы реализуем полноценную JWT-аутентификацию и закладываем фундамент для дальнейшей авторизации.

### Общая схема

Схема работы выглядит следующим образом:

```text
1. Клиент отправляет email и пароль на POST /api/auth/login.
2. API проверяет пароль и возвращает access token + refresh token.
3. Клиент передаёт access token в Authorization: Bearer <token>.
4. Через 15 минут access token истекает.
5. Клиент отправляет refresh token на POST /api/auth/refresh.
6. API отзывает старый refresh token и возвращает новую пару токенов.
7. При выходе клиент отправляет refresh token на POST /api/auth/logout.
```

Access token живёт недолго и используется при каждом запросе. Refresh token живёт дольше и нужен только для получения новой пары токенов.

```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "RfGJmK8...",
  "accessTokenExpiresAt": "2026-07-22T12:15:00Z",
  "refreshTokenExpiresAt": "2026-08-21T12:00:00Z",
  "tokenType": "Bearer"
}
```

### Модель пользователя

Для хранения пользователей добавим две сущности: `UserEntity` и `UserLoginEntity`.

```csharp
public class UserEntity : BaseEntity
{
    public string Email { get; set; }

    public string DisplayName { get; set; }

    public string? AvatarUrl { get; set; }

    public string? PasswordHash { get; set; }

    public ICollection<UserLoginEntity> Logins { get; set; } = [];
}
```

`PasswordHash` nullable неслучайно. Пользователь может входить не только по паролю, но и через внешний провайдер: например, Google, GitHub или Microsoft. Для связи с такими провайдерами подготовлена отдельная сущность:

```csharp
public class UserLoginEntity : BaseEntity
{
    public Guid UserId { get; set; }

    public string LoginProvider { get; set; }

    public string ProviderKey { get; set; }

    public UserEntity User { get; set; }
}
```

Пара `(LoginProvider, ProviderKey)` имеет уникальный индекс. Один внешний аккаунт не сможет быть привязан сразу к нескольким пользователям.

В текущей части внешний OAuth-вход ещё не реализован. `UserLoginEntity` пока только подготавливает модель данных для следующего шага.

### Вход по email и паролю

Вход начинается с простого DTO, который использует уже знакомую нам [валидацию](https://lmcorner.net/ru/posts/coding-validation/):

```csharp
public record LoginDto(
    [property: Required, EmailAddress, StringLength(320)] string Email,
    [property: Required, StringLength(200)] string Password);
```

Сам пароль в базе, конечно, не хранится. Для его проверки используется стандартный `PasswordHasher<TUser>` из `Microsoft.AspNetCore.Identity`:

```csharp
public sealed class PasswordService : IPasswordService
{
    private readonly PasswordHasher<UserEntity> _passwordHasher = new();

    public bool VerifyPassword(UserEntity user, string password)
    {
        if (user.PasswordHash is null)
            return false;

        return _passwordHasher.VerifyHashedPassword(user, user.PasswordHash, password)
            is not PasswordVerificationResult.Failed;
    }
}
```

В `LoginCommandHandler` email нормализуется, после чего пользователь загружается из базы и пароль проверяется через `IPasswordService`.

```csharp
var normalizedEmail = command.Login.Email.Trim().ToLowerInvariant();
var user = await userRepository.FirstOrDefaultAsync(
    x => x.Email.ToLower() == normalizedEmail,
    ct);

if (user?.PasswordHash is null || !passwordService.VerifyPassword(user, command.Login.Password))
    return InvalidCredentials();
```

Важно, что API возвращает одну и ту же ошибку и для неизвестного email, и для неправильного пароля:

```csharp
new ErrorDetail(
    "invalid-credentials",
    "The email or password is invalid.",
    ErrorType.Unauthorized);
```

По ответу нельзя определить, зарегистрирован ли конкретный email. Это уменьшает количество информации, которую API раскрывает потенциальному атакующему.

После успешной проверки создаётся пара токенов, а hash refresh token сохраняется в PostgreSQL.

### Access token и асимметричная подпись

Access token - это JWT с информацией о пользователе:

```csharp
var claims = new[]
{
    new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
    new Claim(JwtRegisteredClaimNames.Email, user.Email),
    new Claim(JwtRegisteredClaimNames.Name, user.DisplayName),
    new Claim("client_id", _options.ClientId),
    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
    new Claim(JwtRegisteredClaimNames.Iat,
        EpochTime.GetIntDate(now).ToString(),
        ClaimValueTypes.Integer64)
};
```

Токен подписывается алгоритмом `RS256`:

```csharp
_signingCredentials = new SigningCredentials(
    new RsaSecurityKey(_privateKey),
    SecurityAlgorithms.RsaSha256);
```

Для подписи используется приватный RSA-ключ, а для проверки - публичный. В отличие от симметричного секрета, публичный ключ можно безопасно передать другим сервисам, которым необходимо валидировать токен. При этом выпустить новый токен они не смогут: приватного ключа у них нет.

JWT содержит `issuer`, `audience`, время начала действия и время окончания. При каждом запросе `JwtBearer` проверяет:

- подпись токена;
- `issuer`;
- `audience`;
- срок действия;
- наличие обязательного времени истечения.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidIssuer = jwtOptions.Issuer,
    ValidateAudience = true,
    ValidAudience = jwtOptions.Audience,
    ValidateIssuerSigningKey = true,
    IssuerSigningKey = new RsaSecurityKey(publicKey),
    ValidateLifetime = true,
    RequireExpirationTime = true,
    RequireSignedTokens = true,
    ClockSkew = TimeSpan.FromMinutes(1)
};
```

Минута `ClockSkew` оставляет небольшой допуск на рассинхронизацию часов между сервисами.

### Конфигурация JWT

Параметры токенов описаны в `JwtOptions`:

```json
"Jwt": {
  "Issuer": "JardiTips",
  "Audience": "JardiTips.Api",
  "ClientId": "JardiTips.WebApi",
  "AccessTokenLifetimeMinutes": 15,
  "RefreshTokenLifetimeDays": 30
}
```

Приватный и публичный ключи намеренно не лежат в `appsettings.json`. Их необходимо передавать через User Secrets, переменные окружения или внешний secret storage:

```text
Jwt:PrivateKeyPem
Jwt:PublicKeyPem
```

При старте приложения конфигурация проверяется через `ValidateOnStart()`. Если отсутствует `Issuer`, `Audience`, `ClientId` или один из RSA-ключей, приложение сразу завершит запуск с понятной ошибкой.

Это лучше, чем узнать об отсутствующем ключе при первом пользовательском запросе.

### Refresh token

Refresh token не является JWT и не содержит пользовательских данных. Это 64 случайных байта, сгенерированных криптографическим генератором:

```csharp
public GeneratedToken CreateRefreshToken()
{
    var bytes = RandomNumberGenerator.GetBytes(64);
    var token = Base64UrlEncoder.Encode(bytes);

    return new GeneratedToken(
        token,
        DateTime.UtcNow.AddDays(_options.RefreshTokenLifetimeDays));
}
```

В базу сохраняется не сам refresh token, а его SHA-256 hash:

```csharp
public string HashRefreshToken(string refreshToken)
{
    return Convert.ToHexString(
        SHA256.HashData(Encoding.UTF8.GetBytes(refreshToken)));
}
```

Это тот же принцип, что и с паролем: утечка базы не должна автоматически превращаться в набор готовых токенов для входа. При обновлении токена API хеширует полученное значение и ищет hash в базе.

```csharp
public class RefreshTokenEntity : BaseEntity
{
    public Guid UserId { get; set; }

    public string Token { get; set; }

    public DateTime? RevokedAt { get; set; }

    public DateTime ExpiresAt { get; set; }
}
```

### Ротация refresh token

При каждом вызове `/auth/refresh` старый refresh token отзывается и создаётся новый. Это называется ротацией токенов.

```csharp
if (storedToken is null ||
    storedToken.RevokedAt is not null ||
    storedToken.ExpiresAt <= now)
{
    return Result<AuthTokenDto>.Failure(InvalidRefreshToken());
}

storedToken.RevokedAt = now;

var accessToken = tokenService.CreateAccessToken(user);
var refreshToken = tokenService.CreateRefreshToken();

await tokenRepository.AddAsync(new RefreshTokenEntity
{
    UserId = user.Id,
    Token = tokenService.HashRefreshToken(refreshToken.Value),
    CreatedAt = now,
    ExpiresAt = refreshToken.ExpiresAt
}, ct);
```

Вся операция выполняется в транзакции. Отзыв старого и сохранение нового токена должны произойти вместе. Иначе при ошибке между этими действиями можно получить два активных токена или, наоборот, оставить пользователя без возможности обновить сессию.

Повторно использовать старый refresh token уже нельзя: он получит `RevokedAt` и следующий запрос завершится с `401 Unauthorized`.

### Logout

JWT access token нельзя удалить на сервере: после выпуска он остаётся валидным до истечения срока действия. Поэтому logout отзывает refresh token и запрещает продлевать сессию.

```csharp
if (storedToken is null || storedToken.RevokedAt is not null)
    return Result.Success();

storedToken.RevokedAt = DateTime.UtcNow;
await unitOfWork.SaveChangesAsync(ct);
```

Logout сделан идемпотентным. Если токен уже отозван или не существует, API всё равно возвращает успешный ответ. Для клиента важен результат - сессия больше не может быть продлена, - а не предыдущее состояние записи в базе.

Access token при этом может прожить ещё максимум 15 минут. Если требуется мгновенная блокировка, понадобится отдельный blacklist, проверка версии сессии или более короткое время жизни access token. У каждого из этих вариантов есть своя стоимость.

### Защита endpoint-ов по умолчанию

Вместо добавления `.RequireAuthorization()` к каждому endpoint-у используется fallback policy:

```csharp
services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());
```

Такой подход безопаснее по умолчанию. Новый endpoint автоматически будет закрыт, даже если разработчик или AI-агент забудет явно добавить авторизацию.

Открытые endpoint-ы необходимо обозначить явно:

```csharp
var group = builder
    .MapGroup("/auth")
    .WithTags("Auth")
    .AllowAnonymous();
```

Это важная инверсия: мы не пытаемся вспомнить, что нужно защитить, а явно перечисляем немногочисленные публичные маршруты.

### `401`, `403` и ProblemDetails

Для неаутентифицированного запроса API возвращает `401 Unauthorized`, а для аутентифицированного пользователя без необходимых прав - `403 Forbidden`.

Оба ответа формируются в едином формате `ProblemDetails`:

```json
{
  "type": "/errors/unauthorized",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Authentication is required or the provided token is invalid.",
  "instance": "/api/categories",
  "code": "unauthorized"
}
```

Для ошибок внутри login/refresh в существующий `Result Pattern` добавлен тип `ErrorType.Unauthorized`, который также маппится в `401`.

В результате ошибки аутентификации выглядят одинаково независимо от того, где они возникли: в JWT middleware или в обработчике команды.

### Swagger

Для ручной проверки защищённых endpoint-ов в OpenAPI добавлена Bearer security scheme. В Swagger UI можно нажать `Authorize` и указать:

```text
Bearer <access_token>
```

После этого Swagger будет добавлять заголовок `Authorization` к запросам. Без такой настройки JWT-аутентификация работает, но проверять её через UI довольно неудобно.

### AI-инструкции

Изменения в ветке дополняют `.github/copilot-instructions.md` правилами для входных DTO: positional records, `[property: ...]` и `StringLength`. Благодаря этому `LoginDto` и `RefreshTokenDto` следуют тем же правилам, что и остальные команды проекта.

Но правила самой аутентификации также важно зафиксировать перед дальнейшей генерацией кода:

```markdown
## Authentication rules

- All API endpoints are protected by the fallback authorization policy.
- Mark only intentionally public endpoint groups with `AllowAnonymous()`.
- Never store raw passwords or refresh tokens.
- Create access tokens through `ITokenService`; do not sign JWTs in handlers.
- Rotate refresh tokens inside a transaction and reject expired or revoked tokens.
- Return the same `invalid-credentials` response for an unknown email and a wrong password.
- Keep private signing keys outside tracked configuration files.
```

В агентском программировании безопасность тоже масштабируется из шаблона. Если разрешить публичный доступ по умолчанию или однажды сохранить refresh token в открытом виде, агент начнёт аккуратно воспроизводить эту ошибку во всех следующих сценариях.

## Болтовня

На первый взгляд JWT-аутентификация выглядит просто: проверили пароль, создали токен, добавили `UseAuthentication()` - готово. Но основная сложность находится не в выпуске JWT, а вокруг него.

Нужно решить, где хранить ключи, сколько живёт access token, как продлевать сессию, можно ли повторно использовать refresh token, что происходит при logout и какие endpoint-ы считаются публичными.

Мне нравится fallback policy, потому что она меняет цену ошибки. Если мы забудем что-то указать, новый endpoint окажется закрыт, а не случайно опубликован наружу. Клиент заметит `401`, и мы быстро исправим конфигурацию. Обратную ошибку можно не заметить очень долго.

При этом текущая реализация - только фундамент. Здесь ещё нет регистрации, восстановления пароля, подтверждения email, ролей, permissions и входа через внешние провайдеры. `UserLoginEntity` лишь подготавливает базу под OAuth.

Есть и более мелкий, но важный момент: email ищется без учёта регистра, а уникальный индекс PostgreSQL на строковом поле по умолчанию регистр учитывает. Перед добавлением регистрации стоит выбрать единую стратегию: хранить `NormalizedEmail`, использовать `citext` или создать функциональный уникальный индекс по `lower(email)`.

Отдельный вопрос - хранение токенов на клиенте. Для браузерного приложения refresh token обычно лучше передавать через защищённую `HttpOnly` cookie, чтобы JavaScript не имел к нему прямого доступа. Текущий API возвращает оба токена в JSON, что удобно для первого этапа и ручного тестирования, но клиентскую модель угроз всё равно придётся разобрать отдельно.

Авторизация по ролям и permissions появится позже. Но теперь у нас уже есть главное: API умеет надёжно установить личность пользователя, защитить маршруты по умолчанию и безопасно продлевать сессию через ротацию refresh token.

---

P.S. Хочу вставить свои три копейки 😇. Статья полностью написана моделью "GPT-5.6 Sol High" на основе моих предыдущих статей с просьбой сохранить авторский стиль. И, в принципе, получилось очень даже неплохо. Вручную я внёс лишь минимальные корректировки.

Почему написал не сам?

Во-первых, хотелось протестировать новую модель в реальных условиях.

Во-вторых, сами аутентификация и авторизация тоже были почти полностью реализованы AI-агентом. Да, с использованием ранее добавленных скиллов. Да, не всё было идеально: приходилось вносить корректировки и, например, просить добавить возможность использовать заголовок `Authorization` в запросах через Swagger UI.

В-третьих, аутентификация пока реализована не до конца и, так сказать, ещё не готова к боевому запуску. Я хотел написать статью, когда будет добавлен вход через внешних провайдеров. Ну и регистрация, в конце концов 😁.

В целом, это был интересный эксперимент, но в дальнейшем статьи планирую писать сам 🤞.

#### Спасибо! Улыбаемся и пашем! 🚀