# Testing ASP.NET Core MVC Applications: Unit, Integration, Functional, and End-to-End Tests

The most useful distinction is not “unit vs. functional vs. end-to-end” as three perfectly separate boxes.

- **Unit** describes isolation and scope.
- **End-to-end (E2E)** describes how much of the deployed system participates.
- **Functional** describes what is being verified: externally observable behavior.

A functional test might be an API-level integration test or a browser-based E2E test. Teams often use “functional test” to mean “automated browser test,” but that is convention rather than a strict definition.

## The practical testing spectrum

| Test type | Starts at | Uses real database? | Uses browser? | Typical speed | Main question |
|---|---|---:|---:|---:|---|
| Unit | C# method/class | No | No | Milliseconds | Is this business rule correct? |
| Integration | Service, repository, controller, HTTP endpoint | Often | Usually no | Milliseconds–seconds | Do these components work together? |
| Functional/API | HTTP request or public interface | Usually | Not necessarily | Seconds | Does this user-facing capability behave correctly? |
| E2E/UI | Browser | Usually | Yes | Seconds–minutes | Can a real user complete this workflow? |

Microsoft defines good unit tests as fast, isolated, repeatable, and independent of infrastructure. Its ASP.NET Core integration-testing guidance uses `WebApplicationFactory<TEntryPoint>` and `TestServer` to run the real application pipeline without needing a browser.

- [Microsoft unit-testing guidance](https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices)
- [ASP.NET Core integration tests](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-10.0)

## 1. Unit tests

A unit test exercises one meaningful unit of behavior while replacing external dependencies.

That unit does not have to be one method. It might be:

- A domain service
- A validator
- A pricing policy
- A query/filter translator
- A permission rule
- A mapping function
- A controller action, although controllers are frequently more valuable at the integration level

A unit test should not normally use:

- SQL Server
- Entity Framework against a real database
- HTTP
- The file system
- The current clock directly
- A browser
- Kendo or AG Grid itself

### Real-world example: grid filtering rule

Suppose your grid sends a filter requesting active customers with a balance of at least $1,000. Your application translates that request into a domain/query specification.

```csharp
public sealed class CustomerFilterService
{
    public IQueryable<Customer> Apply(
        IQueryable<Customer> customers,
        CustomerFilter filter)
    {
        if (filter.ActiveOnly)
            customers = customers.Where(x => x.IsActive);

        if (filter.MinimumBalance is not null)
            customers = customers.Where(
                x => x.Balance >= filter.MinimumBalance.Value);

        return customers;
    }
}
```

A unit test can verify the rule without ASP.NET, JavaScript, or a database:

```csharp
public class CustomerFilterServiceTests
{
    [Fact]
    public void Apply_ActiveWithMinimumBalance_ReturnsMatchingCustomers()
    {
        var customers = new[]
        {
            new Customer { Name = "Alice", IsActive = true,  Balance = 1500m },
            new Customer { Name = "Bob",   IsActive = true,  Balance = 500m },
            new Customer { Name = "Carol", IsActive = false, Balance = 2000m }
        }.AsQueryable();

        var filter = new CustomerFilter(
            ActiveOnly: true,
            MinimumBalance: 1000m);

        var result = new CustomerFilterService()
            .Apply(customers, filter)
            .ToList();

        Assert.Single(result);
        Assert.Equal("Alice", result[0].Name);
    }
}
```

Other unit-test examples for your application:

- A user without `Orders.Edit` cannot change an order.
- An order over $10,000 requires supervisor approval.
- Invalid date ranges are rejected.
- A Kendo request’s filter descriptor maps to the appropriate internal filter.
- A grid column identifier maps only to approved sortable database fields.
- Monetary totals round according to your business rules.
- CSV export formatting handles commas, quotes, and nulls.
- A timezone conversion handles a daylight-saving boundary.
- An AG Grid request calculates the correct page from `startRow` and `endRow`.

### What to mock

Mock boundaries such as:

- Email/SMS provider
- External REST client
- Current time through `TimeProvider`
- Current user abstraction
- Message queue
- File storage

Avoid mocking every class merely because you can. Testing five mocks and verifying twenty calls often tests implementation structure rather than behavior.

> If a dependency executes important behavior that you own, consider using the real implementation. If it crosses an external boundary, replace it in a unit test.

## 2. Integration tests

An integration test verifies that multiple real components work together.

For ASP.NET Core MVC, this commonly means:

```text
HTTP request
  → middleware
  → authentication/authorization
  → model binding
  → controller
  → application service
  → EF Core
  → test database
  → HTTP response
```

You normally use:

- xUnit, NUnit, or MSTest
- `Microsoft.AspNetCore.Mvc.Testing`
- `WebApplicationFactory<Program>`
- A controlled test database
- Optionally Testcontainers for the same database engine used in production

### Real-world grid endpoint example

Imagine AG Grid calls:

```http
POST /api/customers/search
```

with sorting, filtering, and paging parameters. An integration test should verify that the complete server pipeline interprets that request correctly:

```csharp
public class CustomerSearchTests :
    IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public CustomerSearchTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Search_FilteredAndSorted_ReturnsCorrectPage()
    {
        var request = new
        {
            startRow = 0,
            endRow = 25,
            sortModel = new[]
            {
                new { colId = "balance", sort = "desc" }
            },
            filterModel = new
            {
                status = new
                {
                    filterType = "text",
                    type = "equals",
                    filter = "Active"
                }
            }
        };

        var response = await _client.PostAsJsonAsync(
            "/api/customers/search", request);

        response.EnsureSuccessStatusCode();

        var result =
            await response.Content.ReadFromJsonAsync<CustomerGridResult>();

        Assert.NotNull(result);
        Assert.True(result.Rows.Count <= 25);
        Assert.All(result.Rows, row => Assert.Equal("Active", row.Status));
        Assert.Equal(
            result.Rows.OrderByDescending(x => x.Balance),
            result.Rows);
    }
}
```

This catches problems that a unit test cannot:

- JSON property names do not match the AG Grid request.
- MVC model binding fails.
- A required authorization policy is missing.
- EF Core cannot translate a LINQ expression into SQL.
- Sorting treats numeric data as strings.
- Database collation changes case sensitivity.
- Pagination returns duplicate or missing records.
- A controller returns the wrong response shape.
- Serialization produces a date format the grid cannot consume.

### Use the production database engine when semantics matter

EF Core’s in-memory provider is not equivalent to SQL Server, PostgreSQL, or Oracle. It can hide issues involving:

- SQL translation
- Constraints
- Transactions
- Case sensitivity and collation
- Decimal precision
- Null handling
- Date/time behavior
- Raw SQL
- Unique indexes
- Concurrency tokens

For repository and complex grid-query tests, a disposable instance of your production database engine is generally more valuable than an in-memory substitute.

## 3. Functional tests

A functional test verifies that the system satisfies a business capability through a public interface.

Examples:

- Searching for a customer returns matching results.
- An authorized manager can approve an order.
- An invalid edit displays validation errors.
- Exporting a filtered grid downloads only filtered records.
- A user cannot view another tenant’s data.

A functional test may operate through:

1. An HTTP API, or
2. The web UI.

This is why “functional” overlaps with integration and E2E.

### API-level functional test

A test might:

1. Authenticate as a sales user.
2. Create a customer through an API.
3. Search for the customer.
4. Assert it appears.
5. Attempt an admin-only operation.
6. Assert the server returns `403 Forbidden`.

It tests a complete capability but does not prove that the Kendo or AG Grid UI is wired correctly.

API-level functional tests are often an excellent middle ground: much faster and less fragile than UI tests while covering most of the application stack.

## 4. End-to-end tests

An E2E test drives the application from the outermost user interface and uses the full system.

For your application:

```text
Real browser
  → MVC page
  → JavaScript
  → Kendo/AG Grid
  → HTTP
  → ASP.NET Core
  → database
```

For .NET, Playwright is generally the strongest default browser-testing option. It supports automatic waiting and resilient locators. Its documentation recommends user-facing locators such as role, label, text, and explicit test IDs rather than brittle CSS chains.

- [Playwright .NET locators](https://playwright.dev/dotnet/docs/locators)

### Real-world AG Grid E2E test

```csharp
[Test]
public async Task UserCanFilterCustomersAndOpenDetails()
{
    await Page.GotoAsync($"{BaseUrl}/Customers");

    await Page.GetByTestId("customer-grid-filter-name")
        .FillAsync("Alice");

    await Page.GetByRole(
            AriaRole.Button,
            new() { Name = "Apply filters" })
        .ClickAsync();

    var grid = Page.GetByTestId("customer-grid");

    await Expect(grid.GetByRole(AriaRole.Row))
        .ToHaveCountAsync(2); // header plus one data row

    await grid.GetByRole(
            AriaRole.Link,
            new() { Name = "Alice Smith" })
        .ClickAsync();

    await Expect(Page.GetByRole(
            AriaRole.Heading,
            new() { Name = "Alice Smith" }))
        .ToBeVisibleAsync();
}
```

The exact locators depend on the rendered markup and accessibility support of your grid configuration.

### Valuable Kendo/AG Grid E2E scenarios

Use E2E tests for things that genuinely require a browser:

- The grid initializes and loads data.
- Clicking a column header changes the visible sort order.
- A filter entered through the UI reaches the backend.
- Editing a cell sends the new value and displays the saved result.
- Client-side validation appears correctly.
- A server-side validation error is displayed in the grid.
- Row selection enables the correct toolbar buttons.
- Double-clicking a row opens the correct details view.
- Keyboard navigation works for an important accessibility workflow.
- Column menus, date pickers, and dropdown editors work together.
- Virtual scrolling requests subsequent blocks correctly.
- Export/download actually produces a browser download.
- Authentication redirects to login and returns to the intended page.
- A modal confirmation prevents or completes a destructive action.
- A JavaScript exception does not break page initialization.

### What not to test extensively through E2E

Do not create separate browser tests for every combination of:

- Every validation boundary
- Every role and permission
- Every grid filter operator
- Every sort direction
- Every page size
- Every calculation
- Every null/empty input

Those combinations belong mainly in unit or integration tests. Browser tests should prove that the system is connected and that critical journeys work.

## A complete example: editing an order in a grid

Suppose a user can edit an order quantity inline.

### Unit tests

Test the business rules:

- Quantity below 1 is invalid.
- Quantity above available stock is invalid.
- Total is recalculated correctly.
- A locked order cannot be modified.
- Only an appropriate role may request the change.

### Integration tests

Test the server behavior:

- The update endpoint accepts the grid’s JSON shape.
- Invalid model state returns the agreed error format.
- A nonexistent order returns `404`.
- An unauthorized user receives `403`.
- A valid update changes the database.
- The concurrency token prevents overwriting somebody else’s edit.
- The returned row contains the recalculated total.

### E2E tests

Test a few representative UI journeys:

- The user edits quantity, clicks Save, and sees the new total.
- An invalid quantity shows the error next to the row.
- A concurrent update displays a useful conflict message.
- A read-only user cannot see or activate the edit control.

Notice the distribution: perhaps 15–30 unit tests, 6–10 integration tests, and only 2–4 E2E tests for this feature.

## How to decide which test to write

Use this sequence.

### 1. Does the behavior require a browser to prove it?

Examples:

- JavaScript event wiring
- Grid rendering
- Popup behavior
- Drag/drop
- Client-side validation
- Keyboard interaction
- CSS visibility
- Browser downloads

If yes, write an E2E test.

If no, continue downward.

### 2. Does the risk occur at a boundary between components?

Examples:

- JSON/model binding
- MVC routing
- Authorization policies
- EF Core SQL translation
- Database constraints
- Serialization
- Dependency injection configuration
- Middleware behavior

If yes, write an integration or API-level functional test.

### 3. Is it a business rule that can be evaluated in memory?

Examples:

- Calculations
- Validation
- State transitions
- Permissions
- Mapping
- Filtering rules
- Formatting
- Decisions

If yes, write a unit test.

### 4. Is it extremely important?

For high-risk behavior, write tests at more than one level.

For example, tenant isolation should probably have:

- Unit tests for the tenant-access rule.
- Integration tests proving database queries are tenant-filtered.
- One or two E2E tests proving users cannot access another tenant’s records.

The tests are not redundant because each protects against a different failure.

## A useful risk-based matrix

| Failure being considered | Best starting test |
|---|---|
| Discount calculation is wrong | Unit |
| AG Grid request maps incorrectly | Unit plus integration |
| LINQ filter fails when executed by SQL Server | Integration |
| Endpoint allows the wrong role | Integration |
| Button is incorrectly enabled | E2E |
| Clicking Save calls the wrong endpoint | E2E |
| Server rejects valid edited data | Integration |
| Grid displays server validation incorrectly | E2E |
| Pagination skips records | Integration |
| Virtual scrolling does not request another block | E2E |
| Date conversion is wrong | Unit plus integration |
| Date picker emits an unexpected timezone value | E2E plus integration |
| Export contains incorrect records | Integration |
| Browser download is never initiated | One E2E |
| Database uniqueness constraint is missing | Integration |
| Required field rule is incorrect | Unit |
| Validation message is not displayed | E2E |

## Recommended test architecture

A practical solution might look like:

```text
src/
  MyApp.Web/
  MyApp.Application/
  MyApp.Domain/
  MyApp.Infrastructure/

tests/
  MyApp.UnitTests/
  MyApp.IntegrationTests/
  MyApp.EndToEndTests/
```

Typical tools:

- **Test runner:** xUnit, NUnit, or MSTest
- **Assertions:** built-in assertions or FluentAssertions
- **Mocks/stubs:** NSubstitute, Moq, or small handwritten fakes
- **ASP.NET integration:** `WebApplicationFactory<Program>`
- **Database:** Testcontainers or another isolated real database
- **Browser E2E:** Playwright for .NET
- **Coverage:** `dotnet test --collect:"XPlat Code Coverage"`

The particular unit-test framework matters much less than good boundaries and maintainable tests.

## Test data strategy

Reliable tests require controlled data.

### Unit tests

Construct only the data required by the scenario.

### Integration tests

Each test should begin from a known database state. Common approaches:

- Roll back a transaction after each test.
- Recreate or clean the schema between tests.
- Give each test an isolated database.
- Seed only scenario-specific data.

Avoid integration tests that depend on execution order.

### E2E tests

Give each test unique data, such as a customer name containing a generated identifier. Avoid relying on “the third row” unless row position is the behavior under test.

For tests running in parallel, separate data by:

- Tenant
- User
- Unique identifier
- Database/schema, when necessary

## Making grid E2E tests maintainable

Third-party grids generate complicated markup. Tests that use selectors such as:

```css
div.ag-root > div:nth-child(2) > div > div:nth-child(4)
```

will be extremely fragile.

Prefer:

1. Accessible roles and names
2. Visible labels
3. Stable application-owned `data-testid` attributes
4. Grid API helpers only when the behavior cannot be accessed cleanly as a user would

Example Razor markup:

```html
<div id="customer-grid"
     data-testid="customer-grid">
</div>
```

If you need to identify a business row, render or expose a stable identifier:

```html
<a data-testid="customer-link-@customer.Id">
    @customer.Name
</a>
```

Do not use generated DOM class names as your primary testing contract.

Also avoid fixed delays:

```csharp
await Page.WaitForTimeoutAsync(3000);
```

Wait for an observable outcome instead:

```csharp
await Expect(
    Page.GetByText("25 customers found"))
    .ToBeVisibleAsync();
```

## How many tests of each kind?

There is no correct percentage, but the shape should usually be:

```text
Many fast unit tests
A moderate number of integration/API tests
A small, carefully selected E2E suite
```

For a business-heavy MVC application, integration tests may provide more value than unit tests for thin controllers and repositories. Do not force everything into a textbook testing pyramid.

A reasonable starting policy:

- Unit-test meaningful business rules and edge cases.
- Integration-test every important endpoint, authorization boundary, and nontrivial database query.
- E2E-test each critical user journey and a small number of representative error paths.
- Add a regression test at the lowest level capable of reproducing every production bug.

That last point is especially useful: if a defect can be reproduced in a unit test, do not rely solely on a slow browser test.

## What I would test first in your application

For an existing ASP.NET MVC application with Kendo and AG Grid, prioritize:

1. **Authorization and tenant isolation** at the integration level.
2. **Server-side grid filtering, sorting, paging, and grouping** using a real database.
3. **Business validation and calculations** with unit tests.
4. **One E2E smoke test per critical screen** proving the grid loads.
5. **Critical grid journeys** such as search, edit, save, delete, export, and error display.
6. **Regression tests** for historically troublesome date, decimal, null, and concurrency behavior.
7. **A small production-like smoke suite** that runs after deployment.

## Central principle

> Test each behavior at the lowest level that can detect the failure with confidence, then add higher-level tests only for important wiring and user journeys.

This gives you fast diagnosis from unit tests, confidence in ASP.NET/database integration, and a focused E2E suite proving that Kendo and AG Grid actually work from the user’s perspective.
