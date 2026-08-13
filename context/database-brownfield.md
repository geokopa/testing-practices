# Testing a Brownfield .NET Application with NHibernate and Stored Procedures

You should not run automated tests against shared Dev, QA, or Production databases. Shared databases make tests nondeterministic, create collisions between developers, and can damage valuable data.

For this architecture, the best default is:

> Run DAL and stored-procedure integration tests against a temporary, isolated database created from the same database code as the current Git branch.

Because important logic lives in stored procedures, mocking the database would bypass the exact behavior you need to verify.

## What should be unit-tested vs. integration-tested?

### Unit-test code outside the database

Unit tests are appropriate for DAL-adjacent C# behavior such as:

- Translating application parameters into a stored-procedure call
- Mapping a result set into a domain object
- Handling empty results
- Converting database errors into application exceptions
- Retry or timeout policy
- Application calculations performed after retrieving data
- Choosing which repository operation to execute

You can mock an NHibernate `ISession` or, preferably, an application-owned abstraction such as:

```csharp
public interface IOrderRepository
{
    Task<OrderSummary?> GetSummaryAsync(
        int orderId,
        CancellationToken cancellationToken);
}
```

However, such tests cannot prove that:

- The stored procedure exists.
- Its parameters match the application.
- The SQL compiles.
- NHibernate maps its result correctly.
- Its calculations are correct.
- Transactions behave correctly.
- Database-specific null, decimal, collation, or date behavior is correct.

### Integration-test database behavior

Use a real database integration test for:

- Stored-procedure calculations
- Stored-procedure data manipulation
- NHibernate mappings
- Named queries
- Parameter types and directions
- Multiple result sets
- Transactions
- Constraints and triggers
- Concurrency behavior
- Database-specific semantics

Calling a mocked stored procedure is not a test of the stored procedure. It tests only what you instructed the mock to return.

## Recommended architecture

Each developer or CI test run should have its own disposable database:

```text
Git branch
   │
   ├── Application code
   ├── NHibernate mappings
   ├── Database schema and stored procedures
   └── Test scenarios and seed data
             │
             ▼
   Temporary isolated database
             │
             ├── Deploy branch's schema
             ├── Insert minimal test data
             ├── Execute tests
             └── Destroy database
```

Examples of database names might be:

```text
MyApp_Test_george_4f9c2
MyApp_Test_PR_184_7ab91
MyApp_Test_CI_96381
```

Two branches can then run simultaneously without interacting:

```text
feature/order-discount → MyApp_Test_OrderDiscount_123
feature/customer-grid  → MyApp_Test_CustomerGrid_456
```

Each receives exactly the database version checked into its branch.

## The database must be versioned with the application

The main underlying problem is not the database size. It is whether the database definition is reproducible.

The repository should contain the authoritative definitions for:

- Tables
- Views
- Stored procedures
- Functions
- User-defined types
- Triggers
- Indexes and constraints
- NHibernate mappings
- Database migrations or deployment scripts
- Reference-data scripts
- Test-data builders

For SQL Server, this is commonly implemented using either:

1. A SQL Database Project producing a `.dacpac`, or
2. Ordered migration scripts using a migration tool.

A database project can be built and deployed to a new isolated database. Microsoft recommends integrating SQL projects into CI/CD so the database definition is built and deployed consistently.

- [SQL Projects automation](https://learn.microsoft.com/en-us/sql/tools/sql-database-projects/sql-projects-automation?view=sql-server-ver17)

If your database is currently managed by people manually changing Dev and emailing scripts, introducing database source control is the most important improvement you can make.

### Database project approach

A typical repository could contain:

```text
src/
  MyApp/
  MyApp.Database/
    Tables/
    Views/
    Stored Procedures/
    Functions/
    MyApp.Database.sqlproj

tests/
  MyApp.UnitTests/
  MyApp.DatabaseIntegrationTests/
    Scenarios/
    SeedData/
```

The test pipeline would:

1. Build `MyApp.Database.sqlproj`.
2. Produce `MyApp.Database.dacpac`.
3. Create a unique empty database.
4. Publish the DACPAC.
5. Insert minimal scenario data.
6. Configure NHibernate to use that database.
7. Run integration tests.
8. Drop the database.

Microsoft also documents testing stored procedures by deploying a database project into an isolated development database.

- [SQL Server database unit testing](https://learn.microsoft.com/en-us/sql/ssdt/walkthrough-creating-and-running-a-sql-server-unit-test?view=sql-server-ver17)

### Migration-based approach

If you use migrations, each branch contains its own sequence:

```text
001_CreateCustomers.sql
002_CreateOrders.sql
003_AddOrderDiscount.sql
004_UpdateCalculateOrderTotal.sql
```

A new test database starts empty, then all migrations from that branch are applied.

This validates two things:

- A new database can be constructed.
- The stored procedures and application code in that branch are compatible.

You should additionally test upgrading a database from the currently released version. Building an empty database and upgrading an existing database expose different classes of problems.

## Do not reproduce the entire production dataset

A huge production database does not mean every test database must also be huge.

Separate these two concepts:

- **Schema and database code:** should normally be complete.
- **Data:** should be small and specific to each test.

For example, testing a stored procedure that calculates an order total might need:

```text
1 customer
2 products
1 order
3 order lines
2 tax rules
1 discount rule
```

It does not need 400 million production rows.

A test should create the smallest dataset that expresses its scenario.

```csharp
[Fact]
public async Task CalculateOrderTotal_PreferredCustomer_AppliesDiscount()
{
    // Arrange
    var customerId = await TestData.CreateCustomer(
        preferred: true);

    var orderId = await TestData.CreateOrder(
        customerId,
        new OrderLineSeed(productId: 101, quantity: 2, price: 50m));

    // Act
    var result = await _repository.CalculateOrderTotalAsync(orderId);

    // Assert
    Assert.Equal(90m, result.Total);
}
```

This test owns the rows it creates. It does not depend on a customer that happens to be present in Dev.

## Test data should be expressed as scenarios

Avoid one enormous shared `TestData.sql` file containing thousands of mysterious rows. It becomes another fragile legacy database.

Prefer scenario builders:

```csharp
var customer = await DatabaseScenario
    .Customer()
    .AsPreferred()
    .WithCreditLimit(10_000m)
    .CreateAsync();

var order = await DatabaseScenario
    .OrderFor(customer)
    .WithLine("PRODUCT-A", quantity: 2, price: 50m)
    .CreateAsync();
```

Alternatively, use small SQL scripts:

```text
Scenarios/
  CalculateOrderTotal/
    preferred-customer.sql
    expired-discount.sql
    tax-exempt-customer.sql
```

Good test data is:

- Small
- Explicit
- Deterministic
- Owned by the test
- Free of production personal information
- Easy to understand when a test fails

## How to create isolated databases

The best mechanism depends on your database engine and infrastructure.

### Option 1: One database container per test suite

If your production system uses a container-compatible database edition, start a database container for the integration-test suite.

For SQL Server:

```text
Test run starts
  → start SQL Server container
  → deploy DACPAC/migrations
  → insert reference data
  → run tests
  → stop and remove container
```

Testcontainers for .NET can automate container lifecycle. This gives developers and CI runs independent database servers without managing shared infrastructure.

The important benefit is not Docker itself. The benefit is a reproducible, private database lifecycle.

Considerations include:

- Container startup has a cost, so usually use one container per test suite, not per test.
- Database edition and version should be compatible with Production.
- Some enterprise-only features may not be available in the chosen image.
- Host architecture can matter. Microsoft notes that SQL Server Linux container images are supported on x86-64 hosts and that emulation environments are not supported.

- [SQL Server containers](https://learn.microsoft.com/en-us/sql/linux/containers/deploy?view=sql-server-ver16)

### Option 2: One database per test run on a dedicated test server

If containers are unavailable, use a dedicated SQL Server instance, but create a separate database for every run:

```text
Shared server:
  MyApp_Test_PR_101
  MyApp_Test_PR_102
  MyApp_Test_George_FeatureA
  MyApp_Test_Maria_FeatureB
```

The server is shared, but databases and users are isolated.

This is acceptable if:

- Each run receives a unique database name.
- Credentials are limited to that database.
- Tests never use three-part names to access other databases.
- Cleanup removes expired databases.
- Resource contention is monitored.
- Tests do not depend on server-global state.

This is much safer than having everyone modify one `MyApp_Dev` database.

### Option 3: LocalDB or a local database installation

For SQL Server on Windows, LocalDB can work for developer integration tests. Confirm that your stored procedures do not depend on features unavailable in LocalDB.

CI should still use a reproducible isolated database.

### Option 4: Database copy/template

For a large, difficult-to-create schema, maintain a sanitized database template containing:

- The schema
- Stored procedures
- Required reference data
- No production personal data
- No mutable business transactions

Each test run creates a database from that template and then applies the branch’s migrations.

This can reduce setup time, but it introduces a serious requirement:

> The template must have a known version, and the branch must be able to migrate it deterministically.

A randomly refreshed copy of Dev is not a valid template.

## Branch-specific database changes

Consider two branches:

```text
main
  └── migration 100

feature-A
  └── migration 101A: modifies CalculateInvoice

feature-B
  └── migration 101B: adds customer classification
```

Each test run should create its database from its own branch:

```text
feature-A tests → schema 100 + 101A
feature-B tests → schema 100 + 101B
```

Neither developer needs the other unmerged database change.

When both branches merge, migration conflicts must be resolved just like C# conflicts. The merged pipeline creates another new database and applies the combined definition.

Database objects should therefore travel in the same pull request as the code that requires them:

```text
Pull request:
  Application changes
  NHibernate mapping changes
  Stored-procedure changes
  Schema migration
  Tests
```

Do not merge application code that assumes a stored-procedure change without including or referencing that database change.

## A practical integration-test fixture

A conceptual xUnit fixture might look like this:

```csharp
public sealed class DatabaseFixture : IAsyncLifetime
{
    public string ConnectionString { get; private set; } = null!;
    public ISessionFactory SessionFactory { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        // Start a database container or create a unique database.
        ConnectionString = await TestDatabase.CreateAsync();

        // Deploy the schema and stored procedures from this branch.
        await DatabaseDeployment.DeployAsync(ConnectionString);

        // Insert immutable reference data required by most procedures.
        await ReferenceData.SeedAsync(ConnectionString);

        // Build the same NHibernate configuration used by the application,
        // with only the connection string replaced.
        SessionFactory =
            NhConfigurationFactory.Build(ConnectionString);
    }

    public async Task DisposeAsync()
    {
        SessionFactory.Dispose();
        await TestDatabase.DestroyAsync(ConnectionString);
    }
}
```

A test then uses the real NHibernate mapping and stored procedure:

```csharp
public class InvoiceProcedureTests :
    IClassFixture<DatabaseFixture>
{
    private readonly DatabaseFixture _database;

    public InvoiceProcedureTests(DatabaseFixture database)
    {
        _database = database;
    }

    [Fact]
    public async Task CalculateInvoice_MultipleTaxRates_ReturnsCorrectTotal()
    {
        await using var scenario =
            await InvoiceScenario.CreateAsync(_database.ConnectionString);

        using var session = _database.SessionFactory.OpenSession();

        var result = await session
            .CreateSQLQuery(
                "EXEC dbo.CalculateInvoice :invoiceId")
            .SetParameter("invoiceId", scenario.InvoiceId)
            .SetResultTransformer(/* your mapping */)
            .UniqueResultAsync<InvoiceCalculation>();

        Assert.Equal(125.50m, result.Total);
    }
}
```

Use your existing NHibernate named queries or mappings rather than duplicating production query text in every test. Otherwise, the test might validate different SQL from the application.

## How to clean data between tests

There are several strategies.

### Transaction rollback

Start a transaction before the test and roll it back afterward:

```csharp
using var session = sessionFactory.OpenSession();
using var transaction = session.BeginTransaction();

try
{
    // Arrange, execute, and assert.
}
finally
{
    await transaction.RollbackAsync();
}
```

This is fast, but it works only when all tested work participates in that transaction and connection.

It becomes unreliable when stored procedures:

- Open autonomous or separate connections
- Explicitly commit transactions
- Use linked servers
- Send messages
- Start asynchronous work
- Interact with files
- Call external services
- Depend on transaction isolation behavior you are trying to test

Transaction rollback is an optimization, not the fundamental isolation boundary.

### Delete/truncate scenario data

After each test, delete the rows created by that test. This is workable but foreign keys and failed cleanup can make it fragile.

### Reset the database

Reset mutable tables between tests or test classes. Tools or custom scripts can delete data in dependency order and reseed identities.

### Database per test class or test

This provides the strongest isolation but costs more setup time.

A common compromise is:

- One database/container per test suite
- Reset mutable data between tests
- Disable parallel execution for tests sharing that database
- Run multiple suites in parallel using separate databases

## Parallel execution rules

If tests share one temporary database, they can still collide with each other.

Choose one of these models.

### Model A: Sequential database tests

Simplest for a brownfield system:

```text
One temporary database
One integration-test collection
Tests run sequentially
```

This may be slower but is easy to trust.

### Model B: Parallel tests with unique scenario identifiers

Every test creates records with unique keys:

```csharp
var testRunId = Guid.NewGuid();
```

Stored procedures must filter correctly by those identifiers. This is risky when procedures operate on global tables or broad ranges.

### Model C: Database per parallel worker

Create several temporary databases:

```text
Worker 1 → MyApp_Test_1
Worker 2 → MyApp_Test_2
Worker 3 → MyApp_Test_3
Worker 4 → MyApp_Test_4
```

This is usually the best route once test execution time becomes important.

Start sequentially. Parallelize only after the suite is reliable.

## Stored-procedure testing strategy

Your stored procedures deserve their own focused tests because they contain domain logic.

For each important stored procedure, cover:

- Normal case
- Boundary values
- Empty input/data
- Null values
- Invalid state
- Permission or ownership restrictions
- Decimal precision and rounding
- Date/time boundaries
- Multiple matching rows
- Duplicate data
- Concurrent modification, where applicable
- Expected rows modified
- Expected result sets
- Expected error behavior

For a procedure that modifies data, assert both its output and its database effects:

```csharp
[Fact]
public async Task CloseOrder_OpenOrder_ChangesStatusAndCreatesAuditRecord()
{
    var order = await Scenario.CreateOpenOrderAsync();

    await Procedures.CloseOrderAsync(order.Id, userId: 42);

    var savedOrder = await Queries.GetOrderAsync(order.Id);
    var audit = await Queries.GetOrderAuditAsync(order.Id);

    Assert.Equal(OrderStatus.Closed, savedOrder.Status);
    Assert.Single(audit);
    Assert.Equal("Order closed", audit[0].Action);
}
```

Do not assert only that the procedure returned `0`. Verify the business outcome.

## Reference data

Legacy stored procedures often assume large reference tables exist:

- Country codes
- Tax tables
- Status definitions
- Product classifications
- Configuration entries

Separate reference data into two categories.

### Required invariant reference data

Version it alongside the schema and seed it automatically:

```text
Database/
  ReferenceData/
    OrderStatuses.sql
    TaxCategories.sql
    SystemConfiguration.sql
```

### Scenario-specific transactional data

Create it inside the test.

Avoid importing all Dev data merely because a procedure expects a few configuration rows. Identify and version those dependencies explicitly.

## What about testing against production-like volumes?

Small integration tests validate correctness, but they do not reveal all performance problems.

Use a separate performance-test strategy:

- Dedicated, isolated environment
- Sanitized or synthetically generated large dataset
- Production-like indexes and statistics
- No concurrent manual testing
- Explicit performance thresholds
- Scheduled or pre-release execution rather than every unit-test run

Examples:

- Procedure completes in under two seconds for one million orders.
- Paging does not scan the entire table.
- Execution plan uses an expected index.
- Memory and `tempdb` usage remain acceptable.

Do not mix performance tests with ordinary deterministic integration tests.

A practical split is:

| Suite | Data size | Frequency |
|---|---:|---|
| Unit | None | Every build/local run |
| Database integration | Small scenario data | Every pull request |
| Schema upgrade | Representative prior schema | Every pull request or nightly |
| Performance | Large synthetic/sanitized data | Nightly or before release |
| QA acceptance | Broader environment data | Before release |

## External database dependencies

Stored procedures sometimes access:

- Linked servers
- Other databases
- Reporting databases
- Queues
- File shares
- External CLR code

These complicate disposable testing.

Recommended options, in order:

1. Make the dependency another isolated test database.
2. Provide a test implementation behind a synonym, view, or configuration boundary.
3. Stub only the external boundary while keeping the procedure under test real.
4. Run a smaller, separate environment-level suite for behavior that truly cannot be isolated.

Do not silently allow integration tests to call shared or Production systems.

Use credentials with strict permissions so an incorrectly configured test physically cannot connect to Production.

## Safety protections

Add multiple safeguards because connection-string mistakes happen.

### Separate test configuration

```json
{
  "Database": {
    "AllowDestructiveTestOperations": true
  }
}
```

Do not enable that setting in normal application configuration.

### Validate the database before tests

```csharp
private static void ValidateTestDatabase(string connectionString)
{
    var builder = new SqlConnectionStringBuilder(connectionString);

    if (!builder.InitialCatalog.StartsWith(
            "MyApp_Test_",
            StringComparison.OrdinalIgnoreCase))
    {
        throw new InvalidOperationException(
            "Integration tests may run only against MyApp_Test_* databases.");
    }
}
```

Also consider verifying:

- Server name is on an allowlist.
- Database contains a test-only marker table.
- Login is a test-specific login.
- Production hostnames are explicitly rejected.
- Destructive setup requires both a test name and an environment flag.

### Restrict credentials

The test login should not have access to Dev, QA, or Production. A code bug then cannot damage those environments.

## A realistic brownfield adoption plan

Do not attempt to modernize the entire database before writing the first test.

### Phase 1: Stop using shared databases for automated tests

- Provision a dedicated database server or container capability.
- Create one unique database per developer/test run.
- Add connection-string safety checks.
- Run database tests sequentially initially.

### Phase 2: Put the tested database objects in source control

Start with one vertical slice:

```text
Orders tables
CalculateOrderTotal procedure
Required reference data
NHibernate mapping
Integration tests
```

You do not initially need to import every historical table if the chosen slice can be deployed independently. In tightly coupled schemas, you may need to import more of the schema, but still seed only minimal data.

### Phase 3: Establish a reproducible baseline

If the schema is too large or interdependent to build cleanly:

1. Create a sanitized baseline DACPAC, backup, or template database.
2. Give it an explicit version.
3. Restore it into a temporary database.
4. Apply migrations stored in the current branch.
5. Run tests.
6. Destroy the database.

This is often the most practical first step for a large brownfield database.

### Phase 4: Add characterization tests

Before changing an important stored procedure, write tests describing its current behavior—even if the behavior looks strange.

These tests protect you while refactoring. They are often called characterization tests.

For example:

```csharp
[Theory]
[InlineData("2026-01-01", 100.00)]
[InlineData("2026-06-01", 107.50)]
public async Task LegacyTaxCalculation_CurrentBehaviorIsPreserved(
    string invoiceDate,
    decimal expected)
{
    // Build the exact minimal scenario and call the real procedure.
}
```

Do not calculate the expected value by copying the stored procedure’s algorithm into C#. Use known business examples and fixed expected results.

### Phase 5: Add branch-aware CI

For every pull request:

```text
Build application
  → build database project
  → create isolated database
  → deploy branch database definition
  → seed reference data
  → run NHibernate/database integration tests
  → destroy isolated database
```

### Phase 6: Add upgrade testing and performance testing

Once clean-database testing is dependable:

- Test upgrading the last released schema to the branch schema.
- Add sanitized large-data performance runs.
- Add migration rollback/recovery procedures where applicable.

## Recommended starting architecture

Assuming SQL Server, choose this starting architecture:

1. Store procedure and schema definitions in a SQL Database Project or versioned migrations.
2. Build a versioned DACPAC or migration artifact from each branch.
3. Start one SQL Server container per integration-test suite, or create one uniquely named database on a dedicated test SQL Server.
4. Deploy the current branch’s database definition.
5. Seed only required reference data.
6. Create small scenario-specific data for each test.
7. Configure the real NHibernate mappings against this database.
8. Run stored-procedure/DAL tests sequentially at first.
9. Destroy the database after the run.
10. Maintain a separate large-data performance environment.

If reconstructing the database from source is currently impossible, use a versioned, sanitized baseline backup as an interim measure:

```text
Sanitized baseline v37
  + current branch migrations
  + minimal scenario data
  = isolated test database
```

That solves the immediate concurrency problem while you gradually move more database definitions under proper source control.

## Decisive principle

> Share the database server if necessary, but never share the mutable test database.

Dev and QA environments remain useful for exploratory and acceptance testing. They should not be the foundation of deterministic automated integration tests.
