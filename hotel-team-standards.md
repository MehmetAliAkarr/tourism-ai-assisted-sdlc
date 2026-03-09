# Setur Tourism — Hotel Team Development Standards & Tech Stack

> **Generated**: March 7, 2026
> **Purpose**: Agent-consumable standards reference for AI-assisted code generation and task decomposition
> **Scope**: Development standards applied across Hotel domain services
> **Azure DevOps Project**: SeturTourism (`seturcloud.visualstudio.com`)

---

## Table of Contents

1. [Tech Stack Summary](#1-tech-stack-summary)
2. [Architecture Patterns](#2-architecture-patterns)
3. [Project Structure Convention](#3-project-structure-convention)
4. [API Standards](#4-api-standards)
5. [Service Layer Standards](#5-service-layer-standards)
6. [Data Access Patterns](#6-data-access-patterns)
7. [Caching Strategy](#7-caching-strategy)
8. [Validation](#8-validation)
9. [Middleware Pipeline](#9-middleware-pipeline)
10. [Configuration Management](#10-configuration-management)
11. [Messaging & Event Bus](#11-messaging--event-bus)
12. [Git & Branching Strategy](#12-git--branching-strategy)
13. [CI/CD Pipeline](#13-cicd-pipeline)
14. [Security](#14-security)
15. [Testing Standards](#15-testing-standards)
16. [Logging & Monitoring](#16-logging--monitoring)
17. [HTTP Client Patterns](#17-http-client-patterns)
18. [Error Handling](#18-error-handling)
19. [Performance Tuning](#19-performance-tuning)
20. [Shared NuGet Packages](#20-shared-nuget-packages)

---

## 1. Tech Stack Summary

| Category | Technology | Version / Details |
|---|---|---|
| **Runtime** | .NET 8.0 | `<TargetFramework>net8.0</TargetFramework>` (LTS) |
| **Web Framework** | ASP.NET Core | Minimal hosting model (`Program.cs`) or `Host.CreateDefaultBuilder` + `Startup.cs` |
| **ORM / Data Access** | Dapper | 2.1.44 + Dapper.Contrib — micro-ORM with raw parameterized SQL. **NOT Entity Framework** in hotel services |
| **Primary Database** | SQL Server | MSSQL via `Microsoft.Data.SqlClient` — shared `TourismDb` for hotels, rooms, images, providers |
| **Secondary Database** | PostgreSQL | Npgsql 8.0.3 — used by Hotel Price (price CRUD) and channel managers (logging/data) |
| **Document Store** | MongoDB | ScaleGrid hosted — used by BFF for reservations, wishlists, Firebase tokens |
| **Distributed Cache** | Redis | StackExchange.Redis + RedLock.net — distributed caching and locking |
| **Search Engine** | Azure Cognitive Search | REST API — hotel metadata, URL resolution, autocomplete |
| **Message Broker (Primary)** | Azure Service Bus | MassTransit 8.3.6–8.5.2 — event-driven communication between services |
| **Message Broker (Secondary)** | RabbitMQ / CloudAMQP | Used by `tourism-setur-context` for hotel metadata events |
| **Secrets Management** | Azure Key Vault | `ClientSecretCredential` or `DefaultAzureCredential` fallback to `AzureCliCredential` |
| **Observability** | Elastic APM | `Elastic.Apm.NetCoreAll` — HTTP, SQL, ASP.NET Core diagnostics |
| **Logging** | Setur.Logger.WebLogger | Custom logging framework → Elasticsearch |
| **Serialization** | Newtonsoft.Json | 13.0.3 — UTC dates, ISO format, `StringEnumConverter`, `CamelCaseNamingStrategy`, `NullValueHandling.Ignore` |
| **API Documentation** | Swashbuckle | AspNetCore 6.6.2 — Swagger + ReDoc |
| **Validation** | FluentValidation | Auto-registered validators with Turkish error messages |
| **Mapping** | AutoMapper | Ordered profile registration via `IOrderedMapperProfile` |
| **HTTP Client (Internal)** | Setur.HttpClientHelper | 1.0.9 — `ISeturHttpClient` wraps `HttpClient` with project token headers and logging |
| **HTTP Client (Typed)** | Refit | 8.0.0 — Type-safe REST clients for Azure Search and Basket API |
| **Rate Limiting** | AspNetCoreRateLimit | IP/ClientId-based throttling with endpoint whitelisting |
| **Memory** | RecyclableMemoryStream | Microsoft.IO 3.0.0 — optimized memory allocation |
| **Phone Validation** | libphonenumber-csharp | 8.13.55 |
| **CMS** | Kontent.Ai.Delivery | 17.9.0 — Kentico headless CMS integration |
| **CDN** | Azure CDN + Cloudinary | Image URL rewriting |

---

## 2. Architecture Patterns

### 2.1 Clean Architecture (All Hotel Services)

Every hotel service follows Clean Architecture with this layer structure:

```
src/
├── Core/
│   ├── {Service}.Domain/          # Domain entities, value objects
│   ├── {Service}.Application/     # Business logic, interfaces, DTOs, configs, MassTransit consumers
│   └── {Service}.Shared/          # Shared utilities, Result<T> pattern
├── Infrastructure/
│   ├── {Service}.Persistence/     # Dapper repos, ConnectionFactory, raw SQL query classes
│   └── {Service}.ExternalServices/ # HTTP clients to downstream services
├── Presentation/
│   ├── {Service}.API/             # ASP.NET Core Web API (main entry point)
│   └── {Service}.Jobs/            # Background Workers / WebJobs
├── Migrated/                       # Legacy NuGet package models (netstandard2.0/2.1)
└── tests/
    └── {Service}.Test/            # xUnit/NUnit + Moq
```

**Dependency direction**: Presentation → Application → Domain (Infrastructure implements Application interfaces)

### 2.2 BFF Pattern (Backend for Frontend)

The BFF (`b2cbackendforfront-api-svc`) is the single gateway for all B2C web and mobile traffic. It orchestrates across 24+ downstream HTTP microservices. No other client directly calls domain services except through the BFF.

**BFF project structure differs from domain services:**
```
src/
├── Libraries/
│   ├── Setur.Tourism.B2C.Core/     # Domain models, caching, config, utilities
│   ├── Setur.Tourism.B2C.Service/  # Business logic (40+ services)
│   └── Setur.Tourism.B2C.Data/     # MongoDB repositories
└── Presentation/
    ├── Setur.Tourism.B2C.Api/      # 26 controllers, models, validators, helpers
    ├── Setur.Tourism.B2C.Api.Core/ # BaseController, middleware, filters, mappers
    └── Setur.Tourism.B2C.Api.Mobile/ # Mobile-specific API
```

### 2.3 CQRS via MediatR

Used by channel manager services (Elektra, HotelRunner, Reseliva, HMS, Tatilbudur, Akgunler):
- **Queries**: `GetRoomListQuery`, `GetRoomInventoryQuery`, etc.
- **Commands**: `UpdateRoomInventoryCommand`, `PushReservationCommand`, `ConfirmCommand`, etc.
- All handlers registered via MediatR DI

### 2.4 Repository Pattern with Dapper

All hotel services use the same data access pattern:
- `IConnectionFactory` → `ConnectionFactory` (creates `SqlConnection` or `NpgsqlConnection`)
- `I{Entity}Repository` → `{Entity}Repository` (Dapper queries via static `{Entity}Queries` class)
- Raw parameterized SQL stored in static classes (e.g., `HotelQueries.cs`, `PriceQueries.cs`)
- Command timeout: 60 seconds default

### 2.5 Outbox Pattern

Sales Service uses SQL outbox table for reliable event publishing:
```
Sales DB (outbox table) → SaleReservationPublisher reads → publishes to ASB per ChannelManagerType
```

### 2.6 Result Pattern

Services return `Result<T>` wrapper for operation outcomes, enabling standardized success/failure handling.

---

## 3. Project Structure Convention

### 3.1 Solution Layout

```
{repo-name}/
├── src/
│   ├── Core/
│   │   ├── {Service}.Domain/
│   │   ├── {Service}.Application/
│   │   ├── {Service}.Shared/
│   │   └── {Service}.SharedModels/
│   ├── Infrastructure/
│   │   ├── {Service}.Persistence/
│   │   └── {Service}.ExternalServices/     # or individual service projects
│   ├── Presentation/
│   │   ├── {Service}.API/
│   │   └── {Service}.{JobName}Job/         # WebJobs / background workers
│   └── Migrated/
│       └── {Service}.Models/               # NuGet package source (netstandard2.0)
├── tests/
│   └── {Service}.Test/
├── NuGet.Config
├── {Solution}.sln
├── azure-pipelines.yml
├── Dockerfile
└── helm-values.yml
```

### 3.2 DI Registration Convention

Each layer exposes an `AddX()` extension method in `ServiceCollectionExtensions.cs`:
```csharp
// Presentation layer composes all:
builder.Services.AddDependencies(configuration)
    ├── Configure<AppSettings>(configuration)
    ├── AddPersistance()          // Infrastructure: IConnectionFactory, repositories
    ├── AddExternalServices()     // Infrastructure: HTTP service clients
    ├── AddApplication()          // Core: business services
    ├── AddStackExchangeRedisCache(...)
    ├── AddMassTransit(...)
    ├── AddApiVersioning(...)
    └── AddSwagger(...)
```

### 3.3 Package Sources

```xml
<!-- NuGet.Config -->
<packageSources>
    <add key="nuget" value="https://api.nuget.org/v3/index.json" />
    <add key="SeturfeedV3" value="https://seturcloud.pkgs.visualstudio.com/_packaging/SeturFeed/nuget/v3/index.json" />
</packageSources>
```

---

## 4. API Standards

### 4.1 BaseController

All controllers inherit from a base controller:
```csharp
[ApiController]
[Route("[controller]")]
[Route("v{version:apiVersion}/[controller]")]
[Consumes("application/json")]
[Produces("application/json")]
[ProducesResponseType(typeof(ResponseErrorViewModel), 400)]
[ProducesResponseType(typeof(ResponseErrorViewModel), 500)]
public class BaseController : Controller
{
    public IActionResult Error(ResponseStatus status, string message, string code = null);
}
```

### 4.2 API Versioning

- URL segment versioning: `v{version:apiVersion}` (e.g., `/api/v1.1/hotels/search`)
- Supported versions: v1.0, v1.1, v2.0, v3.0, v4.0
- Declared via `[ApiVersion("1.1")]` attribute on controllers
- Package: `Asp.Versioning.Mvc`
- Old routes preserved for backward compatibility (V1.0 at `/`, V1.1 at `api/v{version}/hotels`)

### 4.3 Endpoint Convention

- **All endpoints use `[HttpPost]`** — even read-only search operations use POST due to complex request bodies
- Exceptions: health checks (`GET /healthcheck`), some log endpoints (`GET`)
- Channel manager endpoints accept XML for inventory operations, JSON for reservation push

### 4.4 Swagger Documentation

```csharp
[SwaggerResponse(200, Type = typeof(HotelListViewModel))]
[SwaggerOperation(Summary = "...", Description = "...", OperationId = "listHotels")]
```
- Bearer token security configured in `SwaggerGenConfigureOptions`
- ReDoc alternative UI enabled

### 4.5 Response Envelope

BFF uses `Response<T>` wrapper:
```csharp
public class Response<T>
{
    public ResponseStatus Status { get; set; }  // Success, Error
    public T Data { get; set; }
    public string Message { get; set; }
}
```

Channel manager integration services use `ApiResponse<T>` with `ReturnSuccess()` / `ReturnError()` helpers from `BaseController<T>`.

---

## 5. Service Layer Standards

### 5.1 Return Types

```csharp
// BFF services
public async Task<Response<(IPagedList<HotelListModel> Hotels, IList<HotelListFiltersModel> Filters)>>
    List(HotelListSearchModel model, int uniqueId, int pageIndex = 0, int pageSize = int.MaxValue)

// Domain services
public async Task<Result<SearchResponseModel>> Search(SearchRequestModel model)
```

### 5.2 Async-First Design

All service methods are `async Task<T>`. No synchronous I/O operations.

### 5.3 Dependency Injection

- **Constructor injection only** — no service locator pattern
- Interface-driven: every service registered as `IService → ServiceImpl`
- Lifetime: `Scoped` for most services and repositories
- Heavy injection (10–13 dependencies per controller is common in BFF)

### 5.4 CacheKey Pattern

```csharp
// BFF uses composite cache keys
var cacheKey = new CacheKey("hotel:list:{0}:{1}:{2}:{3}:{4}",
    dates, locationId, guests, loyaltyHash, projectToken);
var result = await _cacheManager.GetAsyncCompressed<T>(cacheKey, async () => { /* load */ });
```

---

## 6. Data Access Patterns

### 6.1 Dapper + ConnectionFactory

```csharp
// IConnectionFactory
public interface IConnectionFactory
{
    Task<NpgsqlConnection> CreatePgConnectionAsync();   // PostgreSQL — eager open
    Task<SqlConnection> CreateSqlConnectionAsync();      // SQL Server — eager open
}

// Repository usage
public async Task<IEnumerable<HotelPriceModel>> GetHotelPrices(PriceSearchRequestModel request)
{
    await using var connection = await _connectionFactory.CreatePgConnectionAsync();
    return await connection.QueryAsync<HotelPriceModel>(
        PriceQueries.GetHotelPrices,
        new { request.HotelId, request.CheckIn, request.CheckOut },
        commandTimeout: 60);
}
```

### 6.2 SQL Query Classes

Raw SQL stored in static classes with parameterized queries:
```csharp
public static class HotelQueries
{
    public const string GetHotelsByCity = @"
        SELECT h.Id, h.Name, h.StarRating, ...
        FROM Hotel h
        WHERE h.CityId = @CityId AND h.IsActive = 1";

    public const string GetMultipleHotelImagesTop10 = @"...";
}
```

### 6.3 Batch Processing Convention

- SQL param batch size: 2000 (Local Hotels)
- Batch updates chunk into 30-day batches processed via `Task.WhenAll`
- `Parallel.ForEach` + `ConcurrentBag<Task>` for price batch updates

### 6.4 Dual-Database Pattern (Hotel Price Service)

Some services use both SQL Server and PostgreSQL:
```
ConnectionFactory
  ├── PgPriceDbConnectionString → PostgreSQL (price CRUD via Dapper)
  └── TourismDbConnectionString → SQL Server (quota logs, project lookups)
```

---

## 7. Caching Strategy

### 7.1 BFF — 3-Tier Cache with Pub/Sub Invalidation

```
Layer 1: Redis (primary distributed cache)
Layer 2: In-Memory cache (15-minute TTL, synchronized via Redis pub/sub)
Layer 3: Per-request cache (HttpContext-scoped)

Pub/Sub Channel: "setur:b2c:cache:mem:invalidation"
```

**`ICacheManager` Interface:**
```csharp
Task<T> GetAsyncCompressed<T>(CacheKey, Func<Task<T>> acquire);  // Get-or-set with GZip
Task<T> GetAsync<T>(CacheKey, Func<T> acquire);
Task SetAsyncCompressed(CacheKey, object data);
Task RemoveAsync(CacheKey, params object[] keys);
Task<bool> IsSetAsync(CacheKey);
```

**Cache TTL Tiers:**

| Tier | Duration | Use Case |
|---|---|---|
| `ShortestTermCacheTime` | 5 min | High-frequency changing data |
| `ShortTermCacheTime` | 30 min | Search results |
| `DefaultCacheTime` | 60 min | Standard data |
| `LongTermCacheTime` | 130 min | Slowly changing data |
| `DailyTermCacheTime` | 1440 min (24h) | Static reference data |

**Fallback Strategy:**
```csharp
if (redisConfig.Enabled && redisConfig.UseCaching)
    → CacheManagerRedis (Redis + Memory + HttpContext)
else
    → CacheManagerMemory (in-process only)
```

### 7.2 Hotel Wrapper — Redis Distributed Cache

```csharp
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = appSettings.AzureRedisCacheConnection;
    options.InstanceName = "HotelWrapper:";
});
```
- `IDistributedCache` for search results and hotel metadata
- Compressed storage for large result sets
- Background worker for proactive cache warming via MassTransit `HotelMetaDataCached` consumer

### 7.3 Hotel Price — NO Caching

**Critical gap**: No Redis, no MemoryCache, no `IDistributedCache` anywhere in DI. Every request hits PostgreSQL directly. Hotel Wrapper caches on its side, but direct callers (Locals, channel managers, Google Hotel Ads, B2E) always hit DB.

### 7.4 Other Services

| Service | Cache |
|---|---|
| Expedia | Azure Redis Cache (`AzureRedisCacheConnection`) |
| Akgunler | Redis via `ICacheHelper` (Tournate session token cache) |
| Tatilbudur | `MemoryCache` for `TatilbudurSecurityKeyCacheKey` |

---

## 8. Validation

### 8.1 FluentValidation (BFF)

Auto-registered from assembly (name starts with "app"):
```csharp
services.AddFluentValidation(config =>
    config.RegisterValidatorsFromAssemblyContaining<Startup>());
```

All validators inherit `AbstractValidator<TViewModel>` with **Turkish error messages**:
```csharp
public class HotelListSearchValidator : AbstractValidator<HotelListSearchModel>
{
    public HotelListSearchValidator()
    {
        RuleFor(m => m.Slug).NotEmpty().WithMessage("Otel url alanı zorunludur.");
        RuleFor(m => m.In)
            .Must(field => !field.HasValue || field.Value.CompareTo(DateTime.Now.AddDays(-1)) > 0)
            .WithMessage("Giriş tarihi bugünden küçük olamaz.");
        RuleFor(m => m.Out)
            .Must((m, field) => !field.HasValue || field.Value > m.In.Value)
            .WithMessage("Çıkış tarihi, giriş tarihinden büyük olmalıdır.");
    }
}
```

Integrated into filter pipeline — returns 400 `ResponseErrorViewModel` before action execution via `ModelStateFilter`.

### 8.2 Validation Helpers

- `RequestHelper.ValidateChildrenBirthDates(string[] rooms)` — parse & validate child dates
- `RequestHelper.ValidateChildsWithAdults(string[] rooms)` — ensure adults present
- `RequestHelper.IsMultipleRoomSearch(string[] rooms)` — detect multi-room searches

### 8.3 Channel Manager Validation

Channel managers like HMS use `ValidationHelper` for inventory updates (date format, price plan currency, quota validation).

---

## 9. Middleware Pipeline

### BFF Middleware Order

```
1. Elastic APM (instruments all diagnostics)
2. ResponseTimeMiddleware (timing via Stopwatch)
3. ForwardedHeaders (X-Forwarded-For, X-Forwarded-Proto)
4. B2CRequestResponseLoggingMiddleware (logs req/response with body, headers, elapsed time)
5. AntiXssMiddleware (XSS prevention)
6. Custom Exception Handler (centralized error handling → ResponseErrorViewModel)
7. Static Files (wwwroot/)
8. CORS
9. Swagger / ReDoc documentation
10. Authentication (JWT validation)
11. Authorization (ApiScope policy)
12. MVC Controllers
```

### Domain Service Middleware (Hotel Wrapper, Hotel Price)

```
1. Elastic APM (UseAllElasticApm with Http, SqlClient, AspNetCore, EfCore, MongoDb subscribers)
2. CustomExceptionMiddleware
3. MapControllers
```

Thread pool tuning happens before middleware:
```csharp
ServicePointManager.DefaultConnectionLimit = isProduction ? 500 : 100;
ThreadPool.SetMinThreads(isProduction ? 250 : 50, isProduction ? 250 : 50);
ServicePointManager.UseNagleAlgorithm = false;
ServicePointManager.Expect100Continue = false;
```

---

## 10. Configuration Management

### 10.1 Configuration Sources (Priority Order)

```
1. appsettings.json                        (base defaults)
2. appsettings.{EnvironmentName}.json     (environment overrides: staging, production, testing, playground, preprod)
3. Environment variables
4. Azure Key Vault                         (secrets — connection strings, API keys, tokens)
```

### 10.2 Azure Key Vault Integration

```csharp
// Try ClientSecretCredential first
var credential = new ClientSecretCredential(tenantId, clientId, clientSecret);
// Fallback to AzureCliCredential for local development
var credential = new AzureCliCredential();

builder.Configuration.AddAzureKeyVault(new Uri(keyVaultUrl), credential);
```

### 10.3 Strongly-Typed AppSettings

All configuration bound to `AppSettings` class via `Configure<AppSettings>()`:

```
AppSettings
├── ProjectConfig: ProjectId, ProjectToken, EmailGroup, DefaultUserId, MajorVersion, MinorVersion
├── CacheConfig: DefaultCacheTime, ShortTermCacheTime, LongTermCacheTime, ShortestTermCacheTime, DailyTermCacheTime
├── RedisConfig: ConnectionString, Enabled, UseCaching
├── LogConfig: ElasticSearch URLs, LogSource
├── ElasticApm: ServerUrl, SecretToken, TransactionSampleRate (0.1), ServiceName
├── IdentityServerConfig: Authority URL
├── AuthConfig: EmployeeEmailDomains (setur.com.tr, setair.com.tr, seturbiz.com, bcdtravel.com.tr)
├── FeatureFlagConfig: Runtime feature toggles (e.g., BypassFraudCheck)
├── MongoDbConfig, AzureKeyVaultConfig, SeturMailConfig, FirebaseConfig
├── RateLimitingConfig, KvkkConfig, OtpConfig, RecaptchaConfig
└── Service URLs: SeturHotelServiceUrl, HotelPriceServiceUrl, BasketApiUrl, etc.
```

### 10.4 Environments

| Environment | Config File | K8s Namespace |
|---|---|---|
| Development | `appsettings.development.json` | local |
| Testing | `appsettings.testing.json` | `.testing.setur.k8s` |
| Playground | `appsettings.playground.json` | `.playground.setur.k8s` |
| Staging / Preprod | `appsettings.staging.json` / `appsettings.preprod.json` | `.staging.setur.k8s` |
| Production | `appsettings.production.json` | `.production.setur.k8s` |

---

## 11. Messaging & Event Bus

### 11.1 MassTransit + Azure Service Bus

```csharp
services.AddMassTransit(x =>
{
    x.UsingAzureServiceBus((context, cfg) =>
    {
        cfg.Host(appSettings.ServiceBusSharedAccess);
        cfg.UseServiceBusMessageScheduler();

        cfg.ReceiveEndpoint(appSettings.QueueName, e =>
        {
            e.PrefetchCount = 1;
            e.ConcurrentMessageLimit = 1;
            e.LockDuration = TimeSpan.FromMinutes(5);
            e.MaxAutoRenewDuration = TimeSpan.FromMinutes(10);
            e.MaxDeliveryCount = 5;
            e.DefaultMessageTimeToLive = TimeSpan.FromDays(7);
            e.EnablePartitioning = false;
            e.EnableBatchedOperations = false;

            e.UseMessageRetry(r => r.Incremental(5, TimeSpan.FromSeconds(5), TimeSpan.FromSeconds(5)));
            e.ConfigureConsumer<MyConsumer>(context);
        });
    });
});
```

### 11.2 Message Types

Common NuGet package: `SeturTourismCommon.EventBus.Messages`

Key message types:
- `HotelMetaDataCached` — triggers cache warming in Hotel Wrapper
- `HotelPriceQuotaChanged` — price/quota change events consumed by Hotel Price
- `PromotionPriceChanged` — promotion changes consumed by Hotel Price
- `{ChannelManager}HotelReservationCreated` — reservation events per channel manager
- `HotelCreated/Updated/Deleted` — hotel metadata events (22 types)

### 11.3 Outbox Pattern (Sales Service)

```
SQL outbox table → SaleReservationPublisher → ASB per ChannelManagerType
```

---

## 12. Git & Branching Strategy

### 12.1 MultiRepo

Every service has its own Git repository in Azure DevOps (`seturcloud.visualstudio.com/SeturTourism`). No monorepo.

### 12.2 GitFlow Branch Model

| Branch | Purpose |
|---|---|
| `master` / `main` | Released production code |
| `development` | Next version in progress |
| `feature-{name}` | Individual feature development (branched from development) |
| `release-{version}` | Release candidates shared with stakeholders |
| `hotfix-{name}` | Patches for released code |

### 12.3 Workflow

1. Create `feature-{name}` branch from `development`
2. Develop, commit with meaningful messages
3. Create Pull Request from `feature-{name}` → `development` in Azure DevOps
4. PR approved → merge to `development`
5. Delete local feature branch (branch prune)
6. Never continue development on a merged feature branch

### 12.4 CI Triggers

Pipelines trigger on push to any branch. Branch filtering in pipeline YAML:
- Fortify (SAST) only runs on: `master`, `main`, `release/master`, `development`
- SonarQube runs on all branches
- Build runs on all branches

---

## 13. CI/CD Pipeline

### 13.1 Pipeline Structure

```yaml
# azure-pipelines.yml — typical structure
resources:
  repositories:
  - repository: templates
    type: git
    name: DevOpsCommon/devops-common    # Shared pipeline templates

stages:
  - stage: Fortify            # SAST security scanning
    jobs:
    - template: fortify/fortify-backend.yml@templates

  - stage: Sonarqube          # Code quality analysis
    jobs:
    - template: sonarqube/sonarqube.yml@templates
      parameters:
        dotnetversion: 8.0.x

  - stage: build              # Docker build + Helm package
    jobs:
    - template: azure-pipeline-templates/templates/temp8.yaml@templates
      parameters:
        serviceName: {service-name}-svc
        projectPath: src/Presentation/{Service}.API
        containerRegistry: seturtourism
```

### 13.2 K8s Deployment Pipeline

```yaml
# azure-pipelines-new-k8s.yml — Helm-based K8s deployment
pool: SeturAzLinuxAgentPool

jobs:
- template: azure-pipeline-templates/net8.0-ubuntu-helm-prisma.yml@templates
  parameters:
    serviceName: {service-name}-svc
    projectPath: src/Presentation/{Service}.API
    containerRegistry: seturtourism
```

### 13.3 Build Agents

| Pool | Purpose |
|---|---|
| `SeturAzLinuxAgentPool` | Linux agents for Docker builds, SonarQube, Helm |
| `SeturFortifyScaBackendAgentPool` | Fortify SAST scanning |
| `AzureAgentPool` | Windows agents (Build Server 1, 2, 10–15) |
| `OnPremiseLinuxAgentPool` | Docker image build & push |

### 13.4 CI/CD Flow

```
Push to any branch
  → Azure Pipelines picks up azure-pipelines.yml
    ├── Fortify SAST (master/development only)
    ├── SonarQube code quality (all branches)
    └── Build
        ├── dotnet restore (NuGet + SeturfeedV3)
        ├── dotnet build --configuration Release
        ├── dotnet test
        ├── dotnet publish
        ├── Docker build
        ├── Docker push → seturtourism container registry
        └── Helm package

Release pipeline (post-build):
  ├── Development (auto-deploy)
  ├── Testing (approval-gated)
  ├── Staging (approval-gated, master only)
  └── Production (approval-gated, master only)
```

### 13.5 Path Filters

BFF pipeline triggers only on changes to:
```yaml
trigger:
  paths:
    include:
    - 'src/Presentation/Setur.Tourism.B2C.Api/**'
    - 'src/Presentation/Setur.Tourism.B2C.Api.Core'
    - 'src/Libraries/**'
    - 'Setur.Tourism.B2C.sln'
    - 'NuGet.Config'
    - 'azure-pipelines.yml'
```

---

## 14. Security

### 14.1 Authentication

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = AppSettings.IdentityServerConfig.Authority; // dsso.setur.com.tr
        options.RequireHttpsMetadata = false;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false
        };
    });

services.AddAuthorization(options =>
{
    options.AddPolicy("ApiScope", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("scope", "openid", "profile");
    });
});
```

### 14.2 Employee Detection

Recognized employee email domains: `@setur.com.tr`, `@setair.com.tr`, `@seturbiz.com`, `@bcdtravel.com.tr`

### 14.3 Channel Manager API Auth

Inbound authentication varies by channel manager:

| Service | Auth Method |
|---|---|
| Elektra | Username/Password headers |
| HotelRunner | Username/Password + SeturAuth headers |
| Reseliva | Username/Password headers |
| HMS | Username/Password headers |
| Akgunler | PublicKey token array with URL whitelisting per consumer |

### 14.4 Anti-XSS

`AntiXssMiddleware` in the BFF pipeline strips XSS payloads from all incoming requests.

### 14.5 Rate Limiting

```json
{
  "RateLimitingConfig": {
    "EnableEndpointRateLimiting": false,
    "RealIpHeader": "X-Real-IP",
    "ClientIdHeader": "X-ClientId",
    "IpWhitelist": ["127.0.0.1", "::1/10", "192.168.0.0/24"],
    "EndpointWhitelist": ["*:/healthcheck"],
    "GeneralRules": [
      {"Endpoint": "*", "Period": "1s", "Limit": 5},
      {"Endpoint": "*", "Period": "1m", "Limit": 60}
    ]
  }
}
```

### 14.6 SAST & Security Scanning

Fortify SAST runs in pipeline on `master` and `development` branches.

### 14.7 TLS

TLS 1.2 enforced across all services.

---

## 15. Testing Standards

### 15.1 BFF Tests (NUnit + Moq + AutoFixture)

```csharp
[TestFixture]
public class HotelServiceTests
{
    private IFixture _fixture;
    private Mock<ISeturHttpClient> _httpClientMock;
    private Mock<IWorkContext> _workContextMock;
    private Mock<ICacheManager> _cacheManager;
    // ... mocks for all dependencies

    [SetUp]
    public void SetUp()
    {
        _fixture = new Fixture().Customize(new AutoMoqCustomization());
        _httpClientMock = _fixture.Freeze<Mock<ISeturHttpClient>>();
        // ... freeze all mocks, create SUT
    }

    [Test]
    public async Task GetAdvancePayment_WhenApiCallSucceeds_ReturnsSuccessResponse()
    {
        // Arrange
        var requestModel = _fixture.Create<HotelGetAdvancePaymentRequestModel>();
        // Act & Assert ...
    }
}
```

### 15.2 Domain Service Tests (xUnit + Moq)

```csharp
public class PriceSearchServiceTests
{
    [Fact]
    public async Task CombinedPriceSearch_WithValidRequest_ReturnsPrices()
    {
        // Arrange: Moq + Moq.Dapper for DB mocking
        // Act
        // Assert
    }
}
```

### 15.3 Naming Convention

```
{MethodName}_{Condition}_{ExpectedResult}
```
Examples:
- `GetAdvancePayment_WhenApiCallSucceeds_ReturnsSuccessResponse`
- `CombinedPriceSearch_WithValidRequest_ReturnsPrices`
- `BatchUpdate_WithEmptyList_ReturnsZero`

### 15.4 Test Libraries

| Framework | Used By |
|---|---|
| NUnit (`[TestFixture]`, `[Test]`, `[SetUp]`) | BFF |
| xUnit (`[Fact]`, `[Theory]`) | Hotel Price, Hotel Wrapper, domain services |
| Moq | All services |
| AutoFixture + AutoMoq | BFF |
| Moq.Dapper | Hotel Price (Dapper mock support) |

### 15.5 Pipeline Test Execution

```bash
dotnet test --configuration Release --no-build
```

---

## 16. Logging & Monitoring

### 16.1 Logging Framework

**Setur.Logger.WebLogger** → Elasticsearch:
```json
{
  "LogConfig": {
    "ElasticSearchUrls": [
      "http://monitoring1.setur.software:9200/",
      "http://monitoring2.setur.software:9200/"
    ],
    "LogSource": "B2CBFF"
  }
}
```

### 16.2 Log Object Structure

```csharp
LogMessage
{
    LogType,          // ExternalRequest, ExternalResponse, Error
    CorrelationId,    // Request correlation
    SessionId,        // User session
    IP, HttpMethod, Path, QueryString,
    Headers, Body,
    ElapsedMilliseconds,
    LogLevel          // Info, Warning, Error
}
```

### 16.3 Elastic APM

```csharp
app.UseAllElasticApm(builder.Configuration,
    new HttpDiagnosticsSubscriber(),
    new EfCoreDiagnosticsSubscriber(),
    new SqlClientDiagnosticsSubscriber(),
    new AspNetCoreDiagnosticsSubscriber(),
    new MongoDiagnosticsSubscriber());
```

| Setting | Value |
|---|---|
| Server | Elastic Cloud (`elastic-cloud.com:443`) |
| Service Name | Per service (e.g., "BFF", "HotelWrapper") |
| Transaction Sample Rate | 0.1 (10%) in production |
| Captures | Body + Headers |

### 16.4 Request/Response Logging (BFF)

`B2CRequestResponseLoggingMiddleware` captures:
- All request bodies, response bodies, headers
- IP address, HTTP method, path, query string
- Elapsed time via `Stopwatch`
- Sends to Setur.Logger as `LogMessage` objects
- Errors logged at `LogLevel.Error`

---

## 17. HTTP Client Patterns

### 17.1 ISeturHttpClient (Internal NuGet)

Primary HTTP client wrapper used across all services:
```csharp
public interface ISeturHttpClient
{
    Task<T> GetAsync<T>(string url, string method, IDictionary<string, string> headers = null);
    Task<T> PostAsync<T>(string url, string method, object model, IDictionary<string, string> headers = null);
    Task<T> PutAsync<T>(string url, string method, object model, IDictionary<string, string> headers = null);
    Task<T> DeleteAsync<T>(string url, string method, IDictionary<string, string> headers = null);
    Task<T> SendAsync<T>(HttpRequestMessage request);
}
```
- From `Setur.HttpClientHelper` 1.0.9 NuGet
- Adds project token headers automatically
- Built-in logging of requests/responses

### 17.2 Refit Typed Clients (BFF)

```csharp
services.AddRefitClient<IAzureSearchFlightApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri(azureSearchUrl));
services.AddRefitClient<IBasketServiceApi>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri(basketUrl));
```
From `Refit` 8.0.0.

### 17.3 External Service Pattern (Domain Services)

Each downstream dependency gets its own external service class:
```csharp
// Infrastructure/ExternalServices/
public class HotelExternalService : IHotelExternalService
{
    private readonly ISeturHttpClientHelper _httpClient;

    public async Task<SearchResponseModel> Search(SearchRequestModel model, Dictionary<string, string> headers)
    {
        return await _httpClient.PostAsync<SearchResponseModel>(
            $"{_appSettings.SeturHotelServiceUrl}/api/search", model, headers);
    }
}
```

---

## 18. Error Handling

### 18.1 Centralized Exception Handler (BFF)

`UseSeturExceptionHandler()` middleware wraps all unhandled exceptions and returns `ResponseErrorViewModel`:
```csharp
public class ResponseErrorViewModel
{
    public List<ErrorItem> Errors { get; set; }
}
```
HTTP status codes mapped appropriately (400, 401, 403, 404, 500).

### 18.2 ModelState Filter

`ModelStateFilter` (`IActionFilter`) returns 400 `ResponseErrorViewModel` before action execution if model validation fails.

### 18.3 Domain Service Exceptions (Akgunler Pattern)

```
AppException (base)
├── FlightException
│   ├── BadRequestFlightException
│   ├── NullRequestFlightException
│   ├── ValidationFailedFlightException
│   ├── ServiceFaultFlightException
│   ├── AirlineFlightException
│   └── PriceChangedException
```

### 18.4 Channel Manager Error Handling

- `PushReservationRetryCount: 3` with configurable delay
- Email notification via Setur Mail Service on failure
- `CcList` distribution for error email alerts

---

## 19. Performance Tuning

### 19.1 Thread Pool Configuration

| Setting | BFF | Hotel Price (Production) | Hotel Price (Non-Prod) |
|---|---|---|---|
| `ThreadPool.SetMinThreads` (worker) | 2000 | 250 | 50 |
| `ThreadPool.SetMinThreads` (IO) | — | 250 | 50 |
| `DefaultConnectionLimit` | — | 500 | 100 |
| Nagle Algorithm | — | Disabled | Disabled |
| Expect100Continue | — | Disabled | Disabled |

### 19.2 Compression

- Redis cache stores compressed results (GZip) for large search responses
- `Microsoft.IO.RecyclableMemoryStream` 3.0.0 for optimized memory allocation

### 19.3 Concurrency Patterns

| Pattern | Service | Use Case |
|---|---|---|
| `Task.WhenAll` | Hotel Core, Hotel Wrapper | Parallel provider fan-out, parallel metadata + pricing fetch |
| `Parallel.ForEach` + `ConcurrentBag<Task>` | Hotel Price | Batch price updates |
| `Task.Run` within `Parallel.ForEach` | Hotel Price | ProcessPlanList inner parallelism |
| Fire-and-forget (`_ = ...`) | Hotel Price | MassTransit log publishes (⚠️ risk: unobserved exceptions) |

### 19.4 Batch Processing

- Batch updates chunk into 30-day batches
- SQL param batches of 2000 (Local Hotels)
- Dapper command timeout: 60 seconds

---

## 20. Shared NuGet Packages

### 20.1 Internal Packages (SeturFeed)

| Package | Version | Purpose |
|---|---|---|
| `SeturTourismCommon.Enumerations` | 2.1.55 | Shared enums across tourism platform |
| `SeturTourismCommon.Helpers` | 1.1.0 | Common utility functions |
| `SeturTourismCommon.WebUtilities` | 2.0.0–2.0.1.1 | Web middleware, exception handling, `ISeturHttpClientHelper` |
| `SeturTourismCommon.EventBus.Messages` | 1.2.2 | MassTransit message contracts |
| `SeturLocalHotelService.CalculateEngine` | 1.0.3 | Price calculation logic (shared with Local Hotels) |
| `Setur.HttpClientHelper` | 1.0.9 | `ISeturHttpClient` — HTTP abstraction with project token headers |
| `Setur.Logger.WebLogger` | 1.0.7 | Logging framework → Elasticsearch |
| `Setur.BasketApi.Shared` | 3.0.74 | Basket API shared models |
| `SeturTourismCurrencyService.Models` | 1.1.7 | Currency service shared models |
| `SeturHotelPriceApi.Models` | 2.0.2 | Hotel Price API shared models (netstandard2.0/2.1) |
| `SeturHotelWrapperService.Models` | — | Hotel Wrapper shared models |

### 20.2 Package Source Configuration

```xml
<!-- NuGet.Config -->
<configuration>
  <packageSources>
    <add key="nuget" value="https://api.nuget.org/v3/index.json" />
    <add key="SeturfeedV3" value="https://seturcloud.pkgs.visualstudio.com/_packaging/SeturFeed/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

### 20.3 Globalization

- **Fixed to Turkish**: `CultureInfo("tr-TR")` set globally
- All error messages in Turkish
- Date/number formatting follows Turkish locale
