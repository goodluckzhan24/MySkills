---
name: dotnet-webapi-improver
description: Audit and improve existing ASP.NET Core / .NET Web API projects — architecture, error handling, validation, security, data access (EF Core), performance, logging, and testing. Use whenever the user mentions .NET Web API, ASP.NET Core, dotnet controllers, Program.cs, minimal APIs, EF Core in a web service, or asks to improve / refactor / review / optimize / harden such a project — even if they never say the word "improve".
---

# .NET Web API Improver

Improve an existing .NET Web API project without breaking what already works. The workflow is always **Survey → Audit → Report → Improve incrementally**. Never start editing before steps 1–3 are done.

## Step 1 — Survey the project

Build a mental map before touching anything:

- Locate `*.sln` / `*.csproj`. Read each csproj: `TargetFramework`, package references, and whether a test project exists.
- Read `Program.cs` end to end: registration order, middleware pipeline order, auth setup.
- Skim folder layout (`Controllers/`, `Services/`, `Domain/`, `Data/`). Note whether business logic lives directly in controllers — that is a finding.
- Check .NET version: if `TargetFramework` is end-of-life, flag an upgrade as a candidate item but never upgrade it on your own initiative.

State the map back in 3–5 sentences so the user can correct misunderstandings early.

## Step 2 — Audit

Work through the dimensions below. For each one, open the matching section of `references/audit-checklist.md` and record findings with `file:line` evidence:

| Dimension | Typical finding |
|---|---|
| Structure & Program.cs | business logic in controllers, giant Program.cs, no layering |
| DI & configuration | `new`-ing services, `new HttpClient()`, secrets in appsettings |
| API design | entities returned directly, inconsistent routes, no ProblemDetails |
| Validation | missing model validation, unbounded inputs |
| Error handling | try/catch in every action, stack traces leaked to clients |
| Logging & observability | `Console.WriteLine`, no logs, no health checks |
| Security | no auth, `AllowAnyHeader/AnyMethod` CORS, Swagger exposed in prod |
| Data access (EF Core) | sync queries, tracking reads, N+1 loops |
| Performance | sync IO, no caching, blocking calls in actions |
| Testing | zero tests, controllers untestable |

Only report dimensions that have real findings; skip clean ones. Read `references/common-fixes.md` when you need the idiomatic fix pattern for a finding.

## Step 3 — Report & prioritize

Present one table: finding, evidence (`file:line`), impact, suggested fix, effort (S/M/L). Order by value, not by file order:

1. **Correctness & security** — auth gaps, secrets in source, error details leaked to clients.
2. **Reliability** — global error handling, validation, EF Core async.
3. **Maintainability** — layering, DI hygiene, test coverage.
4. **Performance polish** — caching, compression, query tuning.

Then ask the user which items to apply. If the user already said "just do it", apply tiers 1–2 and report back.

## Step 4 — Improve incrementally

- One finding per change-set. Run `dotnet build` after each change-set and `dotnet test` when tests exist. If the build breaks, fix it before moving on — never leave the project red.
- Do not change public API contracts (routes, status codes, response shapes) unless the user asked for it.
- Match the codebase's naming, style, and comment density. Do not reformat code you are not changing.
- Adapt patterns from `references/common-fixes.md` to the project's .NET version and conventions — do not paste blindly. Patterns assume .NET 8+; for older runtimes, use the era-appropriate variant (e.g. exception middleware instead of `IExceptionHandler`).
- Finish with a summary per batch: what changed, why, and the verification result.

## Scope guardrails

- If the project does not compile or has no `.csproj` at all, stop and tell the user — auditing a broken tree wastes effort.
- Time-sensitive choices (framework upgrade, major refactors, adopting Clean Architecture) are the user's decision; present options with trade-offs instead of acting.
