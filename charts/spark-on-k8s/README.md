# openshift-spark-on-k8s

## What is Spark

Apache Spark is an open-source distributed computing framework designed to process vast datasets at scale. It utilizes Resilient Distributed Dataset (RDD) as its core data structure, enabling parallel processing across multiple executors. RDDs are fault-tolerant, with the capacity to regenerate lost data using lineage—a sequence of transformations applied to the data.

## High Level Spark Components

- **Driver** - Orchestrates the Spark application. It initializes the SparkContext, coordinates task execution across executors, and returns results.
- **Master** - In standalone cluster mode, the Master is responsible for resource allocation, monitoring worker nodes, and coordinating the distribution of Spark applications.
- **Worker** - These are nodes in the cluster designated for executing tasks. They provide the necessary resources and host executor processes.
- **Executor** - JVM processes located on worker nodes. Executors run tasks, store intermediate and cached data locally, and communicate results back to the Driver.

## About Spark on Kubernetes

Apache Spark's integration with Kubernetes facilitates distributed data processing tasks in a Kubernetes ecosystem. By tapping into Kubernetes' container orchestration prowess, Spark adeptly manages and schedules its driver and executor pods. This ensures optimal resource allocation, dynamic scaling, and streamlined deployment in cloud-native environments.

Within this setup, the Spark driver can either coexist with worker pods in the same Kubernetes namespace or function within interactive platforms and workflow orchestrators like Jupyter Notebook and Apache Airflow.

When operating Spark drivers in client mode on platforms like Jupyter Notebook or Apache Airflow, the driver begins on a port specified by spark.driver.port. Should that port be occupied, the driver sequentially searches for the next open port. For example, if spark.driver.port is set to 4041 and it's occupied, the driver will try 4042, then 4043, continuing until it identifies an available port.

Initiating a Spark driver entails allocating CPU and memory resources. In situations where Apache Airflow DAGs (Directed Acyclic Graphs) launch concurrent Spark jobs, several drivers might start simultaneously. Such concurrent operations could lead to abrupt surges in CPU and memory usage, potentially exhausting the cluster's resources.

Scaling solutions include:

- **Auto-scaling Airflow Workers**: A method allowing Airflow workers, responsible for running the Spark drivers, to auto-scale depending on the workload.
- **Using Cluster Mode**: 
	- **Cluster Mode** - Here, driver pods are created within the same Kubernetes namespace as the worker pods. 
	- **Client Mode** - This method is especially potent when the Spark driver is set to client mode and Kubernetes orchestrates the Spark worker pods.

### Reducing Risk

**Outside of Cluster**: When Spark drivers operate outside of the cluster, they do not benefit from pod-to-pod communication and they compete for port availability. In such setups, it's crucial to ensure a range of ports are open based on the anticipated volume of concurrently running drivers. 

**Workflow Orchestrator**: When running in the cluster on a workflow orchestrator like apache airflow, issues can arise if a driver encounters an error and ends up in a stalled or "zombie" state. Such occurrences can monopolize resources, hinder other ongoing tasks, retain active DAGs, and might even crash the worker.

**Cluster Mode**: On the other hand, when drivers operate in `cluster` mode and are deployed in their own separate pods, several risks, such as port contention and zombie states, are substantially minimized. In this mode, each driver obtains a unique IP and typically employs the `spark.driver.port` for communication with worker pods. Furthermore, by segregating Spark's driver and worker pods from the Airflow workers, competition for system resources is also diminished.

## Other Considerations

- **Bandwidth** - Network saturation can occur, varying based on projects.
- **Storage** - Azure Blobs Gen2 with ABFSS drivers are recommended over Gen1 with WASB drivers. Azure Blob Storage is cost-effective and managed but might introduce latency. HDFS, while integrated with big data tools and potentially faster, introduces operational complexity and costs.
- **Logging and Monitoring** - Centralized monitoring and logging are crucial for ephemeral Spark entities.
- **Namespace isolation** - Use Kubernetes namespaces for isolation especially when you have different teams or projects sharing the same Kubernetes cluster.
- **Integration with Other Data Services** - Often, Spark isn't working in isolation. Consider how it integrates with databases, data lakes, message queues, and other services, both from a performance and a security perspective.

## Cluster vs Client Deployment Mode

- Cluster Deployment Mode

In this configuration, both workflow orchestrators possess the ability to spawn Spark drivers within the cluster, independent of the orchestrator itself. These drivers take on the responsibility of coordinating Spark job executions.

Throughout the execution phase of a job, every executor (operating within its own JVM) inside a worker pod has the capability to write concurrently to Azure Blob Storage. It's important to note that each of these parallel writes corresponds directly to a distinct dataframe partition. Before the write operation takes place, these partitions may either be coalesced or repartitioned to form a unified dataframe.

A significant advantage of this architecture is the placement of the Spark driver within the same namespace as its corresponding workers. This not only ensures that potential risks linked with a worflow orchestrator are effectively mitigated but also guarantees a more secure and stable environment for the execution of Spark jobs.

- Client Deployment mode

In client mode, Spark drivers are initiated by a workflow orchestrator, running on the same machine from where the job is launched. Although the driver operates locally, tasks are dispatched for parallel execution across cluster executors. Each executor, in its JVM, can write concurrently to Azure Blob Storage, with every write reflecting a dataframe partition.

The client mode offers distinct benefits when integrated with workflow orchestrators. Post Spark job execution, the orchestrator can directly access job results, facilitating post-processing, logging, or alerting. The Spark driver's local presence permits real-time feedback, easing debugging and monitoring processes. Additionally, the SparkSession's state remains persistent, allowing consecutive operations or caching. Nevertheless, this approach may consume significant machine resources in the orchestrator, and disruptions on the orchestrator side could impact active Spark jobs.

## Version Management

Apache Spark regularly releases new versions, often with enhanced features, bug fixes, and performance improvements. Managing these versions effectively is crucial to ensure the stability and performance of Spark applications. Here are some key considerations:

- **Version Bundling**: Apache Spark releases often come as bundles that need to be built using command-line tools. Ensure compatibility among the components; for example, the versions of Python and PySpark should align.

- **Workflow Orchestration Compatibility**

	- **Cluster Mode**: If the Spark driver is run inside the cluster, it decouples the Spark environment from the Airflow environment. This means that the specific version of PySpark or other Spark-related dependencies within Airflow become less critical, as the driver within the cluster can have its own set of dependencies.
  
	- **Client Mode**: Whatever workflow orchestrator you use, like Apache Airflow, needs to be compatible with the chosen Spark version. Regularly review and test updates in a controlled environment before rolling them out widely.

- **Backward Compatibility**: Older Spark jobs or applications might have dependencies on previous Spark versions. It's important to ensure that newer versions don't break existing functionalities.

## Integration with CI/CD

Continuous Integration and Continuous Deployment (CI/CD) streamline the development and deployment processes, ensuring that applications are robust, up-to-date, and efficiently managed.

## Script Handling `(Cluster mode)

```
spark-submit \ 
--master k8s://<KUBERNETES_API_ENDPOINT> \ 
--deploy-mode cluster \ 
--name my-spark-job \ 
--class org.apache.spark.examples.SparkPi \ 
--conf spark.executor.instances=5 \ 
--conf spark.kubernetes.container.image=my-spark-image \ 
--conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \ 
--conf spark.hadoop.fs.s3a.access.key=<YOUR_S3_ACCESS_KEY> \ 
--conf spark.hadoop.fs.s3a.secret.key=<YOUR_S3_SECRET_KEY> \ 
--conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \ 
```

- Loaded Dynamically from Object Storage

```
--py-files s3a://my-bucket/path/to/dependencies.zip \ 
s3a://my-bucket/path/to/main_script.py
```

- Passing Script from Workflow Orchestrator

`main_scipt.py`