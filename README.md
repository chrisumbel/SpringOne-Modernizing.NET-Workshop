Workshop Context 
----------------

In this exercise we’re going to perform a lightweight modernization on a fictitious full framework ASP.NET MVC application who's front-end consumes a WebAPI backend. The application is presumed to run on IIS today using standard virtual or physical infrastructure and our task is to migrate the application to Cloud Foundry. Our modernization efforts will involve building the necessary artifacts to get the application to run at all in the cloud and to allow the application to scale once it gets there.  

All the while we’ll try to use a lightweight approach and make as few changes to application code as possible.  

Current State 
----------------

Our sample application for this working is a single solution containing two projects. The first project is a WebAPI project exposing a single endpoint. The second project is an ASP.NET MVC front-end that consumes the WebAPI endpoint from JavaScript. 

The application is designed with an IIS topology in mind with both applications sharing a common hostname, the frontend responding to the root, and the backend living in a virtual directory located at the path “/api”. 

The frontend exposes an endpoint that tracks session-level view counts and is configured to use “InProc” session state. 

Desired State: 
----------------

Our goals are  

    Get this application operational on Cloud Foundry  

    Ensure the application is scalable  

    Use solutions that are minimally invasive to the code base 

Exercise 1: Push the application and try to scale (WIP @Chris)
----------------
 
Create a manifest for the backend in the root of its directory.

```
---
applications:
- stack: windows
  instances: 1
  buildpacks: 
    - hwc_buildpack
  routes:
    - route: <host-name.domain-name>/api
```    

Push the service to cloud foundry

```
cf push <api-app-name>
```

Create a manifest for the web frontend in the root of its directory.

```
---
applications:
- stack: windows
  instances: 1
  buildpacks: 
    - hwc_buildpack
```

Push the web app to cloud foundry

```
cf push <web-app-name>
```

See the app work

```
cf scale -i 3 <app-name> 
```

See the app fail

Buildpacks:  @Brian 
----------------
A `buildpack` is an artifact that configures or provides your app's hosting environment.  For example, the `HWC_BUILDPACK` provides the Hostable Web Core for IIS app support.

Buildpacks come in two flavors:
1. A `supply builpack` helps bootstrap your app's dependencies for startup.
1. A `final buildpack` is responsible for launching your app
    * The HWC_BUILDPACK is an example of a `final buildpack`.
    
Buildpacks provide a series of scripts for the platform to invoke during the buildpack detection and execution phases.  These scripts are referred to as `hooks`.  

#### Lifecycle

A `buildpack` is run by the platform prior to app instance startup if either of the following conditions are met:
1. The buildpack's `detect` hook returns true (implicit)
    * Drives the app detection that allows us to `cf push` certain app types without specifying a buildpack.
2. The buildpack is specified in the manifest (explicit)
    * Most common use-case in a prod environment

If a buildpack is detected or specified in the manifest, the buildpack's remaining hooks will be invoked:
1. `supply`:
    * Dependency injection, environment setup.
    * Must be implemented by each `supply buildpack`
2. `finalize`
    * Provide app startup concerns (webserver, etc.)
    * Must be implemented by `final buildpack`
3. `release`
    * Provide metadata to control app startup - out of scope for today.

#### Composability
Multi-buildpack allows us to compose an app startup pipeline where `supply buildpacks` are invoked in the order provided prior to the `final buildpack`, which starts the app.
* Analogous to Middleware concept in WebAPI

Multi-buildpack can be specified in yml:

```yml
---
applications:
- stack: windows
  instances: 1
  buildpacks:
    - first_supply_buildpack
    - second_supply_buildpack
    - final_buildpack
```

Or, via command line:

```CLI
cf push APP-NAME -b FIRST-BUILDPACK -b SECOND-BUILDPACK -b FINAL-BUILDPACK
```

#### Redis Buildpack
If you've ever pushed an app to Cloud Foundry that started successfully, you've already made use of a `final buildpack`.  In our sample web app, we've chosen to use the HWC_BUILDPACK as our `final buildpack` because the app we're pushing is a full framework .NET app.  And, while the HWC_BUILDPACK has solved our problem of getting our app to run on TAS, it's revealed a new problem regarding our app's design: its use of in-proc session.  Thankfully, there's a buildpack for that, and it's called the Redis for Session Buildpack.

The Redis for Session Buildpack does a very simple trick: when its `supply` hook is invoked, the buildpack transform's your app's web.config and replaces use of in-proc session with use of a Redis backed session.  For the buildpack to be effective, we must do two things to our app:
1. Add the Redis for Session buildpack to our manifest as a supply buildpack
2. Add the RedisSessionStateProvider nuget package to your app.
3. Bind our app to a Redis service instance.

When the app is pushed with the buildpack, the Redis buildpack will lookup the details for connecting to Redis; this is only possible because we've bound our app to a Redis service instance.  If no Redis instances are found by the buildpack during startup, the web.config will not be modified.

Assuming Redis is bound to the app, the buildpack will then transform the app's web.config, replacing the in-proc session configuration with a complete and credentialed configuration for using Redis intsead.  For this to work, the buildpack must find the RedisSessionStateProvider in your app's bin directory; if the provider is not found, the web.config will not be modified.

If we've followed the happy path (installed the nuget and added a redis service to our manifest), the next time we push our app, horizontal scale wont' be a concern even with our reliance on session.

##### Some caveats
The .NET framework requires that objects stored in session state must be serializable; the one exception to this rule is if session is maintained in process.  If your app was not designed with externalization of session in mind, you may encounter exceptions for any objects stored in session that cannot be serialized.



Exercise 2 (WIP @Chris)
----------

Create the shared redis service to store session state 

```
cf create-service p-redis shared-vm session 
```

Bind the application to redis

```
---
applications:
- stack: windows
  instances: 3
  buildpacks:
    - hwc_buildpack
  services:
    - session
```

Add the redis session buildpack

```
---
applications:
- stack: windows
  instances: 3
  buildpacks:
    - https://github.com/cloudfoundry-community/redis-session-aspnet-buildpack/releases/download/v1.0.5/Pivotal.Redis.Aspnet.Session.Buildpack-win-x64-1.0.5.zip 
    - hwc_buildpack
  services:
    - session
```

Push the web app to cloud foundry

```
cf push <web-app-name>
```
