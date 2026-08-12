## Test-Type Decision Matrix

| Signal in the code/feature | Unit | Integration | Functional | E2E |
|---|---|---|---|---|
| Pure logic, no I/O once dependencies mocked (calculations, validation rules, mapping, branching) | **Yes — primary** | No | No | No |
| Correctness depends on real DB engine behavior (dynamic LINQ, raw SQL, joins, EF query translation, case sensitivity, paging edge cases) | No (mocked repo hides the real bug) | **Yes — primary** | Maybe (if you want it via HTTP too) | No |
| Verifying HTTP contract: routing, model binding, status codes, JSON/response shape | No | No | **Yes — primary** | No |
| Verifying auth/authorization middleware, filters, headers | No | Maybe | **Yes** | No |
| Verifying Kendo/AG-Grid request format is parsed correctly server-side (DataSourceRequest, IServerSideGetRowsRequest) | No | Maybe | **Yes — primary** | No |
| Verifying grid renders, JS executes, async load completes, user can filter/sort/edit in the actual browser | No | No | No | **Yes — primary** |
| Cross-service workflow (e.g., order service calls pricing service calls inventory service) | No | **Yes — primary** | Maybe | No |
| Third-party API integration (payment gateway, external data provider) | No (mock the client) | **Yes** (contract test / sandbox) | No | No |
| Regression for a specific past production bug | Depends on layer where bug occurred — write it there | | | |
| CSS/layout/visual correctness | No | No | No | **Yes** (visual regression, separate tool ideally) |
| Message queue / background job processing | No (mock the queue) | **Yes** | Maybe | No |
| Full critical user journey (login → filter grid → edit → save → verify persisted) | No | No | No | **Yes — but sparingly** |

## Decision tree (walk top to bottom, stop at first "yes")

1. **Can I test this with everything mocked, and does that mocked test still catch the bugs I care about?**
   → Yes: **Unit test.** This is true for almost all business logic, calculations, validators, mappers, controller-level branching (given service output X, controller returns Y).
   → No, because the real behavior of a dependency (DB, external API, message broker) is what's actually being tested → go to 2.

2. **Does the bug live in how components interact with real infrastructure (query translation, transaction behavior, concurrency, actual schema constraints)?**
   → Yes: **Integration test**, real dependency via Testcontainers/sandbox, not a fake/in-memory substitute.
   → No, if what you actually care about is the app's HTTP-level contract → go to 3.

3. **Does correctness require going through the actual ASP.NET pipeline (middleware, model binding, routing, serialization) but not a browser?**
   → Yes: **Functional test** via `WebApplicationFactory`.
   → No, if what you care about is whether a human can actually complete the task in the rendered UI → go to 4.

4. **Does correctness depend on JS execution, DOM rendering, or third-party grid widget behavior (Kendo/AG-Grid rendering, async data load, client-side filtering)?**
   → Yes: **E2E**, and only for the critical path, not exhaustively.
   → No → you've mis-scoped the test; go back to step 1.

## Coverage ratio to target (pyramid, not diamond, not inverted pyramid)

- **70% unit** — cheap, fast, pinpoint failures. Every business rule and branch.
- **20% integration** — every distinct real-infrastructure interaction pattern (grid query variants, cross-service calls).
- **8% functional** — one happy path + one failure path per endpoint. You're confirming wiring already covered logically by unit tests.
- **2% E2E** — a short list of workflows where a regression = a real incident. Not per-feature, not per-grid-column.

If you find yourself writing E2E tests to catch logic bugs, or unit tests that mock so much they test nothing but the mock, you've picked the wrong layer — push the test down the pyramid to where the actual risk lives.

## Common misassignments people make in exactly your stack

- Testing grid query logic (filter/sort/paging translation to SQL) only via E2E → **wrong**, move it to integration against a real DB; E2E won't tell you *why* the wrong rows came back.
- Testing controller status-code logic via `WebApplicationFactory` instead of a unit test with mocked service → **wasteful**, slower for no added confidence; unit test it.
- Skipping integration tests because "EF in-memory provider passed" → **false confidence**; in-memory provider doesn't enforce constraints or translate LINQ the way real SQL Server/Postgres does.
- Writing E2E for every grid column/filter combination → **maintenance tax with no proportional value**; cover combinations at the integration layer, use E2E only for one or two representative interactions.
