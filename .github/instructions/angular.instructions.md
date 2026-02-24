---
applyTo: "**/*.component.ts,**/*.service.ts,**/*.guard.ts,**/*.pipe.ts,**/*.directive.ts,**/*.component.html,**/angular.json"
---

# Angular Coding Standards

## Naming Conventions

| Element | Convention | Example |
|---|---|---|
| Components | PascalCase class, kebab-case selector | `UserProfileComponent`, `app-user-profile` |
| Services | PascalCase + Service suffix | `OrderService` |
| Interfaces | PascalCase, NO "I" prefix | `UserData`, `OrderSummary` |
| Enums | PascalCase name and values | `UserStatus.Active` |
| Files | kebab-case | `user-profile.component.ts` |
| Constants | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |

## Angular 19 Rules

- **Always** use `standalone: true` — no NgModules in new code
- **Always** use `ChangeDetectionStrategy.OnPush` in all components
- Use signal-based APIs: `input()`, `output()`, `viewChild()`, `contentChild()` instead of decorators
- Use `inject()` function instead of constructor injection
- Lazy load all routes with `loadComponent`
- Never use `any` — use `unknown` when type is uncertain

## Component Pattern

- Smart (container) components: handle data, inject services, pass data to presentational
- Presentational components: pure, only inputs/outputs, no service injection
- One component = one file
- Extract complex logic into custom hooks (services)

## Code Quality

- `const` by default, `let` only when needed, never `var`
- Optional chaining: `user?.address?.city`
- Nullish coalescing: `value ?? defaultValue`
- Early return to avoid nested conditions
- Expressive variable names for complex conditions

## RxJS

- Use `catchError` for error handling in observables
- Use `takeUntilDestroyed()` for subscription cleanup
- `switchMap` for cancellable requests, `mergeMap` for parallel
- `shareReplay(1)` for shared observables
