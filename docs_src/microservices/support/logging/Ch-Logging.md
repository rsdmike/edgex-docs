# Logging
# (Deprecated in Geneva Release)

![image](EdgeX_SupportingServicesLogging.png)

---
**Deprecation Notice**

Please note that the logging service has been deprecated with the Geneva release (v1.2).  The EdgeX community feels that there are better log aggregation services available in the open source community or by deployment/orchestration tools.

Starting with the Geneva release, logging service will no longer be started as part of the reference implementations provided through the EdgeX Docker Compose files (the service is still available but commented out in those files).

By default, all services now log to standard out (EnableRemote is set to false and File is set to '').  If users wish to still use the central logging service, they must configure each service to use it (set EnableRemote=true).  Users can still alternately choose to have the services log to a file with additional configuration changes (set File to the appropriate file location).

The Support Logging Service will removed in a future release of EdgeX Foundry.

---

## Introduction

Logging is critical for all modern software applications. Proper logging
provides the users with the following benefits:

-   Ability to monitor and understand what systems are doing
-   Ability to understand how services interact with each other
-   Problems are detected and fixed quickly
-   Monitoring to foster performance improvements

The graphic shows the high-level design architecture of EdgeX Foundry
including the Logging Service.

## Minimum Product Feature Set

1.  Provides a RESTful API for other microservices to request log
    entries with the following characteristics:

-   The RESTful calls should be non-blocking, meaning calling services
    should fire logging requests without waiting for any response from
    the log service to achieve minimal impact to the speed and
    performance to the services.
-   Support multiple logging levels, for example trace, debug, info,
    warn, error, fatal, and so forth.
-   Each log entry should be associated with its originating service.

2.  Provide RESTful APIs to query, clear, or prune log entries based on
    any combination of following parameters:

-   Timestamp from
-   Timestamp to
-   Log level
-   Originating service

3.  Log entries should be persisted in either file or database, and the
    persistence storage should be managed at configurable levels.
    Querying via the REST API is only supported for deployments using
    database storage.
4.  Take advantage of an existing logging framework internally and
    provide the "wrapper" for use by EdgeX Foundry
5.  Follow applicable standards for logging where possible and not
    onerous to use on the gateway

## High Level Design Architecture

![image](EdgeX_SupportingServicesLoggingArchitecture.png)

The above diagram shows the high-level architecture for EdgeX Foundry
Logging Service. Other microservices interact with EdgeX Foundry Logging
Service through RESTful APIs to submit their logging requests, query
historical logging, and remove historical logging. Internally, EdgeX
Foundry's Logging Service utilizes the [GoKit
logger](https://github.com/go-kit/kit/tree/master/log) as its internal
logging framework. Two configurable persistence options exist supported
by EdgeX Foundry Logging Service: file or MongoDB.

## Configuration Properties

The following configuration properties are specific to the support-logging service. Please refer to the general Configuration [documentation](https://docs.edgexfoundry.org/1.2/microservices/configuration/Ch-Configuration/#configuration) for configuration properties common across all services.
  
  |Configuration  |     Default Value     |             Dependencies|
  | --- | --- | -- |
| **Entries in the Writable section of the configuration can be changed on the fly while the service is running if the service is running with the `–configProvider/ -cp=<url>` flag** |
  |Writable Persistence  |database  |"file" to save logging in file; "database" to save logging in MongoDB
  | | | |
  

## Logging Service Client Library for Go

As the reference implementation of EdgeX Foundry microservices is
written in Go, we provide a Client Library for Go so that Go-based
microservices could directly switch their Loggers to use the EdgeX
Foundry Logging Service.

The Go LoggingClient is part of the [go-mod-core-contracts
module](https://github.com/edgexfoundry/go-mod-core-contracts). This
module can be imported into your project by including a reference in
your go.mod. You can either do this manually or by executing "go get
github.com/edgexfoundry/go-mod-core-contracts" from your project
directory will add a reference to the latest tagged version of the
module.

After that, simply import
"github.com/edgexfoundry/go-mod-core-contracts/clients/logger" into a
given package where your functionality will be implemented. Declare a
variable or type member as logger.LoggingClient and it's ready for use.

    package main

    import "github.com/edgexfoundry/go-mod-core-contracts/clients/logger"

    func main() {
        client := logger.LoggingClient

        //LoggingClient is now ready for use. A method is exposed for each LogLevel
        client.Trace("some info")
        client.Debug("some info")
        client.Info("some info")
        client.Warn("some info")
        client.Error("some info")
    }

Log statements will only be written to the log if they match or exceed
the minimum LogLevel set in the configuration (described above). This
setting can be changed on the fly without restarting the service to help
with real-time troubleshooting.

Log statements are currently output in a simple key/value format. For
example:

    level=INFO ts=2019-05-16T22:23:44.424176Z app=edgex-support-notifications source=cleanup.go:32 msg="Cleaning up of notifications and transmissions"

Everything up to the "msg" key is handled by the logging
infrastructure. You get the log level, timestamp, service name and the
location in the source code of the logging statement for free with every
method invocation on the LoggingClient. The "msg" key's value is the
first parameter passed to one of the Logging Client methods shown above.
So to extend the usage example a bit, the above calls would result in
something like:

    level=INFO ts=2019-05-16T22:23:44.424176Z app=logging-demo source=main.go:11 msg="some info"

You can add as many custom key/value pairs as you like by simply adding
them to the method call:

    client.Info("some info","key1","abc","key2","def")

This would result in:

    level=INFO ts=2019-05-16T22:23:44.424176Z app=logging-demo source=main.go:11 msg="some info" key1=abc key2=def

Quotes are only put around values that contain spaces.

## EdgeX Logging Keys

Within the EdgeX Go reference implementation, log entries are currently
written as a set of key/value pairs. We may change this later to be more
of a struct type than can be formatted according to the user's
requirements (JSON, XML, system, etc). In that case, the targeted struct
should contain properties that support the keys utilized by the system
and described below.

  
  |Key                   |Intent|
  |------------------------- |---------------------------------------------|
  |level                     |Indicates the log level of the individual log entry (INFO, DEBUG, ERROR, etc)|
  |ts                        |The timestamp of the log entry, recorded in UTC|
  |app                       |This should contain the service key of the service writing the log entry|
  |source                    |The file and line number where the log entry was written|
  |msg                       |A field for custom information accompanying the log entry. You do not need to specify this explicitly as it is the first parameter when calling one of the LoggingClient's functions.|
  |correlation-id            |Records the correlation-id header value that is scoped to a given request. It has two sub-ordinate, associated fields (see below).|
  |correlation-id path       |This field records the API route being requested and is utilized when the service begins handling a request. \* Example: path=/api/v1/event When beginning the request handling, by convention set "msg" to "Begin request".|
  |correlation-id duration   |This field records the amount of time taken to handle a given request. When completing the request handling, by convention set "msg" to "Response complete".|
  

Additional keys can be added as need warrants. This document should be
kept updated to reflect their inclusion and purpose.
