# Common fix patterns — .NET Web API

Idiomatic fix patterns for the most common findings. Patterns assume .NET 8+; for older runtimes use the era-appropriate variant noted where relevant. Adapt names and namespaces to the project — never paste blindly.

## Global exception handling (.NET 8+)

```csharp
// Program.cs
builder.Services.AddProblemDetails();                 // RFC 7807 bodies
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
var app = builder.Build();
app.UseExceptionHandler();                            // no lambda arg → uses DI handler
```

```csharp
public sealed class GlobalExceptionHandler(IProblemDetailsService problemService)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex,
        CancellationToken ct)
    {
        ctx.Response.StatusCode = ex switch
        {
            ArgumentException => StatusCodes.Status400BadRequest,
            UnauthorizedAccessException => StatusCodes.Status403Forbidden,
            _ => StatusCodes.Status500InternalServerError,
        };
        // Never pass ex.Message for 500s in production; log it instead
        return await problemService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = ctx,
            ProblemDetails = new ProblemDetails
            {
                Status = ctx.Response.StatusCode,
                Title = ReasonPhrases.GetReasonPhrase(ctx.Response.StatusCode),
            },
        });
    }
}
```

Pre-.NET 8: an `ExceptionMiddleware` invoking `_next` in try/catch, writing ProblemDetails manually.

## Validation

`[ApiController]` auto-returns 400 with ProblemDetails on invalid ModelState — prefer adding it over manual checks. For rules beyond attributes:

```csharp
// FluentValidation registration
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserRequestValidator>();
builder.Services.AddFluentValidationAutoValidation();
```

Always bound inputs: `[Required]` + `[MaxLength]` on strings, `[Range]` on numbers, `IFormFile` with size checks.

## DTO + CreatedAtAction pattern

```csharp
[HttpPost]
[ProducesResponseType(typeof(UserDto), StatusCodes.Status201Created)]
[ProducesErrorResponseType(typeof(ProblemDetails))]
public async Task<ActionResult<UserDto>> Create(CreateUserRequest req, CancellationToken ct)
{
    var user = await _service.CreateAsync(req, ct);
    return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
}
```

## HttpClient via factory

```csharp
builder.Services.AddHttpClient<IPaymentApiClient, PaymentApiClient>(c =>
{
    c.BaseAddress = new Uri(builder.Configuration["Payment:BaseUrl"]!);
    c.Timeout = TimeSpan.FromSeconds(10);
});
```

Never `new HttpClient()` per request (socket exhaustion) and never a singleton `HttpClient` ignoring DNS changes.

## EF Core hygiene

```csharp
// Read-only list: async + no tracking + projection (no Include needed)
var items = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderListItemDto(o.Id, o.Total, o.CreatedAt))
    .ToListAsync(ct);

// Write
db.Orders.Add(order);
await db.SaveChangesAsync(ct);
```

Register once: `builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlServer(connStr));` (scoped by default — keep it that way).

## Options pattern

```csharp
public sealed class SmtpOptions { public required string Host { get; init; } public int Port { get; init; } }

builder.Services.AddOptions<SmtpOptions>()
    .BindConfiguration("Smtp")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Secrets: `dotnet user-secrets` in Development; environment variables in production. Never committed keys.

## Health checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>("database");
// ...
app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = r => r.Tags.Contains("ready") is false, // liveness vs readiness split if needed
});
```

## CORS done right

```csharp
const string CorsPolicy = "frontend";
builder.Services.AddCors(o => o.AddPolicy(CorsPolicy, p => p
    .WithOrigins(builder.Configuration.GetSection("Cors:Origins").Get<string[]>() ?? [])
    .AllowAnyHeader().AllowAnyMethod()));
// ...
app.UseCors(CorsPolicy);   // after UseRouting, before UseAuthorization
```

Swagger gated to Development:

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

## Rate limiting (built in, .NET 7+)

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddFixedWindowLimiter("fixed", w => { w.Window = TimeSpan.FromMinutes(1); w.PermitLimit = 100; });
});
app.UseRateLimiter();
// per-endpoint: .RequireRateLimiting("fixed") or [EnableRateLimiting("fixed")]
```

## Output caching + response compression

```csharp
builder.Services.AddOutputCache(o => o.AddPolicy("products", p => p
    .Expire(TimeSpan.FromMinutes(5))
    .SetVaryByQuery("page")));
builder.Services.AddResponseCompression(o => o.EnableForHttps = false); // let TLS/proxy decide
// ...
app.UseResponseCompression();
app.UseOutputCache();
// endpoint: .CacheOutput("products")
```

## Structured logging

```csharp
_logger.LogInformation("Order {OrderId} created for {CustomerId}", order.Id, customerId);
// wrong: _logger.LogInformation($"Order {order.Id} created"); — loses structure
```

## Testing slice

```csharp
public class OrdersApiTests(WebApplicationFactory<Program> factory) : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Create_returns_201()
    {
        var client = factory.WithWebHostBuilder(b =>
            b.ConfigureTestServices(s => s.RemoveAll<DbContextOptions<AppDbContext>>()
                .AddDbContext<AppDbContext>(o => o.UseInMemoryDatabase("t")))).CreateClient();
        var resp = await client.PostAsJsonAsync("/api/v1/orders", new { customerId = 1 });
        Assert.Equal(HttpStatusCode.Created, resp.StatusCode);
    }
}
```

For `Program` visibility from the test project: `public partial class Program { }` at the bottom of Program.cs.
