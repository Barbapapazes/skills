---
name: cloudflare-hono-layered-server
description: 'Scaffold or extend a layered Cloudflare Workers server with Hono, D1 repositories, shared Zod validators, module-based services, mappers, custom exceptions, and optional workflows or external integrations. The architecture is based on a practical three-tier split between controllers, services, and repositories. Use when creating a layered Worker backend, adding a new server domain, or refactoring toward a controller/service/repository architecture.'
argument-hint: 'Describe the domain, routes, persistence, external services, and whether the server needs Workflows or other async jobs.'
user-invocable: true
---
# Cloudflare Hono Layered Server

Use this skill when building a Cloudflare Workers server that benefits from a layered architecture:
- Cloudflare Workers as the runtime
- Hono for HTTP routing
- a layered structure with controllers, services, repositories, mappers, and exceptions
- shared Zod validators in `src/shared`
- optional Durable Workflows for long-running orchestration
- optional integrations such as Discord, OpenAI, Cloudflare APIs, or other HTTP services

This skill should help another LLM produce a server that is easy to extend, keeps business logic out of route handlers, and follows a practical Cloudflare-first architecture.

At its core, this skill leans on a simple three-tier architecture:
- controllers act as the presentation or transport layer
- services hold business logic and orchestration
- repositories own persistence and database access

This is not meant to be fancy enterprise ceremony. It is simply a clear separation of concerns that is often useful in Cloudflare Workers projects, even if this style is less common in the JavaScript world than more ad hoc route-driven backends.

## What this skill should produce

Create or update a server that:
1. Uses a practical three-tier split of controllers, services, and repositories.
2. Uses thin controllers and keeps orchestration in services.
3. Uses repositories for persistence concerns instead of mixing SQL into controllers.
4. Uses mappers when API responses need derived or presentation-ready fields.
5. Uses custom exceptions for expected failure cases such as validation and missing resources.
6. Uses shared validators when request contracts are relevant to both client and server.
7. Uses optional workflow classes for long-running or retry-heavy async jobs.
8. Follows local project conventions before introducing new ones.

## Default target structure

Prefer this structure unless the host project already has a stronger convention:

```text
src/
  server/
    index.ts
    context.ts
    constants.ts
    utils.ts
    controllers/
    exceptions/
    mappers/
    repositories/
    services/
    workflows/
  shared/
    validators.ts
```

If the project does not need one of these layers yet, skip it cleanly instead of adding empty ceremony.

## Procedure

### 1. Read the local project first

Before writing code:
- inspect `package.json`, runtime config, and nearby backend files
- confirm whether the project already uses Hono, Cloudflare Workers, D1, R2, `ofetch`, `zod`, `ms`, or logging middleware
- reuse existing naming, error shapes, and import conventions
- prefer extending the current architecture over forcing this one into an incompatible project

If the local codebase has a stronger established pattern than this skill, follow the codebase.

### 2. Clarify the domain slice

Identify the smallest useful slice to build:
- resource names and route paths
- which operations are synchronous CRUD versus asynchronous processing
- whether persistence is required and where it lives
- which external services are involved
- which validation rules should be shared with the frontend

Translate business nouns into folder-level modules. For example, a domain named `articles` usually implies files such as:
- `controllers/article.controller.ts`
- `services/article.service.ts`
- `repositories/article.repository.ts`
- `mappers/article.mapper.ts`

### 3. Set the runtime boundary

Create or update the Worker entrypoint in `src/server/index.ts`.

Preferred responsibilities:
- create the `Hono` app with typed bindings
- register middleware such as structured logging
- add a single `app.onError(...)` handler
- mount controllers under stable route prefixes
- export any `WorkflowEntrypoint` classes used by Wrangler bindings
- keep the entrypoint thin and declarative

When using Hono in this style:
- use `AppEnv` from `context.ts` to type the app
- return JSON for handled failures
- keep generic 500 responses consistent and machine-readable

### 4. Model expected failures explicitly

Create custom exceptions when failure cases are part of normal business behavior.

Use custom exceptions for cases such as:
- invalid input
- missing resources
- external service states that should stop retries
- domain-specific invariants

Keep the root error handler responsible for mapping these exceptions to HTTP responses.

### 5. Put reusable request validation in `src/shared/validators.ts`

Prefer shared Zod schemas when the same input contract matters across frontend and backend.

Typical pattern:
- define a schema in `src/shared/validators.ts`
- parse or `safeParse` input in the service layer
- convert validation failures into a `ValidationException`
- use transforms at schema level when normalization should happen before persistence, for example URL cleanup

### 6. Keep controllers thin

Controllers should mainly:
- read request params or JSON
- call a service function
- map the result if needed
- return `c.json(...)` or `c.body(...)`

Controllers should not:
- contain SQL
- embed non-trivial validation logic
- contain retry logic for external APIs
- perform large domain transformations

### 7. Build services as modules of named functions

Prefer module-based services over classes.

Pattern to follow:
- export named functions such as `list`, `getById`, `create`, `reprocess`, `markAsProcessed`
- import other services as namespaces when helpful, for example `import * as articleService from '../services/article.service'`
- keep domain decisions and orchestration in services
- let services coordinate repositories, external integrations, workflows, and mappers as needed

Service responsibilities usually include:
- schema validation
- existence checks
- business invariants
- triggering workflows or async side effects
- formatting content for storage or delivery

### 8. Isolate persistence in repositories

Use repositories for D1 or database access.

Preferred repository conventions:
- keep SQL in repository files
- type query results explicitly with `.first<T>()` and `.all<T>()`
- return plain data objects, not HTTP responses
- keep timestamp writes consistent through a shared helper when the project stores text timestamps
- map application field names to storage field names in one place

If the host project uses D1 in the same style, prefer direct `env.DB.prepare(...).bind(...).run()` calls over adding an ORM.

### 9. Add mappers when response shape differs from storage shape

Use mapper functions when the API response needs:
- derived URLs
- renamed fields
- computed booleans
- formatting that should not live in controllers or repositories

Mapper files should stay deterministic and avoid side effects.

### 10. Create integration services for external APIs

When the server talks to external systems, create dedicated service modules for each integration.

Examples:
- `openai.service.ts`
- `discord.service.ts`
- `cloudflare.service.ts`

Keep these services focused on API interaction details such as:
- payload construction
- auth headers
- response parsing
- retry-related error classification

Do not spread raw HTTP calls across controllers and workflows.

### 11. Add Workflows only for real asynchronous orchestration

Create `src/server/workflows/*.ts` only when the domain truly needs long-running, retried, or multi-step jobs.

A workflow is a good fit when the process:
- spans minutes or longer
- depends on multiple external systems
- needs step-specific retry and timeout policies
- should survive worker restarts

Workflow conventions to follow:
- define named step profiles in one config object
- use `ms(...)` for human-readable durations
- use `step.do(...)` with descriptive step names
- convert known terminal failures into non-retryable errors when appropriate
- log enough structured context to debug failures later

### 12. Wire environment bindings deliberately

Keep platform bindings explicit.

Typical bindings for a similar server may include:
- `DB` for D1
- `FILES` or `CONTENT` for R2
- workflow bindings such as `ARTICLE_WORKFLOW` or `ORDER_WORKFLOW`
- secrets or vars for external APIs and URLs

When adding new bindings:
- update `wrangler.jsonc`
- regenerate worker types when the project uses generated env declarations
- keep `context.ts` aligned with the generated `Env`

### 13. Finish with integration-aware checks

Before considering the work done, verify that:
- controllers stay thin
- services contain the business decisions
- repositories own SQL and typed query shapes
- expected failures use custom exceptions
- shared validators are reused instead of duplicated
- workflows exist only when they solve a real async problem
- environment bindings are declared and typed
- route paths, response codes, and error envelopes are consistent
- README or setup docs were updated if the architecture or developer workflow changed

## Decision points

Use these branches while generating the server:

### If the project is HTTP-only
- create `index.ts`, `context.ts`, controllers, services, and optionally mappers
- skip workflows entirely

### If the project has persistence but no long-running jobs
- add repositories
- keep async orchestration in services
- do not create workflow files just to look sophisticated

### If the project has long-running or retry-heavy jobs
- add a workflow class and a workflow binding
- let services trigger the workflow instead of inlining the job

### If the frontend shares the same request contracts
- place validators in `src/shared/validators.ts`
- import them into the server instead of duplicating schemas

### If an external API has tricky behavior
- hide it behind a dedicated service and a domain-specific exception
- encode any protocol quirks in one place instead of scattering them across the app

## Practical conventions for this pattern

These conventions keep the architecture consistent without adding unnecessary ceremony:
- the main architectural backbone is a simple three-tier split: controller, service, repository
- `src/server/index.ts` is the main composition root and error boundary
- services are function modules, not classes
- repositories use typed D1 queries directly
- workflow step timing can use `ms(...)` when readable durations help
- timestamp writes use a shared helper when storing UTC text values
- response mappers are separate from repositories and controllers

## Quality checks

Before finishing, verify that:
- no controller contains database queries
- no repository returns Hono responses
- no service returns `c.json(...)`
- database result types are explicit where practical
- external integrations live in dedicated services
- the error handler understands the custom exceptions you introduced
- workflow step names are descriptive enough for logs and dashboards
- imports and filenames follow the local lint and naming conventions

## Response behavior

When using this skill, the response should:
- state which layers were created or intentionally skipped
- briefly mention any assumptions about persistence, workflows, or external APIs
- call out which local conventions were reused
- keep scaffolding practical instead of generating large amounts of speculative business logic

## Good prompts for this skill

- Scaffold a Cloudflare Workers + Hono server for an article review app with D1 and shared Zod validators.
- Add an `orders` domain to this server using controller, service, repository, mapper, and exceptions files.
- Refactor this Worker backend toward a layered Hono architecture without introducing an ORM.
- Create a new workflow-backed server module that uses OpenAI and Discord-style external services.
