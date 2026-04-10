---
name: php-conventions
description: Personal PHP conventions enforced when creating, modifying, or planning code that will touch PHP files. Covers strict types, function imports, testing philosophy, class design, observability, and planning practices. Activate whenever working on PHP code.
---

# PHP Conventions

These are non-negotiable personal conventions unless explicitly overridden by the user.

## Language & Syntax

1. **Strict types** — Every PHP file must declare `declare(strict_types=1);` at the top. If it is an existing file, you do not need to add it, but mention it to the user.

2. **Prefer explicit falsy checks** — Use the narrowest comparison that matches the real condition, such as `=== null`, `=== ''`, or `=== 0`, instead of broad checks like `if (! $someValue)`. Negation is fine when the value is genuinely `bool` (for example, `if (! $isEnabled)`). This avoids surprising type coercion and makes intent clearer to humans and tools.

3. **Import function calls and constants** — Always use function and constant imports rather than calling global functions inline:
   ```php
   use function array_key_exists;
   use function sprintf;
   use function count;

   use const JSON_THROW_ON_ERROR;
   ```

4. **`#[Override]` attribute** — Always add `#[Override]` to methods that override a parent or implement an interface method.

5. **`final readonly` classes** — Prefer `final readonly` by default. Drop `readonly` only when mutability is genuinely needed (e.g., mutable DTOs, test cases). Drop `final` only when extension is a deliberate design choice. If either worsens developer experience, relax — but call it out.

6. **PHPStan generics & iterables** — Document iterables and generic types explicitly. Never mark iterables as `array<array-key, mixed>` — take the time to define the actual shape. Use `@phpstan-type` annotations when the type will likely be imported by other files:
   ```php
   /**
    * @phpstan-type OrderContext array{orderId: int, shopId: int, items: list<LineItemData>}
    */
   ```
   Reference the phpstan-type (`phpstan-type-import` if defined in another class) instead of duplicating the array shape.

7. **PHPStan discipline** — Do not add entries to PHPStan baseline files. Fix the errors properly. When a `@phpstan-ignore` is genuinely necessary, always include a parenthetical explanation of *why*:
   ```php
   // @phpstan-ignore argument.type (We have already verified the type above)
   ```
   A bare `@phpstan-ignore` without justification is not acceptable.

## Class Design & Dependencies

8. **Single-use classes over service bloat** — Prefer small, focused classes that encapsulate one piece of functionality. These may be called "UseCases" or "Actions" depending on team convention, but when possible, name them as `-er` nouns that describe what they do: `OrderCreator`, `FinancialOutcomeDeterminer`, `RefundCalculator`. Avoid adding methods to already-large service classes — you will encounter many of these, and they may be acceptable as-is, but don't contribute to the sprawl.

9. **Dependency injection over service location** — Avoid `resolve()`, `app()`, and similar service locator calls inside methods. These obscure dependencies and prevent unit testing. Acceptable only in rare infrastructure-layer bootstrap code — flag it to the user if encountered.

10. **Prefer composition over deep inheritance** — Avoid complex inheritance hierarchies. Before creating an abstract class with children, ask: could one child be replaceable with another? What is the hierarchy intended to communicate? If the answer is unclear, prefer:
    - **Interfaces** with **traits** that fulfill them, giving concrete classes opt-in behavior.
    - **Coordinator/orchestrator classes** that accept an interface and act upon it, rather than embedding orchestration logic in an abstract parent.

    Deep inheritance chains make code harder to reason about, test, and extend. Flat, composable designs are almost always preferable.

11. **Rich, contextual exceptions** — Exceptions should carry meaningful messages and contextual data. Custom exception classes are good, but they shine when they expose public properties with the relevant context — the entity being operated on, whether the failure is retryable, the external request/response that caused it, etc. This also makes testing easier since you can assert against those properties rather than parsing message strings.

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

12. **Domain knowledge as comments** — When a planning doc, JIRA ticket, or external context reveals domain knowledge not obvious from the code, add it as a comment. This helps future developers and LLM agents. These are among the most valuable comments possible.

13. **Not everything needs an interface** - If there is likely going to be exactly one instance of an implementation, an interface may not be necessary. Remember: hand-rolled mocks/spies/stubs/fakes used in testing are a second implementation. Not using an interface may potentially make testing harder, so you'll need to weigh these tradeoffs. Since this requires taste, you may need input from the user.

## Testing Philosophy

14. **Prefer PHPUnit (unit) tests** — Unit tests are dramatically faster than feature tests. When writing *new* code, design it to be unit-testable: use dependency injection, avoid Laravel facades in business logic, accept interfaces. If code is untestable without booting Laravel, that's a design signal worth discussing.

15. **Avoid mocking frameworks** — Mocking frameworks (Mockery, PHPUnit mocks) usually test the wrong layer and indicate a design flaw. Acceptable uses:
    - **Stubs**: providing a canned return value from a dependency
    - **Classical mocks**: verifying a specific interaction that *is* the point of the test (e.g., "did we dispatch this event?")

    If you find yourself reaching for a mock, pause and discuss refactoring opportunities with the user before proceeding. Mocks are a last resort.

16. **NEVER mock Data objects. NEVER mock Laravel models.** — Instantiate them. If a Data object or Model is hard to construct, that's a test-design or code-design problem to address, not a reason to mock.

17. **Test structure** — Delimit test sections with comments:
    ```php
    // Given a user without permissions
    $user = User::factory()->create(['role' => 'viewer']);

    // When they attempt to update settings
    $response = $this->actingAs($user)->put('/settings', ['theme' => 'dark']);

    // Then the request is forbidden
    $response->assertForbidden();
    ```
    Add a brief description after `Given`, `When`, or `Then` when it clarifies intent.

18. **Test method naming** - The method of a test name should be `{condition}_{theMethodBeingCalled}_{then}` and should use the `#[Test]` attribute rather than prefixing the method with `test_`. Be descriptive, but avoid making the test method name more than 60 characters.

19. **Reference assertion methods statically** - PHPUnit allows you to call either `$this->assertEquals()` or `self::assertEquals()`. Unless the rest of the test class is already using `$this->assert*`, prefer calling them statically.

20. **Mark tests as final** - This is the recommendation from PHPUnit's creator.

21. **Never test private/protected methods directly** — If you feel the need to test a private or protected method on code we own, that's a design smell. Extract the logic into a collaborator class with public methods and test that instead. The need to reach into private internals means the class is doing too much or the boundaries are wrong.

22. **Don't assert against log statements** — Unless you're testing a logger or logging infrastructure itself, avoid asserting that specific log messages were produced. Logs are observability, not behavior.

## Working in Existing Code

23. **Accept the team's conventions** — When modifying existing code, follow the conventions already in place in that area. You may *mention* that refactoring opportunities exist and how the current code conflicts with these rules, but deliver value first. Being opinionated is secondary to shipping.

24. **No drive-by refactors in ticketed work** — Never make unrelated refactors inside a PR/MR tied to a ticket. Unrelated changes increase blast radius for bugs and make reviews harder. Instead:
    - During planning, identify code that would benefit from refactoring.
    - Propose a *separate* refactoring MR that ships independently, ahead of the feature work.
    - Follow the principle: **"Make the change easy, then make the easy change."**
    - This requires judgment — always flag it to the user.

## Observability

25. **Think about the 3 AM oncall engineer** — When writing code, imagine a sleep-deprived human investigating an incident involving this code. Add context to logs, use structured logging keys (following key structure per repo convention; if none is defined, default to snake-case), and make error paths descriptive.

26. **Metrics cardinality** — Be mindful of high-cardinality tags on metrics (e.g., user IDs, order IDs). These explode storage costs and degrade query performance.

27. **Suggest monitors** - If during coding there is an obvious opportunity for "this would make a great monitor to signal application health," you should mention it to the user. If you have awareness of the user's observablity platform, offer to help them construct the monitor. Features are not done until observability is in place.

## Planning

28. **Upfront planning docs** — When planning work, produce a planning document *before* writing code. Structure it for humans who will skim:
    - **TL;DR section at the top** — summary of approach, key decisions, risks.
    - Detailed sections below for those who want depth.

29. **Multi-model plan review** — If sub-agents are available, pass the plan to a different thinking model (ideally from a different provider) for review with *no context about your findings*. Iterate between multiple models until general consensus emerges. Challenge sub-agent feedback when it's wrong — consensus doesn't mean capitulation.

30. **Include failure scenarios** - Make sure to think of different failure scenarios, how we should recover from them (or IF we should attempt to recover from them), and how we will communicate those failures.

31. **Identify refactoring prerequisites during planning** — If the current code structure will make the planned change difficult, call this out early. Propose preparatory refactoring MRs that ship first.

32. **Plans should mention observability** - Iterate with the user to determine how the feature or code change can be observed and what monitors would indicate code health.
