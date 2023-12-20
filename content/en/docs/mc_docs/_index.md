---
title: Multi-cluster Documentation
approvers:
- adamkus319
linkTitle: "Multi-cluster Documentation"
main_menu: true
weight: 70
content_type: concept
no_list: true
---

<!-- overview -->

This section of the Kubernetes documentation contains information about using and setting up a multi-cluster Kubernetes environment.

<!-- body -->

## Introduction

This project extends traditional Kubernetes to a multi-cluster environment, providing 
an interface for several different clusters to communicate and update each other. 
Clusters are separated into regional groups that represent a geographic locality (ex: 
the US region, or the Europe region). Once the multi-cluster extension is running on 
each cluster, a user can input the number of service replicas desired in a region. Any 
cluster can accept an update from a user, making the system very decentralized and easy 
to access. This change is then scheduled to specific clusters in the region, and the 
new global state is propagated throughout the global multi-cluster system. When the 
relevant clusters receive the updated global configuration, they update their deployments 
to reflect the rule input by the user.

## Software Structure

Let's look at the major software components that make up this extension.

**Global Configuration:**
The global configuration of the multi-cluster Kubernetes system is described by a
pointer to the GlobalConfigLocal struct (which is defined in configHelper.cpp); each 
cluster has a GlobalConfigLocal pointer that describes the current global state. The 
GlobalConfigLocal struct consists of a vector of pointers to RegionConfigLocal structs, 
which each describe a region in the multi-cluster system that services can be scheduled 
to. The RegionConfigLocal struct consists of a regionName as an identifier (ex: “us”) 
and a vector of pointers to ClusterConfigLocal structs, which each describe a cluster 
within the region. Each ClusterConfigLocal struct consists of a clusterName as an 
identifier (ex: “us-east”) and a vector of pointers to ServiceConfigLocal structs, which
each describe a service running on the cluster. Each ServiceConfigLocal struct consists 
of the service name (ex: “nginx”) and how many copies (replicas) of the service are running
on the cluster. gRPC is used to flatten the GlobalConfigLocal struct into a message format 
that can then be sent to the other clusters, providing a simple parsing interface.

**Scheduler:**
The scheduler (defined in scheduler.cpp) operates using a simple algorithm that serves
as a proof-of-concept for multi-cluster global scheduling. The current global configuration
is input into the scheduler function, along with a regionName and the number of copies of
a service that should be running in that region after scheduling. Upon receiving this, the 
scheduler goes through each service desired in the new region and counts the number of copies
that are already running in the region. The scheduler will then start adding or removing 
copies of that service (one at a time) in clusters within the region, until the desired 
number of services in the region is achieved. During scheduling, the global configuration 
variable is updated so that after all scheduling is complete, the new global configuration 
can be used to modify the deployments running on each Kubernetes cluster.

**Local Updater:**
Once the global configuration is updated, each cluster must update the deployments running 
on it according to the new global configuration it has received. This is done through the 
updateLocal function (defined in updateLocal.cpp), which takes in the regionName and clusterName
of the calling cluster and the global configuration that includes the services/deployments 
the cluster should be running. Upon identifying the portion of the global configuration that 
involves the calling region/cluster, the local updater uses Python scripts to call the 
Kubernetes API and create, update, or remove deployments. For a new deployment, the API sets 
up a default configuration that sets the deployment to listen on port 8080 (assuming a valid 
Docker container of the deployment exists).

## API

Currently, this project's API supports the user inputting a regionName, a serviceName, and a count.
This input sets the number of services (denoted by serviceName) that should run within region 
regionName. Though simple, this provides opportunity for substantial orchestration across a 
multi-cluster system. Future plans include adding to the API grammar for more complex service
assignments, such as the ability to exclude a cluster.




