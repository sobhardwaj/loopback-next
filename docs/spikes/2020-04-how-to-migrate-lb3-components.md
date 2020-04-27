# How to migrate LB3 components

_GitHub issue: https://github.com/strongloop/loopback-next/issues/4099_

## Out of scope

Migration of the following component is out of scope of this spike:

- [OAuth2](https://loopback.io/doc/en/lb3/OAuth-2.0.html) - will be covered by
  auth&auth migration guide
- [Passport](https://loopback.io/doc/en/lb3/Third-party-login-using-Passport.html) -
  will be covered by auth&auth migration guide
- [Synchronization](https://loopback.io/doc/en/lb3/Synchronization.html) - won't
  be implemented in LB4. It's important to mention this fact in the migration
  guide.

## Table of contents

- [Techniques](#techniques)
  - [General](#general)
  - [Models, Entities and Repositories](#models-entities-and-repositories)
  - [REST API](#rest-api)
  - [Services (local and remote)](#services-local-and-remote)
  - [API transports](#api-transports)
  - [Authentication & authorization](#authentication--authorization)
  - [Introspection](#introspection)
- [Migration](#migration)
  - [Component layout & instructions for mounting to an app](#component-layout--instructions-for-mounting-to-an-app)
  - [LB3 component layout & mounting](#lb3-component-layout--mounting)
  - [Migration to LB4](#migration-to-lb4)
  - [Migrate context](#migrate-context)
  - [Migrate mixins](#migrate-mixins)
  - [Migrate Models, Entities and Repositories](#migrate-models-entities-and-repositories)
  - [Migrate REST API](#migrate-rest-api)
  - [Migrate API transports](#migrate-api-transports)
  - [Authentication & authorization](#authentication--authorization-1)
  - [Migrate Introspection](#migrate-introspection)
- [Overview of existing LB3 components](#overview-of-existing-lb3-components)
  - [Push notifications](#push-notifications)
  - [Storage component](#storage-component)
  - [Logging components](#logging-components)
  - [Access control](#access-control)
  - [Connectors](#connectors)
  - [Custom transports](#custom-transports)
  - [Inspecting models](#inspecting-models)
  - [Unsorted](#unsorted)
  - [Mixins](#mixins)

## Techniques

_This section provides a list of techniques compiled from existing LoopBack 3
components. Please check
[Overview of existing LB3 components](#overview-of-existing-lb3-components)
below if you are interested in learning more about where the techniques are
coming from._

### General

1. Expected component module layout and exports. How to structure the extension
   code to receive user-provided configuration & target app instance. What are
   the instructions for adding a LB4 component to a LB4 app.

2. A context shared by all parts of the application, allowing different layers
   to store and retrieve values like "the current user". In LB3, we have
   `loopback-context` based on continuation-local-storage and `options`-based
   context propagation.

3. How to migrate a component exporting mixins. See
   [Migrating model mixins ](https://loopback.io/doc/en/lb4/migration-models-mixins.html)
   for migration guide for app developers (mixin consumers).

### Models, Entities and Repositories

4. Contribute custom entities (Application, Installation) to be persisted via
   CRUD, exposed via REST and possibly further customized by the app.
   Customization options include which datasource to use, the base path where
   REST API is exposed (e.g. `/api/apps` and `/api/devices`), additional fields
   (e.g. `Application.tenantId`) and changes in persistence behavior (e.g. via
   Operation Hooks).

5. Contribute custom models (Notification) describing shape of data expected by
   the services (Push service). These models are not backed by any datasource,
   they are primarily used to describe data fields.

6. Add a custom Operation Hook to given models, with a config option to
   enable/disable this feature. The list of models can be provided explicitly in
   the component configuration or obtained dynamically via introspection (e.g.
   all models having a "belongsTo" relation with the Group model)

7. Add a custom Operation Hook to all app models matching given criteria (have a
   "belongsTo" relation with the Group model, are listed in the component
   config), plus a config option to enable/disable this feature.

8. Add new relations, e.g. between an app-provided entity `User` and a
   component-provided entity `File`. In this variant, the relation is added on
   small fixed number of models.

9. A model mixing adding new relations (`hasMany ModelEvents`), installing
   Operation Hooks (to generate model events/audit log entries), adding new
   Repository APIs (for working with related model events).

   _(The mixin-based design may be difficult to accomplish in LB4, we may want
   to use introspection and a model setting instead. The trickier part is how to
   apply changes to models added after the component was mounted.)_

10. For all models with a flag enabled in model settings, setup a custom
    `afterRemote` hook to modify the HTTP response (e.g. add additional
    headers).

### REST API

11. Add a new REST API endpoint to every (public) model/entity provided by the
    app, e.g. `GET /api/{model}/schema` returning JSON schema of the model.

12. Add a new REST API endpoint at a configured path, e.g. `/visualize`
    returning an HTML page.

13. REST API endpoints providing file upload & download.

14. Add a new local service (e.g. `Ping.ping()`) and expose it via REST API
    (e.g. `/ping`), allow the user to customize the base path where the API is
    exposed at (e.g. `/pong`).

15. Intercept REST requests handled by an endpoint contributed by a 3rd-party
    controller to invoke additional logic before the controller method is called
    (e.g. automatically create containers when uploading files) and after the
    controller method returned (e.g. post-process the uploaded file).

16. Add new REST API endpoints using Express `(req, res, next)` style.

### Services (local and remote)

17. Add a new local service (a class providing JS/TS API), e.g. Push service or
    (database) Migration service.

    - The service must be configurable by the target app (e.g. Apple & Google
      credentials for push notifications, transport config for logger
      frameworks, Amazon S3 credentials for storage component, etc.).

      _LB3 typically uses component configuration. I think in LB4, we can use
      `@configure` specific to each service (see the logger extension for an
      example)._

18. Inject LB3 models (LB4 entities & repositories) to the service. These models
    can be provided either by the extension or by the target application. When
    using models from the target application, the developer needs an option to
    specify which models (entities) and associated repositories to use.

    _Note: In LB4, models/entities are not injectable now, we used them
    primarily for the type information (to describe the shape of model data). As
    part of the follow-up research, we will need to decide if it's enough to
    inject LB4 Repository to a component or if we need to make model/entity
    classes injectable too. Also note that `DefaultCrudReposiory` class provides
    a public instance property `entityClass` referencing the model/entity it's
    bound to._

19. Provide a service factory to create services dynamically at runtime, e.g.
    named child loggers created via `app.log('name')` in LB3.

20. Provide a custom service connector requiring (environment-specific)
    configuration and providing a static service API (static means the API is
    always the same, as opposed to API generated dynamically from WSDL or
    OpenAPI).

### API transports

21. A new server type to handle requests coming from a non-REST source, e.g.
    Primus/WebSockets and RabbitMQ.

22. WebSockets: mount WebSocket-handler on the underlying http server used by
    `@loopback/rest`.

23. Message queues: allow consumers to also produce (publish) new messages, e.g.
    as a response to the incoming message.

### Authentication & authorization

24. Add a custom step to the default LB3 authentication/authorization workflow,
    so that we can compute additional information about the current user (e.g.
    groups the user belongs to).

    _Note: I think this step could be replaced by lazy initialization, where the
    current user groups are computed on the first access and cached for
    subsequent use._

25. A new role resolver (e.g. using groups), the app developer needs a way how
    to configure which models/repositories to use for role resolution.

### Introspection

26. Get a list of all models (entity classes) used by the application

27. Get definition of a given model (information about properties, relations,
    mixins, etc.)

28. Get a list of all remote methods (REST API endpoints) provided by a given
    model.

29. Obtain a list of all REST endpoints provided by the application, including
    metadata about request parameters & response schema, and any other
    information necessary to configure reverse-proxy routing rules (Kong, nginx,
    etc.).

## Migration

### Component layout & instructions for mounting to an app

TODOs for the migration guide based on @raymondfeng comments:

- Describe what a LB3 component can contribute to the application when it's
  mounted.
- Maybe a diagram would help if it shows the handshake between an application
  and a component as well as typical artifacts exported from a component.
- To some extent, a LB3 application asks its components to extend/patch the app.
  In contrast, a LB4 application imports bindings (representing the component's
  contribution to the app) from its components.

#### LB3 component layout & mounting

A LB3 component is implemented as a function accepting the target app object and
the configuration options.

```ts
module.exports = function initializeComponent(loopbackApplication, options) {
  // impl
};
```

A component is typically added to a LB3 application by creating a new entry in
`server/component-config.json`, see
[LoopBack components](https://loopback.io/doc/en/lb3/LoopBack-components.html).

For example:

```json
{
  "loopback-explorer": {
    "mountPath": "/explorer"
  }
}
```

This allows the component to receive configuration. App developers can provide
environment-specific configuration by using `component-config.{env}.json` files.

#### Migration to LB4

- How to add a component to a LB4 application?
- How to configure the component?
- How can a component receive the configuration object and the target
  application?

See the existing solutions:

- https://github.com/strongloop/loopback-next/tree/master/extensions/logging#basic-use
- https://github.com/strongloop/loopback-next/tree/master/extensions/logging#configure-the-logging-component
- https://github.com/strongloop/loopback-next/tree/master/extensions/apiconnect#usage
- https://github.com/strongloop/loopback-next/tree/master/extensions/health#basic-use
- https://github.com/strongloop/loopback-next/tree/master/extensions/metrics#basic-use

Here is the current convention for adding a component to a LB4 app & configuring
it:

Start by importing the component class:

```ts
import {MetricsComponent} from '@loopback/extension-metrics';
```

In the constructor, add the component to your application:

```ts
this.component(MetricsComponent);
```

Configure the component by adding the following code to the app:

```ts
this.configure(MetricsBindings.COMPONENT).to({
  // the configuration
});
```

Obtaining the target app & the custom configuration inside the component:

```ts
import {Application, Component, CoreBindings} from '@loopback/core';
import {bind, config, ContextTags, inject} from '@loopback/context';
import {MetricsBindings} from './keys';
import {DEFAULT_METRICS_OPTIONS, MetricsOptions} from './types';

@bind({tags: {[ContextTags.KEY]: MetricsBindings.COMPONENT}})
export class MetricsComponent implements Component {
  constructor(
    @inject(CoreBindings.APPLICATION_INSTANCE)
    private application: Application,
    @config()
    options: MetricsOptions = DEFAULT_METRICS_OPTIONS,
  ) {
    // ...
  }
  // ...
}
```

### Migrate context

> A context shared by all parts of the application, allowing different layers to
> store and retrieve values like "the current user". In LB3, we have
> `loopback-context` based on continuation-local-storage and `options`-based
> context propagation.

In LoopBack 4, extensions should use `@inject` decorators to access contextual
information. For example:

- `@inject(key)` and `@inject.getter(key)` to receive values from the context
- `@inject.setter(key)` to obtain a setter function for writing values to the
  context

Components can keep using the old `options`-based approach where it makes more
sense than Dependency Injection and especially when working with existing
`options`-based code like Repository APIs.

### Migrate mixins

1. Follow the steps described in
   [Migrating model mixins ](https://loopback.io/doc/en/lb4/migration-models-mixins.html)
   to convert your mixin implementation to LB4 style.

2. Modify your component to export ES6 class mixins - add `export` statements to
   components `src/index.ts` file.

3. Update your documentation to show how to apply the mixins in LoopBack 4
   applications,
   see[Migrating model mixins ](https://loopback.io/doc/en/lb4/migration-models-mixins.html)
   for more details.

### Migrate Models, Entities and Repositories

> Contribute custom models (Notification) describing shape of data expected by
> the services (Push service). These models are not backed by any datasource,
> they are primarily used to describe data fields.

1. Create a small LB3 app using your component.

2. In your LB4 extension project, run `lb4 import-lb3-models` to import model(s)
   contributed by the component from the LB3 app. Change the base class of the
   imported model from `Entity` to `Model` if needed.

3. Ensure that your main index file exports all models:

   ```ts
   // src/index.ts
   export * from './models';
   ```

4. Update your documentation to instruct users to import the models directly
   instead of relying on loopback-boot to pick them up.

   ```ts
   import {MyModel} from 'my-extension';
   ```

> Contribute custom entities (Application, Installation) to be persisted via
> CRUD, exposed via REST and possibly further customized by the app.
> Customization options include which datasource to use, the base path where
> REST API is exposed (e.g. `/api/apps` and `/api/devices`), additional fields
> (e.g. `Application.tenantId`) and changes in persistence behavior (e.g. via
> Operation Hooks)

TBD

> Add a custom Operation Hook to given models, with a config option to
> enable/disable this feature. The list of models can be provided explicitly in
> the component configuration or obtained dynamically via introspection (e.g.
> all models having a "belongsTo" relation with the Group model)

TBD

> Add a custom Operation Hook to all app models matching given criteria (have a
> "belongsTo" relation with the Group model, are listed in the component
> config), plus a config option to enable/disable this feature.

TBD

> Add new relations, e.g. between an app-provided entity `User` and a
> component-provided entity `File`. In this variant, the relation is added on
> small fixed number of models.

TBD

> A model mixing adding new relations (`hasMany ModelEvents`), installing
> Operation Hooks (to generate model events/audit log entries), adding new
> Repository APIs (for working with related model events).
>
> _(The mixin-based design may be difficult to accomplish in LB4, we may want to
> use introspection and a model setting instead. The trickier part is how to
> apply changes to models added after the component was mounted.)_

TBD

> For all models with a flag enabled in model settings, setup a custom
> `afterRemote` hook to modify the HTTP response (e.g. add additional headers).

### Migrate REST API

In general, we are missing documentation explaining extension authors:

1. How can their extension contribute controllers to the targe app.
2. How to make the base path configurable by the target application.
3. How to add/remove controller methods based on user-provided configuration.

At the moment, we are using the following workaround in
[packages/rest-explorer/src/rest-explorer.component.ts](../../packages/rest-explorer/src/rest-explorer.component.ts):

- In extension constructor, register all controller routes at the given base
  path.

This is not great, but solves the following two requirements:

- Customize the base path of controller endpoints
- Exclude certain controller endpoints based on configuration

We should look into ways how to make this registration step simpler. Few options
that come to my mind:

1. At the moment, components are exporting an array of controller classes:

   ```ts
   export interface Component {
     /**
      * An array of controller classes
      */
     controllers?: ControllerClass[];
     // ...
   }
   ```

   Can we enhance this contract to allow components to specify additional
   metadata like the base path where to mount the controller and a list of
   endpoints which to expose?

2. Create the controller class dynamically using the approach pioneered by
   [`defineCrudRestController()`](../../packages/rest-crud/src/crud-rest.controller.ts).
   This makes it easy to tweak endpoint paths, I am not sure how easy it is to
   add/remove controller methods based on config.

We need to create follow-up tasks to:

- decide what should be the recommended solution
- implement the necessary runtime support
- update existing code (e.g. `packages/rest-explorer`) to follow the new style
- write documentation for extension authors - add a new page to
  [Extending LoopBack 4](https://loopback.io/doc/en/lb4/Extending-LoopBack-4.html).

We can approach this incrementally:

1. Start with documenting the basic setup where the REST API is not configurable
   (just add the controller class to `this.controllers` array).

2. Find & document a solution for configurable REST APIs.

**The following content is based on the assumption that such documentation
already exists.**

> Add a new REST API endpoint to every (public) model/entity provided by the
> app, e.g. `GET /api/{model}/schema` returning JSON schema of the model.

First, we need to find all publicly exposed models. This is tricky in LB4,
because there is no direct mapping between models and REST API endpoints. REST
APIs are implemented by Controllers, there may be more than one Controller for
each Model (e.g. `ProductController` and `ProductCategoryController`).

I am arguing that the particular use case of accessing JSON schema of a model is
not relevant in LB4, because model schema can be obtained from the OpenAPI spec
document provided by all LB4 applications out of the box.

Let's not provide any migration guide for this technique for now. We can wait
until there is user demand and then build a better understanding of user
requirements before looking for LB4 solution.

> Add a new REST API endpoint at a configured path, e.g. `/visualize` returning
> an HTML page.

We have a working example at
[packages/rest-explorer/src/rest-explorer.component.ts](../../packages/rest-explorer/src/rest-explorer.component.ts).

1. Create a new Controller class

2. Move REST API endpoints to this controller class

3. In the controller constructor, inject the HTTP response.

4. In controller methods, use Express API like `res.contentType()` and
   `res.send()` to send back the result. (This is a workaround until
   https://github.com/strongloop/loopback-next/issues/436 is implemented).

5. Decide if you want these endpoints included in OpenAPI spec. Use the
   following operation spec to hide them:

   ```ts
   {
     'x-visibility': 'undocumented',
     responses: {},
   }
   ```

6. Follow the guide-to-be-written in "Extending LoopBack 4" to let the extension
   contribute this new controller to the target app in a configurable way.

> REST API endpoints providing file upload & download.

Rewrite your LB3 remote methods to LB4 controllers, see
[Upload and download files ](https://loopback.io/doc/en/lb4/File-upload-download.html)
for more details.

Follow the guide-to-be-written in "Extending LoopBack 4" to let the extension
contribute this new controller to the target app in a configurable way.

> Add a new local service (e.g. `Ping.ping()`) and expose it via REST API (e.g.
> `/ping`), allow the user to customize the base path where the API is exposed
> at (e.g. `/pong`).

1. Use `lb4 service` to create the new service class.
2. Move the implementation from LB3 model methods to LB4 service methods.
3. TBD: how can an extension contribute service classes? (see below)
4. Create a new Controller class to expose service methods via REST API.
5. Follow the guide-to-be-written in "Extending LoopBack 4" to let the extension
   contribute this new controller to the target app in a configurable way.

We are missing documentation for extension authors showing how to contribute
services to application context.

Here is what I found in the existing code:

- [ServiceBooter](../../packages/boot/src/booters/service.booter.ts) requires
  service classes to be decorated with `@bind()`, which is true for service
  classes scaffolded via `lb4 services`.
- The booter calls `app.service()` API under the hood.
- `app.service()` uses a (public?) helper `createServiceBinding()` that handles
  the special case where the service class is a provider, otherwise delegates
  the work to `createBindingFromClass`.
- Once we have a binding, we can add it to extension's `bindings` property which
  contains a list o bindings to be added to the target application.

Based on the above, I think we should recommend the following solution:

```ts
import {createServiceBinding} from '@loopback/core';
import {MyService} from './services/my.service.ts';
import {GeocodeServiceProvider} from './services/geocoder.service.ts';

export class SampleComponent implements Component {
  bindings = [
    createServiceBinding(MyService),
    createServiceBinding(GeocoderServiceProvider),
  ];
}
```

Additional tasks to create:

- Write documentation for extension authors on contributing services - add a new
  page to
  [Extending LoopBack 4](https://loopback.io/doc/en/lb4/Extending-LoopBack-4.html).

> Intercept REST requests handled by an endpoint contributed by a 3rd-party
> controller to invoke additional logic before the controller method is called
> (e.g. automatically create containers when uploading files) and after the
> controller method returned (e.g. post-process the uploaded file).

TBD

> Add new REST API endpoints using Express `(req, res, next)` style.

1. Rework your extension to create a new `express.Router` instance and define
   your REST API endpoints on that router instance.

2. In extension constructor, use `app.mountExpressRouter` to add your routes to
   the target LB4 application. Optionally you can also provide OpenAPI spec to
   describe the new endpoints contributed by your extension. Learn more in
   [Mounting an Express router](https://loopback.io/doc/en/lb4/Routes.html#mounting-an-express-router).

As a better alternative, we recommend extension authors to use
[Controllers](https://loopback.io/doc/en/lb4/Controllers.html) instead of
low-level Express handler API.

TODO: Add documentation for extension authors to explain how an extension can
contribute new controllers to the target app.

### Migrate API transports

TBD

### Authentication & authorization

TBD

### Migrate Introspection

TBD

## Overview of existing LB3 components

### Push notifications

- LB3 docs: https://loopback.io/doc/en/lb3/Push-notifications.html
- GitHub repo: https://github.com/strongloop/loopback-component-push

This is not a typical LoopBack component because it's not configured via
`component-config.json`. Instead, the component provides a connector to be
configured via `server/datasources.json` and `server/config.json`, plus a set of
(base) models that are expected to be added to the target application.

Models:

- `application` - Holds app-specific push settings like Apple and Google
  credentials, this model is typically persisted in the "main" database. A
  single LoopBack project (application) can host multiple push-notification
  client application instances.

- `installation` - Represents a device registered for receiving push
  notifications, it's typically persisted in the "main" database. Client
  applications (iOS, Android) are consuming REST APIs of this model.

- `notification` - describes the shape (data fields) of a single notification to
  be sent. This is essentially a plain `Model` that does not need to be attached
  to any datasource.

- `push` - a service-like model to hold methods for sending notifications, e.g.
  `notifyById`, it's attached to the datasource using the push component as the
  connector.

#### Techniques

1. Contribute custom entities (Application, Installation) to be persisted via
   CRUD, exposed via REST and possibly further customized by the app.
   Customization options include base path where REST API is exposed (e.g.
   `/api/apps` and `/api/devices`), additional fields (e.g.
   `Application.tenantId`) and changes in persistence behavior (e.g. via
   Operation Hooks).

2. Contribute custom behavior (Push service) requiring user-provided
   configuration (Apple & Google credentials, dependant models/repositories to
   use).

3. Contribute custom model (Notification) to describe shape of data expected by
   the services (Push service).

### Storage component

- LB3 docs: https://loopback.io/doc/en/lb3/Storage-component.html
- GitHub repo: https://github.com/strongloop/loopback-component-storage

Again, this is not a typical LB3 component.

- The component itself behaves like a connector and is configured via
  `server/datasources.json`.
- It provides models to be added to the target application.

Models:

- `Container` is a service-like model that should be attached to the storage
  datasource and provides JS & REST APIs for working with both containers and
  files.

The concept of a Container (groups files, similar to a directory or folder) and
a File (stores the data, such as a document or image) is implicit, there are no
LB3 models to formally describe these entities.

#### Techniques

1. Contribute a new service, it requires user-provided configuration (Amazon
   S3/IBM COS credentials).

2. Contribute a REST API for this new service, allow the user to customize the
   base path where the API is exposed at (e.g. `/api/containers`).

3. REST API implements endpoints for file upload & download.

### Logging components

_NOTE: We already have a Winston-based logging extension, see
[@loopback/logging](./extensions/logging/README.md)._

https://www.npmjs.com/package/loopback-component-bunyan

> Creates a Bunyan logger based on configuration in component-config.json file.

This component contributes:

- `app.logger()` factory function for creating named loggers
- custom transports (bunyan streams)
- custom request-logger middleware

It uses the configuration provided in `server/component-config.json` to
configure the underlying Bunyan logging library (most notably the transports).

```ts
const log = app.logger('MyComponent');
log.debug({}, 'My log message');
```

https://www.npmjs.com/package/loopback-component-winston

> Creates winston logger based on configuration in component-config.json file.

This component contributes:

- `app.log()` function for producing logs, the method name can be customized.
- custom request-logger middleware

It uses the configuration provided in `server/component-config.json` to
configure the underlying Winston logging library (most notably the transports).

#### Techniques

1. Configure log transports.
2. Provide getter/factory functions for accessing (child) loggers, e.g. a global
   app-level logger, a child request-level logger.

### Access control

https://www.npmjs.com/package/loopback-component-access-groups

> This loopback component enables you to add multi-tenant style access controls
> to a loopback application. It enables you to restrict access to model data
> based on a user's roles within a specific context.

This component is rather complex.

1. The component is heavily depending on loopback-context, which uses
   continuation-local-storage (CLS) to provide per-request context storage. Note
   that CLS is not reliable, it can loose context e.g. when connection pools are
   involved.

2. It defines a custom strong-remoting phase executed after regular
   authorization but before method invocation, this phase adds extra properties
   to strong-remoting context.

   See
   [setupRemotingPhases](https://github.com/fullcube/loopback-component-access-groups/blob/75ad90190898305839b7cd76ef138dd875b1cc20/lib/utils.js#L45-L56)

3. Applications are expected to provide two additional models `Group` and
   `GroupAccess`, names of these models can be customized. Application models
   (e.g. `Product` and `Category`) are expected to have a `belongsTo Group`
   relation set up to enable group-based access control.

4. The component installs a new role resolver that uses group membership to
   resolve user roles.

5. It also provides some middleware:

   - `userContextMiddleware` to set the current user & their groups in the
     CLS-based context
   - `accessLoggerMiddleware` to log HTTP requests with additional information.

6. Optionally, it installs an `access` Operation Hook to all models that have a
   `belongsTo` relation to `Group` model, this hook modifies `filter` and
   `where` queries to limit the results only to model instances allowed by
   access control settings.

There is a partial example app showing the component in practice, see
https://github.com/fullcube/loopback-example-advanced-access-control

#### Techniques

1. A context shared by all parts of the application, allowing different layers
   to store and retrieve values like "the current user".

2. A custom step added to the default authentication/authorization sequence, so
   that we can compute additional information about the current user (e.g.
   groups the user belongs to).

   _Note: I think this step could be replaced by lazy initialization, where the
   current user groups are computed on the first access and cached for
   subsequent use._

3. A new role resolver, the app developer needs a way how to provide custom
   models/repositories to use for role resolution.

4. A custom Operation Hook applied to all app models matching given criteria
   (have a "belongsTo" relation with the Group model), plus a config option to
   enable/disable this feature.

### Connectors

https://www.npmjs.com/package/loopback-component-braintree

> The [Braintree](https://www.braintreepayments.com) connector for the LoopBack
> framework

AFAICT, this is a typically Service Proxy connector, similar to our SOAP/REST
connectors.

#### Techniques

- A service connector requiring (environment-specific) configuration and
  providing a static service API.

### Custom transports

https://www.npmjs.com/package/@oneflow/loopback-component-primus

> Primus adapter for loopback. It allows you to call loopback's remote methods
> via websocket.

https://www.npmjs.com/package/loopback-component-rabbitmq

> This component provides a convenient way to work with RabbitMQ within a
> loopback application. This includes:
>
> - Defining a RabbitMQ topology using the component-config.json
> - Registering message producers and consumers handlers using a mixin.
> - Inspecting RabbitMQ stats and queue statuses using a RabbitMQ loopback
>   model.
>
> In addition, an optional mixin is provided that provides an easy way to attach
> message producer and consumer helper methods directly to your loopback models.

#### Techniques

1. A new server type to handle requests coming from a non-REST source, e.g.
   Primus/WebSockets and RabbitMQ.
2. WebSockets: mount WebSocket-handler on the underlying http server used by
   `@loopback/rest`.
3. Message queues: allow consumers to also produce (publish) new messages, e.g.
   as a response to the incoming message.

### Inspecting models

https://www.npmjs.com/package/loopback-component-meta

> Component for LoopBack that adds a Meta model that can be used to retrieve
> meta data about the model definitions.

https://www.npmjs.com/package/loopback-jsonschema-generator

> Generates JSON schemas for your LoopBack models Access the generated JSON
> schema url http://yourapi.com/api/{model-plural-name}/json-schema

https://www.npmjs.com/package/loopback-component-visualizer

> Visualizing a model is sometimes a difficult task. When the data model gets
> larger, it becomes even more difficult to understand how models relate to each
> other.
>
> loopback-component-visualizer helps you in creating a model diagram with a
> representation of all the properties, methods and relationships of your models
> for your loopback application.

#### Techniques

1. Get a list of all models (entity classes) used by the application
2. Get definition of a given model (information about properties, relations,
   mixins, etc.)
3. Add a new REST API endpoint to every (public) model/entity provided by the
   app
4. Add a new REST API endpoint at a configured path (e.g. `/visualize`)
5. Get a list of all remote methods (REST API endpoints) provided by a model

### Unsorted

https://www.npmjs.com/package/loopback-component-migrate

> A library to add simple database migration support to loopback projects.
>
> Migrations that have been run will be stored in a table called 'Migrations'.
> The library will read the loopback datasources.json files based on the
> NODE_ENV environment variable just like loopback does. The usage is based on
> the node-db-migrate project.

https://www.npmjs.com/package/loopback-component-ping

> Component for LoopBack that adds a model for retrieving the internal state of
> the Node process.
>
> It is a wrapper around the express-ping package.

https://www.npmjs.com/package/component-storage

> This loopback storage component is an extension to loopback-component-storage,
> it uses 3rd party image processing library Sharp for recreating images.
>
> Usage:
> https://github.com/pbalan/component-storage/blob/master/test/fixtures/simple-app/server/boot/component-storage.js

https://www.npmjs.com/package/loopback-component-changestreamer

> The component observes a number specified models and notifies about the
> changes by SSE.
>
> The main difference with Loopback /change-stream channels is that this
> implementation creates only two observers (after save and after delete) per
> model and then streams the changes to keep-alive registered connections. In
> contrast Loopback creates two same observers for each connection.

https://www.npmjs.com/package/@aliatech/loopback-component-traceability

> Module for Loopback Framework that allows to keep a persisted traceability of
> custom operations over models

https://www.npmjs.com/package/@joinbox/loopback-component-remote-microservice

> Loopback component to expose and consume models of remote microservices.

_This component is too complex to understand, I am leaving it out of the initial
version of the migration guide._

https://www.npmjs.com/package/loopback-component-kong-sync

> This is a Loopback JS component to synchronize routes to Kong API Gateway.
>
> This component map all Models and Methods from Loopback standardizing it and
> creating the related routes in Kong.

https://www.npmjs.com/package/@joinbox/loopback-component-custom-headers

> Sets cache headers according to the model configuration

#### Techniques

1. Add entities with CRUD persistence to the app, allow the app developer to
   decide which datasource to use.
2. Add new JS/TS API (a local service) utilizing the models provided by the
   extension, e.g. `Migration.migrateTo`.
3. Add a new local service (e.g. `Ping.ping()`) and expose it via REST API (e.g.
   `/ping`).
4. Add new relations, e.g. between an app-provided entity `User` and a
   component-provided entity `File`.
5. Intercept REST requests handled by an endpoint contributed by a 3rd-party
   controller to invoke additional logic before the controller method is called
   (e.g. automatically create containers when uploading files) and after the
   controller method returned (e.g. post-process the uploaded file).
6. Add new REST API endpoints using Express `(req, res, next)` style.
7. Set up custom Operation Hooks for the given app models/entities
8. A model mixing adding new relations (`hasMany ModelEvents`), installing
   Operation Hooks (to generate model events/audit log entries), adding new
   Repository APIs (for working with related model events).
9. Obtain a list of all REST endpoints, including metadata about request
   parameters and response schema and any other information necessary to
   configure reverse-proxy routing rules (Kong, nginx, etc.).
10. For all models with a flag enabled in model settings, setup a custom
    `afterRemote` hook to modify the HTTP response (e.g. add additional
    headers).

### Mixins

Migration of mixin implementation is already covered by
[Migrating model mixins ](https://loopback.io/doc/en/lb4/migration-models-mixins.html).

However, we are missing instructions for migrating a component exporting mixins
and apps consuming mixins from 3rd party components.

Let's use the most popular mixin
[loopback-ds-timestamp-mixin](https://www.npmjs.com/package/loopback-ds-timestamp-mixin)
as an example component to drive the migration docs.
