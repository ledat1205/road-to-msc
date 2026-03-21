

![](https://substackcdn.com/image/fetch/$s_!Ir_N!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6489490c-01ad-4754-8323-f20738cde0e1_2000x1428.png)


---

## Intro

As a data engineer, I’m often told that understanding the business and bringing actual value to the organization is the most important thing. I totally agree with that. From time to time, vendors and communities have done a great job by providing more and more abstraction to hide the low-level complexity from us, from frameworks (e.g., Spark, Airflow) to managed services (e.g., Databricks, Snowflake).

That’s great. All you need might be just a few clicks. However, I am the one who always wants to understand the things that happen behind the scenes, especially with the infrastructure. I realized there is one thing that everyone (who loves infrastructure like me) should learn, as it encompasses a vast amount of practical knowledge.

It is Kubernetes.

In this week's article, we will delve into this system, learn its basic concepts, and how it works. Of course, the goal is not to turn you into a DevOps engineer; instead, it is to equip you with practical skills so you can apply them to your daily job as a Data Engineer. To make things more interesting, I will run an Apache Spark application on Kubernetes to connect everything we learn.

But first, let’s understand the concept of a container.

---

## Containers

VM was the game-changer, as it allowed more than one application to run securely and in isolation on the server. However, it had some flaws:

- The VM required a dedicated OS system, and the resources (CPU and RAM) consumed by the OS could have otherwise made room for more applications.
    
- The OS also requires patching and monitoring, and it necessitates a license fee, which may be inconvenient for specific use cases.
    

Here comes the container technologies. Containers are lightweight, portable methods for packaging and running applications. They bundle everything your application needs to operate, from the code to its dependencies, such as libraries and environment variables, ensuring it runs smoothly across different environments.

[

![](https://substackcdn.com/image/fetch/$s_!zOaY!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe8fe2531-4f4c-4c4f-9bf1-64b96d378a06_514x360.png)



](https://substackcdn.com/image/fetch/$s_!zOaY!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe8fe2531-4f4c-4c4f-9bf1-64b96d378a06_514x360.png)

Unlike VMs, containers do not require a dedicated OS: all containers share the host’s OS. Users don’t need to worry about the CPU and RAM requirements for the OS, patching and monitoring processes, or license fees.

[

![](https://substackcdn.com/image/fetch/$s_!-TT6!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F60e6c75b-a44d-4388-b6ac-d9034c899b40_828x498.png)



](https://substackcdn.com/image/fetch/$s_!-TT6!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F60e6c75b-a44d-4388-b6ac-d9034c899b40_828x498.png)

Docker is the most popular containerization platform, making it easy to create and run containers. Running a few containers manually with Docker Compose on a single server is acceptable, but what if you need to run hundreds or thousands of containers across multiple servers? How do you manage the container’s scaling and communication across servers?

This is where Kubernetes steps in, providing the tools and abstractions needed to manage containers at scale.

> _**Note:** Docker itself has a solution for this called Docker Swarm, but I found it is less widely adopted compared to Kubernetes._

## Kubernetes Overview

Kubernetes, often abbreviated as k8s, is an open-source platform designed to automate the deployment, scaling, and operation of containerized applications. Initially developed by Google, it is now maintained by the Cloud Native Computing Foundation (CNCF).

> _**Note**: I will use both Kubernetes and k8s in this article._

The official definition might be quite overwhelming to some people (I know this because I used to be that way when I first learned about k8s). To make things more straightforward for me, a k8s cluster is a single, unified resource pool.

[

![](https://substackcdn.com/image/fetch/$s_!axKu!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F23ee8320-5f21-4010-a421-63a0a2eb17ca_684x500.png)



](https://substackcdn.com/image/fetch/$s_!axKu!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F23ee8320-5f21-4010-a421-63a0a2eb17ca_684x500.png)

It abstracts away the underlying hardware, allowing us to deploy applications (which must be containerized) without caring much about the server behind the scenes.

At the heart of Kubernetes are **controllers**, which are control loops that monitor the state of your cluster and make adjustments to achieve the desired state. Here’s how they work:

[

![](https://substackcdn.com/image/fetch/$s_!E1B9!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F38f286e2-358c-4cab-b54e-5126a5c5c07e_906x406.png)



](https://substackcdn.com/image/fetch/$s_!E1B9!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F38f286e2-358c-4cab-b54e-5126a5c5c07e_906x406.png)

---

> To celebrate Lunar New Year (the true New Year holiday in Vietnam), I’m offering _**50% off the annual subscription**_. The offer ends soon; grab it now to get full access to nearly 200 high-quality data engineering articles.
> 
> [50% off the annual subscription](https://vutr.substack.com/subscribe)

- **Desired State**: You declaratively define your application's desired state. For example, you might specify that you want three web server instances running.
    
- **Actual State**: Kubernetes continuously monitors the system's actual state. It checks the number of instances your application runs and compares this with the desired state.
    
- **Reconciliation Loop**: If there’s a difference between the desired and actual states, the Kubernetes controller reconciles this difference. It might start a new instance, replace a failed one, or perform other necessary adjustments. This feedback loop ensures that your applications always run as intended.
    

K8s will continuously check (the reconciliation loop); do the users have (the actual state) what they wanted (the desired state)? If not, it tries to fix it. You ne| Feature                 | Native spark-submit | Spark Operator (Apache/Kubeflow)         |
| ----------------------- | ------------------- | ---------------------------------------- |
| Style                   | Imperative CLI      | Declarative CRD/YAML                     |
| Job as K8s object       | No                  | Yes                                      |
| GitOps                  | Hard                | Excellent                                |
| Restart/Cron            | Manual              | Built-in                                 |
| Dynamic alloc + shuffle | Yes (tracking)      | Yes (tracking)                           |
| Production choice       | Simple/legacy       | Most teams (scalability + observability) |ant, K8s sees it’s missing 1 Airflow’s scheduler instance. It then created 1 one more instance to ensure satisfying the desired state.

## The overview architecture

A Kubernetes cluster consists of two main components: **the Control Plane and the** **Worker Nodes**.

[

![](https://substackcdn.com/image/fetch/$s_!UZ1b!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F966b980a-b019-43b6-8970-0f01e80a9f7a_1350x664.png)



](https://substackcdn.com/image/fetch/$s_!UZ1b!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F966b980a-b019-43b6-8970-0f01e80a9f7a_1350x664.png)

### Control Plane

The **Control Plane** manages the cluster's overall state and has:

- **API Server**: The entry point for all Kubernetes API interactions, validating and processing requests, allowing users, administrators, and other components to interact with the k8s cluster.
    
- **etcd**: A key-value store holding the cluster’s configuration, state, and metadata information, ensuring consistency across the system.
    
- **Controller Manager**: Runs controllers that monitor and reconcile the state of resources like Pods and Nodes to match the desired configuration.
    
- **Scheduler**: Assigns Pods to Nodes based on resource availability and policies, optimizing workload distribution.
    
- **Cloud control manager:** This component integrates cloud-specific control logic into Kubernetes. It connects your cluster to the cloud provider's API, separating cloud-specific interactions from those within the cluster. This component runs controllers tailored to the cloud provider, managing tasks like provisioning resources and handling cloud-specific operations.
    
    - If your Kubernetes cluster runs on local machines, the Cloud Controller Manager is unnecessary.
        

### Worker Nodes

**Worker Nodes** run your containerized applications and include:

- **Kubelet**: An agent ensuring containers run as the Control Plane specifies. It is an essential component of the Kubernetes node responsible for managing the pods running on it. (Pod will be discussed later)
    
    - It ensures that containers within those pods are running and healthy by interacting with the Kubernetes control plane and taking care of tasks like container lifecycle management, resource allocation, and monitoring.
        
- **Container Runtime**: The software (e.g., containerd, CRI-O) responsible for running the containers on the node. It is responsible for managing the execution and lifecycle of containers within the Kubernetes environment.
    
- **Kube Proxy**: Manages networking for Pods, handling communication within the cluster and with external resources. It is responsible for implementing the Kubernetes Service concept by maintaining network rules to forward incoming service requests to the appropriate backend pods.
    

## Pod

The **Pod** is the smallest and most basic deployable object in Kubernetes. It represents a single instance of a running process in the cluster and can contain one or more containers that share the same network namespace and storage.

[

![](https://substackcdn.com/image/fetch/$s_!dCdp!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc43a21a7-89ad-4558-b496-e84d535e3c07_1124x340.png)



](https://substackcdn.com/image/fetch/$s_!dCdp!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc43a21a7-89ad-4558-b496-e84d535e3c07_1124x340.png)

You might wonder, “Why not just manage containers directly?”

Kubernetes uses Pods to group containers that need to work together, simplifying networking and storage management. For instance, a Pod might include a web server container alongside a helper container that handles logging.

Pods are ephemeral, meaning they can be created, destroyed, or replaced at any time. This is where higher-level abstractions, such as Deployments, come into play. They manage the lifecycle of Pods.

## Deployment

**Deployments** are one of the most commonly used Kubernetes abstractions. They provide a declarative way to manage the lifecycle of Pods, handling tasks like scaling, updates, and rollbacks.

[

![](https://substackcdn.com/image/fetch/$s_!DC6L!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5515a59f-7989-4b3e-bfd2-a45713652906_1140x502.png)



](https://substackcdn.com/image/fetch/$s_!DC6L!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5515a59f-7989-4b3e-bfd2-a45713652906_1140x502.png)

In k8s, a ReplicaSet is an abstraction that maintains a stable set of replica pods running at any given time. In most cases, we will use a Deployment to manage ReplicaSets.

For example, if you need to scale up the application, a Deployment ensures that the correct number of Pods are running at all times. It also facilitates rolling updates, allowing you to update your application without downtime (pod by pod). If something goes wrong, you can easily roll back to a previous version.

Deployment is the most common resource for running stateless workloads, such as web servers, APIs, and microservices.

## StatefulSet

Unlike Deployment, which is ideal for applications where all pods are interchangeable and have no unique identity. K8s has an abstraction called StatefulSet, which is designed for applications that require a unique, stable identity for each pod. This is essential for databases, message queues, and other distributed systems that rely on a specific order of operations and persistent data.

[

![](https://substackcdn.com/image/fetch/$s_!rII_!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2db102f4-f16d-4f78-ab20-f05296abde3a_894x340.png)



](https://substackcdn.com/image/fetch/$s_!rII_!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F2db102f4-f16d-4f78-ab20-f05296abde3a_894x340.png)

Each pod is given a stable, unique name. This identity persists even if the pod is rescheduled. Each pod is associated with its own PersistentVolumeClaim (PVC), guaranteeing a dedicated and stable storage volume. When a pod is rescheduled, its storage is re-attached to the new pod, so it retains its state.

[

![](https://substackcdn.com/image/fetch/$s_!mXSK!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F84d9c916-4bdc-49dc-95e8-98868bdf9258_812x458.png)



](https://substackcdn.com/image/fetch/$s_!mXSK!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F84d9c916-4bdc-49dc-95e8-98868bdf9258_812x458.png)

> _PV and PVC will be discussed later._

Scaling is performed in a controlled and ordered manner. When scaling up, pods are created one by one in increasing ordinal order. When scaling down, pods are terminated in reverse order. This strict ordering is critical for many stateful applications.

[

![](https://substackcdn.com/image/fetch/$s_!3ooP!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff072ef4a-fb42-4598-88e6-0ce4356389a0_854x388.png)



](https://substackcdn.com/image/fetch/$s_!3ooP!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Ff072ef4a-fb42-4598-88e6-0ce4356389a0_854x388.png)

Imagine we’re running a PostgreSQL primary-replica setup (one accepts writes and one handles data replication) with a Deployment abstraction. We could have the following issues:

- **Identity Loss**: When a pod fails and is recreated, it gets a new, random name and IP address. The replicas wouldn't know how to connect to the primary, and the primary wouldn't be able to track its replicas.
    
- **No Persistent State**: Each pod would get new, empty storage when it's rescheduled. The new pods would not have the database's data, making replication impossible.
    
- **No Ordering**: You can't guarantee that the primary starts before the replicas, which is essential for setting up the replication links.
    

A StatefulSet is helpful here:

- **Stable Names**: Each pod is named postgres-0, postgres-1, etc.
    
- **Stable Storage**: Each pod is tied to a specific PersistentVolumeClaim (PVC), so its data persists across pod restarts and rescheduling.
    
- **Ordered Scaling**: The pods are created and deleted in order. (e.g., postgres-0—the primary, will always start first.)
    

_**Note**: Deployment and StatefulSet are not the only abstractions for managing pods; there are others, such as DaemonSet or Job._

## Services

Services provide a stable network endpoint for a set of Pods. This is important because Pods are ephemeral; their IP addresses can change as they are created or replaced.

[

![](https://substackcdn.com/image/fetch/$s_!b6X_!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbc841b9a-8f87-4bd8-847e-9592d444164e_994x500.png)



](https://substackcdn.com/image/fetch/$s_!b6X_!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbc841b9a-8f87-4bd8-847e-9592d444164e_994x500.png)

Services ensure that other applications or clients can reliably communicate with your Pods, even as the underlying Pods are added and removed. They also handle load balancing and distribute incoming traffic across multiple pods to ensure even load distribution and high availability.

## ConfigMaps and Secret

As a data engineer, you often need to configure your applications differently depending on the environment—whether it's dev, staging, or production. **ConfigMaps** in Kubernetes let you decouple configuration data from your application code, making it easier to manage and update configurations dynamically.

[

![](https://substackcdn.com/image/fetch/$s_!RCZI!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F67af18d6-a957-43ca-abc2-a177dd02a356_622x384.png)



](https://substackcdn.com/image/fetch/$s_!RCZI!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F67af18d6-a957-43ca-abc2-a177dd02a356_622x384.png)

For example, you can store the application’s feature flags in a ConfigMap and inject them into the Pods as environment variables. This approach makes your applications more portable and reduces the need to rebuild container images when configuration changes.

It is crucial to securely manage sensitive data, such as passwords, API keys, and certificates. Kubernetes provides **Secrets**, which allow us to store sensitive information separately from application code, and ConfigMaps. Secrets are stored in a base64-encoded format and can be mounted into Pods as files or exposed as environment variables.

## Persistent Volumes and Persistent Volume Claims

While Pods are ephemeral, some applications require persistent storage that outlives a Pod's lifecycle. Kubernetes provides **Persistent Volumes (PV)** and **Persistent Volume Claims (PVC)** to manage this aspect.

[

![](https://substackcdn.com/image/fetch/$s_!mumE!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9cc4d738-353a-46a2-b764-2f1740b99a84_870x502.png)



](https://substackcdn.com/image/fetch/$s_!mumE!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9cc4d738-353a-46a2-b764-2f1740b99a84_870x502.png)

- **Persistent Volumes (PV)**: A Persistent Volume is a piece of storage in the cluster that has been provisioned manually by the user or dynamically via storage classes. It is independent of the Pod that uses it, allowing storage to persist even if the Pod is destroyed. PVs abstract the underlying storage infrastructure, which can range from local disk storage to cloud-based solutions, such as AWS EBS, Google Persistent Disks, or NFS.
    
- **Persistent Volume Claims (PVC)**: A Persistent Volume Claim is a request from a Pod for persistent storage. It specifies the size, access modes, and sometimes the storage class needed. When a PVC is created, Kubernetes binds it to an available PV that meets the requested criteria. This decouples storage management from application deployment, allowing developers to request storage resources without needing to understand the underlying infrastructure.
    

## Why So Many Abstractions?

As you can see, there are different abstractions to manage various aspects of an application. The reason for this approach lies in its design philosophy, which emphasizes the separation of concerns.

This modular approach facilitates the management of complex applications. By isolating concerns, Kubernetes enables you to independently scale, secure, and update different parts of the application, or deploy an application in the most suitable way.

For example, you can update configurations without redeploying applications, use StatefulSet when you need the statefulness, restart the pod without worrying about losing data in the PV, or not care about the IP of the pod, as Service will handle the connection to the Pod for us.

## How to run an application on Kubernetes

Running an application on k8s can be done in several ways. The most common methods are by directly applying YAML manifest files and by using a package manager, such as Helm, for more complex or reusable deployments.

The Manifest File describes the desired state of our application. Which abstraction we use, how many pods, each pod's resources, the disk configuration, the container’s image, the environment variable we want to input, or the credentials we want to protect.

After having all the necessary manifest files, we submit them to the Kubernetes cluster. We can define each resource in a separate YAML file. The Deployment, Service, ConfigMap, Secret, PV, and PVC can each have their own YAML file.

Here is an example of a Deployment definition ([source](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)):

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Here is an example of a PV definition ([source](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)):

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc
    namespace: foo
  ...
```

For complex applications, Helm streamlines the process by bundling all necessary YAML manifest files into a single, reusable package called a Chart. Helm charts utilize a templating engine to make manifests configurable and dynamic.

Imagine you need five different manifest files to deploy an application. Without Helm, when you need to deploy a similar application with some changes, you copy those five manifest files, edit them as needed, and submit them to k8s. Helm Chart addresses this problem by allowing you to templatize the manifest files and specify placeholders, enabling customization and reuse.

## Make it more fun by running a Spark application

> _Yayyy, the section I am looking for._

### Revisiting

Before running some commands, it’s crucial to revisit some Spark fundamentals, especially when working with the Spark cluster. Technically, there are two distinct “clusters”; the first is the Spark cluster:

[

![](https://substackcdn.com/image/fetch/$s_!HaHQ!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd88e701b-0b7f-4cfb-a7f9-ef49fbdba1a6_458x410.png)



](https://substackcdn.com/image/fetch/$s_!HaHQ!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd88e701b-0b7f-4cfb-a7f9-ef49fbdba1a6_458x410.png)

- **Driver:** This JVM process manages the entire Spark application, from handling the user application, planning the execution, to distributing tasks to the executors.
    
- **Executors:** These processes execute tasks the driver assigns and report their status and results. Each Spark application has its own set of executors.
    

Each Spark cluster is associated with a Spark application. The Spark cluster must run somewhere, physically. And, it is the job of the resource cluster. This cluster is a set of physical servers that are managed by the cluster manager. Spark can work with multiple resource clusters, one of which is the system we focus on in this article: Kubernetes.

### Spark on Kubernetes at a high level

When we submit a Spark application, we send the request to the k8s API server. Like any Spark submit command, the user can include configurations such as driver/executor resources.

[

![](https://substackcdn.com/image/fetch/$s_!wwBj!,w_2400,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F70762537-fa1c-4b2b-b929-6f526089f285_1328x660.png)



](https://substackcdn.com/image/fetch/$s_!wwBj!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F70762537-fa1c-4b2b-b929-6f526089f285_1328x660.png)

If the deploy mode is cluster, a driver pod will be initiated in the Kubernetes cluster. Then, the driver asks the Kubernetes cluster (which now acts as the cluster manager) to launch the executors based on the input configurations. If the application is run in client mode, it is our responsibility to ensure that the [executor pods from k8s can connect to the driver process on our side](https://spark.apache.org/docs/latest/running-on-kubernetes.html#client-mode-networking).

> _Spark has different modes of execution, which are distinguished by the location of the driver:_
> 
> - _**Cluster Mode:** The driver process is launched on the resource cluster alongside the executor processes in this mode._
>     
> - _**Client Mode:** The driver remains on the client machine. This setup requires the client machine to maintain the driver process throughout the application’s execution._
>     

What happens next will be a typical Spark process; the application’s logic is planned and scheduled by the driver, and the physical execution will be handed over to the executors.

After the application finishes, k8s terminates and cleans up the executor pods. The driver pod is persisted in the “completed” state until it’s manually cleaned up. This helps the user when debugging the application, as the log is retained in the driver.

### The hands-on

First things first, we need a Kubernetes cluster. Self-deploying a real Kubernetes locally or renting from a cloud vendor is expensive. An alternative solution is [minikube. It could help us set up a local Kubernetes cluster on macOS, Linux, and Windows](https://minikube.sigs.k8s.io/docs/). Based on my observation, minikube will launch a Docker container that includes the minimal setup of the Kubernetes cluster. [Please review the document for the installation guide specific to your platform](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download).

When you successfully start minikube, the Kubernetes command-line tool, kubectl, will also be installed and configured to work with the cluster that Minikube starts. I almost forgot something: in Kubernetes, there is a concept of a namespace to help with the logical grouping of resources on the same cluster. When users don’t specify the namespace, the `default` namespace will be used. In the scope of this exercise, the `default` namespace is enough.

To run a Spark application on k8s, we need to do three things:

- Retrieve the Spark image: Recall that the application running on k8s must be containerized. We need a Spark image.
    
- Know where to submit: We need to know the details of the K8s api server so we can specify when submitting.
    
- RBAC: With a k8s cluster that enabled RBAC (it’s true for the minikube cluster), we need to create the service account with the required roles so we can tell the k8s to create the driver and executor pods.
    
- An application to run.
    

#### Retrieve the Spark image

For the image, Spark provides a script to build and push the image to our repo. You can download the [Spark package](https://spark.apache.org/downloads.html), unzip it, and navigate to that folder. Then, you can run these two commands (make sure you have Docker installed and running):

```
./bin/docker-image-tool.sh -r <repo> -t <tag> build
./bin/docker-image-tool.sh -r <repo> -t <tag> push
```

If they run successfully, you will have the image with the identifier like this in your Docker repo:

```
<repo>/spark:<tag>
```

#### Know where to submit

The next step is to obtain the Kubernetes API endpoint. After running the minikube successfully, open the terminal, and run:

```
❯ kubectl cluster-info
Kubernetes control plane is running at https://127.0.0.1:64371
```

In my case, the result is https://127.0.0.1:64371

> _**Note**: the port is not always 64371_

#### RBAC

We need to create a Kubernetes service account with sufficient roles. You can run the following roles, which create a service account named `spark` on the `default` namespace.

```
kubectl create serviceaccount spark
kubectl create rolebinding spark-edit --clusterrole=edit --serviceaccount=default:spark --namespace=default
```

The service account is then assigned an edit role. We will specify this role when we submit our application.

> _**Note:** For a real-life setup, the edit role is not always the ideal role. Please grant the role based on your needs and ensure it follows [the principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege)._

#### An application

For the application, we will run the existing Spark example in the folder ./examples/jars inside the Spark package we’ve just downloaded.

#### Submit the application

Now, we have all the required ingredients. Let’s submit the Spark application; stand inside the Spark package folder, and run this (please fill in the placeholder with your input):

```
./bin/spark-submit \
  --master k8s://<your-k8s-api-endpoint> \
  --deploy-mode cluster \
  --name spark-pi \
  --class org.apache.spark.examples.SparkPi \
  --conf spark.kubernetes.container.image=<repo>/spark:<tag> \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
  local:///opt/spark/examples/jars/spark-examples_2.13-4.0.0.jar
```

You can validate it is actually run by running:

```
kubectl get pods
```

…to check for running pods. At first, you will see the driver pod is up and running:

[

![](https://substackcdn.com/image/fetch/$s_!hclH!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc8eb3569-7fc7-45b9-ad47-8272805f272e_1002x90.png)



](https://substackcdn.com/image/fetch/$s_!hclH!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc8eb3569-7fc7-45b9-ad47-8272805f272e_1002x90.png)

After that, the executor pods are created:

[

![](https://substackcdn.com/image/fetch/$s_!7d9C!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F59ebf868-4e31-4cf6-aa63-7c6a247e366c_1112x178.png)



](https://substackcdn.com/image/fetch/$s_!7d9C!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F59ebf868-4e31-4cf6-aa63-7c6a247e366c_1112x178.png)

After the application is finished, the executor pods are garbage collected, and only the driver pod persists in the “Completed“ status:

[

![](https://substackcdn.com/image/fetch/$s_!6S9V!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F18dcc02b-2433-448a-81a9-d003643ec45e_994x100.png)



](https://substackcdn.com/image/fetch/$s_!6S9V!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F18dcc02b-2433-448a-81a9-d003643ec45e_994x100.png)

And that’s it —you've run a Spark application on Kubernetes. When working with PySpark, things are not much different; all you need to do is pass the proper configuration. Here is the code for my SparkSession object, which also requires the Kubernetes API server, the service account, and the Spark image.

[

![](https://substackcdn.com/image/fetch/$s_!LuGw!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1109f081-b4e6-4497-bfd3-74b91bc14008_1008x302.png)



](https://substackcdn.com/image/fetch/$s_!LuGw!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F1109f081-b4e6-4497-bfd3-74b91bc14008_1008x302.png)

### A different way to run a Spark application on k8s

You might wonder, “Why can’t we run a Spark application like a regular k8s application with the YAML files?“

In the past, we couldn’t do this as Spark applications don't use standard Kubernetes abstractions like Deployments or StatefulSets, because their architecture is fundamentally different from that of a typical long-running microservice.

Spark jobs are transient, and their components—the driver and executors—have a unique, dynamic lifecycle. Later, the community allowed us to run Spark via the YAML files by introducing the [Spark Operator](https://www.kubeflow.org/docs/components/spark-operator/overview/).

An Operator in k8s extends the functionality of the Kubernetes API to automate the lifecycle of applications. In other words, you can create your own custom abstraction, in addition to the existing ones, such as Deployment and StatefulSet. An Operator works by defining two things:

- Custom Resource Definitions (CRDs): A CRD is an extension of the Kubernetes API that allows you to define new types of resources. For example, with Spark Operator, we have a new resource called SparkApplication.
    
- Custom Controllers: The Operator's custom controller watches for changes to the custom resources defined by the CRD. When it detects a change, it takes action to ensure the cluster's actual state aligns with its desired state. The application’s domain knowledge can also be implemented in custom controllers.
    

With Spark Operator, you can declaratively define how a Spark application looks on the k8s cluster by editing the YAML manifest files:

```
apiVersion: 'sparkoperator.k8s.io/v1beta2'
kind: SparkApplication
...
```

---

## Outro

In this article, we first revisit the concept of containers, and then we delve into Kubernetes, answering what it is and how it works. Next, we explore common abstractions, including ways to instruct Kubernetes to deploy our application using manifest files or Helm charts.

Finally, we run a toy Spark application on the local Kubernetes cluster, which is run by minikube. We also explore the option, in addition to the traditional Spark submit approach, of running Spark applications in a more Kubernetes-native manner by leveraging the Spark Operator.

Thank you for reading this far. See you next time.