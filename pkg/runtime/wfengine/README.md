# Dapr Workflow Engine

The Dapr Workflow engine enables developers to author workflows using code and execute them using the Dapr sidecar. You can learn more about this project here: [[Proposal] Workflow building block and engine (#4576)](https://github.com/dapr/dapr/issues/4576).

This README is designed to be used by maintainers to help with getting started. This README will be updated with more information as the project progresses.

## Building Daprd

The workflow engine is entirely encapsulated within the [dapr sidecar (a.k.a. daprd)](https://docs.dapr.io/concepts/dapr-services/sidecar/). All dependencies are compiled directly into the binary.

Internally, this engine depends on the [Durable Task Framework for Go](https://github.com/microsoft/durabletask-go), an MIT-licensed open-source project for authoring workflows as code. Use the following command to get the latest build of this dependency:

```bash
go get github.com/microsoft/durabletask-go@dapr
```

The following command can be used to build a version of Daprd that supports the workflow engine.

```bash
DEBUG=1 make build
```
* `DEBUG=1` is required to attach debuggers. This should never be set for production or performance testing workloads.

After building, the following command can be run from the project root to test the code:

```bash
./dist/linux_amd64/debug/daprd --app-id foo --dapr-grpc-port 4001 --placement-host-address :50005 ~/.dapr/components/
```
* The gRPC port is set to `4001` since that's what the Durable Task test clients default to.
* This assumes a placement service running locally on port `50005` (the default).
* This assumes a basic actor-compatible state store is configured in `~/.dapr/components`.
* You should see logs with `scope=dapr.runtime.wfengine` if the workflow engine is enabled in your build.

Here's an example of the log output you'll see from Dapr when the workflow engine is enabled:

```
INFO[0000] configuring workflow engine gRPC endpoint         app_id=foo instance=XYZ scope=dapr.runtime.wfengine type=log ver=edge
INFO[0000] configuring workflow engine with actors backend   app_id=foo instance=XYZ scope=dapr.runtime.wfengine type=log ver=edge
INFO[0000] worker started with backend dapr.actors/v1-alpha  app_id=foo instance=XYZ scope=dapr.runtime.wfengine type=log ver=edge
INFO[0000] workflow engine started                           app_id=foo instance=XYZ scope=dapr.runtime.wfengine type=log ver=edge
```

If you want to see the full set of logs, run daprd with verbose logging enabled (`--log-level debug`). You'll see a few additional logs in this case, indicating that the workflow engine is waiting for new work items:

```
DEBU[0000] orchestration-processor: waiting for new work items...  app_id=foo instance=XYZ scope=dapr.runtime.wfengine type=log ver=edge
DEBU[0000] activity-processor: waiting for new work items...       app_id=foo instance=XYZ scope=dapr.runtime.wfengine type=log ver=edge
```

## Running tests

There are no tests that directly target Dapr Workflows yet. However, this engine is fully compatible with .NET and Java Durable Task SDKs.

| Language/Stack | Package | Project Home | Samples |
| - | - | - | - |
| .NET | [![NuGet](https://img.shields.io/nuget/v/Microsoft.DurableTask.Client.svg?style=flat)](https://www.nuget.org/packages/Microsoft.DurableTask.Client/) | [GitHub](https://github.com/microsoft/durabletask-dotnet) | [Samples](https://github.com/microsoft/durabletask-dotnet/tree/main/samples) |
| Java | [![Maven Central](https://img.shields.io/maven-central/v/com.microsoft/durabletask-client?label=durabletask-client)](https://search.maven.org/artifact/com.microsoft/durabletask-client) | [GitHub](https://github.com/microsoft/durabletask-java) | [Samples](https://github.com/microsoft/durabletask-java/tree/main/samples/src/main/java/io/durabletask/samples) |

You can also run the samples above and have them execute end-to-end with Dapr running locally on the same machine. The samples connect to gRPC over port `4001` by default, which will work without changes as long as Dapr is configured with `4001` as its gRPC port (like in the example above).

In the future, instructions will be added for how to tests in a more automated way.

## How it works

The Dapr Workflow engine introduced a new concept of *internal actors*. These are actors that are registered and implemented directly in Daprd with no host application dependency. Just like regular actors, they have turn-based concurrency, support reminders, and are scaled out using the placement service. Internal actors also leverage the configured state store for actors. The workflow engine uses these actors as the core runtime primitives for workflows.

Each workflow instance corresponds to a single `dapr.wfengine.workflow` actor instance. The ID of the workflow instance is the same as the internal actor ID. The internal actor is responsible for triggering workflow execution and for storing workflow state. The actual workflow logic lives outside the Dapr sidecar in a host application. The host application uses a new gRPC endpoint on the daprd gRPC API server to send and receive workflow-specific commands to/from the actor-based workflow engine. The workflow app doesn't need to take on any actor dependencies, nor is it aware that actors are involved in the execution of the workflows. Actors are purely an implementation detail.

TODO: More information on activity actors, the details of state storage, and information about how reminders are used to ensure reliability.