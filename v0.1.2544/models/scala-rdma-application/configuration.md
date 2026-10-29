---
title: "Configuration"
description: "This page lists the Scala RDMA application attributes that change a run, with their types and defaults, and shows how to set the application up."
---

This page lists the Scala RDMA application attributes that change a run, with their types and defaults, and shows how to set the application up.

The Scala RDMA application is an application in a simulation configuration. Its `typedParameters` hold `distributionType` at the top level and two components: `client`, the Initiator, and `server`, the Responder. Its distribution ratios are set apart from `typedParameters`. The page ends with the rules that stop a simulation. [Traffic generation](./traffic-generation.md) explains the behavior behind the attributes.

## How to read the tables

**Default** is the value an attribute carries in an RDMA application the platform adds to a configuration with `POST /api/v1/configurations/{config_id}/applications` ([Add the application](#add-the-application)). Such an application carries every attribute in these tables at its Default until you change it. If you supply the application's `typedParameters` yourself instead, for example in the `applications` of `POST /api/v1/configurations`, an attribute you leave out takes its Default.

**Type** is the value of the attribute's `type` field in the API, which a patch must repeat. Values of type `bytes` carry a `unit`: `B`, `KB` (1,000 bytes), `KiB` (1,024 bytes), `MB` (1,000,000 bytes), or `MiB` (1,048,576 bytes). Values of type `uint` are whole numbers, 0 or more.

## Application

This attribute sits at the top level of the application's `typedParameters`.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `distributionType` | `string` | `ClientServer` | How the application's components are paired: each Initiator is paired with a Responder on another host ([Pairing](./traffic-generation.md#pairing)). The components API marks it read-only. |

## Distribution ratios

The distribution ratios set the share of the hosts that run each component ([Host counts](./traffic-generation.md#host-counts)). They are not in the application's `typedParameters`: each is an entry of the configuration's application distribution, named by its `distributionId`, and is changed with `PATCH /api/v1/configurations/{config_id}/distribution` ([Change the distribution ratios](#change-the-distribution-ratios)).

| `distributionId` | Default | Description |
| --- | --- | --- |
| `prScalaRdmaInitiator` | An even share: `0.5` for the application on its own | Share of the hosts that run an Initiator: more than 0.0 and less than 1.0, since every ratio must be above 0 and the ratios must add up to 1. |
| `prScalaRdmaResponder` | An even share: `0.5` for the application on its own | Share of the hosts that run a Responder: more than 0.0 and less than 1.0, since every ratio must be above 0 and the ratios must add up to 1. |

When you add an application, the platform splits 1.0 evenly over every client and server entry of every application in the configuration. A configuration body you build yourself, for example for `POST /api/v1/configurations`, needs a distribution entry for each component in `topology.appDistribution.randomApplicationDistribution`: a component without one gets a ratio of 0, which makes the configuration invalid. The ratios of all the applications in a configuration must add up to 1, and none can be 0.

## Initiator

These attributes sit in the `client` object of the application's `typedParameters`, whose `name` is `ScalaRdmaInitiator`.

| Attribute | Type | Default | Description |
| --- | --- | --- | --- |
| `NumConnections` | `uint` | `1` | Number of connections between the Initiator and each Responder it is paired with, from 1 to 1,024. Each connection has its own queue pair and its own `MaxPendingWQEs` WQEs outstanding ([Connections](./traffic-generation.md#connections)). |
| `StartType` | `string` | `simultaneous` | When each connection starts in the client start window: `simultaneous` (all at the window's start), `uniform` (each at its own random time in the window), or `sequential` (evenly spaced across the window). Whatever the value, WQE posting starts at the end of the window, or when the connection's setup completes if that is later ([Start types](./traffic-generation.md#start-types)). |
| `OpCode` | `string` | `RDMA_OPCODE_WRITE_FIRST` | The only allowed value. The NIC chooses each packet's opcode ([WQEs](./traffic-generation.md#wqes)). |
| `RdmaVerb` | `string` | `RDMA_VERB_POST_WRITE` | The only allowed value: every WQE is an RDMA WRITE of 250,000 bytes. |
| `DoorbellPayloadSize` | `bytes` | `64B` | Payload of the doorbell message that posts each WQE to the NIC over the host's PCIe link. The message adds 28 bytes of UDP and IPv4 headers, and it never reaches the network ([WQEs](./traffic-generation.md#wqes)). |
| `RdmaDataPacketTC` | `uint` | `0` | On the Scala RoCE NIC, the traffic class of the RDMA WRITE data packets: it selects the NIC's transmit queue and sets their DSCP codepoint to CSn for class n. Use a class from 0 to 7 that has a transmit pool on the Initiator's NIC and a receive pool on the Responder's NIC: classes 0 and 1 with the NIC's defaults ([Traffic class on the Scala RoCE NIC](./traffic-generation.md#traffic-class-on-the-scala-roce-nic)). |
| `MaxPendingWQEs` | `uint` | `10` | WQEs each connection keeps outstanding: it posts this many when data transfer starts, and one more each time one completes ([Closed-loop generation](./traffic-generation.md#closed-loop-generation)). |

## Responder

The `server` object of the application's `typedParameters`, whose `name` is `ScalaRdmaResponder`, is the Responder. A Responder accepts the connections of the Initiators paired with it, and its NIC acknowledges the data they write. The Initiator's attributes set the connections and the traffic ([Initiator](#initiator)).

## Setting up an RDMA simulation through the platform

An RDMA simulation needs a configuration whose topology gives every host a Scala RoCE NIC, or every host a Scala UET NIC, and that has the Scala RDMA application. The reference pages for the APIs involved are [Components](../../api-reference/components.md) and [Configurations](../../api-reference/configurations.md). Get an API token first ([Authentication](../../authentication.md#using-your-token)) and export it as `SCALA_API_TOKEN`.

### Read the application's parameters

`GET /api/v1/components?category=application` lists the application models the platform offers; the `ScalaRdmaApplication` entry gives the RDMA application's `id` and its `versions`. `GET /api/v1/components/{component_id}`, with the `version` query parameter set to the model version these pages describe, returns the application's `typedParameters`: `distributionType` at the top level, and `client` and `server` objects that each hold the component's `name` and its own `typedParameters`. Each attribute in the tables above appears there as an object with its `value`, its `type`, a `unit` for `bytes` values, and, where the model sets them, its `default` (with `defaultUnit` for `bytes` values), `description`, `min` and `max` (each an object with a `value`), `readonly`, and allowed values in `enum`.

```bash
curl -X GET "https://api.scalacomputing.com/api/v1/components?category=application" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"

# Set COMPONENT_ID to the id of the ScalaRdmaApplication entry in that response.
export COMPONENT_ID="model_..."

curl -X GET "https://api.scalacomputing.com/api/v1/components/$COMPONENT_ID?version=3.5.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

### Add the application

`POST /api/v1/configurations/{config_id}/applications` adds the RDMA application to a configuration, with every attribute at its Default ([Add application to configuration](../../api-reference/configurations.md#add-application-to-configuration)). The query parameters are `modelId`, the application's component `id`; `name`, a name for the application that is unique in the configuration, 1 to 64 characters; and `version`. Set `CONFIG_ID` to the configuration's ID.

```bash
curl -X POST "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/applications?modelId=$COMPONENT_ID&name=rdma-app&version=3.5.0" \
  -H "Authorization: Bearer $SCALA_API_TOKEN"
```

The platform also adds the application's two distribution entries, `prScalaRdmaInitiator` and `prScalaRdmaResponder`, and splits 1.0 evenly over every entry in the configuration: `0.5` each when the RDMA application is the only application. The Scala RDMA application cannot share a simulation with the [Scala Traffic Generator](../scala-traffic-generator/index.md), the Scala UDP Echo application, or the [Chakra workload](../chakra-workload/index.md).

### Change the distribution ratios

`PATCH /api/v1/configurations/{config_id}/distribution` sets one distribution ratio ([Update distribution entry value](../../api-reference/configurations.md#update-distribution-entry-value)). Its body has three fields, all required:

- `parentApplication`: the application's `name`.
- `distributionId`: `prScalaRdmaInitiator` or `prScalaRdmaResponder`.
- `value`: the ratio, from 0.0 to 1.0. A value outside that range returns 400.

The request returns 404 when the configuration has no entry for that `parentApplication` and `distributionId`, and a 200 response returns the configuration's application distribution. A write does not need the ratios to add up to 1, but the configuration is `invalid` until they do, so change both entries. This example gives a quarter of the hosts to Initiators and three quarters to Responders, which at 16 hosts gives 4 Initiators and 12 Responders ([Worked example: 16 hosts](./traffic-generation.md#worked-example-16-hosts)):

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/distribution" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "parentApplication": "rdma-app", "distributionId": "prScalaRdmaInitiator", "value": 0.25 }'

curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/distribution" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "parentApplication": "rdma-app", "distributionId": "prScalaRdmaResponder", "value": 0.75 }'
```

### Change values in a configuration

`PATCH /api/v1/configurations/{config_id}/applications/{app_name}/parameters` merges a patch into the `typedParameters` of the application named `app_name` ([Update application parameters](../../api-reference/configurations.md#update-application-parameters)); `GET /api/v1/configurations/{config_id}/applications?withDetails=true` lists the configuration's applications with their parameters. Set `APP_NAME` to the application's `name`.

The patch repeats the nesting of the application's `typedParameters`, down to each attribute you change:

- An Initiator attribute sits in the `client` object, which carries the component's `name`, `ScalaRdmaInitiator`, beside its `typedParameters`. Without `name`, the request returns 400.
- Only keys the application already has can be updated. An unknown key returns 400; the distribution ratios are not keys of `typedParameters` ([Change the distribution ratios](#change-the-distribution-ratios)).
- Each attribute's `type` must match the stored type, or the request returns 400. The request also returns 400 for an empty patch, or when a nested object is sent where an attribute is expected, or the reverse.
- The `value` and the `unit` you send replace the stored ones, so send the `unit` with every `bytes` value.
- A value outside the attribute's allowed values (`enum`) or range (`min`, `max`), a `uint` value that is negative or fractional, or a `bytes` value other than 0 without a valid unit returns 400.
- A 200 response returns the application's full, updated `typedParameters`.

This example keeps 4 WQEs outstanding on each connection and opens 4 connections per pair, for 16 WQEs outstanding per pair:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/applications/$APP_NAME/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "client": {
      "name": "ScalaRdmaInitiator",
      "typedParameters": {
        "MaxPendingWQEs": { "value": 4, "type": "uint" },
        "NumConnections": { "value": 4, "type": "uint" }
      }
    }
  }'
```

### Change the global simulation parameters

The run's timeline (`StartTime`, `ClientStartOffset`, `ClientStartInterval`, `WarmUp`, `RunTime`, and `CoolDown`), `RandomSeed`, and `BeaconReportInterval` are global simulation parameters, not attributes of this model ([Run timeline](./traffic-generation.md#run-timeline)). They sit in the configuration's `SimulationParameters`, and `PATCH /api/v1/configurations/{config_id}/parameters` changes them ([Update global parameters](../../api-reference/configurations.md#update-global-parameters)). The patch names the section, then the parameter. This example lengthens the recording window to 1 second:

```bash
curl -X PATCH "https://api.scalacomputing.com/api/v1/configurations/$CONFIG_ID/parameters" \
  -H "Authorization: Bearer $SCALA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "SimulationParameters": {
      "RunTime": { "value": 1.0, "type": "double" }
    }
  }'
```

## Configuration rules that stop a simulation

A simulation can start only from a configuration whose `status` is `validated`; `POST /api/v1/simulations` returns 422 for any other. `POST /api/v1/configurations/{config_id}/validate` returns the configuration's validation errors and warnings ([Validate configuration](../../api-reference/configurations.md#validate-configuration)). A configuration with the Scala RDMA application is `invalid` when:

- A host has a Scala NIC PCIe.
- The distribution ratios of all the applications in the configuration do not add up to 1.
- A distribution ratio is 0.
- The topology has fewer hosts than there are components with a ratio above 0: 2 for the Scala RDMA application on its own.
- The configuration also has the Scala Traffic Generator, the Scala UDP Echo application, or the Chakra workload.

A write that sets a value outside an attribute's allowed values (`enum`) or range (`min`, `max`), or a `bytes` value other than 0 without a valid unit, returns 400.
