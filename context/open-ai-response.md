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
