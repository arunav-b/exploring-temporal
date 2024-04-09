# Exploring Temporal

## What is a Workflow?

Conceptually, a workflow defines a sequence of steps. With Temporal, those steps are defined by writing code, known as a _**Workflow Definition**_, and are carried out by running that code, which results in a **_Workflow Execution_**.

<br>
<br>

## Temporal Architecture Overview

### Temporal Server
![temporalServer](/images/temporal-server-diagram.png)

<br>

### Communication between Temporal Cluster and Temporal Application

![communications](/images/communication-v2.png)

Clients communicate with the Temporal Server by issuing requests to this Frontend Service. The Frontend Service then communicates with backend services, as necessary to fulfill the request, and then returns a response to the client. Communication to and within the Cluster is done using gRPC, a popular high-performance open source RPC framework originally developed at Google and now part of the Cloud Native Computing Foundation ecosystem. The messages themselves are encoded using Protocol Buffers, an open source serialization mechanism also originally developed at Google.

<br>

### Temporal Cluster

![temporalCluster](/images/temporal-cluster-diagram.png)

<br>

### Workers

One thing that people new to Temporal may find surprising is that the Temporal Cluster does not execute your code. While the platform guarantees the durable execution of your code, it achieves this through orchestration. The execution of your application code is external to the cluster, and in typical deployments, takes place on a separate set of servers, potentially running in a different data center than the Temporal Cluster.

The entity responsible for executing your code is known as a Worker, and it's common to run Workers on multiple servers, since this increases both the scalability and availability of your application. The Worker, which is part of your application, communicates with the Temporal Cluster to manage the execution of your Workflows.

Since the Worker uses a Temporal Client to communicate with the Temporal Cluster, each machine running a Worker will require connectivity to the Cluster’s Frontend Service, which listens on TCP port 7233 by default.

![Workers](/images/temporal-platform-diagram.png)

<br>
<br>

## Writing a Workflow Definition

There are two steps for turning a Java interface and implementation into a **_Workflow Definition_**:

1. Import the `io.temporal.workflow.WorkflowInterface` and `io.temporal.workflow.WorkflowMethod` annotation types provided by the SDK
2. Annotate the interface with `@WorkflowInterface`
3. Annotate the method signature with `@WorkflowMethod`

<br>

## Initializing Worker

### Role of a Worker

- Workers execute your Workflow code. 
- The Worker itself is provided by the Temporal SDK, but your application will include code to configure and run it. 
- When that code executes, the Worker establishes a persistent connection to the Temporal Cluster and begins polling a Task Queue on the Cluster, seeking work to perform.

### Initializing a Worker

There are typically three things you need in order to configure a Worker:

1. A **Temporal Client**, which is used to communicate with the Temporal Cluster
2. The name of a **Task Queue**, which is maintained by the Temporal Server and polled by the Worker
3. The name of the **Workflow Definition interface**, used to register the Workflow implementation with the Worker

```java
public class GreetingWorker {

    public static void main(String[] args) {

        // Represents the GRPC connection to the Temporal Cluster.
        // For Temporal Cluster is running on the same machine, we'll use newLocalServiceStubs(). 
        // When Temporal Cluster is on a dedicated server, we'll use newServiceStubs(WorkflowServiceStubsOptions options)
        WorkflowServiceStubs service = WorkflowServiceStubs.newLocalServiceStubs();
        
        // Create a Temporal Client using WorkflowServiceStubs
        WorkflowClient client = WorkflowClient.newInstance(service);
        
        // Creates one or more Worker instances
        WorkerFactory factory = WorkerFactory.newInstance(client);

        // Specify the name of the Task Queue that this Worker should poll
        Worker worker = factory.newWorker("greeting-tasks");

        // Specify which Workflow implementations this Worker will support
        worker.registerWorkflowImplementationTypes(GreetingImpl.class);

        // Begin running the Worker
        factory.start();
    }
}
```

### Lifetime of a Worker

The lifetime of the Worker and the duration of a Workflow Execution are unrelated. The start function used to start this Worker is a blocking function that doesn't stop unless it is terminated or encounters a fatal error. The Worker's process may last for days, weeks, or longer. If the Workflows it handles are relatively short, then a single Worker might execute thousands or even millions of them during its lifetime. On the other hand, a Workflow can run for years, while the server where a Worker process is running might be rebooted after a few months by an administrator doing maintenance. If the Workflow Type was registered with other workers, one or more of them will automatically continue where the original Worker left off. If there are no other Workers available, then the Workflow Execution will continue where it left off as soon as the original Worker is restarted. In either case, the downtime will not cause the Workflow Execution to fail.

<br>

## Code to start a Workflow

```java
public class Starter {
    public static void main(String[] args) throws Exception {

        WorkflowServiceStubs service = WorkflowServiceStubs.newLocalServiceStubs();
        WorkflowClient client = WorkflowClient.newInstance(service);
        
        WorkflowOptions options = WorkflowOptions.newBuilder()
                    .setWorkflowId("my-first-workflow")
                    .setTaskQueue("greeting-tasks")
                    .build();
       
        // Creating a new Workflow instance
        HelloWorkflowWorkflow workflow = client.newWorkflowStub(HelloWorkflowWorkflow.class, options);
        
        // Blocking call on the Workflow
        String greeting = workflow.greetSomeone(args[0]);
        
        // Retrieving information of the workflow when execution is complete
        String workflowId = WorkflowStub.fromTyped(workflow).getExecution().getWorkflowId();

        System.out.println(workflowId + " " + greeting);

    }
}
```

> NOTE:
> 1. The code used to create and configure the `client` here is identical to the code used during Worker initialization. You can structure your application such that the same client is shared between those two parts of the code. In fact, this is common for real-world Temporal applications.
> 2. Workflow code must be **_deterministic_**, and must produce the same output each time, given the same input. 

<br>

## What are activities ?

- Activities encapsulate business logic that is prone to failure. Unlike the Workflow Definition, there is no requirement for an Activity Definition to be deterministic.
- While Activities are executed as part of Workflow Execution, they have an important characteristic: they're retried if they fail.
- The code within that Activity Definition will be executed, retried if necessary, and the Workflow will continue its progress once the Activity completes successfully.

### Activity Definition

- The interface must be annotated with `@ActivityInterface`.
- Optionally, you can annotate your methods with `@ActivityMethod`, although this is not required unless you are attempting to specify optional arguments to the Activity.

```java
@ActivityInterface
public interface GreetingActivities {

    public String greetInSpanish(String name);
}
```

### Registering Activities

You may recall that you must register your Workflows when initializing the Worker. You must also perform a similar step for Activities. The process for registering the Activity is slightly different to that for registering a Workflow, with the only difference being the name of the function you call to register it, and by passing in an instance of the Activity implementation to the registration method.

```java
public class GreetingWorker {
    public static void main(String[] args) {

        WorkflowServiceStubs service = WorkflowServiceStubs.newLocalServiceStubs();
        WorkflowClient client = WorkflowClient.newInstance(service);
        WorkerFactory factory = WorkerFactory.newInstance(client);
        Worker worker = factory.newWorker("greeting-tasks");
        worker.registerWorkflowImplementationTypes(GreetingWorkflowImpl.class);

        // Registering an activity
        worker.registerActivitiesImplementations(new GreetingActivitiesImpl());

        factory.start();
    }
}
```

### Specifying Activity Options 
The first step to executing an Activity as part of your Workflow is to specify the options that govern its execution. The following code would be written in the implementation of the Workflow Definition.

```java
ActivityOptions options = ActivityOptions.newBuilder()
        .setStartToCloseTimeout(Duration.ofSeconds(5))
        .build();
```

### Executing Activities

- Temporal Activities can be executed either synchronously or asynchronously, depending on your use case.
- 

