
## Intro

To resume the Apache Spark series, we will explore how Spark schedules the data processing for us this week.

The article begins with the anatomy of a Spark job. Then, we will explore the overview of a typical scheduling process and its related components and concepts.

---

## Jobs, Stages, and Tasks

[

![](https://substackcdn.com/image/fetch/$s_!JRcV!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Febd9d6e7-d8ef-4ace-85a7-8f6c738c7e04_1644x952.png)



](https://substackcdn.com/image/fetch/$s_!JRcV!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Febd9d6e7-d8ef-4ace-85a7-8f6c738c7e04_1644x952.png)

Image created by the author.

- **Job:** In Spark, a job represents a series of transformations applied to data. It encompasses the entire workflow from start to finish. A single Spark application can have more than one Spark job.
    
- **Stage:** A stage is a job segment executed without data shuffling. Spark splits the job into different stages when a transformation requires shuffling data across partitions. Speaking of transformations, there are two categories we need to explore:
    
    [
    
    ![](https://substackcdn.com/image/fetch/$s_!JWcf!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9dbeb15f-7bf2-4c9c-abc5-03acc286aad4_1352x892.png)
    
    
    
    ](https://substackcdn.com/image/fetch/$s_!JWcf!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F9dbeb15f-7bf2-4c9c-abc5-03acc286aad4_1352x892.png)
    
    Image created by the author.
    
    - Transformations with **narrow dependencies** are those where each partition in the child RDD has a limited number of dependencies on partitions in the parent RDD. These partitions may depend on a single parent (e.g., the map operator) or a specific subset of parent partitions known beforehand (such as with coalesce). This means that operations like map and filter do not require data shuffling. RDD operations with narrow dependencies are pipelined into one set of tasks in each stage.
        
    - Transformations with **wide dependencies** require data to be partitioned in a specific way, where a single partition of a parent RDD contributes to multiple partitions of the child RDD. This typically occurs with operations like groupByKey, reduceByKey, or join, which involve shuffling data. Consequently, wide dependencies result in stage boundaries in Spark's execution plan.
        
- **Task:** A task is the smallest unit of execution within Spark. Each stage is divided into multiple tasks, which execute processing in parallel across different data partitions.
    
- **DAG:** In Spark, the DagScheduler (more on this later) uses RDD dependencies to build a Directed Acyclic Graph (DAG) of stages for a Spark job. The DAG ensures that stages are scheduled in topological order.
    

The following section will provide an overview of the scheduling process.

---

## The scheduling process

When we submit a Spark application, the SparkContext is first initialized

> _SparkContext is the entry point to all Spark’s components._

### SparkContext

The SparkContext then initializes the TaskScheduler (for task-oriented scheduling) and the SchedulerBackend (which interacts with the cluster manager and provides resources to the TaskScheduler). After that, the DAGScheduler (for stage-oriented scheduling) is created.

[

![](https://substackcdn.com/image/fetch/$s_!Ywm-!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5220bea0-c0c6-44f8-b264-ec45a963684b_1490x804.png)



](https://substackcdn.com/image/fetch/$s_!Ywm-!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5220bea0-c0c6-44f8-b264-ec45a963684b_1490x804.png)

Image created by the author.

> To celebrate Lunar New Year (the true New Year holiday in Vietnam), I’m offering _**50% off the annual subscription**_. The offer ends soon; grab it now to get full access to nearly 200 high-quality data engineering articles.
> 
> [50% off the annual subscription](https://vutr.substack.com/subscribe)

### DagScheduler

The scheduling process begins with the DAGScheduler building the DAG based on the dependencies between RDD objects.

The DAGScheduler traverses the RDD lineage from the final RDD (with action) back to the source RDD, building up a **DAG of stages** based on shuffle boundaries. Stages are formed where wide dependencies (shuffle boundaries) exist. Each stage consists of parallel tasks that can be executed on different partitions. Stages are created as [ResultStages](https://books.japila.pl/apache-spark-internals/scheduler/ResultStage/) (final stage) or [ShuffleMapStages](https://books.japila.pl/apache-spark-internals/scheduler/ShuffleMapStage/) (intermediate stages that perform shuffles).

[

![](https://substackcdn.com/image/fetch/$s_!eU2N!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6d620b07-258d-42c4-a94e-6f405c22345e_1692x462.png)



](https://substackcdn.com/image/fetch/$s_!eU2N!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F6d620b07-258d-42c4-a94e-6f405c22345e_1692x462.png)

Image created by the author.

The stage is then scheduled according to the DAG's topological order. Each stage is submitted once all its "parent stages" (upstream dependencies) are completed. The DAGScheduler handles failures due to lost shuffle output files, in which previous stages may need resubmit. Failures in a stage not caused by shuffle file loss are handled by the TaskScheduler, which will retry each task several times before canceling the whole stage.

[

![](https://substackcdn.com/image/fetch/$s_!O0v7!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd1c8c6e0-3ba9-4252-a300-9b32b2694af3_1702x706.png)



](https://substackcdn.com/image/fetch/$s_!O0v7!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fd1c8c6e0-3ba9-4252-a300-9b32b2694af3_1702x706.png)

An example of the stage scheduler order. Image created by the author.

The DAGScheduler creates a TaskSet for each stage. A TaskSet includes fully independent tasks of a stage that are uncomputed.

Then, TaskSet is sent to the TaskScheduler for execution. In addition, the DAGScheduler determines the preferred locations to run each task based on the current cache status and sends these to the TaskScheduler.

[

![](https://substackcdn.com/image/fetch/$s_!FYYG!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F50b0c5a8-5007-4282-863f-bc4b4bee018a_1544x698.png)



](https://substackcdn.com/image/fetch/$s_!FYYG!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F50b0c5a8-5007-4282-863f-bc4b4bee018a_1544x698.png)

Image created by the author.

> _**Cache tracking**: the DAGScheduler detected with RDDs are cached to avoid recomputing them and remembers which shuffle map stages have produced which output files to avoid duplicate process._
> 
> _**Preferred locations**: the DAGScheduler also computes where to run each task in a stage based on the preferred locations of its underlying RDDs, or the location of cached or shuffle data._

### TaskScheduler

Next, the process continues with the TaskScheduler. The TaskScheduler is responsible for scheduling tasks (in the TaskSet received from the DAGScheduler) on available executors. It interacts with a SchedulerBackend to schedule tasks across various cluster types. The SchedulerBackend is started and stopped as part of the TaskScheduler’s initialization and stopping process.

In more detail, when the DAGScheduler submits a TaskSet to the TaskScheduler, the TaskScheduler registers a new TaskSetManager and requests the SchedulerBackend to handle resource allocation offers. (More on this later when we discuss the SchedulerBackend.)

[

![](https://substackcdn.com/image/fetch/$s_!Olkb!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa8086802-229c-4e5f-953f-e03df7e9af18_1684x848.png)



](https://substackcdn.com/image/fetch/$s_!Olkb!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fa8086802-229c-4e5f-953f-e03df7e9af18_1684x848.png)

Image created by the author.

The TaskSetManager schedules the tasks in a single TaskSet. It keeps track of each task, retries tasks if they fail (up to a limited number of times), and handles locality-aware scheduling (using each task’s locality preference obtained from the DagScheduler). TaskSetManger tries to assign tasks to executors as close to the data as possible. There are several data locality types (nearest to farthest):

- **PROCESS_LOCAL**: Task runs on the same executor where the data resides.
    
- **NODE_LOCAL**: The task runs on the same node as the data but on a different executor.
    
- **NO_PREF** data is accessed equally quickly from anywhere and has no local preference.
    
- **RACK_LOCAL**: The task runs on the same rack but on a different node.
    
- **ANY**: The task can run on any executor when no locality preferences are satisfied.
    

TasksetManager tries to achieve local locality-aware scheduling for a TaskSet by leveraging delay scheduling. This optimization technique has a simple idea: if a task cannot be scheduled on an executor with the desired locality level, TasksetManager will wait a short period before scheduling the task.

[

![](https://substackcdn.com/image/fetch/$s_!akWD!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5ed36cbe-01b1-44df-8af9-bfe60965e7b0_1746x1208.png)



](https://substackcdn.com/image/fetch/$s_!akWD!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F5ed36cbe-01b1-44df-8af9-bfe60965e7b0_1746x1208.png)

Image created by the author. [Reference](https://youtu.be/rpKjcMoega0?t=1169)

When a task is ready to be scheduled, it first checks if an available executor has the desired level of data locally. If no executors satisfy the desired locality level, TaskSetManager doesn't immediately assign the task. Instead, it delays the task for a short, configurable amount of time, hoping that a local executor will become available during that delay. If, after the delay, no satisfied executor becomes available, TaskSetManager will launch a task on an executor that has a more “relaxed” data locality.

There are cases when some tasks might take longer than other tasks (e.g., due to hardware problems). In the TaskSetManager, there is a health-check procedure called _Speculative execution of tasks_ (enabled by setting _**spark.speculation = true**_ )that checks for tasks to be _speculated_. Such slow tasks will be re-submitted to another executor. (This means that issues caused by hardware problems can be mitigated with speculative execution.)

[

![](https://substackcdn.com/image/fetch/$s_!kLAA!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F67dfe8c6-b4aa-4615-8a4e-eda53c7edebf_1114x646.png)



](https://substackcdn.com/image/fetch/$s_!kLAA!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F67dfe8c6-b4aa-4615-8a4e-eda53c7edebf_1114x646.png)

Image created by the author.

TaskSetManager will not stop the slow tasks but launch a copy of that task in parallel. The first copy of the task that is completed successfully will be used, and other copies will be killed.

### SchedulerBackend

As mentioned, the SchedulerBackend is started and stopped as part of the TaskScheduler’s initialization and stopping process. At first, the SchedulerBackend requests executors from the cluster manager, which then launches executors based on the application's requirements. Once started, the executors attempt to register with the SchedulerBackend through an RPC endpoint. If successful, the SchedulerBackend receives a list of the application's desired executors.

[

![](https://substackcdn.com/image/fetch/$s_!DdUq!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe00ebdc7-533d-4a56-bb15-25088dbb5d5d_1408x828.png)



](https://substackcdn.com/image/fetch/$s_!DdUq!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fe00ebdc7-533d-4a56-bb15-25088dbb5d5d_1408x828.png)

Image created by the author.

When the TaskScheduler creates the TaskSetManager, it requests resources from the SchedulerBackend to schedule the tasks. Based on the list of active executors, the SchedulerBackend retrieves WorkerOffers, each representing an executor's available resources.

> _Based on the Spark source code, active executors are those that are registered and are not pending removal, have not been lost without reason, and are not being decommissioned._

Then, the SchedulerBackend passes the WorkerOffers back to the TaskScheduler. The TaskScheduler assigns tasks from the TaskSet to the resources from the WorkerOffers, resulting in a list of task descriptions. These task descriptions are then returned to the SchedulerBackend, which launches tasks based on this task description list.

[

![](https://substackcdn.com/image/fetch/$s_!6dES!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8999cd8b-a421-4b52-b7b1-5400f7b31577_1430x1016.png)



](https://substackcdn.com/image/fetch/$s_!6dES!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8999cd8b-a421-4b52-b7b1-5400f7b31577_1430x1016.png)

Image created by the author.

For each entry in this list, the SchedulerBackend serializes the task description. Additionally, it pulls the executor ID assigned to the task from the entry and uses this ID to retrieve information for that executor (e.g., hostname, cores, executor address, executor endpoint).

Finally, the SchedulerBackend sends the serialized task descriptions to the executor endpoints.

### **Task Execution on Executors**

When receiving a serialized task description from the SchedulerBackend, the executor deserializes the task description and begins launching the task using the information provided.

During its lifecycle, the executor runs user-defined code, reads data from local or remote storage, performs computations, and writes out intermediate results, such as shuffle data.

### Things go on

The process continues until all stage tasks are finished, with stages being processed in the DAG order. A Spark job is considered complete when all stages have finished.

Before we exit, let's review the typical end-to-end Spark scheduling process

[

![](https://substackcdn.com/image/fetch/$s_!y0o-!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F128ac1e0-6a1f-4341-a148-8d9501c32e79_1374x1160.png)



](https://substackcdn.com/image/fetch/$s_!y0o-!,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F128ac1e0-6a1f-4341-a148-8d9501c32e79_1374x1160.png)

Image created by the author.

---

## Outro

Thank you for reading this far.

We’ve just explored the anatomy of a Spark job and walked through the Spark scheduling process, from RDD dependencies to how tasks are assigned to the executors.

Hope I can bring some value.

If you notice I missed something, feel free to let me know.

Now it’s time to say goodbye; see you on my next blog!

---

## Reference

_[1] Xingbo Jiang, [Deep Dive into the Apache Spark Scheduler](https://www.youtube.com/watch?v=rpKjcMoega0) (2018)_

_[2] Holden Karau, Rachel Warren, [High Performance Spark: Best Practices for Scaling and Optimizing Apache Spark](https://www.amazon.com/High-Performance-Spark-Practices-Optimizing/dp/1491943203) (2017)_

_[3] [Spark GitHub Repo](https://github.com/apache/spark)_

_[4] Mounika Tarigopula, [Understanding speculative execution](https://kb.databricks.com/scala/understanding-speculative-execution) (2022)_

_[5] Jacek Laskowski, [The Internals of Spark Core](https://books.japila.pl/apache-spark-internals/)_

_[6] Mallikarjuna_g Gitbooks, [CoarseGrainedSchedulerBackend](https://mallikarjuna_g.gitbooks.io/spark/content/spark-overview.html)_

_[7] Bimalendu Choudhary, [Starting with Spark code](https://bimalenduc.medium.com/starting-with-spark-code-46093c976a4) (2021)_