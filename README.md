# Insurance Claims API

A backend application for managing insurance claims, built with C# and .NET.

## Overview

This project demonstrates the implementation of an insurance claims management API: creating and managing insurance covers, filing claims against them, validating business rules, computing premiums, and recording an audit trail for every write operation.

Two data stores are used deliberately: claims and covers are persisted in MongoDB (via the EF Core MongoDB provider), while audit records are written to SQL Server. Audit dispatch is asynchronous — a bounded in-memory channel decouples the request path from the audit writer, so auditing never blocks or fails a request.

## Architecture

The solution follows a layered architecture with dependencies pointing inward only:

```
Claims (host)          ASP.NET Core controllers, DI wiring, exception handling, Swagger
Claims.Application     services, repository interfaces, DTOs, FluentValidation validators
Claims.Domain          entities, enums, CoverPeriod, PremiumCalculator (no dependencies)
Claims.Infrastructure  EF Core contexts (MongoDB + SQL Server), repositories,
                       audit channel + background worker, EF migrations
Claims.Tests           unit and integration tests (xUnit, NSubstitute, WebApplicationFactory)
```

Controllers depend only on service abstractions (`IClaimService`, `ICoverService`); no `DbContext` or persistence types leak past `Claims.Infrastructure`. `Claims.Domain` has no project references.

## Features

- Claim management — create, retrieve, list, and delete claims against covers
- Cover management — CRUD plus a `POST /covers/compute` endpoint that prices a hypothetical cover without persisting it
- Business rule validation — FluentValidation rules for cover duration, start dates, claim amounts, damage types, and cover/claim date containment
- Premium calculation — cover-type-specific base rates with duration-band discounts (`Claims.Domain/PremiumCalculator.cs`)
- Asynchronous auditing — every create/delete is enqueued on a bounded `Channel<AuditEntry>` and persisted to SQL Server by a background service
- Automated tests — unit tests for domain logic, validators, services, and the audit pipeline, plus integration tests via `WebApplicationFactory`
- REST API with Swagger UI and RFC 7807 problem-details error responses

## Technologies

- C# / .NET 9
- ASP.NET Core (controllers, `TimeProvider`, `BackgroundService`)
- Entity Framework Core (SQL Server provider + MongoDB EF Core provider)
- FluentValidation
- xUnit v3, NSubstitute, `Microsoft.Extensions.TimeProvider.Testing`
- Testcontainers (ephemeral SQL Server + MongoDB on startup)
- Docker, Swagger/OpenAPI

## Running the Application

Requires the .NET 9 SDK and a running Docker daemon (Docker Desktop on Windows). The host provisions ephemeral SQL Server and MongoDB containers via Testcontainers on startup — no manual database setup is needed.

```
dotnet run --project Claims
```

Swagger UI is available at the root URL in the Development environment.

## Running Tests

```
dotnet test
```

Most tests are pure unit tests and run instantly. The `ClaimsControllerTests` integration tests spin up the full host (and therefore require Docker) via `WebApplicationFactory`.

## Design Decisions

A few non-obvious choices worth highlighting:

- **Band-3 premium discounts stack additively** (5% + 3% off the base rate), matching the original constants rather than compounding multiplicatively.
- **The billable cover period is exclusive of the end date** (Jan 1 → Jan 31 bills 30 days), isolated in `CoverPeriod` so the convention lives in one place — while claim coverage uses a *closed* interval, since an incident on a policy's last day should still be covered even though that day isn't billable.
- **The one-year cover limit is calendar-aware** (`AddYears(1)`), not a fixed 365 days, so a one-year cover spanning a leap day is still valid.
- **The audit queue drops on saturation rather than blocking.** `TryWrite` never stalls the request thread; a saturated queue logs the dropped entry instead of propagating an error.
- **`TimeProvider` is injected everywhere "now" matters**, making date-boundary tests deterministic via `FakeTimeProvider`.
- **The audit background worker catches exceptions per-item**, so one bad write is logged without stopping the host-wide worker.
- **Services throw `ValidationException` for invalid input but return `null`/`false` for not-found**, mapping cleanly to `400` vs `404` at the controller boundary.

## Disclaimer

This is a personal demonstration project showcasing backend development, layered architecture, and testing practices.
