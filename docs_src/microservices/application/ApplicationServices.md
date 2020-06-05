# Application Services

![image](ApplicationServices.png)

Application Services are a means to get data from EdgeX Foundry to
external systems and process (be it analytics package, enterprise or
on-prem application, cloud systems like Azure IoT, AWS IoT, or Google
IoT Core, etc.). Application Services provide the means for data to be
prepared (transformed, enriched, filtered, etc.) and groomed (formatted,
compressed, encrypted, etc.) before being sent to an endpoint of choice.
Endpoints supported out of the box today include HTTP and MQTT
endpoints, but will include additional offerings in the future and could
include a custom endpoints.

!!! note
    Application Services has replaced Export Services

Application Services are based on the idea of a "Functions Pipeline".
A functions pipeline is a collection of functions that process messages
(in this case EdgeX event/reading messages) in the order that you've
specified. The first function in a pipeline is a trigger. A trigger
begins the functions pipeline execution. A trigger is something like a
message landing in a watched message queue.

```mermaid
graph LR;
    A(EdgeX Core Data)-. message bus .-> B
    subgraph <br/>Application Service
    B(Trigger<br/>Function) --> C;
    C(Function A) --> D(Function B);
    D --> E(...)
    end
```

An Applications Functions Software Development Kit (or App Functions
SDK) is available to help create Application Services. Currently the
only SDK supported language is Golang, with the intention that community
developed and supported SDKs will come in the future for other
languages. It is currently available as a Golang module to remain
operating system (OS) agnostic and to comply with the latest EdgeX
guidelines on dependency management.

Any application built on top of the Application Functions SDK is considered an App Service. This SDK is provided to help build Application Services by assembling triggers, pre-existing functions and custom functions of your making into a pipeline.

## Application Service Improvements

Providing an SDK that connects directly to a message bus by which Core
Data events are published eliminates performance issues as well as allow
the developers extra control on what happens with that data as soon as
it is available. Furthermore, it emphasizes configuration over
registration for consuming the data. The application services can be
customized to a client's needs and thereby also removing the need for
client registration.

## Standard Functions

As mentioned, an Application Service is a function pipeline. The SDK
provides some standard functions that can be used in a functions
pipeline. In the future, additional functions will be provided
"standard" or in other words provided with the SDK. Additionally,
developers can implement their own custom functions and add those to their
Application Service functions pipeline.

```mermaid
graph LR;
    subgraph <br/>Application Service
    B(Trigger<br/>Function) --> C(Device Name Filter Function)
    C(Device Name<br/>Filter Function) --> D(Value Descriptor Filter Function)
    D(Value Descriptor<br/>Filter Function) --> E(XML Transform<br/>Function)
    OR(OR)
    D(Value Descriptor<br/>Filter Function) --> F(JSON Transform<br/>Function)
    end
    style OR fill:none,stroke:none
```

One of the most common use cases for working with data that comes from
Core Data is to filter data down to what is relevant for a given
application and to format it. To help facilitate this, four primary
functions ported over from the existing services today are included in
the SDK. The first is the `DeviceNameFilter` function which
will remove events that do not match the specified IDs and will cease
execution of the pipeline if no event matches. The second is the
`ValueDescriptorFilter` which exhibits the same behavior as
`DeviceNameFilter` except filtering event readings on Value Descriptor
instead of DeviceID. The third and fourth provided functions in the SDK
transform the data received to either XML or JSON by calling
`XMLTransform` or `JSONTransform`.

Typically, after filtering and transforming the data as needed,
exporting is the last step in a pipeline to ship the data where it needs
to go. There are two primary functions included in the SDK to help
facilitate this. The first is the`HTTPPost` function that will POST the provided data 
to a specified endpoint, and the second is an `MQTTSecretSend()` function that will publish
the provided data to an MQTT Broker as specified in the configuration.

There are two primary triggers that have been included in the SDK that
initiate the start of the function pipeline. First is via a POST HTTP
Endpoint `/api/v1/trigger` with the EdgeX event data as the body.
Second is the MessageBus subscription with connection details as
specified in the configuration.

Finally, data may be sent back to the message bus or HTTP response by
calling `.complete()` on the context. If the trigger is HTTP, then it will
be an HTTP Response. If the trigger is MessageBus, then it will be
published to the configured host and topic.

### Unsupported existing export service functions

**Valid Event Check**--The first component in the pipe and filter,
before the copier (described in the previous section) is a filter that
can be optionally turned on or off by configuration. This filter is a
general purpose data checking filter which assesses the device- or
sensor-provided Event, with associated Readings, and ensures the data
conforms to the ValueDescriptor associated with the Readings.

- For example, if the data from a sensor is described by its metadata profile as adhering to a "Temperature" value descriptor of floating number type, with the value between -100° F and 200° F, but the data seen in the Event and Readings is not a floating point number, for example if the data in the reading is a word such as "cold," instead of a number, then the Event is rejected (no client receives the data) and no further processing is accomplished on the Event by the Export Distro service.
