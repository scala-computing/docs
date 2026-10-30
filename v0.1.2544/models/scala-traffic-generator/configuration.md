---
title: "Configuration"
description: "This page lists the Scala Traffic Generator attributes that change a run, with their types and defaults, and shows how to set the application up."
---

This page lists the Scala Traffic Generator attributes that change a run, with their types and defaults, and shows how to set the application up.

The Scala Traffic Generator is an application in a simulation configuration. Its `typedParameters` hold two components: `client` and `server`. Its distribution ratios are set apart from `typedParameters`. The page ends with the rules that stop a simulation. [Traffic generation](./traffic-generation.md) explains the behavior behind the attributes.

## How to read the tables

**Default** is the value an attribute carries in a Traffic Generator application the platform adds to a configuration with `POST /api/v1/configurations/{config_id}/applications` ([Add the application](#add-the-application)). Such an application carries every attribute in these tables at its Default until you change it. If you supply the application's `typedParameters` yourself instead, for example in the `applications` of `POST /api/v1/configurations`, an attribute you leave out takes its Default.

**Type** is the value of the attribute's `type` field in the API, which a patch must repeat. Values of type `uint` are whole numbers, 0 or more; the sizes are in bytes. Values of type `double` are numbers. Values of type `milliseconds` are numbers of milliseconds of simulation time, sent with no `unit`. Values of type `bool` are `true` or `false`.

## Distribution ratios

The distribution ratios set the share of the hosts that run each component ([Host counts](./traffic-generation.md#host-counts)). They are not in the application's `typedParameters`: each is an entry of the configuration's application distribution, named by its `distributionId`, and is changed with `PATCH /api/v1/configurations/{config_id}/distribution` ([Change the distribution ratios](#change-the-distribution-ratios)).

| `distributionId` | Default | Description |
| --- | --- | --- |
| `prScalaTrafficGenerator` | An even share: `0.5` for the application on its own | Share of the hosts that run a client: more than 0.0 and less than 1.0, since every ratio must be above 0 and the ratios must add up to 1. |
| `prScalaTrafficServer` | An even share: `0.5` for the application on its own | Share of the hosts that run a server: more than 0.0 and less than 1.0, since every ratio must be above 0 and the ratios must add up to 1. |

When you add an application, the platform splits 1.0 evenly over every client and server entry of every application in the configuration. A configuration body you build yourself, for example for `POST /api/v1/configurations`, needs a distribution entry for each component in `topology.appDistribution.randomApplicationDistribution`: a component without one gets a ratio of 0, which makes the configuration invalid. The ratios of all the applications in a configuration must add up to 1, and none can be 0.

## Client

These attributes sit in the `client` object of the application's `typedParameters`, whose `name` is `TrafficGeneratorClient`. The tables group them by what they control.

### Connections and start

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `NumConnections` | `uint` | `8` | Number of connections between the client and each server it is paired with, from 1 to 8. Each connection is its own TCP connection, with its own requests at the full `RequestRate` ([Connections](./traffic-generation.md#connections)). |
| `StartType` | `string` | `uniform` | When each connection starts in the client start window: `uniform` (each at its own random time in the window), `simultaneous` (all at the window's start), or `sequential` (evenly spaced across the window). A connection generates its first request when it starts ([Start types](./traffic-generation.md#start-types)). |

### Requests

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `RequestRate` | `double` | `500.0` | Mean number of requests per second each connection generates during On periods. The gaps between requests are exponentially distributed with mean 1 / `RequestRate` seconds ([Transactions](./traffic-generation.md#transactions)). |
| `RequestSizeDistribution` | `string` | `normal` | How request sizes are drawn: `constant`, `uniform`, or `normal` ([Request and reply sizes](./traffic-generation.md#request-and-reply-sizes)). |
| `RequestSizeMin` | `uint` | `400` | Smallest request size, in bytes, and the size of every request with `constant`. Keep it less than `RequestSizeMax`. |
| `RequestSizeMax` | `uint` | `3000` | Largest request size, in bytes. |
| `AverageRequestSize` | `uint` | `1500` | Mean of the normal distribution request sizes are drawn from with `normal`, in bytes. `constant` and `uniform` do not use it. |

### Replies

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `ReplySizeDistribution` | `string` | `normal` | How reply sizes are drawn: `constant`, `uniform`, or `normal` ([Request and reply sizes](./traffic-generation.md#request-and-reply-sizes)). The client draws each reply's size when it generates the request, and the server replies with that size. |
| `ReplySizeMin` | `uint` | `1000` | Smallest reply size, in bytes, and the size of every reply with `constant`. Keep it no greater than `ReplySizeMax`. |
| `ReplySizeMax` | `uint` | `10000` | Largest reply size, in bytes. |
| `AverageReplySize` | `uint` | `6000` | Mean of the normal distribution reply sizes are drawn from with `normal`, in bytes. `constant` and `uniform` do not use it. |

### Burst cycle

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `BurstOnCycleTime` | `milliseconds` | `100` | Length of an On period, measured by the request gaps drawn in it. On average an On period has 1 + `RequestRate` x `BurstOnCycleTime` requests, with `BurstOnCycleTime` in seconds: 1 + 500 x 0.1 = 51 with the defaults ([Burst cycle](./traffic-generation.md#burst-cycle)). |
| `BurstOffCycleTime` | `milliseconds` | `0` | Length of an Off period, in which the connection generates no requests. At `0`, requests are continuous. |

### Records

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `ServerType` | `string` | `ScalaApp` | The label written in the `metric_name` column of `perf` ([perf](./traffic-generation.md#perf)). The only allowed value. |
| `EnableTcpStats` | `bool` | `true` | With the global simulation parameter `EnableTcpStackReporting` also `true`, writes a `tcp-stats` row for each client connection every `TcpStackReportInterval` ([tcp-stats](./traffic-generation.md#tcp-stats)). |

## Server

The `server` object of the application's `typedParameters`, whose `name` is `TrafficGeneratorServer`, is the server. A server listens on its own port for each connection of each client paired with it, and answers each request with a reply of the size the client drew. The client's attributes set the connections, the requests, and the replies ([Client](#client)).

## Setting up a Traffic Generator simulation through the platform

A Traffic Generator simulation needs a configuration whose topology gives every host a Scala NIC PCIe, and that has the Scala Traffic Generator. The reference pages for the APIs involved are [Components](../../api-reference/components.md) and [Configurations](../../api-reference/configurations.md). Get an API token first ([Authentication](../../authentication.md#using-your-token)) and export it as `SCALA_API_TOKEN`.

### Read the application's parameters

`GET /api/v1/components?category=application` lists the application models the platform offers; the `ScalaTrafficGeneratorApplication` entry gives the Traffic Generator's `id` and its `versions`. `GET /api/v1/components/{component_id}`, with the `version` query parameter set to the model version these pages describe, returns the application's `typedParameters`: `client` and `server` objects that each hold the component's `name` and its own `typedParameters`. Each attribute in the tables above appears there as an object with its `value`, its `type`, and, where the model sets them, its `default`, `description`, `min` and `max` (each an object with a `value`), and allowed values in `enum`.

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/components?category=application" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

# Set COMPONENT_ID to the id of the ScalaTrafficGeneratorApplication entry in that response.
export COMPONENT_ID="model_..."

curl -X GET "https://api.scalacomputing.com/api/v1/components/$COMPONENT_ID?version=3.5.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Add the application

`POST /api/v1/configurations/{config_id}/applications` adds the Traffic Generator to a configuration, with every attribute at its Default ([Add application to configuration](../../api-reference/configurations.md#add-application-to-configuration)). The query parameters are `modelId`, the application's component `id`; `name`, a name for the application that is unique in the configuration, 1 to 64 characters; and `version`. Set `CONFIG_ID` to the configuration's ID.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/applications?modelId=$COMPONENT_ID&name=traffic-gen&version=3.5.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

The platform also adds the application's two distribution entries, `prScalaTrafficGenerator` and `prScalaTrafficServer`, and splits 1.0 evenly over every entry in the configuration: `0.5` each when the Traffic Generator is the only application. The Scala Traffic Generator cannot share a simulation with the [Scala RDMA application](../scala-rdma-application/index.md) or the [Chakra workload](../chakra-workload/index.md).

### Change the distribution ratios

`PATCH /api/v1/configurations/{config_id}/distribution` sets one distribution ratio ([Update distribution entry value](../../api-reference/configurations.md#update-distribution-entry-value)). Its body has three fields, all required:

- `parentApplication`: the application's `name`.
- `distributionId`: `prScalaTrafficGenerator` or `prScalaTrafficServer`.
- `value`: the ratio, from 0.0 to 1.0. A value outside that range returns 400.

The request returns 404 when the configuration has no entry for that `parentApplication` and `distributionId`, and a 200 response returns the configuration's application distribution. A write does not need the ratios to add up to 1, but the configuration is `invalid` until they do, so change both entries. This example gives nine tenths of the hosts to clients and one tenth to servers, which at 16 hosts gives 15 clients and 1 server ([Worked example: 16 hosts](./traffic-generation.md#worked-example-16-hosts)):

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/distribution" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "parentApplication": "traffic-gen", "distributionId": "prScalaTrafficGenerator", "value": 0.9 }'

curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/distribution" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "parentApplication": "traffic-gen", "distributionId": "prScalaTrafficServer", "value": 0.1 }'
```

### Change values in a configuration

`PATCH /api/v1/configurations/{config_id}/applications/{app_name}/parameters` merges a patch into the `typedParameters` of the application named `app_name` ([Update application parameters](../../api-reference/configurations.md#update-application-parameters)); `GET /api/v1/configurations/{config_id}/applications?withDetails=true` lists the configuration's applications with their parameters. Set `APP_NAME` to the application's `name`.

The patch repeats the nesting of the application's `typedParameters`, down to each attribute you change:

- A client attribute sits in the `client` object, which carries the component's `name`, `TrafficGeneratorClient`, beside its `typedParameters`. Without `name`, the request returns 400.
- Only keys the application already has can be updated. An unknown key returns 400; the distribution ratios are not keys of `typedParameters` ([Change the distribution ratios](#change-the-distribution-ratios)).
- Each attribute's `type` must match the stored type, or the request returns 400. The request also returns 400 for an empty patch, or when a nested object is sent where an attribute is expected, or the reverse.
- A value outside the attribute's allowed values (`enum`) or range (`min`, `max`), a `uint` value that is negative or fractional, or a `milliseconds` value sent with a `unit` returns 400.
- A 200 response returns the application's full, updated `typedParameters`.

This example doubles the request rate and turns the burst cycle into 50 ms On and 50 ms Off:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/applications/$APP_NAME/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "client": {
      "name": "TrafficGeneratorClient",
      "typedParameters": {
        "RequestRate": { "value": 1000.0, "type": "double" },
        "BurstOnCycleTime": { "value": 50, "type": "milliseconds" },
        "BurstOffCycleTime": { "value": 50, "type": "milliseconds" }
      }
    }
  }'
```

### Change the global simulation parameters

The run's timeline (`StartTime`, `ClientStartOffset`, `ClientStartInterval`, `WarmUp`, `RunTime`, and `CoolDown`), `RandomSeed`, `BeaconReportInterval`, `EnableTcpStackReporting`, and `TcpStackReportInterval` are global simulation parameters, not attributes of this model ([Run timeline](./traffic-generation.md#run-timeline), [tcp-stats](./traffic-generation.md#tcp-stats)). They sit in the configuration's `SimulationParameters`, and `PATCH /api/v1/configurations/{config_id}/parameters` changes them ([Update global parameters](../../api-reference/configurations.md#update-global-parameters)). The patch names the section, then the parameter. This example turns the `tcp-stats` record on:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "SimulationParameters": {
      "EnableTcpStackReporting": { "value": true, "type": "bool" }
    }
  }'
```

## Configuration rules that stop a simulation

A simulation can start only from a configuration whose `status` is `validated`; `POST /api/v1/simulations` returns 422 for any other. `POST /api/v1/configurations/{config_id}/validate` returns the configuration's validation errors and warnings ([Validate configuration](../../api-reference/configurations.md#validate-configuration)). A configuration with the Scala Traffic Generator is `invalid` when:

- A host has a Scala RoCE NIC or a Scala UET NIC.
- The distribution ratios of all the applications in the configuration do not add up to 1.
- A distribution ratio is 0.
- The topology has fewer hosts than there are components with a ratio above 0: 2 for the Scala Traffic Generator on its own.
- The configuration also has the Scala RDMA application or the Chakra workload.

A write that sets a value outside an attribute's allowed values (`enum`) or range (`min`, `max`) returns 400.

The run stops as soon as a client starts when:

- `RequestSizeMin` is not less than `RequestSizeMax`.
- `ReplySizeMin` is greater than `ReplySizeMax`.
