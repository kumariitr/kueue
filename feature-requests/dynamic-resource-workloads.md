# Dynamic Resource Workload Support in Kueue

## Context
Workloads in batch compute normally tend to be dynamic in nature from a resourcing perspective. When we talk about controllers to run these workloads on Kubernetes, these controllers should support the scheduling of these dynamic workloads. Currently Kueue supports workloads with static resource requirements, i.e. workloads for which resource requirements are decided at the time of creation and later can not be changed.

## Objective
This document explains a high level approach for supporting dynamic resource workloads in Kueue. 
It’s not an engineering design document. A detailed engineering design document will follow after agreement on the high level approach (same doc can be further extended to a design doc or a new one will be written).

## Use Cases
Batch job processing technologies like Spark and Ray mainly depend on operators to schedule and manage workloads to run applications on Kubernetes. These operators interact with the Kubernetes apiserver to create and manage the life cycle of the applications. Also, underlying layers like Spark (Driver) and RayCluster (AutoScaler) also interact with the Kubernetes apiserver for further updates related to resource requirements.

### Spark
Kubernetes Operator for Spark
In Spark 2.3, Kubernetes becomes an official scheduler backend for Spark, additionally to the standalone scheduler, Mesos, and Yarn. Compared with the alternative approach of deploying a standalone Spark cluster on top of Kubernetes and submitting applications to run on the standalone cluster, having Kubernetes as a native scheduler backend offers some important benefits as discussed in SPARK-18278 and is a huge leap forward. However, the way life cycle of Spark applications are managed, e.g., how applications get submitted to run on Kubernetes and how application status is tracked, are vastly different from that of other types of workloads on Kubernetes, e.g., Deployments, DaemonSets, and StatefulSets. The Kubernetes Operator for Apache Spark reduces the gap and allows Spark applications to be specified, run, and monitored idiomatically on Kubernetes.

### Ray Cluster
Ray Kubernetes Operator
KubeRay operator makes deploying and managing Ray clusters on top of Kubernetes painless - clusters are defined as a custom RayCluster resource and managed by a fault-tolerant Ray controller. The Ray Operator is a Kubernetes operator to automate provisioning, management, autoscaling and operations of Ray clusters deployed to Kubernetes.

## Approach
To Support spark workloads in Kueue, we mainly need to watch, validate and mutate (?) two types of K8s objects:
Initial application objects created by the operator (e.g. SparkApplication, ScheduledSparkApplication, RayCluster).
Worker / Executor pods created by the underlying processing layer itself (e.g. executor pod created by Spark driver and Worker Node pods created by Autoscaler in Ray Cluster).
Given these objects are also watched by the controllers in the operator, it’s important to intercept these objects from a capacity perspective before even they are created. Keeping this in mind, following proposal lists the steps for our approach:
Implement an admission controller for the Application CRD objects. This controller will be implemented in Kueue as DynamicWorkloadController (which will cover Spark, Ray and other dynamic capacity workload use cases). 
This controller will also create a DynamicWorkload object in Kueue corresponding to the application objects. This DynamicWorkload object will have the application object and all the information related to resource requirements.
Once the resource requirements are met, the application objects will be created and then the operator of the application will manage them as usual.
The DynamicWorkload object will be moved to a running queue from the original clusterQueue.
Implement another admission controller for the worker/ executor Pods created by the application layer (e.g. for Spark, details in the spark repo KubernetesClusterManager and ExecutorPodAllocator. For Ray, Autoscaler). 
This controller will intercept these worker / executor pods creation and validate if their creation meets the capacity requirements. 
It will update the DynamicWorkload object for the application to which these worker pods belong to, with new resource requirements and move it back to the clusterQueue for further scheduling. These pods will be created once the capacity requirement is met and the DynamicWorkload object will again move to the running queue.
By taking this two admission controller approach, we don’t stop the creation of Application objects but we do control the creation of new worker / executor pdos which are dynamically required and created by the Spark layer. Which objects will go through these admission controllers can be controlled by using objectSelectors

## Next Steps
* Do a PoC with this approach
* Come up with a detailed design doc. 

