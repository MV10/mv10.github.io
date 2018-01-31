---
title: .NET Core Class Library Dependency Injection
tags: c# .net class.library dependency.injection
header:
  image: "/assets/2018/01-31/header1280.png"
  teaser: "/assets/2018/01-31/header1280.png"

---

Adding dependency injection to class libraries in .NET Core 2.0 is almost as easy as DI in ASP.NET, but class libraries may require a few extra tricks that aren't immediately obvious.

<!--more-->

Any MVC developer is familiar with the stack of `services.Add` commands in `Startup`, and many will have encountered `AddTransient`, `AddScoped`, and possibly `AddSingleton` for registering new service interfaces for injection. Most blog articles online never go beyond the `Startup` and controller scenario, and it can be difficult to find good, correct advice about how to extend DI into an external class library.

If you're reading this, it's likely you already understand DI. The primary benefit touted for DI is unit test mocking, but in my opinion the real value is consistency. The older approach of `static` classes and methods for libraries definitely prevents mocking, but it also denies the opportunity to leverage other DI features like scoped and lazy instantation.

## NuGet Package for DI

The good news is Microsoft externalized their DI solution when they began working on ASP.NET Core. Now it lives in the NuGet package [Microsoft.Extensions.DependencyInjection](https://www.nuget.org/packages/Microsoft.Extensions.DependencyInjection/). Pull that package into your class library and you're nearly finished!

This package gives you access to several important tools. We'll focus on three of them.

## Extending `IServiceCollection`

This is the familiar `services` collection used in ASP.NET Core's `Startup` class. It stores all of the available services and various configuration details for each registered service. In DI terminology, that `Startup` process is known as the [composition root](http://blog.ploeh.dk/2011/07/28/CompositionRoot/). In practical terms, it's the place where injectable services are registered.

You don't need ASP.NET to use DI. A simple console program could act as the composition root by creating an instance of this collection and using it to register services.

```
public class Program
{
    public static async Task Main(string[] args)
    {
        var services = new ServiceCollection();
        services.AddUsefulService();
    }
}
```

Your class library should make it easy for the consuming application to register its services. In ASP.NET the many `services.Add` options (like `AddUsefulService` above) are implemented as extensions of `IServiceCollection`, and your class library can do exactly the same thing.

```
namespace Example.Library.Azure
{
    public static class IServiceCollectionExtension
    {
        public static IServiceCollection AddExampleAzureLibrary(this IServiceCollection services)
        {
            services.AddScoped<IGetSecret, GetSecret>();
            services.AddScoped<IKeyVaultCache, KeyVaultCache>();
            services.AddScoped<IBlobStorageToken, BlobStorageToken>();
            services.AddScoped<IBlobWriter, BlobWriter>();
            return services;
        }
    }
}
```

Once this is built and your client application has a reference to the library, it can register the library's services with a single line of code. This is how the example abouve would be used in an ASP.NET Core `Startup` class.

```
using Example.Library.Azure;

namespace Website
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddExampleAzureLibrary();
        }
    }
}
```

I usually take the approach of adding an extension for each namespace in the library, and then one big extension that pulls together all the namespace-level registrations. If your library is particularly complex, you may want to take a different approach, creating several extensions that register various logical commonly-used combinations of your namespaces. It's a decision to be made on a case-by-case basis.

```
using Example.Library.Azure;
using Example.Library.Database;
using Example.Library.Security;

namespace Example.Library.Utilities
{
    public static class IServiceCollectionExtension
    {
        public static IServiceCollection AddExampleLibrary(this IServiceCollection services)
        {
            services.AddExampleAzureLibrary();
            services.AddExampleDatabaseLibrary();
            services.AddExampleSecurityLibrary();
            return services;
        }
    }
}
```

## Embrace the Anti-Pattern

The other two tools we'll discuss from the DI NuGet Package are:

* `ActivatorUtilities`
* `IServiceProvider`

These allow us to retrieve registered services without declaring dependencies in the constructor.

At this point, the Pattern Priesthood will begin freaking out, reciting mystical lore: "Requesting services without dependency injection is the [Service Locator](http://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/) anti-pattern!" This is often a valid concern, but we'll take a look at a couple of equally-valid exceptions to this _advice_. (And _advice_ is all architectural patterns are. All too often [Architecture Astronauts](https://www.joelonsoftware.com/2001/04/21/dont-let-architecture-astronauts-scare-you/) invoke the Secret Truths of Patterns, to be treated by their flock as inviolable rules handed down by Authorities. Hopefully one of these Authorities has declared _that_ to be an anti-pattern!)

All joking aside, I will emphasize that I do agree DI is usually preferable to Service Locator (otherwise there would be little point to writing this article), but class libraries are a special case where "doing the wrong thing for the right reasons" sometimes applies.

.NET Core DI uses the common approach of declaring dependencies in the constructor. The parameters define the dependencies as interface references, and the DI system wires up the parameters with concrete instance references.

```
public class MyUtility
{
    private readonly IUsefulService svc;   // local reference so we can use the service

    public MyUtility(IUsefulService diSvc) // dependency delcared in constructor
    {
        svc = diSvc;                       // save the reference provided by DI
    }

    // other methods
}
```

The Service Locator anti-pattern is when a method inside the class uses some other technique to acquire a reference to an object managed by the DI system. The dependency is not declared in the constructor and the reference is not established when the class is created.

This article was inspired by my efforts to convert an old-style `static` class library to DI. There was existing code in both the library itself and dependent applications with certain patterns of usage relating to the `static` design that would have been very time-consuming and expensive to refactor into a 100% pure DI architecture.

Let's examine two cases where the Service Locator anti-pattern was more useful than harmful.

## Minimizing Complexity

The first case where it's reasonable to apply Service Locator is when the alternative adds significant complexity. Every developer has run into circular references. They almost always indicate there is a problem with the overall design, but on rare occasions they make sense. This is a prime example of "doing the wrong thing for the right reasons" you may encounter while developing class libraries. (If you want examples from a Higher Authority than me, check out the circular references within the .NET Framework; Microsoft even uses multi-pass compilation with [custom tooling](https://stackoverflow.com/a/1316587/152997) to build these DLLs.)

The `static` library I was converting to DI had just such a scenario. The classes were very simple. The separation of concerns was very clean: one encapsulated routine, low-level database access, and the other cached database schema and performed related operations. Some client apps and parts of the library itself would reference the database class first, and others would reference the schema first. If the schema cache wasn't initialized, the schema class would use the database class to open a connection. The very same database method that opened connections checked that the schema was initialized since many other database methods used the schema for things like parameter typing. It was a classic circular reference scenario.

![circular1](/assets/2018/01-31/circular1.jpg)

The standard recommendation for resolving circular dependencies is to analyze both classes and break out the dependencies into a third class. Each of the original classes can then depend on the new third class. However, this just felt wrong. As I described above, both classes felt well-designed and appropriately-sized. Breaking them up purely to make DI happy would also require significant changes throughout the rest of the code base. The standard answer just felt wrong.

![circular1](/assets/2018/01-31/circular2.jpg)

## `IServiceProvider`

It turns out there was exactly one call to the database class from the schema class. It was far cleaner to simply embrace the Service Locator anti-pattern and grab a reference to the database class on the rare occasion it was needed. The DI package includes a `ServiceProvider` class which does exactly that: provides references to services registered for DI.

Before the change, the `static` classes looked something like this.

```
public static class SchemaCache
{
    private static List<TableDefinition> Table { get; }
    private static List<ColumnDefinition> Column { get; }

    public static bool IsInitialized { get => (Table == null); }

    public static async Task Initialize(SqlConnection conn = null;)
    {
        var connection = (conn != null) ? conn : await Database.GetOpenConnection();
        // ... load the schema
    }

    public static async Task SomeOtherMethod()
    {
        if(!IsInitialized) await Initialize();
        // ...
    }
}


public static class Database
{
    public static async Task<SqlConnection> GetOpenConnection()
    {
        var connection =  new SqlConnection(connectionstring);
        await connection.OpenAsync();
        if(!SchemaCache.IsInitialized) SchemaCache.Initialize(connection);
        return connection;
    }
}
```

The naive circular reference version is shown below. This throws a circular dependency exception at runtime the first time either class is referenced.

```
public class SchemaCache
{
    private readonly IDatabase db;
    public SchemaCache(IDatabase diDb)
    {
        db = diDb;
    }

    public async Task Initialize(SqlConnection conn = null;)
    {
        var connection = (conn != null) ? conn : await db.GetOpenConnection();
        // ... load the schema
    }

    // other properties and methods omitted
}


public class Database
{
    private readonly ISchemaCache schema;
    public Database(ISchemaCache diSchema)
    {
        schema = diSchema;
    }

    public async Task<SqlConnection> GetOpenConnection()
    {
        var connection =  new SqlConnection(connectionstring);
        await connection.OpenAsync();
        if(!schema.IsInitialized) schema.Initialize(connection);
        return connection;
    }
}

```

We eliminate the circular reference by removing the `IDatabase` dependency in the `SchemaCache` constructor, replacing it with a dependency on `IServiceProvider` instead. The `Database` class is unchanged from the version shown above -- it remains dependent upon an injected reference to `SchemaCache`.

```
public class SchemaCache
{
    private readonly IServiceProvider provider;
    public SchemaCache(IServiceProvider diProvider)
    {
        provider = diProvider;
    }

    public async Task Initialize(SqlConnection conn = null;)
    {
        var db = diProvider.GetService<IDatabase>();
        var connection = (conn != null) ? conn : await db.GetOpenConnection();
        // ... load the schema
    }

    // other properties and methods omitted
}
```

## Creating Objects with Dependencies

The second case where I felt the Service Locator anti-pattern was appropriate is very easy to define. Part of the library needed to create multiple instances of a class which itself declared a dependency. (Remember, this used to be a `static` library: the class used to simply call the `static` methods it needed from the dependency.) It is very definitely not a good idea to call `new` on a class with dependencies -- it defeats the whole point of doing DI at all: the creating class becomes much more tightly-tied to the target class, the creating class may need to declare dependencies it doesn't otherwise need purely so it can create the target class, and so on.

The library created a `List` of data items, and those objects depended on another class for data-sanitization. The original `static` version had the following structure (greatly simplifed here, obviously).

```
public static class Sanitizer
{
    public static string SanitizeInput(string input) { ... }
}


public class DataItem()
{
    public string Data { get; set; }

    public string ProcessData()
    {
        return Sanitizer.SanitizeInput(Data);
    }
}


public class DataManager()
{
    public List<DataItem> ReadData(List<string> rawData)
    {
        var list = new List<DataItem>;
        foreach(string data in rawData)
        {
            var item = new DataItem();
            item.Data = data;
        }
        return list;
    }
}
```

The DI version of this would register `ISanitizer` as a DI-ready service, and `DataItem` would add a constructor requesting an instance of an `ISanitizer` object. But how does `DataManager` iterate over the raw input data, creating new `DataItem` instances as it goes? Each new `DataItem` has dependencies that need to be resolved.

## `ActivatorUtilities`

The solution is the `CreateInstance` method in the `ActivatorUtilities` class from the DI package. As the name suggests, it will create a new instance of the requested type, processing any DI requirements in the process. The method requires an instance of `IServiceProvider` so the class in question has to declare a dependency on that.

The DI-friendly version of the three classes are as follows.

```
public class Sanitizer
{
    public string SanitizeInput(string input) { ... }
}


public class DataItem()
{
    private readonly ISanitizer sanitizer;
    public DataItem(ISanitizer diSanitizer)
    {
        sanitizer = diSanitizer;
    }

    public string Data { get; set; }

    public string ProcessData()
    {
        return sanitizer.SanitizeInput(Data);
    }
}


public class DataManager()
{
    private readonly IServiceProvider provider;
    public DataManager(IServiceProvider diProvider)
    {
        provider = diProvider;
    }

    public List<DataItem> ReadData(List<string> rawData)
    {
        var list = new List<DataItem>;
        foreach(string data in rawData)
        {
            var item = ActivatorUtilities.CreateInstance<DataItem>(provider, null);
            item.Data = data;
        }
        return list;
    }
}
```

## Conclusion

As you can see, it's very easy to provide first-class support for dependency injection in .NET Core class libraries. The library itself can leverage dependency injection internally, as well as being available service-style to other libraries or applications that consume the library.

