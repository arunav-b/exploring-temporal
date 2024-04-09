# Exploring Temporal

## What is a Workflow?

Conceptually, a workflow defines a sequence of steps. With Temporal, those steps are defined by writing code, known as a _**Workflow Definition**_, and are carried out by running that code, which results in a **_Workflow Execution_**.


## Temporal Architecture Overview

### Temporal Server
![temporalServer](/images/temporal-server-diagram.png)

### Communication between Temporal Cluster and Application using Temporal

![communications](/images/communication-v2.png)

Clients communicate with the Temporal Server by issuing requests to this Frontend Service. The Frontend Service then communicates with backend services, as necessary to fulfill the request, and then returns a response to the client. Communication to and within the Cluster is done using gRPC, a popular high-performance open source RPC framework originally developed at Google and now part of the Cloud Native Computing Foundation ecosystem. The messages themselves are encoded using Protocol Buffers, an open source serialization mechanism also originally developed at Google.

### Temporal Cluster

![temporalCluster](/images/temporal-cluster-diagram.png)

### Workers

One thing that people new to Temporal may find surprising is that the Temporal Cluster does not execute your code. While the platform guarantees the durable execution of your code, it achieves this through orchestration. The execution of your application code is external to the cluster, and in typical deployments, takes place on a separate set of servers, potentially running in a different data center than the Temporal Cluster.

The entity responsible for executing your code is known as a Worker, and it's common to run Workers on multiple servers, since this increases both the scalability and availability of your application. The Worker, which is part of your application, communicates with the Temporal Cluster to manage the execution of your Workflows.

Since the Worker uses a Temporal Client to communicate with the Temporal Cluster, each machine running a Worker will require connectivity to the Cluster’s Frontend Service, which listens on TCP port 7233 by default.

![Workers](/images/temporal-platform-diagram.png)