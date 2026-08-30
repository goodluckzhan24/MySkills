# Audit checklist — .NET Web API

Per-dimension checks for Step 2. Use grep/搜索 to find the signals fast; read the surrounding code before declaring a finding. Every finding needs `file:line` evidence.

## 1. Structure & Program.cs

- `Program.cs` over ~100 lines or containing business logic → candidate for extension methods (`AddApplicationServices`, `UseApiPipeline`).
- Controllers containing EF queries, `HttpClient` calls, or business rules → logic belongs in a service layer (or handler/mediator if the project already uses one).
- No project layering for a solution with 3+ projects of mixed concerns, or all code in one web project with no clear folders.
- Duplicate middleware/DI setup across multiple projects → shared `Web` infrastructure project.

## 2. DI & configuration

- `new SomeService(...)` inside controllers/services instead of constructor injection.
- `new HttpClient(...)` anywhere → must be `IHttpClientFactory` (`AddHttpClient`), static `HttpClient` is acceptable only if noted with BaseAddress/timeout rationale.
- Singleton services holding `DbContext` or `HttpClient` — captive dependency.
- `services.AddScoped` for everything regardless of lifetime — check for services holding state.
- Config read via `configuration["Key"]` scattered in action bodies → Options pattern (`IOptions<T>` / `IOptionsSnapshot<T>`).
- Connection strings, API keys, or passwords in `appsettings.json` committed to source control → move to user-secrets/env vars, and flag rotation if the repo is shared.

## 3. API design

- Domain entities returned directly from actions → return DTOs; avoids over-posting and leaking schema.
- Inconsistent route templates (`api/[controller]` vs `api/v1/users` vs literal mixed case) → pick one convention.
- No API versioning on a project that already has external consumers.
- Actions returning raw entity + 200 for create → `CreatedAtAction` with 201.
- No `Produces`/`ProducesResponseType` metadata → harms Swagger accuracy.
- Synchronous actions doing IO (no `async`/`await`).

## 4. Validation

- No `[ApiController]` attribute (which auto-validates `ModelState`) or controllers not deriving from `ControllerBase`.
- DTOs without any validation attributes and no manual checks → unbounded strings, no range limits.
- File upload endpoints without size/type limits.
- Manual `if (model == null)` style checks duplicated across actions → action filter or FluentValidation.

## 5. Error handling

- try/catch blocks in many actions doing the same thing → global handler.
- Exceptions returned as 200 with an error body, or `500` with `ex.ToString()` / `ex.Message` returned to the client → leaks internals; use ProblemDetails.
- No `app.UseExceptionHandler()` / `AddExceptionHandler()` → unhandled exceptions yield a bare 500 with no correlation for debugging.
- Missing `app.UseStatusCodePages()` or 404/400 responses with empty bodies → clients get no machine-readable error.

## 6. Logging & observability

- `Console.WriteLine` / `Debug.WriteLine` in production paths → `ILogger<T>`.
- Logging string interpolation (`$"...{x}"`) → structured logging templates (`LogInformation("Got {Count}", count)`).
- Logging exceptions without passing the exception object.
- No `/health` endpoint → `AddHealthChecks()`, plus DB readiness check when EF is used.
- Log level not configurable per environment; `appsettings.Development.json` not differing from production.

## 7. Security

- Endpoints with data writes and no `[Authorize]` / `RequireAuthorization()`.
- CORS `AllowAnyOrigin` combined with `AllowCredentials` (invalid + dangerous), or wide-open CORS in production config.
- Swagger (`UseSwaggerUI`) enabled unconditionally — gate it to Development or an env flag.
- JWT configuration: `ValidateIssuer`/`ValidateAudience`/`ValidateLifetime` set to false, symmetric key hardcoded.
- Missing `app.UseHttpsRedirection()` behind a proxy, or trust settings for forwarded headers.
- Bindings in `Bind()` / `GetSection().Bind()` on shapes that allow over-posting of sensitive fields.
- Rate limiting absent on auth-sensitive endpoints (`AddRateLimiter` is built in since .NET 7).

## 8. Data access (EF Core)

- Sync EF calls: `.ToList()`, `.FirstOrDefault()`, `.SaveChanges()` instead of `ToListAsync()` / `FirstOrDefaultAsync()` / `SaveChangesAsync()`.
- Read-only queries with tracking → `AsNoTracking()` (or `AsNoTrackingWithIdentityResolution` when needed).
- N+1: queries inside `foreach` loops, or navigation properties used after the query without `Include`/projection.
- `Include` chains loading whole graphs where a projection (`Select`) would do.
- `DbContext` registered as Singleton, or manually `new`-ed per request.
- Raw SQL with string concatenation → SQL injection; must use parameterized `FromSqlInterpolated`/`ExecuteSqlInterpolated`.
- Missing index hints: frequent `.Where(x => x.Field == ...)` on columns with no migration index (note it, don't create migrations unasked).

## 9. Performance

- `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` anywhere → deadlock risk + thread starvation.
- Heavy in-memory work (`ToList()` then filtering) instead of server-side filtering.
- Frequently-read, rarely-changed data fetched per request → `IMemoryCache` / `OutputCache` (`AddOutputCache` is built in).
- Large responses without pagination.
- Missing `AddResponseCompression` for text-heavy payloads (when self-hosted; IIS/YARP often handles it).
- JSON options not set: `JsonSerializerOptions.WebForReflectableTypes` (or previous `JsonSerializerDefaults.Web`) to get camelCase + loose enum binding on minimal APIs.

## 10. Testing

- No test project in solution at all.
- Only trivial tests, none covering controllers/handlers/services with logic.
- Untestable code: `DateTime.Now` inline, static dependencies, no interfaces on services that call external systems.
- Missing integration-test slice: `WebApplicationFactory<Program>` for a couple of key endpoints.
