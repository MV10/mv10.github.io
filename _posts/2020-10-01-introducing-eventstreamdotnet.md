---
title: Introducing the EventStreamDotNet Library
tags: c# .net eventstream eventsourcing events
header:
  image: "/assets/2020/10-01/header1280.jpg"
  teaser: "/assets/2020/10-01/header1280.jpg"

# standard header size is 1280 x 400
# image paths:
#   publish:                        (/assets/2020/mm-dd/pic.jpg)
#   edit from _post or _templates:  (../assets/2020/mm-dd/pic.jpg)
---

A free, easy-to-use library for Event Stream based data handling.

<!--more-->

Almost a year ago, I posted the article [Event Sourcing with Orleans Journaled Grains]({{ site.baseurl }}{% post_url 2019-12-05-event-sourcing-with-orleans-journaled-grains %}) which demonstrated how to implement the Event Streaming (often also called Event Sourcing) pattern in Orleans. I recently needed to implement this type of data handling in a non-Orleans-based system, so I decided to write a library to handle it. This is a short article introducing that library.

The package is called EventStreamDotNet. The NuGet package is [here](https://github.com/MV10/EventStreamDotNet) and the Github repository is [here](https://www.nuget.org/packages/EventStreamDotNet/1.0.0).

The library [documentation](https://github.com/MV10/EventStreamDotNet/blob/master/Docs/index.md) in the Github repository covers everything you'll need to know about using the library, so this article is more of an announcement than anything.

The Orleans article pretty thoroughly covered the architectural concepts behind Event Streams, and closely related concepts like Domain Driven Design and CQRS, so I won't rehash that here. The library documentation in the Github repository also has a page covering those topics, too. The repository's demo project's basic domain data model and domain events are also the same as the demo from the Orleans article (although how it works is very different, as you might imagine). Unlike the Orleans-based implementation, the library also supports projections (data extractions from snapshot updates).

Consequently, this article will jump straight into library usage, under the assumption that the reader is already up to speed with the concept of a domain data model, why you'd take this approach, and concepts like applying domain events to "evolve" the state of the domain data.

## The Domain Data Model

In order to use the library, you have to create your domain data model -- set of POCOs with properties and the various relationships between those classes. The domain model root class has to inherit from `IDomainModelRoot` and that requires you to provide a `string Id` property representing the unique ID that represents the instance of your domain model data.

Once you have that, you must define domain event classes. These are POCOs with properties that represent the data that changed as a result of the event. This is exactly what was done for the Orleans version, so refer to that article for more details. The library even requires domain event classes to derive from a base class by the same name, `DomainEventBase`.

And again, just like the Orleans example, you will then create a domain event handler -- a class which implements the library's `IDomainEventHandler<TDomainModelRoot>` interface. Unlike the Orleans implementation, however, the library populates a `DomainModelState` property before calling the event handler's `Apply` events.

Projections were not available in the Orleans implementation. Your application can provide a handler which implements the `IDomainModelProjectionHandler<TDomainModelRoot>` interface. Projection methods return `async Task` and just like the domain event handler, the class must provide a `DomainModelState` property which the projection methods use. Methods are marked with `[SnapshotProjection]` and `[DomainEventProjection(typeof(event))]` attributes to define when they should be invoked.

## Library Settings

The library exposes a group of configuration classes which are suited to being populated by the _Microsoft.Extensions.Configuration_ set of libraries -- most commonly used with `appsettings.json` files. The documentation covers the available settings and how to load it up, but it bears mentioning that multiple groups of these settings can be configured to apply to different domain data models within the same application.

The config file used by the repository's demo project looks like this:

```json
{
  "EventStreamDotNet": {
    "Database": {
      "ConnectionString": "Server=(localdb)\\MSSQLLocalDB;Integrated Security=true;Database=EventStreamDotNet",
      "EventTableName": "EventStreamDeltaLog",
      "SnapshotTableName": "DomainModelSnapshot"
    },
    "Policies": {
      "SnapshotFrequency": "AfterAllEvents",
      "DefaultCollectionQueueSize":  10
    },
    "Projection": {
      "ConnectionString": "Server=(localdb)\\MSSQLLocalDB;Integrated Security=true;Database=EventStreamDotNet"
    }
  }
}
```

Once the settings are loaded, you pass them into one of the library services (associated with the domain model root class), which leads to the next topic...

## Library Services

The library uses three services internally -- one that tracks configuration settings, another that tracks domain event handlers, and a third which tracks projection handlers. Each of these associate those elements with a particular domain model root, which is how the library manages multiple domain models within a single application. The services support dependency injection, but the use of dependency injection is not required, thanks to a helper class provided by the library.

An example of setting up the services for two domain models using dependency injection looks like this:

```csharp
// not shown: AppConfig reads appsettings.json

services.AddEventStreamDotNet(
    loggerFactory: null, // no debug logging
    domainModelConfigs: cfg =>
    {
        // settings are instances of EventStreamDotNetConfig read from appsettings.json
        cfg.AddConfiguration<Customer>(AppConfig.Get.CustomerModelSettings);
        cfg.AddConfiguration<HumanResources>(AppConfig.Get.HumanResourcesModelSettings);
    },
    domainEventHandlers: cfg =>
    {
        cfg.RegisterDomainEventHandler<Customer, CustomerEventHandler>();
        cfg.RegisterDomainEventHandler<HumanResources, HumanResourcesEventHandler>();
    },
    projectionHandlers: cfg =>
    {
        cfg.RegisterProjectionHandler<Customer, CustomerProjectionHandler>();
    });
);
```

Configuring the services without dependency injection is very similar (we'll talk about that last line in the next section):

```csharp
// not shown: AppConfig reads appsettings.json

var eventLibraryServices = new DirectDependencyServiceHost(
    loggerFactory: null, // no debug logging
    domainModelConfigs: cfg =>
    {
        cfg.AddConfiguration<Customer>(AppConfig.Get.CustomerEventStream);
        cfg.AddConfiguration<HumanResources>(AppConfig.Get.HumanResourcesEventStream);
    },
    domainEventHandlers: cfg =>
    {
        cfg.RegisterDomainEventHandler<Customer, CustomerEventHandler>();
        cfg.RegisterDomainEventHandler<HumanResources, HumanResourcesEventHandler>();
    },
    projectionHandlers: cfg =>
    {
        cfg.RegisterProjectionHandler<Customer, CustomerProjectionHandler>();
    });
);

var customers = new EventStreamCollection<Customer>(eventLibraryServices);
```

## Working With Your Data

The library provides two ways to work with instances of your domain data. 

The class `EventStreamManager` handles a specific instance of the data -- it has a `string Id` property representing the unique ID assigned to that object model, and has just three methods to interact with the data.

`GetCopyOfState` returns a copy of the domain model object as the manager sees it. The name reinforces the fact that the application doesn't have a reference to the "real" data. This is important in the Event Stream world, because the data model can only be altered by applying domain events to the model -- which only the manager is permitted to do.

`PostDomainEvent` and `PostDomainEvents` are how those changes are made. There are some options you can read about in the docs, but they basically store and apply the domain event objects your application defines, then return an updated copy of the domain data object. After events are stored and applied, any relevant projection methods are invoked.

The library also provides the `EventStreamCollection` class, which is how the application interacts with multiple manager instances (and therefore, multiple domain data model instances). It has the same three methods (which also require an ID), as well as a few methods relating to the underlying collection.

In dependency injection scenarios, you will normally only register an `EventStreamCollection` with singleton scope. There is rarely a scenarion that it will make sense to register an individual `EventStreamManager` object for injection.

## Conclusion

I'm pretty pleased with the state of the 1.0 release of this library. I have some enhancements in mind -- I really want to come up with a way to make projection configuration extensible -- and I might even do things like refactor the database handling into a separate package so that others can add support for more than just SQL Server. But generally speaking, I think EventStreamDotNet is a very clean plug-n-play solution for getting an Event Stream-based application up and running with very little fuss or ceremony.
