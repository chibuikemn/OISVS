# Project Architecture, DI, SSR, and Clean Code

This document guides contributors on architecture, build/run flows, DI usage, testing, and clean-code rules for this starter repo. It applies to the entire repository.

## Scope & Precedence
- This file defines engineering conventions and patterns for the whole repo.
- If instructions here conflict with a task’s explicit requirements, follow the task’s requirements and update this file in a follow-up PR.

## 1) Stateless Server Architecture (Auth Chain)

The server is intentionally stateless. Every request supplies its own authentication context, and data is stored only in `res.locals` for the life of the request. No server sessions.

Authentication chain (DI middleware classes) for routes that require auth (SSR and API where applicable):
- ExtractTokenMiddleware: reads token from, in priority
  - `Authorization: Bearer <token>` header
  - `token` cookie (cookie-parser registered globally)
  - `token` or `sessionToken` query param
  - `token` in request body
- VerifyMondayJwtMiddleware: verifies the token using both secrets (Monday CLIENT_SECRET and MONDAY_SIGNING_SECRET). On success, sets `res.locals.decodedToken`.
- PopulateLocalsMiddleware: validates payload with `yup` and populates
  - `res.locals.accountId`, `res.locals.userId`, `res.locals.appId`
  - `res.locals.shortLivedToken`

All values in `res.locals` are request-scoped and discarded after the response.

## 2) InversifyJS & Controllers

- IoC container: `server/di/container.ts`, tokens in `server/di/types.ts`, bindings in `server/di/bindings/*`.
- We use `inversify-express-utils`. Controllers are classes annotated with `@controller`, actions use `@httpGet`, `@httpPost`, etc.
- Middleware are classes extending `BaseMiddleware`. Bind them in the container and reference their tokens in controller decorators to apply them.
- Request logging: DI `RequestLoggerMiddleware` is registered globally in `server/app.ts` using `server.setConfig`.
- Global error handler is registered with `server.setErrorConfig` so it runs after controllers.

Example: Apply DI auth + billing/permissions on an API route
```
import { controller, httpPost } from 'inversify-express-utils';
import { TYPES } from '../di/types';

@controller('/api/items')
export class ItemsController {
  @httpPost(
    '/',
    TYPES.ExtractTokenMiddleware,
    TYPES.VerifyMondayJwtMiddleware,
    TYPES.PopulateLocalsMiddleware,
    TYPES.MondayCheckSubscriptionMiddleware,
    TYPES.MondayEnforceSubscriptionMiddleware,
    TYPES.MondayCheckPermissionsMiddleware,
    TYPES.MondayEnforcePermissionsMiddleware,
  )
  create() {
    return { ok: true };
  }
}
```

## 3) Server-Side Rendering (SSR)

SSR is orchestrated by `SsrController` and two services:
- SsrService: the “data brain” that builds initial props (e.g., subscription state, permissions) using `res.locals`.
- SsrRenderService: the “HTML builder” that renders the React app and injects app HTML, serialized props, and style tags into `index.html`.

Runtime behavior:
- Dev: uses Vite `ssrLoadModule('/client/entry-server.tsx')` and `vite.transformIndexHtml`.
- Prod: loads the SSR bundle from `dist/ssr/entry-server.mjs` and reads template from `dist/client/index.html`.
- Template placeholders in `index.html`: `<!--SSR-APP-HERE-->`, `<!--SSR-PROPS-HERE-->`, `<!--SSR-STYLES-HERE-->`.

## 4) TypeScript Project Structure

We use TS project references for clear separation and type-safety.
- `tsconfig.server.json`: Node/Express build (CommonJS, ES2022, decorators enabled). Output: `dist/server`.
- `tsconfig.client.json`: Client app built by Vite (ESNext + bundler resolution).
- `tsconfig.shared.json`: Shared code compiled with declarations for cross-project types.
- Path aliases: `@shared/*`, `@client/*`.

## 5) Build & Runtime

- Dev: `npm run dev` starts the Express server and a Vite dev server; `reflect-metadata` is preloaded.
- Build: `npm run build` produces `dist/client`, `dist/server`, and `dist/ssr` bundles; `tsc-alias` fixes path aliases in server build.
- Start: `npm run start` runs production server from `dist/server/index.js`.

## 6) Logging

- LoggerService (Monday SDK Logger) is the standard logger and must be accessed via DI (`@inject(TYPES.LoggerService)`).
- Do not import `server/config/logger.ts` (deprecated and removed). All logging goes through `ILogger` injection.

## 7) Testing Standards

We use Jest with unit tests co-located in `__tests__` folders.
- Mock external dependencies; focus each test on a single unit of behavior.
- Use `@jest-mock/express` for `req`, `res`, `next` objects.
- Auth middleware tests cover token sources, verification success/fail (
  `TokenExpiredError`, `JsonWebTokenError`), and payload schema.
- Billing/permissions middleware tests verify their DI class behavior and `res.locals` side effects.

## 8) Contribution Guide (How‑To)

Add a service
- Define an interface, implement an `@injectable()` class, bind it in `server/di/bindings/*`, and inject via constructor with `@inject(TYPES.YourService)`.

Add a controller
- Create a class with `@controller('/api/your-route')` and methods with `@httpGet`, `@httpPost`, etc. Import the file in `server/app.ts` (side effect) so it’s discovered.
- Apply DI middleware by listing token symbols in the decorator.

Add middleware
- Implement a class extending `BaseMiddleware` with a `handler(req,res,next)` that returns `void`.
- Bind in `server/di/bindings/middlewares.ts` and reference its token in controller decorators or global config.

## 11) Metrics Service (DI)

Avoid importing metrics helpers directly. Prefer the injectable `MetricsService`.

- Service: `server/services/app/metrics.service` (index.ts) with `trackPoint`, `trackCount`, `trackDuration`.
- Middleware: `DurationMetricsMiddleware` (`server/middlewares/metrics.duration.middleware.ts`) injects the service to record request durations.
- DI: Bound in `server/di/bindings/app.ts` and `server/di/bindings/middlewares.ts` with tokens in `server/di/types.ts`.
- Why: Eliminates hidden deps and global state, simplifies testing and composition.

## 9) Current Alignment & Gaps

What’s aligned
- DI-first server: container, tokens, bindings in place.
- Controllers: `HealthController` and SSR catch-all controller.
- Auth chain: class-based DI middlewares applied to SSR; cookie support registered globally.
- SSR: dev/prod import paths stable; template injection consistent.
- Logging: unified via DI-backed adapter.
- Legacy routers and legacy SSR handler removed.

Recent changes
- Converted MondayBilling and MondayPermissions to injectable services.
- DI auth, billing, and permissions middleware are ready to be applied to controllers via decorator tokens.

Opportunities
- Prefer DI over static helpers everywhere; if new helpers emerge, model them as injectable services.
- Apply the DI auth chain consistently to new API controllers as they’re introduced.

## 10) Clean Code Standard

Code is clean if it can be understood easily – by everyone on the team. Clean code can be read and enhanced by a developer other than its original author. With understandability comes readability, changeability, extensibility and maintainability.
_____________________________________

### General rules
1. Follow standard conventions.
2. Keep it simple stupid. Simpler is always better. Reduce complexity as much as possible.
3. Boy scout rule. Leave the campground cleaner than you found it.
4. Always find root cause. Always look for the root cause of a problem.

### Design rules
1. Keep configurable data at high levels.
2. Prefer polymorphism to if/else or switch/case.
3. Separate multi-threading code.
4. Prevent over-configurability.
5. Use dependency injection.
6. Follow Law of Demeter. A class should know only its direct dependencies.

### Understandability tips
1. Be consistent. If you do something a certain way, do all similar things in the same way.
2. Use explanatory variables.
3. Encapsulate boundary conditions. Boundary conditions are hard to keep track of. Put the processing for them in one place.
4. Prefer dedicated value objects to primitive type.
5. Avoid logical dependency. Don't write methods which works correctly depending on something else in the same class.
6. Avoid negative conditionals.

### Names rules
1. Choose descriptive and unambiguous names.
2. Make meaningful distinction.
3. Use pronounceable names.
4. Use searchable names.
5. Replace magic numbers with named constants.
6. Avoid encodings. Don't append prefixes or type information.

### Functions rules
1. Small.
2. Do one thing.
3. Use descriptive names.
4. Prefer fewer arguments.
5. Have no side effects.
6. Don't use flag arguments. Split method into several independent methods that can be called from the client without the flag.
7. Avoid `any` and `as any`. Prefer precise types, generics, `unknown` + narrowing, or dedicated interfaces. If an assertion is unavoidable, assert to the narrowest specific type (never to `any`).

### Comments rules
1. Always try to explain yourself in code.
2. Don't be redundant.
3. Don't add obvious noise.
4. Don't use closing brace comments.
5. Don't comment out code. Just remove.
6. Use as explanation of intent.
7. Use as clarification of code.
8. Use as warning of consequences.

### Source code structure
1. Separate concepts vertically.
2. Related code should appear vertically dense.
3. Declare variables close to their usage.
4. Dependent functions should be close.
5. Similar functions should be close.
6. Place functions in the downward direction.
7. Keep lines short.
8. Don't use horizontal alignment.
9. Use white space to associate related things and disassociate weakly related.
10. Don't break indentation.

### Objects and data structures
1. Hide internal structure.
2. Prefer data structures.
3. Avoid hybrids structures (half object and half data).
4. Should be small.
5. Do one thing.
6. Small number of instance variables.
7. Base class should know nothing about their derivatives.
8. Better to have many functions than to pass some code into a function to select a behavior.
9. Prefer non-static methods to static methods.

### Tests
1. One assert per test.
2. Readable.
3. Fast.
4. Independent.
5. Repeatable.

### Code smells
1. Rigidity. The software is difficult to change. A small change causes a cascade of subsequent changes.
2. Fragility. The software breaks in many places due to a single change.
3. Immobility. You cannot reuse parts of the code in other projects because of involved risks and high effort.
4. Needless Complexity.
5. Needless Repetition.
6. Opacity. The code is hard to understand.

— End —
