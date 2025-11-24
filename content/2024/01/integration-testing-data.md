---
title: "Integration Testing for ASP.NET APIs (2/3) - Data" 
date: 2024-01-23T05:51:55-05:00 
summary: "Using TestContainers, SQLite, and Postgres to perform automated integration tests against an ASP.NET Core API" 
codeMaxLines: 30 .
codeLineNumbers: true 
toc: true
tags:
  - C#
  - API
  - ASP.NET
  - Testing
  - XUnit
  - Bogus  
  - TestContainers
  - FluentValidation
  - SQLite
  - Postgres
---

{{% notice note "Where's the code?" %}}
Fully working examples for this code are in the `02-api-with-sqlite`
and `03-api-with-postgres`
folder of the GitHub repo with the examples for this series
of posts:

- [02-api-with-sqlite](https://github.com/dahlsailrunner/testing-examples/tree/main/02-api-with-sqlite)
- [03-api-with-postgres](https://github.com/dahlsailrunner/testing-examples/tree/main/03-api-with-postgres)
{{% /notice %}}

In the [previous post](../integration-testing) we got started
with some basic integration tests.  In this one we'll use a
real data store for the tests - first SQLite and next Postgres.

## API Changes

I added a simple `ProductsController` that uses an EF Core `DbContext`
to retrieve existing products (two `GET` methods) and to create new ones
(a `POST` method).

Here's the code:

```c#
[HttpGet]
public async Task<IEnumerable<Product>> GetProducts(string category = "all")
{
    return await context.Products
        .Where(p => p.Category == category || category == "all")
        .ToListAsync();
}

[HttpGet]
[Route("{id}")]
public async Task<Product> Get(int id)
{
    var product = await context.Products.FindAsync(id);
    return product ?? throw new KeyNotFoundException($"Product with id {id} not found");
}

[HttpPost]
public async Task<Product> Post(Product product, CancellationToken token)
{
    await validator.ValidateAndThrowAsync(product, token);
    await context.Products.AddAsync(product, token);
    await context.SaveChangesAsync(token);
    return product;
}
```

It's intentionally pretty simple but does have a few nuances:

- Any `GET` by ID with an ID that doesn't exist with return a `404 Not Found` with a `ProblemDetails` response
- The `POST` method uses [FluentValidation](https://docs.fluentvalidation.net/en/latest/)
to perform validation on the submitted object and will return a `400 Bad Request` with
`ProblemDetails` if validation fails. The `ProblemDetails` object will include information
about the validation failure(s).

In our tests, we want to start with some sample data and make sure all of the
above actually works like we want.

## Establishing Sample Data - SQLite

I've been using [SQLite](https://learn.microsoft.com/en-us/ef/core/providers/sqlite)
for most of my demo projects lately -
it doesn't require any installs or anything to be running locally but is
supported by EF Core. So people can simply clone a repo and run it,
and things should work: super handy.

The `DbContext` I created for the demo API is really simple:

```c#
public class LocalContext(DbContextOptions<LocalContext> options) : DbContext(options)
{
    public DbSet<Product> Products { get; set; } = null!;
}
```

When the API starts up (in `Program.cs`) I call a method that will either
simply perform any EF Core migrations or (if we're in the `Development`)
environment, create some hard-coded sample data:

```c#
public static void InitializeDatabase(this IServiceProvider serviceProvider, string environmentName)
{
    // scope is required because dbContext is a scoped service
    using var scope = serviceProvider.CreateScope(); 
    using var context = new LocalContext(scope.ServiceProvider.GetRequiredService<DbContextOptions<LocalContext>>());

    if (environmentName == "Development")
    {
        StartWithFreshDevData(context);
    }
    else
    {
        context.Database.Migrate(); // just make sure we're up-to-date
    }
}
```

You can see the `Program.cs` code and the `StartWithFreshDevData` in the repo -
they're not that interesting and look like what you would expect. One note
about `StartWithFreshDevData` is that it deletes the entire database on start,
then does migrations, and finally inserts some hard-coded data.

The two main points to recall here are:

- A `DbContext` is created during API startup
- Either a migration is completed or (if `Development`) fresh sample data is created

### Using `Fixtures` in XUnit

We can use a [Fixture](https://xunit.net/docs/shared-context)
(see the "Class Fixture" section) to share context between test classes
and conditions in our XUnit tests.

In our `DatabaseFixture`, we'll use an in-memory version of SQLite so that
we don't create new files for the db when running tests, and we'll also
generate our own data for the tests (the `Development` data is only 6 hard-coded
products and we want to test with more).

Here's the starting point for a `DatabaseFixture` class that at least gets
an in-moemory SQLite database created:

```c#
public class DatabaseFixture : IAsyncLifetime
{
    public const string DatabaseName = "InMemTestDb;Mode=Memory;Cache=Shared;";
    private LocalContext? _dbContext;

    public async Task InitializeAsync()
    {
        var options = new DbContextOptionsBuilder<LocalContext>()
            .UseSqlite($"Data Source={DatabaseName}")
            .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)
            .Options;

        _dbContext = new LocalContext(options);

        await _dbContext.Database.EnsureDeletedAsync();
        await _dbContext.Database.EnsureCreatedAsync();
        await _dbContext.Database.OpenConnectionAsync();
        await _dbContext.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        if (_dbContext != null) await _dbContext.DisposeAsync();
    }
}
```

The `DatabaseName` proeprty defines the name of the SQLite database
and specifies that it will be in-memory. The rest of the code should
be pretty familiar-looking - just creating a `DbContext` and
doing migration-type stuff.

### Sample Data Creation with `Bogus`

To create sample data (I wanted more than 6 hard-coded products and didn't
want to have to make them up myself), I used the handy [Bogus NuGet package](https://github.com/bchavez/Bogus).

I added some code to the above `DatabaseFixture` class that works the magic
(I am omitting some of the original code shown above - but it's all in the
GitHub repo):

```c#
public class DatabaseFixture :IAsyncLifetime
{
    //...
    public List<Product> OriginalProducts = [];
    public Faker<Product> ProductFaker { get; } = new Faker<Product>()
        .RuleFor(p => p.Name, f => f.Commerce.ProductName())
        .RuleFor(p => p.Price, f => Convert.ToDouble(f.Commerce.Price()))
        .RuleFor(p => p.Description, f => f.Commerce.ProductDescription())
        .RuleFor(p => p.Category, f => f.Commerce.Categories(1)[0])
        .RuleFor(p => p.ImgUrl, f => f.Image.PicsumUrl());

    public async Task InitializeAsync()
    {
        // ... original code to create dbcontext and do migrations
        CreateFreshSampleData(100);

        OriginalProducts = await _dbContext.Products.ToListAsync();
    }

    private void CreateFreshSampleData(int numberOfProductsToCreate)
    {
        var products = ProductFaker.Generate(numberOfProductsToCreate);
        _dbContext!.Products.AddRange(products);
        _dbContext.SaveChanges();
    }
    //...
}
```

The code above creates a `Faker<Product>` to generate sample `Product`
objects and uses some handy built-in methods to the `Bogus` library
to create things like product names and image URLs.

The invocation of the `CreateFreshSampleData` specifies to create
100 products - handy if I wanted to test paging, and this number can
be whatever I want it to be.  Really useful.

I capture the `OriginalProducts` based on exactly this first set
of data that gets created so that I can reference them during
tests.

### Aside: The `CollectionFixture` in XUnit

{{% notice note "When to Use Collection Fixtures" %}}
This comes straight from the XUnit docs:
When to use: when you want to create *a single test context and share it among tests in several test classes*, and have it cleaned up after all the tests in the test classes have finished.
{{% /notice %}}

Since the database will be used by any / all API routes and we (likey) will
have multiple API routes that we want to put in different test classes, this
makes total sense for us to use.

We create a class definition like this (I generally put mine at the bottom
of the file with the `IClassFixture` implementation - `DatabaseFixture.cs` in
this case):

```c#
[CollectionDefinition("IntegrationTests")]
public class DatabaseCollection : ICollectionFixture<DatabaseFixture>
{
    // This class has no code, and is never created. Its purpose is simply
    // to be the place to apply [CollectionDefinition] and all the
    // ICollectionFixture<> interfaces.
}
```

The generic parameter for the `ICollectionFixture<T>` value should
be your custom fixture - the one for this example is `DatabaseFixture`.

Also, the `[CollectionDefinition("IntegrationTests")]` attribute should
be placed above your test classes as well, so a test class declaration
would look something like this:

```c#
[Collection("IntegrationTests")]
public class ProductControllerTests(CustomApiFactory<Program> factory, 
  ITestOutputHelper output, DatabaseFixture dbFixture) : BaseTest(factory, output), IClassFixture<CustomApiFactory<Program>>
```

All of the above is "plumbing" to make sure your tests "wire up" properly.

### Replacing the `DbContext` Used by the API

Now that we've got the in-memory SQLite database created and have
added some sample data to it, we need to make sure that this
database is used by the API code when we run our tests.

This is where we take more advantage of the `CustomApiFactory` we
established in the [previous post](../integration-tests).

We specified `builder.UseEnvironment("test")` so we should not
be creating any hard-coded sample data (it only does that in
"Development"), and we can add some code into the same `ConfigureWebHost`
to achieve what we want:

```c#
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.UseEnvironment("test");

    builder.ConfigureServices(services =>
    {
        var dbContextDescriptor = services.SingleOrDefault(
            d => d.ServiceType == typeof(DbContextOptions<LocalContext>));
        services.Remove(dbContextDescriptor!);

        var ctx = services.SingleOrDefault(d => d.ServiceType == typeof(LocalContext));
        services.Remove(ctx!);

        // add back the container-based dbContext
        services.AddDbContext<LocalContext>(opts =>
            opts.UseSqlite($"Data Source={DatabaseFixture.DatabaseName}")
                .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
    });
}
```

The code above ***removes** the `LocalContext` that was added to the services
collection in `Program.cs` and then adds the one that we created in the
`DatabaseFixture` class -- note the use of the `DatabaseFixture.DatabaseName`
property in the `UseSqlite` call.

With this code in place, any tests we write should use a new in-memory
instance of SQLite and the sample data we generated in the `DatabaseFixture` --
sweet!

### Writing Tests

With the [`HttpClient` extensions we created in the previous post](../integration-testing/#step-two-add-httpclient-extensions-and-a-base-class)
and the `DatabaseFixture` and the `OriginalProducts` list that we have, Writing
highly readable tests is pretty straight-forward.  Here's a first test for the
`GET` many method:

```c#
[Fact(DisplayName = "Get all products")]
public async Task GetProductsReturnsProducts()
{
    var retrievedProducts = await Client.GetJsonResultAsync<List<Product>>("/v1/products", 
        HttpStatusCode.OK);

    Assert.NotNull(retrievedProducts);

    // NOTE: because the tests are not explicitly ordered and there are
    // tests that add products, we may have more products than we started with
    Assert.True(dbFixture.OriginalProducts.Count <= retrievedProducts.Count);

    var randomProduct = BogusFaker.PickRandom(dbFixture.OriginalProducts);
    Assert.Contains(retrievedProducts, c => c.Id == randomProduct.Id);

    var product = retrievedProducts.First(c => c.Id == randomProduct.Id);
    Assert.Equal(randomProduct.Name, product.Name);
}
```

The above calls the `v1/products` route and expects a status of `OK`.

It also finds a random product from the `OriginalProducts` list that
we created in the `DatabaseFixture` and makes sure that product is
in the list of products we got back.

Here's another sample that checks for a validation error when trying
to `POST` a new product:

```c#
[Fact(DisplayName = "Cannot create product with duplicate name")]
public async Task CreateProductDuplicate()
{
    var existingCompany = BogusFaker.PickRandom(dbFixture.OriginalProducts);

    var newProduct = dbFixture.ProductFaker.Generate();
    newProduct.Name = existingCompany.Name;

    var problem = await Client.PostJsonForResultAsync<ProblemDetails>(
        $"/v1/products", newProduct, HttpStatusCode.BadRequest);

    Assert.Equal("Validation error(s) occurred.", problem.Title);
    Assert.Equal("A product with the same name already exists.", problem.Extensions["Name"]!.ToString());
}
```

The above uses a `Faker` to generate a new `Product` record, and then
overwrites the `Name` property with a random one from the `OriginalProducts` list.

Then it does a `POST` and expects a `Bad Request` response and evaluates the
returned `ProblemDetails` record to make sure the right information was returned
to the API caller.

You can explore the rest of the tests in the GitHub repo.

## Switching to Use Postgres

The technique we used above makes for some great tests - as defined by
reliability, readability, and ease of writing the test conditions.

And while using SQLite is fine for demo apps, most real production applications
will use "eavier" databases like Postgres and SQL Server.

But these techinques apply equally well to those databases too!

In the [03-api-with-postgres](https://github.com/dahlsailrunner/testing-examples/tree/main/03-api-with-postgres)
folder of the GitHub repo, I've updated the API to use Postgres instead of SQLite.

This was a matter of:

- Swapping out the EF Core SQLite NuGet package for the Npgsql one
- Deleting the `Data/Migrations` folder and recreating the EF core migrations
- Updating `Program.cs` to `UseNpsql()` when adding the `LocalContext` to the services collection (which uses a connection string defined in `appsettings.json`)

### Updating the Tests to use Postgres - with `TestContainers`

Postgres doesn't run in-memory (and if it did we probably wouldn't want to
test that way anyway).

But there's a handy NuGet package called [TestContainers](https://testcontainers.com)
that lets you use container technology to host things like a Postgres database
in a container that will exist during your test run and then be eliminated
once the run completes.  Again - super handy.  It also has built-in support for a
[bunch of common modules](https://testcontainers.com/modules/) - which includes
Postgres (and SQL Server (MSSQL) and lots more).  

{{% notice note "Using TestContainers Requires a Container Runtime" %}}
You need to have a Docker (container) runtime set up on your computer
to use TestContainers. On Windows, that is two steps:

- [Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
- Install a container desktop application (like [Rancher Desktop](https://rancherdesktop.io/) or [Docker Desktop](https://www.docker.com/products/docker-desktop/))
{{% /notice %}}

Postgres is in the `Testcontainers.PostgreSql` NuGet package. Using it in our
`DatabaseFixture` class looks like the updated code below:

```c#
public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _dbContainer =
        new PostgreSqlBuilder()
            .WithDatabase("simple_api_tests")
            .WithUsername("postgres")
            .WithPassword("notapassword")
            .Build();

    public string TestConnectionString => _dbContainer.GetConnectionString();

    private LocalContext? _dbContext;

    // OriginalProducts and Faker<Product> declarations

    public async Task InitializeAsync()
    {
        await _dbContainer.StartAsync();

        var optionsBuilder = new DbContextOptionsBuilder<LocalContext>()
            .UseNpgsql(TestConnectionString);
        _dbContext = new LocalContext(optionsBuilder.Options);

        await _dbContext.Database.MigrateAsync();

        //... rest of method and class
```

Note the handy `GetConnectionString` method that will return a connection
string that we can use to connect to the database - which we can use
in the `CustomApiFactory` when we need to replace the `DbContext` that
was wired up in `Program.cs` for the API:

```c#
// add back the container-based dbContext
services.AddDbContext<LocalContext>(options =>
    options.UseNpgsql(dbFixture.TestConnectionString));
```

Those were the only updates needed to use Postgres!  The tests themselves don't
change at all! You could use exactly the same technique if you were using
a different database, including SQL Server, MySql, and more.

## Next

In the [next post](../integration-testing-auth) we'll get into how to add authentication and authorization
logic into the tests.  

Stay tuned!
