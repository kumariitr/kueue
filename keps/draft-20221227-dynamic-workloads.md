# Dynamic Resource Workload Support in Kueue

## Motivation
Workloads in batch compute normally tend to be dynamic in nature from a resourcing perspective. When we talk about controllers to run these workloads on Kubernetes, these controllers should support the scheduling of these dynamic workloads. Currently Kueue supports workloads with static resource requirements, i.e. workloads for which resource requirements are decided at the time of creation and later can not be changed.
