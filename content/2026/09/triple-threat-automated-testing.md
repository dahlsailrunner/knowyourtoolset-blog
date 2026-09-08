---
title: "TUnit, Aspire, and Playwright: A Powerful Triple Threat for Automated Testing and Agentic Workflows" 
date: 2026-09-05T15:29:04-05:00 
summary: "Setting up an amazing automated testing framework for a distributed application can improve both your quality and speed for human- and agent-based delivery." 
featured: true 
toc: true
thumbnail: "/images/triple-threat/tunit-report-trace.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/triple-threat/tunit-report-trace.png" # Designate a separate image for social media sharing.
codeMaxLines: 15 
codeLineNumbers: false 
figurePositionShow: true 
tags:
  - .NET
  - ASP.NET
  - C#
  - Aspire
  - TUnit
  - Playwright
  - GitHub Actions
  - Code Coverage
  - MTP
---

How about an awesome automated testing framework that:

* Allows full testing **without deploying the application**
* Has **complete logs and OpenTelemetry traces from the application** for each test on its own
* Provides **detailed code coverage** information
for all code in your application
* Includes **screenshots and video recordings** of
automated tests of your UI easily
* Can be run both **locally and in a continuous
integration (CI) pipeline** easily
* Is **super fast**
* ***Enables AI-coding agent iteration that results in successful implemetation of features that include tests and video recordings of the updated application***

Everything in this post is working code in the
[automated-testing-aspnetcore10](https://github.com/dahlsailrunner/automated-testing-aspnetcore10)
repo. The app under test ("CarvedRock Fitness") is a distributed ASP.NET Core 10
application: a Razor Pages front end, a REST API, an MCP server, an AI agent,
Postgres, and a mail server -- all composed with Aspire.

To set expectations on the "super fast" claim up front, here's what the full
suite looks like today: **54 tests across three projects, and about 32 seconds
of wall clock** from the first test starting to the last one finishing. That
includes standing up the *entire* distributed application in real containers
and driving Chromium browser sessions through it.

| Test project | Tests | Duration |
| --- | --- | --- |
| `CarvedRock.UnitTests` | 20 | 0.43 s |
| `CarvedRock.ApiTests` | 16 | 8.5 s |
| `CarvedRock.AppTests` | 18 | 40.7 s |

The three projects run concurrently, which is why the wall clock is shorter
than the sum -- and why the `AppTests` number (which is dominated by Aspire
startup) is the one that actually gates the run.

## Overall Setup

The following is what has been set up in a sample repo and is
what I think the building blocks for an amazing test automation
framework is.

1. A TUnit test project for your unit tests
1. A separate TUnit test proejct for each ASP.NET Core
  app you want to test on its own (with `WebApplicationFactory`).
1. A TUnit test project for the entire application that
  will use the Aspire `DistributedApplicationTestingBuilder`.
1. When running tests:
    1. Create coverage outputs
    1. Use a `testconfig.json` file to provide exclusions for code coverage reporting (e.g. source generated content)
    1. Specify output directories for output files
1. Create a coverage report using the excellent [reportgenerator](https://reportgenerator.io/) dotnet tool
1. Create a script that can run all of your tests locally that will:
    1. Delete artifacts from previous runs
    1. Run all of the tests with the above settings
    1. Create the coverage report based on the test run output
    1. Optionally open each of the reports for human review
1. Create a continuous integration (CI) pipeline that will do
  all of the above, include good information in the pipeline
  summary, and provide the ability to download and review the
  detailed artifacts and reports from the test run

One piece of required plumbing: a `global.json` in the root of
the repo that opts into Microsoft Testing Platform (MTP) as the test runner.

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

With that in place, `dotnet test` accepts platform options directly --
`--coverage`, `--coverage-settings`, `--treenode-filter`, `--report-trx` --
rather than needing a `.runsettings` file and VSTest adapter indirection. That
single file is what lets the local script and the CI pipeline share essentially
the same command.

> **Why three test projects instead of one?**
> Mostly because of top-level statements. Referring to `Program` from more than
> one `WebApplicationFactory<Program>` target gets ambiguous, and rather than
> abandon top-level statements or fight the tooling, separate projects are the
> path of least resistance. It turns out to be a happy accident: each project has
> a totally different cost profile and set of dependencies, so you can run just
> the fast ones during a tight edit loop.

## Why TUnit as the Testing Framework?

There are a LOT of reasons to like TUnit as the backbone
of the automated testing framework for your application(s):

* Adds simplicity and features to both `WebApplicationFactory`
  and `DistributedApplicationTestingBuilder`
* Fast - uses source-generation and not reflection; aggressive
  parallelization by default; can be compiled for AOT for even more speed
* Creates an excellent "results report" that shows test execution
  details - how the tests were run - as well as showing logs, traces, and other information about each test individually
* Uses Microsoft Testing Platform and simplifies capture of
  code coverage details
* Easy-to-use API that keeps tests simple but doesn't limit
  capabiltiies
* "Batteries included" approach: built-in mocking (including HTTP mocking) and first class integration with Playwright

That first bullet is the one that ties this whole post together. The three
"threats" in the title aren't three separate tools you glue together yourself --
TUnit ships first-class integration packages for each of them:

| Package | What it gives you |
| --- | --- |
| `TUnit` | The framework: source-generated discovery, assertions, hooks, parallelism |
| `TUnit.Mocks` | Source-generated, AOT-friendly mocking (no Castle.Core proxies) |
| `TUnit.AspNetCore` | `WebApplicationFactory`-based fixtures and a test base class |
| `TUnit.Aspire` | Whole-distributed-app fixtures with resource waiting + OTel capture |
| `TUnit.Playwright` | Browser/context/page lifecycle managed for you |

A few API conventions show up in every snippet below, so they're worth calling
out once:

* `[Test]` replaces `[Fact]` / `[TestMethod]`
* `[Arguments(...)]` replaces `[InlineData(...)]`
* `[Before(Test)]`, `[Before(Class)]`, `[After(TestSession)]` are the hooks
* `[ClassDataSource<T>(Shared = SharedType.PerTestSession)]` is how expensive
  fixtures get created once and shared
* `[DependsOn(nameof(OtherTest))]`, `[NotInParallel]`, and
  `[ParallelLimiter<T>]` are the escape hatches from "everything runs in
  parallel by default"

Assertions are async and chainable, and a single await can check
multiple members of the same result.

```csharp
await Assert.That(result)
    .Member(r => r.Errors, errors => errors.IsEmpty())
    .And
    .Member(r => r.IsValid, valid => valid.IsTrue());
```

One small quality-of-life thing: rather than repeating `using` statements in
every test file, the common namespaces are declared once as `<Using>` items in
each test `.csproj`:

```xml
<ItemGroup>
  <Using Include="System.Net" />
  <Using Include="System.Net.Http.Json" />
  <Using Include="CarvedRock.ApiTests.Utils" />
  <Using Include="Microsoft.AspNetCore.Mvc" />
  <Using Include="TUnit.Core.Logging" />
</ItemGroup>
```

That's why the test files that follow look so short -- they really are that short
in the repo.

## Unit Tests

The unit test project is the boring one, and that's the point. No I/O, no
containers, no HTTP -- just business logic and validators. Twenty tests, 430
milliseconds, and the whole project file is this:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="TUnit" Version="1.6*" />
    <PackageReference Include="TUnit.Mocks" Version="1.65.38" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\CarvedRock.Domain\CarvedRock.Domain.csproj" />
  </ItemGroup>
</Project>
```

The main subject here is `NewProductValidator`, a FluentValidation validator with
a wrinkle that makes it a good example: most of its rules are pure, but one of
them (`UniqueName`) has to ask the repository whether a name is already taken.
That's the dependency we need to mock.

### Interface Mocking

`TUnit.Mocks` is source-generated, so there's no `new Mock<T>()` ceremony and no
`.Object` unwrapping -- you call `.Mock()` directly on the interface, and the
generated mock *is* the interface:

```csharp
public class ProductValidationTests
{
    private static ICarvedRockRepository _mockedRepo = null!;

    [Before(Class)]
    public static void SetupDatabaseMock(ClassHookContext context)
    {
        var mock = ICarvedRockRepository.Mock();
        mock.IsProductNameUniqueAsync(Any()).Returns(true);
        mock.IsProductNameUniqueAsync("duplicate").Returns(false);
        _mockedRepo = mock;
    }
```

Two things worth noticing. First, `Any()` doesn't need a type argument -- the
generated mock already knows the parameter type, so the setup reads like the call
site. Second, the more specific setup for `"duplicate"` wins over the `Any()`
one, which means a single mock configured once in a `[Before(Class)]` hook covers
both the happy path and the collision path for every test in the class.

Now the tests are just arrange/act/assert with no mock noise in them at all:

```csharp
[Test]
public async Task DuplicateNameFails()
{
    IValidator<NewProductModel> validator = new NewProductValidator(_mockedRepo);

    var newProduct = new NewProductModel
    {
        Name = "duplicate",
        Category = "boots",
        Description = "",
        ImgUrl = "https://some.place/image.png",
        Price = 59.99
    };

    var result = await validator.ValidateAsync(newProduct,
        opts => opts.IncludeAllRuleSets());

    await Assert.That(result.Errors)
                .Contains(err => err.ErrorMessage ==
                    "A product with the same name already exists.");
}
```

Verifying that a call *happened* (rather than stubbing what it returns) is a
one-liner, and it reads as the call itself:

```csharp
[Test]
public async Task ClearCartAsyncCallsRepository()
{
    var mockRepo = ICarvedRockRepository.Mock();
    var validator = new AddToCartValidator(mockRepo);
    var cartLogic = new CartLogic(mockRepo, validator, NullLogger<CartLogic>.Instance);

    await cartLogic.ClearCartAsync("user-1");

    mockRepo.ClearCartAsync("user-1").WasCalled();
}
```

Exception assertions use the same `Assert.That` entry point, with a lambda
instead of a value:

```csharp
await Assert.That(async () => await orderLogic.PlaceOrderAsync("user-1", "user@test.com"))
    .Throws<InvalidOperationException>()
    .WithMessageContaining("Cannot place an order with an empty cart.");
```

### Data-Driven Tests

Validators are the classic case where one test method and a table of inputs beats
fifteen near-identical methods. TUnit gives you a few ways to build that table,
and it's worth knowing all of them because they trade off differently.

**Inline `[Arguments]`** is the simplest -- and `DisplayName` is what keeps the
report readable when the arguments themselves are meaningless in a test list:

```csharp
[Test]
[Arguments(null, "boots", "really nice footwear - you'll love them!",
                "https://some.place/image.png", 59.99,
                "Name is required.",
                DisplayName = "Inline - missing product name")]
[Arguments("Fancy Boot", "boots", "really nice footwear - you'll love them!",
                "https://some.place/image.png", 49.99,
                "Price for boots must be between $50.00 and $300.00.",
                DisplayName = "Inline - price too low")]
public async Task LongSingleValidationFailures(string? name, string? category,
    string? description, string? imageUrl, double price, string expectedMessage)
{
    var productToValidate = new NewProductModel { /* ...from the arguments... */ };

    var result = await _validator.ValidateAsync(productToValidate,
        opts => opts.IncludeAllRuleSets());

    await Assert.That(result.Errors)
        .Contains(err => err.ErrorMessage == expectedMessage);
}
```

Six positional parameters is about where inline arguments stop being pleasant,
which is the cue to move to a **`[MethodDataSource]`** and hand over real objects
instead. Wrapping each row in a `TestDataRow<T>` lets you attach a display name
to it:

```csharp
[Test]
[MethodDataSource(nameof(SingleFailureDataSourceWithNames))]
public async Task SingleValidationFailures(NewProductModel productToValidate,
        string expectedMessage)
{
    var result = await _validator.ValidateAsync(productToValidate,
        opts => opts.IncludeAllRuleSets());

    await Assert.That(result.Errors)
        .Contains(err => err.ErrorMessage == expectedMessage);
}

public static IEnumerable<Func<TestDataRow<(NewProductModel Product,
            string ExpectedMessage)>>> SingleFailureDataSourceWithNames()
{
    yield return () => new((new NewProductModel
    {
        Name = "Woods Walker",
        Category = "boots",
        Description = "really nice footwear - you'll love them!",
        ImgUrl = "https://some.place/image.png",
        Price = 49.99 // valid range is 50 - 300
    }, "Price for boots must be between $50.00 and $300.00."),
    DisplayName: "Price too low");
}
```

The `Func<>` wrapper matters: each row is a *factory*, so every test case gets its
own fresh instance rather than sharing one mutable object across a parallel run.

There's also `[MatrixDataSource]` with `[Matrix(...)]` on each parameter when you
genuinely want the cross product of several dimensions -- handy for "every
category x every boundary price" style coverage.

## ASP.NET Core Appplication Tests (non-UI)

This is the layer that earns its keep. These tests run the API **in-process** via
`WebApplicationFactory`, so there's no deployment, no `dotnet run`, and no ports
to coordinate -- but the request still goes through the real middleware pipeline,
the real controllers, the real validation, and real EF Core queries against a
real Postgres.

Two packages do the heavy lifting: `TUnit.AspNetCore` for the factory, and
`Testcontainers.PostgreSql` for the database.

The database fixture is a plain class that implements `IAsyncInitializer`. TUnit
sees the interface and awaits it for you, and `SharedType.PerTestSession` means
the container starts exactly once no matter how many test classes want it:

```csharp
public class TestData : IAsyncInitializer, IAsyncDisposable
{
    public PostgreSqlContainer DbContainer { get; } =
        new PostgreSqlBuilder("postgres:18.3").Build();

    public string ConnectionString =>
        DbContainer.GetConnectionString() + ";SSL Mode=Disable";

    public List<Data.Entities.Product> InitialProducts { get; private set; } = null!;

    public readonly Faker<NewProductModel> NewProductFaker = new Faker<NewProductModel>()
        .UseSeed(2001) // will generate consistent data (with any fixed seed value)
        .RuleFor(p => p.Name, f => f.Commerce.ProductName())
        .RuleFor(p => p.Description, f => f.Commerce.ProductDescription())
        .RuleFor(p => p.Category, f => f.PickRandom("boots", "equip", "kayak"))
        .RuleFor(p => p.Price, (f, p) =>
                p.Category == "boots" ? f.Random.Double(50, 300) :
                p.Category == "equip" ? f.Random.Double(20, 150) :
                p.Category == "kayak" ? f.Random.Double(100, 500) : 0)
        .RuleFor(p => p.ImgUrl, f => f.Image.PicsumUrl());

    public async Task InitializeAsync()
    {
        await DbContainer.StartAsync();

        var options = new DbContextOptionsBuilder<LocalContext>()
                            .UseNpgsql(ConnectionString).Options;
        var context = new LocalContext(options);

        await context.Database.EnsureCreatedAsync();

        var products = NewProductFaker.Generate(100);
        var productMapper = new ProductMapper();

        List<Data.Entities.Product> productsToCreate = [];
        foreach (var product in products)
        {
            productsToCreate.Add(productMapper.NewProductModelToProduct(product));
        }

        context.Products.AddRange(productsToCreate);
        await context.SaveChangesAsync();

        InitialProducts = await context.Products.ToListAsync();
    }
}
```

`UseSeed(2001)` is the detail I'd most encourage you to steal. Bogus with a fixed
seed gives you a hundred products that are *varied* (three categories, realistic
prices, different name lengths) but **identical on every run and every machine**.
You get the coverage benefits of generated data without the flakiness, and
`InitialProducts` gives every test a trustworthy snapshot of what the database
contained before anything ran.

Next, the factory. `TestWebApplicationFactory<Program>` is TUnit's subclass of the
familiar `WebApplicationFactory<Program>`, and the important addition is
`ConfigureStartupConfiguration` -- a hook that runs early enough to inject the
Testcontainers connection string *before* the host reads configuration:

```csharp
public class ApiFactory : TestWebApplicationFactory<Program>
{
    [ClassDataSource<TestData>(Shared = SharedType.PerTestSession)]
    public TestData TestData { get; init; } = null!;

    protected override void ConfigureStartupConfiguration(
                        IConfigurationBuilder configurationBuilder)
    {
        configurationBuilder.AddInMemoryCollection(new Dictionary<string, string?>
        {
            { "ConnectionStrings:CarvedRockPostgres", TestData.ConnectionString }
        });
    }
}

public abstract class ApiTestsBase : WebApplicationTest<ApiFactory, Program>
{
    protected TestData TestData => GlobalFactory.TestData;
    protected static DefaultLogger TestLogger =>
                        TestContext.Current!.GetDefaultLogger();
}
```

Note that the fixture is injected into the *factory* using the same
`[ClassDataSource]` attribute you'd put on a test class -- the factory is just
another fixture as far as TUnit is concerned.

`ApiTestsBase` is the whole reason the individual test files are so small.
Inheriting from `WebApplicationTest<ApiFactory, Program>` gives every test a
`Factory` property, and the two extra members expose the seeded data and a
per-test logger. A complete test file then looks like this:

```csharp
public class ProductApiTests : ApiTestsBase
{
    [Test]
    public async Task GetProductsAnonymous_ReturnsAllProducts()
    {
        var client = Factory.CreateClient();

        var response = await client.GetAsync($"/product");

        await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.OK);
        var content = await response.Content.ReadFromJsonAsync<List<Product>>();

        var randomProduct = TestData.GeneralFaker.PickRandom(TestData.InitialProducts);

        await Assert.That(content!)
            .Count().IsEqualTo(TestData.InitialProducts.Count)
            .And.Contains(p => p.Name == randomProduct.Name);
    }
}

// defined here to validate contract external callers may be
// depending on specific property names / JSON format
public record Product(int Id, string Name, string Category,
                      string Description, double Price, string ImgUrl);
```

That last bit is deliberate and I think underrated: the test declares its **own**
`Product` record rather than referencing the app's DTO. If someone renames a
property or changes the JSON casing, this test fails -- which is exactly what you
want, because a caller out in the world would have broken too. Referencing the
app's own type would have quietly renamed both sides at once and told you nothing.

Because these are records, whole-object assertions work, and there are a few nice
styles to pick from:

```csharp
// check an entire record you create on the fly
var expectedProduct = new Product(2, "Desert Walker", "boots",
        "Breathable and lightweight boots perfect for hot weather hiking and desert exploration.",
        74.99, "https://picsum.photos/id/15/800/600");
await Assert.That(product).IsEqualTo(expectedProduct); // works because it's a record

// ...or compare against the known-good seeded data
var expected = fixture.InitialProducts.Single(p => p.Id == 2);
await Assert.That(product).IsEqualTo(expected);
```

Problem details responses are worth asserting on properly too, since that's the
actual contract your clients consume on a validation failure:

```csharp
[Test]
public async Task PostProductValidationFailure()
{
    var client = Factory.CreateClient();
    client.AddAdminAuthHeaders();

    var newProduct = TestData.NewProductFaker.Generate();
    newProduct.Name = ""; // invalid

    var response = await client.PostAsJsonAsync("/product", newProduct);

    await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.BadRequest);

    var problemDetails = await response.Content.ReadFromJsonAsync<ProblemDetails>();

    await Assert.That(problemDetails).IsNotNull()
        .And.Member(pd => pd.Detail, detail =>
                detail.IsEqualTo("One or more validation errors occurred."))
        .And.Member(pd => pd.Extensions.Keys, keys => keys.Contains("Name"))
        .And.Member(pd => pd.Extensions["Name"]!.ToString(),
                err => err.Contains("Name is required."));
}
```

> **A word about shared state.** All 16 of these tests run in parallel against one
> Postgres container, and some of them mutate it. `DeleteProductAsAdmin_Succeeds`
> deletes product 1, so the cart tests have to filter it out
> (`InitialProducts.Where(p => p.Id != 1)`) or a random pick can 404. TUnit gives
> you `[DependsOn(nameof(OtherTest))]` to sequence cases that truly can't overlap,
> and using a distinct fake user per test is often a cleaner fix than ordering.
> This is real complexity that comes with parallelism -- but it's the cost of a
> suite that finishes in 8 seconds instead of 80, and the exclusions are cheap as
> long as you comment *why* they exist.

### Authentication and Authorization Scenarios

The app authenticates against an external OIDC provider (the public Duende demo
IdentityServer). Dragging that into in-process API tests would be slow, flaky, and
beside the point -- what we actually want to test is *our* authorization logic:
does an admin get through, does a customer get a 403, does an anonymous caller get
a 401?

So we swap the entire authentication scheme for a fake one. The user comes from an
`X-Authorization` header, and **every `X-Test-<claim>` header becomes a claim**:

```csharp
public class TestAuthHandler(IOptionsMonitor<AuthenticationSchemeOptions> options,
                                ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string SchemeName = "TestScheme";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Context.Request.Headers.TryGetValue("X-Authorization", out var value))
        {
            return Task.FromResult(AuthenticateResult.Fail("No X-Authorization Header"));
        }

        var userName = value.First();

        var claims = new List<Claim> { new("name", userName!) };
        claims.AddRange(GetClaimsFromHttpHeaders());

        var identity = new ClaimsIdentity(claims, "TestAuthType");
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, SchemeName);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }

    private IEnumerable<Claim> GetClaimsFromHttpHeaders()
    {
        var headers = Context.Request.Headers;

        return from header in headers
               where header.Key.StartsWith("X-Test-")
               let claimType = header.Key.Replace("X-Test-", "")
               select new Claim(claimType, header.Value!);
    }
}
```

Registering it takes three lines in the factory:

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureTestServices(services => services
           .AddAuthentication(TestAuthHandler.SchemeName)
           .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>
                            (TestAuthHandler.SchemeName, _ => { }));
    // ...
}
```

The generic claim mapping is what makes this so flexible. In this app, admin
rights are granted by an `AdminClaimsTransformation` that looks at the email
local-part, so "make this caller an admin" is just a header value -- no test
users, no seeded identity data:

```csharp
public static class AuthHelpers
{
    public static void AddAdminAuthHeaders(this HttpClient client)
    {
        client.DefaultRequestHeaders.Add("X-Authorization", "Bob Smith");
        client.DefaultRequestHeaders.Add("X-Test-sub", "456");
        client.DefaultRequestHeaders.Add("X-Test-idp", "CarvedRock");
        client.DefaultRequestHeaders.Add("X-Test-email", "bobsmith@someplace.com");
    }

    public static void AddCustomerAuthHeaders(this HttpClient client)
    {
        client.DefaultRequestHeaders.Add("X-Authorization", "Erik Dahl");
        client.DefaultRequestHeaders.Add("X-Test-sub", "123");
        client.DefaultRequestHeaders.Add("X-Test-idp", "CarvedRock");
        client.DefaultRequestHeaders.Add("X-Test-email", "erikdahl@someplace.com");
    }
}
```

And now the three-way authorization matrix is genuinely trivial to cover:

```csharp
[Test]
public async Task GetSampleAnonymous_ReturnsUnauthorized()
{
    var client = Factory.CreateClient();
    var response = await client.GetAsync("/sample/auth?text=hello");

    await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.Unauthorized);
}

[Test]
public async Task GetSampleAdminAsAdmin_ReturnsOK()
{
    var client = Factory.CreateClient();
    client.AddAdminAuthHeaders();

    var response = await client.GetAsync("/sample/auth-admin?text=hello");

    await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.OK);
}

[Test]
public async Task GetSampleAdminAsNonAdmin_ReturnsForbidden()
{
    var client = Factory.CreateClient();
    client.AddCustomerAuthHeaders();

    var response = await client.GetAsync("/sample/auth-admin?text=hello");

    await Assert.That(response.StatusCode).IsEqualTo(HttpStatusCode.Forbidden);
}
```

Also note that the fake handler's rejection shows up in the test's own log output
-- so when a test fails with an unexpected 401 you can see *why* immediately:

```text
info: CarvedRock.ApiTests.Utils.TestAuthHandler[7]
      TestScheme was not authenticated. Failure message: No X-Authorization Header
```

### Mocking External HTTP / API Requests

The API has an endpoint that calls an external dad-joke service. In a test run we
don't want to depend on somebody else's uptime, rate limits, or (entertainingly)
their sense of humor changing the assertion.

The controller reads its base address from configuration:

```csharp
public class DadJokeController(IConfiguration config) : ControllerBase
{
    [HttpGet]
    public async Task<string> Get()
    {
        var client = new HttpClient
        {
            BaseAddress = new Uri(config.GetValue<string>("DadJokeUrl")!)
        };
        // ...
    }
}
```

So the mock is a real HTTP server on localhost, started by
[WireMock.Net](https://github.com/WireMock-Net/WireMock.Net), with its URL pushed
into configuration by the factory:

```csharp
private WireMockServer _wireMockDadJokes = null!;

protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    // ... auth setup ...

    _wireMockDadJokes = WireMockServer.Start();

    _wireMockDadJokes
        .Given(Request.Create().WithPath("/").UsingGet())
        .RespondWith(Response.Create()
            .WithStatusCode(200)
            .WithBody("""{"id": "xxxxxx", "joke": "joke's on you - from the mock!", "status": 200}"""));

    builder.ConfigureAppConfiguration((context, config) =>
    {
        config.AddInMemoryCollection(new Dictionary<string, string?>
        {
            { "DadJokeUrl", _wireMockDadJokes.Url }
        });
    });
}
```

`WireMockServer.Start()` picks a free port and `.Url` hands it back -- so nothing
is hardcoded and parallel runs can't collide. The test itself just asserts, and
logs what came back:

```csharp
[Test]
public async Task GetDadJokeWorks()
{
    var client = Factory.CreateClient();

    var response = await client.GetAsync("/dadjoke");
    var joke = await response.Content.ReadAsStringAsync();

    TestLogger.LogInformation($"JOKE RESPONSE: {joke}");

    await Assert.That(joke).IsNotNullOrWhiteSpace();
}
```

...and the run report shows the mock was really in the loop:

```text
Information: JOKE RESPONSE: joke's on you - from the mock!
```

> **WireMock or `TUnit.Mocks.Http`?** TUnit ships an HTTP mocking package that
> gives you a real `HttpClient` backed by a scriptable handler
> (`Mock.HttpClient("https://api.example.com")` plus
> `client.Handler.OnGet("/users/1").RespondWithJson(...)`). It's quite nice, and it's
> the right call whenever you can *inject* the `HttpClient` -- a typed client
> registered in DI, for instance. Here the controller news up its own client from
> a config value, so there's no seam to inject through. When the only thing you
> can change is the URL, WireMock is the tool that fits.

## Full Application Tests (Aspire, Playwright)

Now the fun part. This project starts the **entire distributed application** --
every project, every container -- once per test session, using Aspire's
`DistributedApplicationTestingBuilder` under the hood.

Here's the topology being stood up, straight from `AppHost.cs`:

```text
db (Postgres 18) ──┐
smtp (MailPit) ────┴─> api ─> mcp ─> agent ─> webapp
                                        mcp-inspector ─> mcp
```

That's two containers and four ASP.NET Core projects (plus the MCP Inspector, a
developer tool we'll come back to), wired with real service
discovery, real health checks, and real HTTPS. The fixture to bring it all up is
one line (the class inheritance from `AspireFixture`):

```csharp
public class AppFixture : AspireFixture<CarvedRock_AppHost>
{
    protected override TimeSpan ResourceTimeout => TimeSpan.FromMinutes(3);
    // ...
}
```

`AspireFixture<T>` builds the app host, starts it, and waits for every resource to
pass its health checks before any test runs. The default `ResourceTimeout` is 60
seconds; three minutes is more forgiving of a cold container pull on a CI runner.

Because `InitializeAsync` is `virtual`, you can hook in *after* everything is
healthy -- which is where this fixture opens a `LocalContext` against the real
Postgres so tests can assert directly on the database:

```csharp
public override async Task InitializeAsync()
{
    await base.InitializeAsync(); // Build, start, wait for resources

    // Post-start: get already-seeded data for test confirmations
    //     could also do migrations or create test data
    var connStr = await App.GetConnectionStringAsync("CarvedRockPostgres");

    var options = new DbContextOptionsBuilder<LocalContext>()
                        .UseNpgsql(connStr).Options;
    TestDbContext = new LocalContext(options);

    InitialProducts = await TestDbContext.Products.Select(p =>
            new Product(p.Id, p.Name, p.Category, p.Description, p.Price, p.ImgUrl))
        .ToListAsync();
}
```

Two Aspire-isms are doing a lot of work there.
`App.GetConnectionStringAsync(...)` and `App.GetEndpoint(...)` /
`App.CreateHttpClient(...)` mean **no test ever hardcodes a port or a host**.
Aspire assigns them at startup and the fixture asks for them by resource name:

```csharp
public async Task<HttpClient> GetAdminApiClient()
{
    var client = App.CreateHttpClient("api");
    var token = await GetClientCredsAccessTokenAsync("m2m.short", "secret"); // admin
    client.SetBearerToken(token); // Duende.IdentityModel convenience method

    return client;
}
```

Unlike the API tests, these use **real access tokens** from the real identity
provider via client credentials. That means the actual JWT validation,
`AdminClaimsTransformation`, and token forwarding through the MCP server are all
genuinely exercised -- not stubbed.

A test class then just takes the fixture as a constructor parameter:

```csharp
[ClassDataSource<AppFixture>(Shared = SharedType.PerTestSession)]
public class ProductApiTests(AppFixture fixture)
{
    [Test]
    public async Task GetProductsAnonymous_ReturnsAllProducts()
    {
        var client = fixture.CreateHttpClient("api");

        var products = await client.GetFromJsonAsync<List<Product>>("/product");
        await Assert.That(products).Count().IsEqualTo(fixture.InitialProducts.Count);
    }
}
```

The payoff for testing the whole system at once is that you can assert on things
that cross service boundaries. `PlacingOrderWorksCompletely` places one order
through the API and then verifies **all** of the consequences:

```csharp
// place order
var orderResult = await client.PostAsJsonAsync("/order", new NewOrder(null));
await Assert.That(orderResult.StatusCode).IsEqualTo(HttpStatusCode.Created);

// 1. the order + details rows landed in Postgres, and the total adds up
var savedOrder = await context.Orders.Include(o => o.Details)
    .SingleAsync(o => o.Id == placedOrder!.Id);

await Assert.That(savedOrder).IsNotNull()
    .And.Member(o => o.Email, e => e.IsEqualTo(placedOrder!.Email))
    .And.Member(o => o.Total, t => t.IsEqualTo(savedOrder.Details.Sum(d => d.LineTotal)))
    .And.Member(o => o.Details.Count, c => c.IsEqualTo(productsToOrder.Count));

// 2. the cart was emptied
var cartAfterOrder = await client.GetFromJsonAsync<List<CartLine>>("/cart");
await Assert.That(cartAfterOrder).IsEmpty();

// 3. the confirmation email was actually sent, and mentions every product
var emailApiEndpoint = fixture.App.GetEndpoint("smtp", "http");
using var mailClient = new HttpClient { BaseAddress = emailApiEndpoint };

var messages = await mailClient.GetFromJsonAsync<MailPitMessageList>("/api/v1/messages");
var sentMessage = messages!.Messages
    .Where(m => m.To.Any(to => to.Address == placedOrder!.Email))
    .OrderByDescending(m => m.Created).FirstOrDefault();

await Assert.That(sentMessage).IsNotNull();
await Assert.That(sentMessage!.Subject).IsEqualTo("Your CarvedRock Order");

var fullMessage = await mailClient
        .GetFromJsonAsync<MailPitMessage>($"/api/v1/message/{sentMessage.ID}");

foreach (var product in productsToOrder)
{
    await Assert.That(fullMessage!.HTML).Contains(product.Name);
}
```

Database rows, an emptied cart, and a real SMTP message with the right contents --
in one test, in under two seconds. MailPit is in the AppHost as the `smtp`
resource specifically so the email is a *testable artifact* instead of a
fire-and-forget side effect.

The MCP server gets the same treatment, including the authorization surface --
which for an MCP server means "which tools are even visible to you":

```csharp
[Test]
public async Task GetToolsIncludesGetProducts()
{
    var mcpClient = await fixture.GetAnonymousMcpClient();

    var tools = await mcpClient.ListToolsAsync();

    var getProductsTool = tools.FirstOrDefault(t => t.Name == "get_products");
    await Assert.That(getProductsTool).IsNotNull();

    var setPriceTool = tools.FirstOrDefault(t => t.Name == "set_product_price");
    await Assert.That(setPriceTool).IsNull();  // hidden from anonymous callers
}
```

> **Shared mutable state, again -- but louder.** Unlike the API project's
> per-session Testcontainers database, this Postgres is the app's *real*
> development database and it survives between runs. Tests here delete products 20
> and 23, update product 22, and mutate carts. The order test filters those ids out
> explicitly:
>
> ```csharp
> var productsToOrder = fixture.GeneralFaker
>         .PickRandom(fixture.InitialProducts.Where(p =>
>                    p.Id != 20 && p.Id != 23  // deleted by webapptest
>                 && p.Id != 22),              // updated by mcp test
>             3)
>         .ToList(); // important!  this locks the list
> ```
>
> If you take one thing from this section: **be careful about mutations**. A comment
> naming the test that deletes the row is the difference between a five-minute fix
> and an afternoon.

### Excluding Resources

By default `AspireFixture<T>` waits for *every* resource in the app model to be
healthy. That's the right default, but it isn't always what you want -- this
AppHost includes an `mcp-inspector` resource, which is a genuinely useful
developer tool for poking at the MCP server by hand and completely useless to an
automated test. Waiting on a container you'll never call is pure startup cost.

`ResourcesToRemove()` removes resources from the app model entirely -- they
are never started at all:

```csharp
public class AppFixture : AspireFixture<CarvedRock_AppHost>
{
    // mcp-inspector is a human debugging tool - no test ever talks to it,
    // so don't pay to start it.
    protected override IEnumerable<string> ResourcesToRemove() => ["mcp-inspector"];
}
```

A couple of related fixture knobs are worth knowing while you're in here:

* `EnableTelemetryCollection` (on by default) starts an OTLP receiver that
  collects telemetry from your services and routes it to the originating test.
  This is the feature behind the traces section below, and it needs **no
  test-specific code in your app** -- standard Aspire `ServiceDefaults` is enough.
* `Options => new() { ForwardResourceLogs = true }` forwards each resource's raw
  console output into the test output as it happens. This is the one that saves
  you when a resource *fails to boot*, because it subscribes before the app starts
  -- so you see the crash even though the OTel exporter never got a chance to
  flush.
* `DumpResourceLogsOnFailure` (on by default) appends recent error lines from each
  waited-on resource to a failing test's output.

### Playwright Recordings

`TUnit.Playwright` manages the browser, context, and page lifecycle, so inheriting
from `PageTest` gives you a ready `Page` property and Playwright's `Expect`
assertions. This app needs a couple of customizations, so there's a
`CustomPageTest` in between (simple version here; more functionality further below):

```csharp
public class CustomPageTest : PageTest
{
    [ClassDataSource<AppFixture>(Shared = SharedType.PerTestSession)]
    public required AppFixture Fixture { get; init; }

    public string WebAppUrl => Fixture.App.GetEndpoint("webapp").ToString();

    // playwright browsers on linux don't play well with the self-signed certs
    // this override is really only to support CI pipelines
    public override BrowserNewContextOptions ContextOptions(TestContext testContext)
    {
        var options = base.ContextOptions(testContext);
        options.IgnoreHTTPSErrors = true;
        return options;
    }
}
```

The `Fixture` property is the important line: the browser tests share the *same*
session-scoped Aspire app as the API and MCP tests, and `WebAppUrl` comes from
Aspire rather than a config file.

Logging in happens often enough to deserve an extension method, and it ends with
an assertion so a failed login fails *there* rather than three steps later with a
confusing selector error:

```csharp
public static async Task Login(this IPage page, string username, string password)
{
    await page.GetByRole(AriaRole.Textbox, new() { Name = "Username" }).FillAsync(username);
    await page.GetByRole(AriaRole.Textbox, new() { Name = "Password" }).ClickAsync();
    await page.GetByRole(AriaRole.Textbox, new() { Name = "Password" }).FillAsync(password);
    await page.GetByRole(AriaRole.Button, new() { Name = "Login" }).ClickAsync();

    await Assertions.Expect(page.GetByRole(AriaRole.Link, new() { Name = "Sign Out" }))
                            .ToBeVisibleAsync();
}
```

Now **the tests read like user stories.** This one places an order in the browser and
then checks the confirmation email in MailPit's web UI -- same browser, same test:

```csharp
[Test]
[RecordVideo]
public async Task CustomerCanPlaceOrderAndGetEmail()
{
    await Page.GotoAsync(WebAppUrl);
    await Page.GetByRole(AriaRole.Link, new() { Name = "Footwear" }).ClickAsync();

    // footwear link should redirect to login page
    await Page.Login("alice", "alice");  // customer

    await Page.GetByRole(AriaRole.Row, new() { Name = "Desert Walker" })
                .GetByRole(AriaRole.Button).ClickAsync();
    await Page.GetByRole(AriaRole.Row, new() { Name = "River Guide" })
                .GetByRole(AriaRole.Button).ClickAsync();

    // implicit assertion that the cart button shows 2 items in it
    await Page.GetByRole(AriaRole.Link, new() { Name = "Cart (2)" }).ClickAsync();

    await Page.GetByRole(AriaRole.Button, new() { Name = "Checkout" }).ClickAsync();
    await Page.GetByRole(AriaRole.Button, new() { Name = "Submit Order" }).ClickAsync();

    await Expect(Page.Locator("h1")).ToContainTextAsync("Thanks for your (fake) order!");

    // now go read the actual email that got sent
    var emailUrl = Fixture.App.GetEndpoint("smtp", "http").ToString();
    await Page.GotoAsync(emailUrl);

    await Page.GetByRole(AriaRole.Link, new() { Name = "to: alicesmith@email.com" })
                .ClickAsync();

    await Expect(Page.Locator("#preview-html").ContentFrame.Locator("body"))
                .ToContainTextAsync("Desert Walker");
    await Expect(Page.Locator("#preview-html").ContentFrame.GetByRole(AriaRole.Heading))
                .ToContainTextAsync("Thank you for your order!");
}
```

Screenshots are a one-liner anywhere in a test:

```csharp
await Page.ScreenshotAsync(new() { Path = "playwright-artifacts/screenshot.png" });
```

![The home page screenshot captured automatically during the test run](/images/triple-threat/playwright-screenshot.png)

Video is where I ended up writing a little custom plumbing, because Playwright's
out-of-the-box behavior has two annoyances: you enable recording per *context*
(not per test), and it names the files `page@<hash>.webm`. Neither is great once
CI has uploaded a dozen of them.

So: a custom `[RecordVideo]` attribute. The interesting bit is that it uses
TUnit's `ITestDiscoveryEventReceiver` and `StateBag` rather than reflection --
which keeps the whole thing source-generation and AOT friendly:

```csharp
// The StateBag is used to avoid reflection and use TUnit source generation approach
[AttributeUsage(AttributeTargets.Method)]
public sealed class RecordVideoAttribute : Attribute, ITestDiscoveryEventReceiver
{
    internal const string StateBagKey = "CarvedRock.RecordVideo";

    public ValueTask OnTestDiscovered(DiscoveredTestContext discoveredTestContext)
    {
        discoveredTestContext.TestContext.StateBag[StateBagKey] = true;
        return default;
    }
}
```

`CustomPageTest` then reads that flag when it builds the browser context -- so a
single attribute on a test method turns recording on for just that test, at a
viewport size that actually looks good in a demo:

```csharp {hl_lines=[6,7,8,9,10]}
public override BrowserNewContextOptions ContextOptions(TestContext testContext)
{
    var options = base.ContextOptions(testContext);
    options.IgnoreHTTPSErrors = true;

    if (testContext.StateBag.ContainsKey(RecordVideoAttribute.StateBagKey))
    {
        options.RecordVideoDir = "playwright-artifacts/";
        options.ViewportSize = new ViewportSize { Width = 1280, Height = 1400 };
    }

    return options;
}
```

Renaming is the fiddly part, and the comment in the repo explains why it happens
at the end of the session rather than after each test:

```csharp
// Playwright names its recordings page@<hash>.webm, which tells you nothing about
// which test produced which video once CI has uploaded a dozen of them. The name
// can't be set through RecordVideoDir, and IVideo.SaveAsAsync waits for the page to
// close - which hasn't happened yet inside an [After(Test)] hook. So note where each
// video is headed while the test runs, then rename them all at the end of the
// session, by which point every browser context has been torn down and flushed.
private static readonly ConcurrentBag<(string TestName, string SourcePath)>
    RecordedVideos = [];

[Before(Test)]
public async Task NoteVideoPathForRenaming(TestContext testContext)
{
    if (Page.Video is null) return;

    // A retried test records once per attempt; number them so the flaky-test videos
    // line up with the attempts shown in the run report instead of overwriting.
    var attempt = testContext.Execution.CurrentRetryAttempt;
    var name = testContext.Metadata.TestName +
               (attempt > 0 ? $"-attempt{attempt + 1}" : string.Empty);

    RecordedVideos.Add((name, await Page.Video.PathAsync()));
}

[After(TestSession)]
public static void RenameRecordedVideos()
{
    foreach (var (testName, sourcePath) in RecordedVideos)
    {
        // ... File.Move to <TestName>.webm, de-duplicating if needed ...
        // A recording we couldn't rename is still a usable recording - never fail
        // a run (or hide the real result) over cosmetic artifact naming.
    }
}
```

The result is a `playwright-artifacts/` folder with files called
`CustomerCanPlaceOrderAndGetEmail.webm` -- and, on a flaky retry,
`CustomerCanPlaceOrderAndGetEmail-attempt2.webm` right next to it, which lines up
with the attempts shown in the run report.

![Directory containing Playwright artifacts](/images/triple-threat/artifacts.png)

One more Playwright-specific practicality.

**Throttle browser parallelism.** Every browser test is a Chromium instance on top
of the whole Aspire app, and CI runners are not generous machines:

```csharp
// Browser tests are the heaviest thing in this suite: every one is its own Chromium
// instance, running on top of the Aspire AppHost (two containers plus four services)
// and whatever the sibling test projects are doing in the same `dotnet test` run.
public record BrowserParallelLimit : IParallelLimit
{
    public int Limit => 3;
}

[ParallelLimiter<BrowserParallelLimit>]
public partial class WebAppTests : CustomPageTest { /* ... */ }
```

Finally: the chat-driven tests assert on **LLM output**, which is
non-deterministic by nature. The pattern that works is a soft assertion on the
chat text plus a hard assertion on the resulting state:

```csharp
await Expect(Page.Locator("#chatMessages"))
        .ToContainTextAsync("successfully",  // be careful - non-deterministic!!
            options: new() { Timeout = 15_000 });

// the real assertion: the products are actually gone from the database
var actualProduct = await Fixture.TestDbContext.Products
                        .FirstOrDefaultAsync(p => p.Id == 20 || p.Id == 23);
await Assert.That(actualProduct).IsNull();
```

## Logs and Traces for Every Test

This is the feature I didn't know I wanted, and now won't do without.

TUnit produces a run report -- one JSON file and one HTML file per test project,
in `TestResults/`. The HTML report contains an `Overview`: an execution
timeline, per-class timelines, slowest tests, where time was spent, parallel
execution, categories, and failures.

![The TUnit run report: execution timeline, class timelines, and slowest tests](/images/triple-threat/tunit-report-overview.png)

Then you click an individual test and get four tabs: **Output**, **Trace**,
**Properties**, and **Source**.

The Output tab is where the Aspire integration pays off. Remember that
`EnableTelemetryCollection` starts an OTLP receiver and correlates telemetry back
to the test that caused it. So for the order test, the output contains log lines
emitted by the **API service running in a separate process**, prefixed by resource
name, interleaved with the test's own logging:

```text
[api] [Information] Adding product 30 to cart.
[api] [Information] Adding product 1 to cart.
[api] [Information] Adding product 42 to cart.
[api] [Information] Adding product 30 (qty 1) to cart for m2m
[api] [Information] Created order 1 with 3 line(s).
[api] [Information] Order confirmation email sent to unknown@carvedrock.com for order 1.
[api] [Information] Placed order 1 for m2m.
Information: ACTUAL: [ 30, Calm Waters Touring Kayak, 1, 699.99 ]
Information: EXPECTED: Product { Id = 30, Name = Calm Waters Touring Kayak, ... }
```

Your own test logging goes through `TestContext`, so it lands in the right test's
output even with everything running in parallel:

```csharp
protected static DefaultLogger TestLogger => TestContext.Current!.GetDefaultLogger();

// ...then anywhere in a test:
TestLogger.LogInformation($"ACTUAL: [ {ordered.ProductId}, {ordered.ProductName}, " +
                          $"{ordered.Quantity}, {ordered.UnitPrice} ]");
```

The Trace tab is the other half. Which means for one test you can see, with timings:

```text
   187.0ms  [Experimental.System.Net.Http.Connections] HTTP wait_for_connection demo.duendesoftware.com:443
   241.7ms  [System.Net.Http] POST                      <- fetching the access token
   558.7ms  [Microsoft.AspNetCore] POST Cart            <- the API handling the request
     1.4ms  [Npgsql] postgresql                         <- the SQL it ran
   134.9ms  [Npgsql] postgresql
```

Database calls, HTTP calls, MCP tool calls, AI calls, and SMTP sends -- attributed
to the individual test that caused them. When an integration test fails in CI at
2am, this is the difference between "something timed out" and "the token request to
the external IdP took 30 seconds."

![The Trace tab for a single test, showing the span waterfall across services](/images/triple-threat/tunit-report-trace.png)

And critically: **none of this requires test-specific code in the application.**
The services export OpenTelemetry because they use Aspire's standard
`ServiceDefaults`. TUnit just points the OTLP endpoint at itself for the duration
of the run.

The JSON report next to the HTML one has all the same data in a machine-readable
shape -- which matters a lot for the agent section at the end of this post.

## Coverage Reporting

Because everything runs on Microsoft Testing Platform, coverage is a command-line
flag rather than a separate tool invocation:

```bash
dotnet test --coverage --coverage-output-format cobertura --coverage-settings testconfig.json
```

That drops one raw cobertura file per test project into `TestResults/`. They're
GUID-named, so don't try to identify projects by filename -- generate a report
instead.

### Report Generation

The excellent [ReportGenerator](https://reportgenerator.io/) turns those cobertura files into
something humans (and PR reviewers) can use. Install it once:

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool
```

Then merge all three projects' coverage into a single report:

```bash
reportgenerator -reports:TestResults/*.cobertura.xml -targetdir:coveragereport -reporttypes:"Html;TextSummary;"
```

Merging matters more than it sounds. A line in `ProductLogic` might only be
covered by an app test, and a line in `NewProductValidator` only by a unit test --
looking at either report alone would badly understate your coverage. The glob
across all three cobertura files gives you the real number.

`TextSummary` is the report type I'd add if you add nothing else: it produces a
`Summary.txt` that's a perfect at-a-glance rollup, and it's plain text so you can
diff it, grep it, or hand it to an agent:

```text
Summary
  Generated on: 9/4/2026 - 9:49:16 AM
  Parser: MultiReport (3x Cobertura)
  Assemblies: 8
  Classes: 60
  Line coverage: 82.3%
  Covered lines: 1128
  Uncovered lines: 242
  Branch coverage: 70.7% (215 of 304)
  Method coverage: 83.7% (134 of 160)

CarvedRock.Api                                                86.0%
  CarvedRock.Api.Controllers.CartController                  100.0%
  CarvedRock.Api.Controllers.ProductController                79.4%
  CarvedRock.Api.ValidationExceptionHandler                   79.3%

CarvedRock.Domain                                            100.0%
  CarvedRock.Domain.CartLogic                                100.0%
  CarvedRock.Domain.NewProductValidator                      100.0%
  CarvedRock.Domain.OrderLogic                               100.0%
  CarvedRock.Domain.ProductLogic                             100.0%

...
```

82.3% overall, with the business logic at 100% -- and the gaps are consciously
omitted areas for someone else to practice with.

![The merged ReportGenerator HTML coverage report](/images/triple-threat/coverage-report.png)

### Excluding Things

An unfiltered coverage number is misleading in both directions, and the noise
makes the report harder to act on. Auto-properties, EF Core migrations,
source-generated mapper output, and DTO/entity files all pad the denominator
without telling you anything.

The `--coverage-settings testconfig.json` flag above points at this:

```json
{
  "Configuration": {
    "CodeCoverage": {
      "SkipAutoProperties": true,
      "Functions": {
        "Exclude": [
          "^Microsoft\\..*",
          "^System\\..*",
          "^CarvedRock\\.Data\\.Migrations\\..*"
        ]
      },
      "Sources": {
        "Exclude": [
          ".*\\\\Riok.Mapperly\\\\.*",
          ".*\\\\.*Models.*",
          ".*\\\\.*Model.*",
          ".*\\\\.*Entities.*",
          ".*\\\\MailKit.Client\\\\.*"
        ]
      }
    }
  }
}
```

Three kinds of exclusion, all worth understanding:

* **`SkipAutoProperties`** -- the single highest-value setting here. Every
  `{ get; set; }` is technically a coverable line, and including them means your
  DTOs quietly inflate the number.
* **`Functions.Exclude`** -- regex on fully-qualified names. Framework code and EF
  Core migrations are the obvious candidates; migrations in particular are
  generated, already ran to produce your schema, and will never be meaningfully
  "tested."
* **`Sources.Exclude`** -- regex on file paths (note the escaped backslashes).
  This is how you drop source-generated output like Riok.Mapperly's mapper
  implementations, plus models/entities and, in this repo, a hand-rolled Aspire
  client integration.

The full set of options is documented in the
[Microsoft code coverage configuration reference](https://github.com/microsoft/codecoverage/blob/main/docs/configuration.md).

My general advice: **exclude things you don't intend to cover, and nothing else.**
The point of excluding isn't a bigger number, it's a report where every red line
is a real decision you get to make.

## Local Execution Script

All of the above collapses into one script, which is what I actually run:

```powershell
param(
    [switch]$ShowReports
)

Remove-Item -Recurse -Force -ErrorAction SilentlyContinue TestResults, coveragereport, "tests/CarvedRock.AppTests/bin/Debug/Net10.0/playwright-artifacts"

dotnet test --coverage --coverage-output-format cobertura --coverage-settings testconfig.json

reportgenerator -reports:TestResults/*.cobertura.xml -targetdir:coveragereport -reporttypes:"Html;TextSummary;"

if ($ShowReports) {
    Invoke-Item ./coveragereport/index.html
    Invoke-Item ./TestResults/*.html
}
```

Fourteen lines, and every one of them earns its place:

* **Delete first.** Stale artifacts from a previous run are how you end up
  debugging a video from twenty minutes ago, or reporting coverage that includes a
  project you deleted. The `-ErrorAction SilentlyContinue` keeps it quiet on a
  clean checkout.
* **One `dotnet test`** runs all three projects, concurrently.
* **Report generation** merges the three cobertura files.
* **`-ShowReports`** is opt-in. During a tight loop you want the exit code and
  nothing else; when you're reviewing, four browser tabs open (three TUnit reports
  plus coverage) and you can see everything.

```bash
./test-with-coverage.ps1              # just run them
./test-with-coverage.ps1 -ShowReports # run them and open everything
```

For a tighter loop, skip the script and target a single project or test. TUnit's
tree-node filter follows `/Assembly/Namespace/Class/Test` and takes wildcards:

```bash
# one project
dotnet run --project tests/CarvedRock.UnitTests

# one class
dotnet run --project tests/CarvedRock.UnitTests --treenode-filter "/*/*/ProductValidationTests/*"

# one test
dotnet run --project tests/CarvedRock.ApiTests --treenode-filter "/*/*/*/GetDadJokeWorks"
```

That last form is what you want when you're iterating on a single Playwright test
and re-watching its video.

## Continuous Integration Pipeline

The CI pipeline runs the same commands and then does the one thing the local
script can't: makes the results visible to people who weren't at the keyboard.

The build and setup steps are mostly what you'd expect, with one Linux-specific
wrinkle. Playwright's Chromium on Linux doesn't trust ASP.NET Core's dev
certificate the way Windows does, so the certs get cleaned, re-created, and
exported for OpenSSL:

```yaml
- name: Ensure browsers are installed
  run: pwsh tests/CarvedRock.AppTests/bin/Release/net10.0/playwright.ps1 install --with-deps chromium

- name: Clear any existing dev certs
  run: dotnet dev-certs https --clean

- name: Export SSL_CERT_DIR for OpenSSL trust
  run: echo "SSL_CERT_DIR=$HOME/.aspnet/dev-certs/trust:/usr/lib/ssl/certs" >> "$GITHUB_ENV"

- name: Create and trust https certificate
  run: dotnet dev-certs https --trust
```

(That's the other half of the `IgnoreHTTPSErrors = true` in `CustomPageTest` --
belt and suspenders for a problem that only shows up in CI.)

The test step is the same command as local, with the platform options after a `--`
separator and the OpenAI key supplied as the Aspire parameter the AppHost expects:

```yaml
- name: Run tests with coverage
  env:
    Parameters__openaiKey: ${{ secrets.OPENAI_KEY }}
  run: |
    dotnet test \
      --configuration Release \
      --no-build \
      --results-directory ./TestResults \
      -- \
      --coverage \
      --coverage-output-format cobertura \
      --coverage-settings testconfig.json \
      --report-trx
```

Note there's **no Docker setup step**. Testcontainers and Aspire both use the
container runtime that GitHub's `ubuntu-latest` runner already provides, so the
full distributed application -- Postgres, MailPit, four services -- comes up with
no extra pipeline configuration at all.

Coverage gets rendered by the ReportGenerator action, with `MarkdownSummaryGithub`
added specifically so it can be posted as a comment:

```yaml
- name: ReportGenerator
  if: always() # Run even if tests fail
  uses: danielpalme/ReportGenerator-GitHub-Action@5.5.11
  with:
    reports: "./TestResults/*.cobertura.xml"
    targetdir: "coveragereport"
    reporttypes: "MarkdownSummaryGithub,Html"
    tag: "${{ github.run_number }}_${{ github.run_id }}"
```

`if: always()` is doing real work there. A failed test run is exactly when you
most want the coverage report and the artifacts, so every reporting step is marked
this way.

Then the good part -- the results go where people will actually see them:

```yaml
- name: Add comment to PR
  if: github.event_name == 'pull_request'
  run: gh pr comment $PR_NUMBER --edit-last --create-if-none --body-file coveragereport/SummaryGithub.md
  env:
    PR_NUMBER: ${{ github.event.number }}

- name: Publish coverage in build summary
  run: cat coveragereport/SummaryGithub.md >> $GITHUB_STEP_SUMMARY
```

`--edit-last --create-if-none` is a small kindness: it updates the existing
coverage comment on each push rather than burying the PR conversation under
fifteen near-identical bot comments.

Everything a human might want to dig into gets staged into one downloadable
artifact -- the merged coverage report, the merged TUnit report, and the Playwright
screenshots and videos:

```yaml
- name: Stage combined artifacts
  if: always()
  run: |
    mkdir -p combined-report/coverage combined-report/merged-report combined-report/playwright
    cp -r coveragereport/. combined-report/coverage/ 2>/dev/null || true
    cp -r ${{ runner.temp }}/tunit-aggregate/**/merged-report.html combined-report/merged-report/ 2>/dev/null || true
    cp -r ./tests/CarvedRock.AppTests/bin/Release/net10.0/playwright-artifacts/. combined-report/playwright/ 2>/dev/null || true

- name: Upload combined test artifacts
  if: always()
  uses: actions/upload-artifact@v7
  with:
    name: test-artifacts
    path: combined-report
```

Being able to download a video of the exact browser session that failed on a CI
runner you can't SSH into is, I think, the single biggest quality-of-life
improvement in this whole setup.

![The GitHub Actions run summary with coverage and downloadable artifacts](/images/triple-threat/ci-artifacts.png)

## Coding Agent Setup

Here's the thesis of this last section: **everything above is even more valuable
to a coding agent than it is to you.** An agent can't eyeball a browser or "just
try it." What it can do is run a command, read structured output, and iterate --
and this framework is unusually good at giving it all three.

Four things to set up.

**1. Initialize the repo for your agent.** Whatever you use, this usually means a
`CLAUDE.md` or `AGENTS.md` in the repo root describing the architecture and
conventions.

**2. Run `aspire agent init`.** This installs the Aspire skills that let an agent
understand and interact with your distributed app properly -- starting and
stopping it, listing resources, reading logs and traces -- instead of guessing at
`dotnet run` and hardcoded ports. See the
[Aspire docs on AI coding agents](https://aspire.dev/get-started/ai-coding-agents/).

**3. Wire up MCP servers.** Two are worth having here:

```json
{
  "mcpServers": {
    "aspire": {
      "command": "aspire",
      "args": ["agent", "mcp"]
    },
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

The Playwright MCP server is the one that surprised me. It lets the agent drive a
real browser to *explore* the app -- find the actual accessible name of a button,
confirm a flow works by hand -- before it writes the test that asserts on it. That
turns "write a Playwright test" from guesswork into transcription.

**4. Tell the agent where the results are.** This is the highest-leverage thing you
can add, and it's a paragraph:

> Running the tests is a single script (`test-with-coverage.ps1`). After a run,
> `coveragereport/Summary.txt` has a per-assembly/class coverage rollup -- read
> that instead of the cobertura XML. Machine-readable test results are in
> `TestResults/*.tunit-report.json` (per-test status, duration, output, and spans).
> Raw cobertura files are GUID-named and don't identify their project, so don't
> grep them directly.

Every one of those sentences prevents a specific failure mode I watched happen.
Without them an agent will cheerfully grep 300KB of cobertura XML, or try to match
a GUID filename to a project, and burn a lot of context doing it. The
`tunit-report.json` files are the important pointer: an agent can read exact
failure output and per-test timings without parsing HTML or scraping console
output.

### Planning and Implementing Features

The workflow that's been working for me is: **plan first, save the plan in the
repo, and make tests non-negotiable in the plan.**

Here's a real prompt from this repo:

```txt
Need to plan the implementation of a new feature: the AI chat accessible
from the listing page should be able to add a recommended item or items
to the cart.  When the chat provides the recommendations, it should have a
way to ask a follow up question about whether they would like any
of the items added to the cart, and if the user says yes in some way,
then the appropriate items should be added to the cart. Make sure that
the text on the cart button is updated to reflect the added item. A playwright
test or test (with a video recording) should be created to verify
the new functionality.  Please create a plan for this work that I can
review and save it in the repo so that I can edit manually if needed.
```

Three details in that prompt are doing the work:

* **"create a plan ... save it in the repo so that I can edit manually"** -- you
  get a review gate before any code is written, in a file you can edit.
* **"A playwright test (with a video recording) should be created"** -- tests are
  part of the deliverable, not a follow-up.
* **The specific UI assertion** ("the text on the cart button is updated") gives
  the agent something concrete and checkable to aim at.

The plan that came back (saved to `docs/plans/chat-add-to-cart-plan.md`) correctly
identified that this wasn't "just add an MCP tool" -- the chat was fully stateless,
so a follow-up "yes, add it" had nothing to resolve "it" against -- and proposed a
design with trade-offs for me to sign off on. That's a conversation worth having
*before* implementation, and it only happened because the plan was a separate
reviewable step.

But the part I want to highlight is what that plan document looked like *after*
implementation. It grew a section called "Implementation notes (found while
building and testing this)," including this:

> **The test's `Cart (1)` assertion is not safe against reruns.** Unlike the
> ApiTests' per-session Testcontainers DB, this AppHost's Postgres survives across
> separate test runs -- carrying a leftover item count from a previous run into the
> next one. The test now clears the cart first.

That's a real bug in the *test*, found by running it repeatedly against the real
app, and it's exactly the class of problem you can't reason your way to from the
source. The final entry in that document reads: *"Full `./test-with-coverage.ps1`
run, 54/54 tests passing, 82.3% overall line coverage."*

And in the repo, a video called `CustomerCanAddRecommendedProductToCartViaChat.webm`
showing the feature working end to end -- produced by the agent's own test run, on
its own machine, without me watching.

{{< video src="/images/triple-threat/CustomerCanAddRecommendedProductToCartViaChat.webm" >}}

That's the loop worth building toward: the agent plans, implements, runs the real
distributed application, reads structured results, fixes what it finds, and hands
you a video of the feature working plus a coverage number. The tests aren't
overhead on agentic development -- they're the feedback signal that makes it work
at all.

## Wrapping Up

The three tools each solve a different problem, and they compound:

* **TUnit** makes tests fast, parallel, and -- through its report -- *legible*,
  with per-test logs and traces you didn't have to instrument.
* **Aspire** means "the whole application" is a thing you can start from a test
  method, with real containers and no hardcoded ports or deployment step.
* **Playwright** covers the last mile through a real browser, with screenshots and
  videos as first-class artifacts.

54 tests. 82.3% coverage. About a minute to run. One script locally, the same commands
in CI (slightly longer to run due to more installs / plumbing), and structured output
that both humans and agents can act on.

The full working repo is at
[dahlsailrunner/automated-testing-aspnetcore10](https://github.com/dahlsailrunner/automated-testing-aspnetcore10)
-- and the `readme.md` has a list of deliberately-omitted tests if you'd like to
practice with the setup yourself.
