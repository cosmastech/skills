---
name: php-conventions
description: Personal PHP conventions enforced when creating or modifying PHP files. Covers strict types, function imports, testing philosophy, class design, and observability. Activate whenever working on PHP code.
---

# PHP Conventions

These are non-negotiable personal conventions unless explicitly overridden by the user.

## Language & Syntax

- **Strict types** — Every PHP file must declare `declare(strict_types=1);` at the top. If it is an existing file, you do not need to add it, but mention it to the user.

- **Prefer explicit falsy checks** — Use the narrowest comparison that matches the real condition, such as `=== null`, `=== ''`, or `=== 0`, instead of broad checks like `if (! $someValue)`. Negation is fine when the value is genuinely `bool` (for example, `if (! $isEnabled)`). This avoids surprising type coercion and makes intent clearer to humans and tools.

- **Import function calls and constants** — Always use function and constant imports rather than calling global functions inline:
   ```php
   use function array_key_exists;
   use function sprintf;
   use function count;

   use const JSON_THROW_ON_ERROR;
   ```

- **`#[Override]` attribute** — Always add `#[Override]` to methods that override a parent or implement an interface method.

- **`final readonly` classes** — Prefer `final readonly` by default. Drop `readonly` only when mutability is genuinely needed (e.g., mutable DTOs, test cases). Drop `final` only when extension is a deliberate design choice. If either worsens developer experience, relax — but call it out.

- **PHPStan generics & iterables** — Document iterables and generic types explicitly. Never mark iterables as `array<array-key, mixed>` — take the time to define the actual shape. Use `@phpstan-type` annotations when the type will likely be imported by other files:
   ```php
   /**
    * @phpstan-type OrderContext array{orderId: int, shopId: int, items: list<LineItemData>}
    */
   ```
   Reference the phpstan-type (`phpstan-type-import` if defined in another class) instead of duplicating the array shape.

- **Avoid unnecessary nullability** — Do not accept `null` as a function/method argument or make a property nullable unless null carries distinct domain meaning (e.g., "not yet set," "intentionally cleared"). When a parameter is nullable only for convenience or because the caller *might* not have a value, push the null-check to the call site and keep the signature non-nullable. The same applies to return types: prefer throwing or returning a dedicated "empty" value type over returning `null` when the absence isn't semantically meaningful. Unnecessary nullability spreads defensive `=== null` checks throughout the codebase and weakens type safety.

- **PHPStan discipline** — Do not add entries to PHPStan baseline files. Fix the errors properly. When a `@phpstan-ignore` is genuinely necessary, always include a parenthetical explanation of *why*:
   ```php
   // @phpstan-ignore argument.type (We have already verified the type above)
   ```
   A bare `@phpstan-ignore` without justification is not acceptable.

## Class Design & Dependencies

- **Single-use classes over service bloat** — Prefer small, focused classes that encapsulate one piece of functionality. These may be called "UseCases" or "Actions" depending on team convention, but when possible, name them as `-er` nouns that describe what they do: `OrderCreator`, `FinancialOutcomeDeterminer`, `RefundCalculator`. Avoid adding methods to already-large service classes — you will encounter many of these, and they may be acceptable as-is, but don't contribute to the sprawl.

- **Dependency injection over service location** — Avoid `resolve()`, `app()`, and similar service locator calls inside methods. These obscure dependencies and prevent unit testing. Acceptable only in rare infrastructure-layer bootstrap code — flag it to the user if encountered.

- **Prefer composition over deep inheritance** — Avoid complex inheritance hierarchies. Before creating an abstract class with children, ask: could one child be replaceable with another? What is the hierarchy intended to communicate? If the answer is unclear, prefer:
    - **Interfaces** with **traits** that fulfill them, giving concrete classes opt-in behavior.
    - **Coordinator/orchestrator classes** that accept an interface and act upon it, rather than embedding orchestration logic in an abstract parent.

    Deep inheritance chains make code harder to reason about, test, and extend. Flat, composable designs are almost always preferable.

- **Rich, contextual exceptions** — Exceptions should carry meaningful messages and contextual data. Custom exception classes are good, but they shine when they expose public properties with the relevant context — the entity being operated on, whether the failure is retryable, the external request/response that caused it, etc. This also makes testing easier since you can assert against those properties rather than parsing message strings.

    When helpful, add static factory methods that build a default message and set properties from the inputs:
    ```php
    final class ShopifyFailure extends RuntimeException
    {
        public function __construct(
            string $message,
            public readonly Response $response,
            public readonly bool $isRetryable = false,
        ) {
            parent::__construct($message);
        }

        public static function fromResponse(Response $response): self
        {
            return new self(
                message: sprintf('Shopify returned %d: %s', $response->status(), $response->body()),
                response: $response,
                isRetryable: $response->status() >= 500,
            );
        }
    }
    ```

- **Domain knowledge as comments** — When a planning doc, JIRA ticket, or external context reveals domain knowledge not obvious from the code, add it as a comment. This helps future developers and LLM agents. These are among the most valuable comments possible.

- **Not everything needs an interface** - If there is likely going to be exactly one instance of an implementation, an interface may not be necessary. Remember: hand-rolled mocks/spies/stubs/fakes used in testing are a second implementation. Not using an interface may potentially make testing harder, so you'll need to weigh these tradeoffs. Since this requires taste, you may need input from the user.

## Testing Philosophy

- **Prefer PHPUnit (unit) tests** — Unit tests are dramatically faster than feature tests. When writing *new* code, design it to be unit-testable: use dependency injection, avoid Laravel facades in business logic, accept interfaces. If code is untestable without booting Laravel, that's a design signal worth discussing.

- **Avoid mocking frameworks** — Mocking frameworks (Mockery, PHPUnit mocks) usually test the wrong layer and indicate a design flaw. Acceptable uses:
    - **Stubs**: providing a canned return value from a dependency
    - **Classical mocks**: verifying a specific interaction that *is* the point of the test (e.g., "did we dispatch this event?")

    If you find yourself reaching for a mock, pause and discuss refactoring opportunities with the user before proceeding. Mocks are a last resort.

- **NEVER mock Data objects. NEVER mock Laravel models.** — Instantiate them. If a Data object or Model is hard to construct, that's a test-design or code-design problem to address, not a reason to mock.

- **Test structure** — Delimit test sections with comments:
    ```php
    // Given a user without permissions
    $user = User::factory()->create(['role' => 'viewer']);

    // When they attempt to update settings
    $response = $this->actingAs($user)->put('/settings', ['theme' => 'dark']);

    // Then the request is forbidden
    $response->assertForbidden();
    ```
    Add a brief description after `Given`, `When`, or `Then` when it clarifies intent.

- **Test method naming** - The method of a test name should be `{condition}_{theMethodBeingCalled}_{then}` and should use the `#[Test]` attribute rather than prefixing the method with `test_`. Be descriptive, but avoid making the test method name more than 60 characters.

- **Reference assertion methods statically** - PHPUnit allows you to call either `$this->assertEquals()` or `self::assertEquals()`. Unless the rest of the test class is already using `$this->assert*`, prefer calling them statically.

- **Mark tests as final** - This is the recommendation from PHPUnit's creator.

- **Never test private/protected methods directly** — If you feel the need to test a private or protected method on code we own, that's a design smell. Extract the logic into a collaborator class with public methods and test that instead. The need to reach into private internals means the class is doing too much or the boundaries are wrong.

- **Don't assert against log statements** — Unless you're testing a logger or logging infrastructure itself, avoid asserting that specific log messages were produced. Logs are observability, not behavior.

## Working in Existing Code

- **Accept the team's conventions** — When modifying existing code, follow the conventions already in place in that area. You may *mention* that refactoring opportunities exist and how the current code conflicts with these rules, but deliver value first. Being opinionated is secondary to shipping.

## Observability

- **Metrics cardinality** — Be mindful of high-cardinality tags on metrics (e.g., user IDs, order IDs). These explode storage costs and degrade query performance.
