+++
date = '2026-07-22T00:00:00+02:00'
title = 'Agentic Coding. Part 2.3: Authentication & Authorization'
tags = ["best-practices", "software-design", "dotnet", "jardi-tips"]
author = ["GPT-5.6 Sol High", "Aleksandr T."]
+++

### Hello there! 🖖

### Overview

* **Ecosystem:** .NET 10
* **Technologies:** ASP.NET Core Web API, JWT, PostgreSQL, Entity Framework Core
* **Improvement:** Authentication with `Access Token` and `Refresh Token`, global authorization policy

The API now has three endpoints:

```text
POST /api/auth/login   - login with email and password
POST /api/auth/refresh - refresh the token pair
POST /api/auth/logout  - revoke the refresh token
```

All other endpoints require an authenticated user by default:

```csharp
services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());
```

To access protected endpoints, the client sends a JWT in the header:

```http
Authorization: Bearer <access_token>
```

You can see the full implementation in commit [`b5d4260`](https://github.com/AleksandrTurkin/api-jardi-tips/commit/b5d4260d81b85879402f997f56376bbd12b318d2).

The main parts of the implementation:

- [AuthEndpoint](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Endpoints/AuthEndpoint.cs) - login, token refresh, and token revocation endpoints.
- [AuthenticationExtensions](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.WebApi/JardiTips.WebApi/Extensions/AuthenticationExtensions.cs) - registration of JWT authentication and the global authorization policy.
- [JwtTokenService](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Infrastructure/Authentication/JwtTokenService.cs) - creation of access tokens and refresh tokens.
- [LoginCommandHandler](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Application/Features/Authentication/LoginCommandHandler.cs) - login and password verification.
- [RefreshTokenCommandHandler](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Application/Features/Authentication/RefreshTokenCommandHandler.cs) - refresh token rotation.
- [RevokeRefreshTokenCommandHandler](https://github.com/AleksandrTurkin/api-jardi-tips/blob/main/JardiTips.Application/Features/Authentication/RevokeRefreshTokenCommandHandler.cs) - refresh token revocation during logout.

## Step-by-Step Implementation

First, let's separate two concepts that are often mixed up.

**Authentication** answers the question: "Who are you?" The user sends an email and password, and the API checks them and issues tokens.

**Authorization** answers the question: "What are you allowed to do?" The current implementation is still basic: the user is either authenticated and gets access to the API, or not. Roles, permissions, and separate policies are not implemented yet.

So, in this part, we implement full JWT authentication and prepare the foundation for future authorization.

### General Flow

The flow looks like this:

```text
1. The client sends an email and password to POST /api/auth/login.
2. The API verifies the password and returns an access token + refresh token.
3. The client sends the access token in Authorization: Bearer <token>.
4. The access token expires after 15 minutes.
5. The client sends the refresh token to POST /api/auth/refresh.
6. The API revokes the old refresh token and returns a new token pair.
7. During logout, the client sends the refresh token to POST /api/auth/logout.
```

The access token has a short lifetime and is used with every request. The refresh token lives longer and is used only to get a new token pair.

```json
{
  "accessToken": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "RfGJmK8...",
  "accessTokenExpiresAt": "2026-07-22T12:15:00Z",
  "refreshTokenExpiresAt": "2026-08-21T12:00:00Z",
  "tokenType": "Bearer"
}
```

### User Model

To store users, we add two entities: `UserEntity` and `UserLoginEntity`.

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

`PasswordHash` is nullable for a reason. A user can sign in not only with a password, but also through an external provider, such as Google, GitHub, or Microsoft. A separate entity is prepared for links with such providers:

```csharp
public class UserLoginEntity : BaseEntity
{
    public Guid UserId { get; set; }

    public string LoginProvider { get; set; }

    public string ProviderKey { get; set; }

    public UserEntity User { get; set; }
}
```

The `(LoginProvider, ProviderKey)` pair has a unique index. One external account cannot be linked to several users at the same time.

External OAuth login is not implemented in this part yet. For now, `UserLoginEntity` only prepares the data model for the next step.

### Login with Email and Password

Login starts with a simple DTO that uses the [validation](https://lmcorner.net/posts/coding-validation/) we already know:

```csharp
public record LoginDto(
    [property: Required, EmailAddress, StringLength(320)] string Email,
    [property: Required, StringLength(200)] string Password);
```

Of course, the password itself is not stored in the database. To verify it, we use the standard `PasswordHasher<TUser>` from `Microsoft.AspNetCore.Identity`:

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

In `LoginCommandHandler`, the email is normalized. Then the user is loaded from the database, and the password is verified through `IPasswordService`.

```csharp
var normalizedEmail = command.Login.Email.Trim().ToLowerInvariant();
var user = await userRepository.FirstOrDefaultAsync(
    x => x.Email.ToLower() == normalizedEmail,
    ct);

if (user?.PasswordHash is null || !passwordService.VerifyPassword(user, command.Login.Password))
    return InvalidCredentials();
```

It is important that the API returns the same error for both an unknown email and a wrong password:

```csharp
new ErrorDetail(
    "invalid-credentials",
    "The email or password is invalid.",
    ErrorType.Unauthorized);
```

The response does not show whether a specific email is registered. This reduces the amount of information that the API exposes to a potential attacker.

After a successful check, a token pair is created, and the refresh token hash is saved in PostgreSQL.

### Access Token and Asymmetric Signature

An access token is a JWT with information about the user:

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

The token is signed with the `RS256` algorithm:

```csharp
_signingCredentials = new SigningCredentials(
    new RsaSecurityKey(_privateKey),
    SecurityAlgorithms.RsaSha256);
```

The private RSA key is used for signing, and the public key is used for verification. Unlike a symmetric secret, the public key can be safely shared with other services that need to validate the token. At the same time, they cannot issue a new token because they do not have the private key.

The JWT contains the `issuer`, `audience`, start time, and expiration time. For every request, `JwtBearer` checks:

- the token signature;
- `issuer`;
- `audience`;
- the lifetime;
- the required expiration time.

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

One minute of `ClockSkew` gives a small tolerance for clock differences between services.

### JWT Configuration

Token settings are described in `JwtOptions`:

```json
"Jwt": {
  "Issuer": "JardiTips",
  "Audience": "JardiTips.Api",
  "ClientId": "JardiTips.WebApi",
  "AccessTokenLifetimeMinutes": 15,
  "RefreshTokenLifetimeDays": 30
}
```

The private and public keys are intentionally not stored in `appsettings.json`. They should be passed through User Secrets, environment variables, or an external secret storage:

```text
Jwt:PrivateKeyPem
Jwt:PublicKeyPem
```

When the application starts, the configuration is checked through `ValidateOnStart()`. If `Issuer`, `Audience`, `ClientId`, or one of the RSA keys is missing, the application immediately stops with a clear error.

This is better than finding out about a missing key during the first user request.

### Refresh Token

A refresh token is not a JWT and does not contain user data. It is made of 64 random bytes generated by a cryptographic random number generator:

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

The database stores the SHA-256 hash of the refresh token, not the token itself:

```csharp
public string HashRefreshToken(string refreshToken)
{
    return Convert.ToHexString(
        SHA256.HashData(Encoding.UTF8.GetBytes(refreshToken)));
}
```

This is the same principle as with a password: a database leak should not automatically provide a set of ready-to-use login tokens. During token refresh, the API hashes the received value and looks for the hash in the database.

```csharp
public class RefreshTokenEntity : BaseEntity
{
    public Guid UserId { get; set; }

    public string Token { get; set; }

    public DateTime? RevokedAt { get; set; }

    public DateTime ExpiresAt { get; set; }
}
```

### Refresh Token Rotation

On every `/auth/refresh` call, the old refresh token is revoked and a new one is created. This is called token rotation.

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

The whole operation runs inside a transaction. Revoking the old token and saving the new one must happen together. Otherwise, an error between these actions can leave two active tokens or, on the other hand, leave the user unable to refresh the session.

The old refresh token cannot be used again. It gets `RevokedAt`, and the next request ends with `401 Unauthorized`.

### Logout

A JWT access token cannot be deleted on the server. After it is issued, it stays valid until it expires. Because of this, logout revokes the refresh token and prevents the session from being extended.

```csharp
if (storedToken is null || storedToken.RevokedAt is not null)
    return Result.Success();

storedToken.RevokedAt = DateTime.UtcNow;
await unitOfWork.SaveChangesAsync(ct);
```

Logout is idempotent. If the token is already revoked or does not exist, the API still returns a successful response. For the client, the result is important - the session can no longer be extended - not the previous state of the database record.

The access token can still live for up to 15 minutes. If immediate blocking is required, we need a separate blacklist, a session version check, or a shorter access token lifetime. Each option has its own cost.

### Protecting Endpoints by Default

Instead of adding `.RequireAuthorization()` to every endpoint, we use a fallback policy:

```csharp
services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());
```

This approach is secure by default. A new endpoint is automatically protected, even if a developer or an AI agent forgets to add authorization explicitly.

Public endpoints must be marked explicitly:

```csharp
var group = builder
    .MapGroup("/auth")
    .WithTags("Auth")
    .AllowAnonymous();
```

This is an important inversion. We do not try to remember what must be protected. Instead, we explicitly list the few public routes.

### `401`, `403`, and ProblemDetails

For an unauthenticated request, the API returns `401 Unauthorized`. For an authenticated user without the required permissions, it returns `403 Forbidden`.

Both responses use the same `ProblemDetails` format:

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

For errors inside login/refresh, the `ErrorType.Unauthorized` type is added to the existing `Result Pattern`. It is also mapped to `401`.

As a result, authentication errors look the same no matter where they happen: in the JWT middleware or in the command handler.

### Swagger

For manual testing of protected endpoints, a Bearer security scheme is added to OpenAPI. In Swagger UI, we can click `Authorize` and enter:

```text
Bearer <access_token>
```

After that, Swagger adds the `Authorization` header to requests. JWT authentication works without this setting, but testing it through the UI is not very convenient.

### AI Instructions

The changes in the branch add rules for input DTOs to `.github/copilot-instructions.md`: positional records, `[property: ...]`, and `StringLength`. Because of this, `LoginDto` and `RefreshTokenDto` follow the same rules as other commands in the project.

But it is also important to record the authentication rules before generating more code:

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

In agentic coding, security also scales from the template. If we allow public access by default or store a refresh token in plain text once, the agent starts carefully reproducing this mistake in all future scenarios.

## Blabber

At first glance, JWT authentication looks simple: verify the password, create a token, add `UseAuthentication()`, and we are done. But the main complexity is not in issuing the JWT. It is everything around it.

We need to decide where to store the keys, how long the access token lives, how to extend a session, whether a refresh token can be reused, what happens during logout, and which endpoints are public.

I like the fallback policy because it changes the cost of a mistake. If we forget to specify something, the new endpoint stays protected instead of being accidentally exposed. The client sees `401`, and we quickly fix the configuration. The opposite mistake can stay unnoticed for a very long time.

At the same time, the current implementation is only a foundation. It does not have registration, password recovery, email confirmation, roles, permissions, or login through external providers yet. `UserLoginEntity` only prepares the database for OAuth.

There is also a smaller but important point: email lookup is case-insensitive, but a PostgreSQL unique index on a string field is case-sensitive by default. Before adding registration, we should choose one strategy: store `NormalizedEmail`, use `citext`, or create a functional unique index on `lower(email)`.

Another question is token storage on the client. For a browser application, it is usually better to send the refresh token through a secure `HttpOnly` cookie, so JavaScript does not have direct access to it. The current API returns both tokens in JSON. This is convenient for the first step and manual testing, but we still need to review the client threat model separately.

Role-based authorization and permissions will come later. But now we already have the main part: the API can reliably identify the user, protect routes by default, and safely extend the session through refresh token rotation.

---

P.S. I want to add my two cents 😇. The article was written entirely by the "GPT-5.6 Sol High" model. It used my previous articles and a request to keep my writing style. And, in general, the result is quite good. I made only a few small changes manually.

Why did I not write it myself?

First, I wanted to test the new model in real conditions.

Second, the authentication and authorization implementation itself was also written almost entirely by an AI agent. Yes, it used the AI skills added earlier. Yes, not everything was perfect. I had to make corrections and, for example, ask it to add support for the `Authorization` header in Swagger UI requests.

Third, authentication is not fully implemented yet and is not ready for production, so to speak. I wanted to write the article after login through external providers was added. And registration too, after all 😁.

Overall, it was an interesting experiment, but I plan to write future articles myself 🤞.

#### Thanks! Keep calm and code on! 🚀