
# Coding rules

- Always use Typescript code.
- Generate object-oriented code, using classes to encapsulate functionality.
- Use enterprise software patterns such as Factory, Observer, etc.
- Each class should live in its own source file, named using camelcase.
- Prefer built-in primitive types (`string`, `number`, `boolean`) over their wrapper object equivalents (`String`, `Number`, `Boolean`).
- Immutable data structures are preferred over mutables. When an object must be assembled incrementally, build it through a dedicated **Builder** class and pass the finished parts to the constructor, or accept that the type is a mutable aggregate and stop labelling it immutable. Do not split the difference with a `public` mutator hidden behind an `/** @internal */` comment -- `@internal` is documentation, not enforcement, and a `public` method is callable by anyone.
- **Knowledge has a single home (DRY).** A constant, list, or rule that more than one class depends on lives in exactly one module (typically under `shared/` or a domain constants file) and is imported. Never copy a literal set or array into a second file -- duplicated knowledge drifts out of sync silently.

## Dependencies (reuse before you build)

- Before writing non-trivial functionality from scratch, **search [npmjs.com](https://www.npmjs.com/) for an existing package** that solves the problem (parsers, validators, date/HTTP/crypto helpers, etc.).
- If a candidate exists and its GitHub repository has **at least 100 stars**, prefer importing it over a hand-rolled implementation. Star count is a proxy for community vetting -- it should not be the only check (see below).
- Before adopting, confirm the package is **actively maintained** (recent commits/releases), carries a **permissive licence** (MIT/Apache-2.0/BSD), ships **TypeScript types**, and has **no critical advisories** (`npm audit`).
- Pin the dependency and add it to the appropriate layer -- wrap third-party APIs behind an **Adapter** in `infrastructure/` so the rest of the code depends on your interface, not the vendor's.
- Build from scratch only when no package clears the bar above, or the dependency would be heavier than the problem warrants. Record the reason in a brief code comment or commit message.

## Naming

- Classes, interfaces, types, enums: `PascalCase` (e.g. `BedrockModelFactory`)
- Variables, functions, methods, parameters: `camelCase` (e.g. `createModel`)
- Global constants: `SCREAMING_SNAKE_CASE` (e.g. `DEFAULT_REGION`)
- Boolean variables and properties: prefix with `is`, `has`, or `can` (e.g. `isReady`, `hasCredentials`)
- Treat acronyms as whole words: `loadHttpUrl` not `loadHTTPURL`, `XmlParser` not `XMLParser`
- No Hungarian notation, no leading/trailing underscores

## SOLID Principles

| Principle | Rule |
|---|---|
| **Single Responsibility (SRP)** | A class has exactly one reason to change. If you find yourself writing "and" when describing what a class does, split it. |
| **Open/Closed (OCP)** | Classes are open for extension, closed for modification. Add behaviour by composing new classes, not by editing existing ones. |
| **Liskov Substitution (LSP)** | Any object implementing a contract must honour the full contract. Do not implement a method only to throw `Error("not implemented")`. |
| **Interface Segregation (ISP)** | Prefer small, focused interfaces over large ones. A class should not be forced to depend on methods it does not use. |
| **Dependency Inversion (DIP)** | High-level classes depend on abstractions, not concrete implementations. Inject dependencies -- never instantiate collaborators inside a class. |

## Classes and Visibility

- Default all members to `private`; only promote to `protected` or `public` when required by callers.
- Never use the `public` modifier on instance methods -- it is the default and adds noise.
- Use TypeScript parameter properties to eliminate constructor boilerplate: `constructor(private readonly model: BedrockModel)`.
- Never use `obj['property']` bracket access to bypass `private` or `protected` visibility.
- Add a JSDoc comment (`/** ... */`) to every public method and class.

## Functions

- Limit parameters to **two or fewer**. When more are needed, pass a single configuration object instead.
- A function does **one thing** -- if you need "and" to describe it, extract a second function.
- Keep method bodies under **~20 lines** and avoid more than **2 levels of nesting**. When a `switch` or `if/else if` chain dispatches on a type tag, extract each branch into a named private method (`handleOpenTag`, `handleText`, ...) so the dispatcher reads as a table of intent rather than a wall of inline logic.
- Never use boolean flag parameters; they signal the function has two responsibilities. Split it.
- Functions must not produce hidden side effects on state outside their scope.
- Use `for...of` over `forEach` for iteration; use `Object.keys()` / `Object.values()` / `Object.entries()` over `for...in`.

## Types

- Avoid `any`; use `unknown` with explicit type guards when the type is truly unknown.
- Always annotate return types on public methods -- catches breaking changes early.
- Mark properties that are never reassigned after construction as `readonly`.
- Use type aliases only for unions.
- Prefer a named constant or an `as const` object for a string or numeric literal repeated across files. Trade-off: for discriminated-union tags (e.g. `"open-tag" | "text"`), inline literals keep exhaustiveness checking sharp, so this rule yields to readability case-by-case -- centralise the *type*, and only extract the *values* when typos would otherwise slip past the compiler.
- Use `@ts-expect-error` (never `@ts-ignore`) when suppressing a type error; add a comment explaining why.

## Variables and Control Flow

- Use `const` by default; use `let` only when reassignment is needed; never use `var`.
- Always use `===` and `!==` for equality checks.
- Always throw `Error` instances -- never throw strings or plain objects (stack traces only populate on `Error`).
- Custom errors exist to make failures diagnosable. Every field on an error must carry a **real, computed value** -- never a placeholder such as a hardcoded `0`. If you cannot supply accurate context for a field, do not add the field; a misleading "at position 0" is worse than no position at all.
- Never leave an empty `catch` block. Either handle the error, rethrow it, or add a comment explaining why it is intentionally swallowed.
- Never ignore promise rejections. Always `.catch()` or `await` inside a `try/catch`.

## Imports and Exports

- Use named exports exclusively; avoid default exports.
- Order imports in three groups separated by a blank line: (1) Node built-ins, (2) external packages, (3) internal/relative paths.
- Remove unused imports immediately; they add noise and slow down the compiler.

## Strict Mode (tsconfig.json)

Enable beyond the default `strict: true`:
- `noUncheckedIndexedAccess` -- array/index access returns `T | undefined`, preventing silent out-of-bounds bugs.
- `exactOptionalPropertyTypes` -- distinguishes between a missing property and one explicitly set to `undefined`.


# Project Structure

Use **layered architecture**. Organise by technical responsibility; each layer depends only on layers below it -- never upward.

```
src/
  domain/           # Pure business logic -- no framework, no AWS deps
    [feature]/
      [feature].service.ts
      [feature].types.ts
      [feature].repository.ts
  infrastructure/   # External systems: AWS Bedrock, S3, secrets, HTTP
    [service]/
  shared/           # Cross-cutting: logging, errors, config, factories
    config/         # Environment-based data -- never mixed with logic
  index.ts          # Public API barrel export -- the only import boundary
tests/              # Mirrors src/ structure exactly
examples/           # Runnable usage examples

```

## Layer rules

- `domain/` must not import from `infrastructure/` -- inject dependencies via DIP
- `infrastructure/` must not contain business logic
- `shared/` must not import from `domain/` or `infrastructure/`
- `shared/config/` holds environment config only -- no logic
- `shared/` organises factories and cross-cutting constructs by capability (e.g. `shared/logging/`, `shared/errors/`), not by type
- `index.ts` is the only public surface of the library -- never import from internal paths

## Path aliases

Configure `@/` in `tsconfig.json` to avoid `../../../` chains:

```json
{
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }
  }
}
```

## Test mirroring

`tests/` mirrors `src/` exactly:
`src/domain/agent/agent.service.ts` -> `tests/domain/agent/agent.service.test.ts`


## Large or multi-package projects: TypeScript Project References

When the codebase splits into multiple packages (e.g. a monorepo), use TypeScript Project References instead of a single `tsconfig.json`:

- Set `composite: true` in each sub-project's `tsconfig.json`
- Add a solution-level `tsconfig.json` at the root with `"files": []` and `references` pointing to each sub-project
- Build with `tsc -b` (not `tsc`) for incremental, dependency-ordered compilation
- Use `declarationMap: true` to support cross-package "Go to Definition"

```json
// root tsconfig.json (solution file)
{
  "files": [],
  "references": [
    { "path": "./src" },
    { "path": "./tests" }
  ]
}
```

## References

| Source | Guidance |
|---|---|
| [TypeScript Handbook -- Project References](https://www.typescriptlang.org/docs/handbook/project-references.html) | Composite builds, project references, `tsc -b` |
| [TypeScript Handbook -- Library Structures](https://www.typescriptlang.org/docs/handbook/declaration-files/library-structures.html) | How to structure declaration files and public APIs for libraries |
| [AWS Prescriptive Guidance -- Organise code for large-scale CDK projects](https://docs.aws.amazon.com/prescriptive-guidance/latest/best-practices-cdk-typescript-iac/organizing-code-best-practices.html) | Factory pattern, common/config/utilities folder layout |
| [AWS Prescriptive Guidance -- TypeScript best practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/best-practices-cdk-typescript-iac/typescript-best-practices.html) | Strict mode, TDD, ESLint for CDK TypeScript |
| [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html) | Naming, visibility, module organisation, exports |
| [Airbnb JavaScript Style Guide](https://github.com/airbnb/javascript) | ES6 modules, naming, grouping related functionality |
| [Zignuts -- TypeScript-First Node.js Backend Architecture 2026](https://www.zignuts.com/blog/typescript-first-nodejs-backend-architecture-2026) | Layered domain/api/infrastructure structure |
| [Clean Code TypeScript](https://labs42io.github.io/clean-code-typescript/) | SOLID, functions, classes, testing |

# Test-Driven Development

Follow the **Red -> Green -> Refactor** cycle strictly -- one test at a time.

## Cycle

1. **Red** -- write a failing test that describes the desired behaviour. Run it and confirm it fails with a clear, expected error. Never skip this step.
2. **Green** -- write the minimum production code needed to make the test pass. Nothing more.
3. **Refactor** -- improve structure, naming, and clarity while keeping all tests green.

> For AI agents: never write implementation code before a failing test exists. Writing tests after the fact produces tests that confirm the code as written, not the behaviour that was required.

## F.I.R.S.T. properties

Every test must be:

| Property | Meaning |
|---|---|
| **Fast** | Runs in milliseconds. Slow tests stop being run. Mock I/O and network to keep them fast. |
| **Independent** | No test depends on another. Each sets up and tears down its own state. |
| **Repeatable** | Produces the same result in any environment -- local, CI, offline. No reliance on external services. |
| **Self-Validating** | Passes or fails without human inspection. No `console.log` comparisons. |
| **Timely** | Written before the production code, not after. |

## Test naming

Use the format `should [expected behaviour] when [condition]`:
- `should return error when credentials are missing`
- `should invoke model with system prompt when provided`

## Structure (AAA)

Every test follows Arrange -> Act -> Assert. One concept per test -- a single logical assertion per test case:

```typescript
it("should return greeting when invoked", async () => {
  // Arrange
  const model = new MockBedrockModel();
  const agent = new Agent({ model });

  // Act
  const result = await agent.invoke("Say hello");

  // Assert
  expect(result.lastMessage).toContain("hello");
});
```

## Isolation

- Mock all external dependencies: model API calls, file system, network, AWS SDK.
- Do **not** mock classes you own -- use real collaborators from your own codebase and mock only at the system boundary.

## Coverage

- Target **80% minimum** line coverage; enforce it in CI.
- Use mutation testing (Stryker) periodically to verify tests actually catch bugs, not just execute code.

## Anti-patterns

- Testing implementation details instead of observable behaviour
- Hardcoding return values to pass a single test, then moving on

# Design Patterns

Use the following patterns when adding or refactoring code:

## Creational

| Pattern | When to use |
|---|---|
| **Factory Method** | You need to create objects but want the caller to remain decoupled from the concrete class. The factory decides what to instantiate based on input or configuration. |
| **Abstract Factory** | You need to create families of related objects that must be used together (e.g. a UI toolkit that produces buttons, dialogs, and inputs all in the same theme). |
| **Singleton** | Exactly one instance must exist for the lifetime of the application -- typically for shared resources like a connection pool, logger, or configuration registry. Use sparingly; prefer dependency injection. |

## Structural

| Pattern | When to use |
|---|---|
| **Adapter** | You have an existing class with the wrong interface and you cannot modify it. Wrap it in an adapter that speaks the interface your code expects. |
| **Facade** | A subsystem is complex or has many moving parts. Expose a single simplified class that covers the common use cases and hides the internals. |
| **Decorator** | You want to add responsibilities to an object at runtime without subclassing. Chain decorators to compose behaviours (e.g. logging, caching, retry around a service call). |
| **Composite** | You have a tree of objects and want to treat individual items and groups of items uniformly (e.g. a file/folder hierarchy, a UI component tree). |
| **Proxy** | You need to control access to an object -- for lazy initialisation, access control, remote calls, or caching -- without changing the object's interface. |

## Behavioural

| Pattern | When to use |
|---|---|
| **Strategy** | A class needs to perform a task that can be done in multiple ways and the algorithm should be swappable at runtime. Extract each algorithm into its own class and inject the desired one. |
| **Observer** | One object changes state and an unknown number of other objects need to be notified automatically. Avoids tight coupling between producer and consumers. |
| **Command** | You need to encapsulate a request as an object -- useful for queuing work, supporting undo/redo, logging operations, or building a task pipeline. |
| **State** | An object's behaviour changes significantly based on its internal state and you want to avoid large `if`/`switch` chains. Each state becomes its own class. |
| **Iterator** | You want to traverse a collection without exposing its internal structure. In TypeScript, prefer the built-in `Iterable`/`for...of` protocol before writing a custom iterator. |
| **Template Method** | Requires inheritance -- avoid per the rules above. Prefer **Strategy** to achieve the same goal through composition. |

# Avoid

- Do not use class inheritance.
- Do not create classes to encapsulate one method call.
- Do not add templates or generics. Only use generics for Abstract Factories if needed.
