---
---
The precursor to the Pirc Platform was a fairly complex web application that
consisted of many different API layers but was not very carefully organized 
from an API perspective.  We spent time proving out a lot of theories, but 
built up some cruft in the process that made it hard to publish the API as
a standalone offering.  We undertook a major refactoring of the code to create
Pirc Platform, with an eye towards making all of this sophisticated 
functionality that we have built available as a modular set of services that
may be used by different applications.

This is a so-called "API First" mentality.  In our case, this means that we
start with Java interface classes to define any and all functionality that
we aim to support in the system.  When reviewing new requirements, the first
question is always to look at the platform-api package to see if the requested
feature can be achieved with the existing interfaces.  If it cannot, then 
the first order of business is to figure out how to augment those interfaces
to offer the required functionality.

The controllers in our web applications are written in Java and deal only 
with the interface classes.  The web applications have no exposure to the 
platform implementation (which is written in Scala), they only have access 
to the interfaces.  But just in the same way that the our controllers know
nothing about the platform implementaiton, our platform implementation also
knows nothing about the servlet API.  The philosophy here is that we do not
want our platform API or implementation to be unnecessarily bound to the 
web execution context.  The result of this is that we may need a wrapper module
around the root API that does any of the servlet magic we may need (digging
through cookies, parsing HTTP request bodies, etc.).  But for the most part,
application requests are handled by the platform implementation classes.

### Application Development Guidelines

1. Controllers in our apps will deal solely with platform API instances.  These instances are obtained by using the `pirc.platform.api.Root` object and the various helpers and factories that live within that Root instance.
1. API interface contracts for domain objects are immutable, meaning that only getter methods are offered as part of the API.  The interfaces may offer other API endpoints for doing mutation (e.g. `changePassword(String from, String to)` or `addMatchup(Coupon)`) but we do not offer per-field setters.  Experience shows that having those setters means that you API will lose its expressiveness over time, with more and more controllers being created to manipulate the objects in different ways without the API needing to reflect what those changes may be.
1. Even though individual fields don&apos;t offer setters, a domain object will typically offer an `update` method which receives as an arguemnt an instance of the object itself.  The idea here is that the update process should first involve obtaining an instance of the object in question, of then making changes to that object by some means, then of submitting the changed object to the `update()` method to be applied to the domain.
1. As is typical for ReST applications, `GET` is used to retrieve objects, `PUT` is used to write update them, and `POST` is used to create new objects.  The same URL that is used to retrieve an object with the `GET` method may be used to delete that object with the `DELETE` method.
1. Controllers return JSON data, and the `POST` and `PUT` methods consume JSON data.  Furthermore, the `GET` and `PUT` methods should be symmetrical, meaning that you should be able to `GET` data from a URL and then you `PUT` it back without error.  It is expected, therefore, that web applications would `GET` an object to obtain its state, then the object state would be changed in some way, and the whole object would then be `PUT` back to the same URL to save those changes.
1. In the controllers, the preferred way of parsing an inbound post is to create a JSON binder object.  This binder will use the JSON API to pick individual fields out of the inbound JSON, and it will implement some domain object in order to present those values back to the application.  In this way, a JSON binder may then be used directly as an argument into an `update()` method.  The result would be code that looks like this:
```
  Person binder = new PersonBinder(rqst);
  API.getPerson(id).update(binder);
```
1. Field validations will take place inside the constructor of the binder, and will be thrown as field validation error and handled by the framework to return the correct HTTP status and payload.
