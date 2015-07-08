# Introduction to Enterprise Container Management: *Atomic Enterprise Platform*

[//]: # (This is a first attempt at framing a story for how AE enhances)
[//]: # (the stock capabilities. Right now it sounds like a whine-fest. It)
[//]: # (will need re-wording to make the limitations of Kubernetes to appear)
[//]: # (-correctly- a natural outcome of its design scoping)

Project Atomic - Atomic Enterprise is an enhanced container
orchestration and management platform, based on a set of open-source
container deployment and management projects, and extended to provide
additional capabilities and services.

Atomic Enterprise is meant to provide the administrators and users the
means to create, deploy and manage complex applications in a container
service environment.

This document lays a foundation for understanding software containers,
container management and the challenges and potential for large scale
container management services. It does not assume a lot of prior
knowledge of existing software container systems. Where necessary,
references are provided so that the reader can choose to read in more
depth about the technologies under discussion. Finally, this document
highlights the enhancements and benefits which Atomic Enterprise
brings above the capabilities of the software on which it is based.

The outline here uses two of the currently prevailing tools, Docker
and Kubernetes, as the basis for discussion. This is largely because
these are also the technologies which underlie Atomic Enterprise, but
the concepts apply equally to related technologies.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Preface - The Layers of Containers and Container Management](#preface---the-layers-of-containers-and-container-management)
  - [Container Basics](#container-basics)
    - [Container Image](#container-image)
    - [Container Service](#container-service)
    - [Container Host](#container-host)
  - [Container Service Orchestration](#container-service-orchestration)
    - [Orchestration Layer 1: Host Resource Management](#orchestration-layer-1-host-resource-management)
    - [Orchestration Layer 2: Host Clustering](#orchestration-layer-2-host-clustering)
    - [Orchestration Layer 3: Network Resources and Publication](#orchestration-layer-3-network-resources-and-publication)
  - [Resource Management, Access Control and Policy](#resource-management-access-control-and-policy)
  - [Summary of Container Management Concepts](#summary-of-container-management-concepts)
- [Containers and Orchestration with Kubernetes](#containers-and-orchestration-with-kubernetes)
  - [Container Deployment](#container-deployment)
  - [Resource Allocation](#resource-allocation)
  - [Networking](#networking)
  - [Publication and External Access](#publication-and-external-access)
  - [Network Storage](#network-storage)
  - [Continuous Integration/Development Hooks](#continuous-integrationdevelopment-hooks)
  - [Installation and Administration](#installation-and-administration)
  - [Summary of Capabilities and Limitations of Kubernetes](#summary-of-capabilities-and-limitations-of-kubernetes)
- [Atomic Enterprise](#atomic-enterprise)
  - [Container Deployment - Regions and Zones](#container-deployment---regions-and-zones)
  - [Software Defined Networking - Open vSwitch](#software-defined-networking---open-vswitch)
  - [Inbound Access and Publication - HAProxy and integrated SkyDNS](#inbound-access-and-publication---haproxy-and-integrated-skydns)
    - [Inbound Access to Container Services](#inbound-access-to-container-services)
    - [Application Publication and Discovery](#application-publication-and-discovery)
  - [Network Storage - native Gluster and Ceph](#network-storage---native-gluster-and-ceph)
  - [Installation and Administration - Unified common services](#installation-and-administration---unified-common-services)
  - [Hooks for CI/CD and build processes.](#hooks-for-cicd-and-build-processes)
- [References](#references)
- [Glossary](#glossary)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Preface - The Layers of Containers and Container Management

The first section is a brief (?) recap of the functional structure of
a generalized container deployment ecosystem. If you're familiar with
the complete stack then you can skip to the
[next section](#containers-and-orchestration-with-kubernetes)

Software container applications are still a young technology (at least
in terms of common use and availablility.) The introduction of Docker
made container development and execution accessible, and opened
peoples' minds to the explore the possibilities that containers may
offer. Since then the ideas and implementations have come in a flood,
with all of the promise for change (and chaos) that this implies.

The chaos of exploration and the rush to implementation and
productization has lead to some confusion. Container applications are
as complex (more complex?) than conventional applications and
services. The development, composition and deployment patterns are
different. The first impulse to treat containers like virtual machines
or like traditional host based processes has masked the actual
patterns that are inherent to the actual characteristics of container
systems.

The characteristics of containers and of the systems which host them
define a set of rings of scope.  Each ring defines a set of resources
and responsibilities which must be provided to the container
system. Each ring has an inward and outward boundary and these
boundaries define the scope of a subsystem of a complete container
management system.

### Container Basics

[Add discussion of the base - out of scope, but background]

#### Container Image

The container image is often ignored as an artifact rather than a
component of the service. In the model here the container image is the
center-most component of the system. It defines the passive contents of
the container as well as providing metadata to the container service
which will be used to create the containment environment and to start
the container processes. This metadata is the first example of the
boundary communications which typify the behavior of the layers of a
container system. These communications will pass information both in
and out of each layer so that the systems above and below. 

#### Container Service

The container service is the first ring. It creates containers on a
single host. Containers are defined by a set of files on a host (from
the container image), a limited view of the host filesystems and
network, and a single root process running in that context.

The container service uses host resources to build the container
run-time context and then to run the container process in that context
on the host.

The container service does not add or remove resources on the host,
though it may modify some host configurations such as adding virtual
network interfaces (to the existing real interfaces) or trigger NFS
automounts to the host and then import the local view into the
container.

The container service adds and removes containers on the request of
some outside agent; a user or an orchestration system.

*NOTE*: The container service may also be responsible for retrieving
 container images and building them. These are outside the scope of
 this discussion.

#### Container Host

The container host is the place where individual containers are
created and executed. It has memory, local storage and at least one
network interface. The container host can run on bare metal or in a
virtual machine. It can be running a general purpose operating system
or a minimized distribution tuned for containers.

The container host provides the local resources that the containers
need to run. It provides connectivity through the network to
additional resources such as distributed filesystems. Any resources
that are not statically configured to the host must be provided
on-demand by the orchestration system. (see below)

### Container Service Orchestration

[Improve segue - here's the first meat]

Orchestration is a blanket term for "making stuff happen automatically
and in concert" (in the non-musical sense.) In terms of the scoping
rings of a complete container management system, orchestration has
three different responsibilities corresponding to three rings: facing
in to the host, connecting hosts into a cluster, and facing out of the
cluster to additional resources and to users.

Each of the elements of container orchestration bridges a gap between
the container and some resource that the container needs to do its job.

#### Orchestration Layer 1: Host Resource Management

The Host Resource Management function of the orchestration system
bridges the boundaries between remote resources or non-existent host
resources and the container system.

The first part of the orchestration system is an agent which runs on
the container host. This agent can make changes to the host to provide
additional resources to its containers. For example, the orchestration
system can manipulate the ISCI targets to add shared physical storage
to the host so that the container system can mount them into new
containers. It can communicate with the SDN service to provide new
virtual interfaces to connect local and remote containers, or to route
inbound network connections to the local containers.

#### Orchestration Layer 2: Host Clustering

The Host Clustering function of the orchestration system also makes
use of the host agent. The clustering function binds container hosts
together, allowing new containers to be run on any host in the
cluster.

Somewhere an orchestration master service runs. Each host that
participates in the cluster is running an agent process which connects
to the master and accepts requests from the master. The master service
places containers onto members of the cluster and then can instruct
the agents to create and manage network connections between
containers.

By itself the host clustering layer does not make intelligent resource
allocation decisions. Without guidance it merely places each new
container on some available host.

#### Orchestration Layer 3: Network Resources and Publication

All of the focus up until now has been on activities within the
container host cluster: Creating and connecting containers with each
other. The third layer of the orchestration system looks outside the
cluster to bring resources in for the containers to use.

The most common external resources are network storage and inbound
connectivity. The first is a pull operation, bringing known resources
to the container hosts and making them available to the container
system. The second is a kind of push operation and requires two steps:

1. Establish an inbound access point and a path to the container(s)
1. Publish the inbound access point

### Resource Management, Access Control and Policy

Properly, resource management is outside the scope of the
orchestration system. The decisions about resource allocation are made
by a policy engine such as _Mesos_. However, the orchestration system
must be able to reflect and report on the resources that are
available, and it must be able to accept allocation requests which
respect the decisions of the policy engine. This means it must be able
to both label resources and allow the policy engine to make requests
using those labels.

### Summary of Container Management Concepts

[TBD]

## Containers and Orchestration with Kubernetes

[Improve Segue - addressing the frameworks - skipping right to orchestration]

When you want to use software containers to build and run complete
applications and services, it quickly becomes evident that creating
single containers on individual hosts will become time consuming and
unwieldy. Containers run on single hosts. To make services span
multiple hosts, each host must be prepared with identical remote
storage and networking resources independent of the container service.

### Container Deployment

_Kubernetes_, an orchestration and clustering layer above the container
systems provides a means to create individual containers and related
groups of containers. It provides a way to describe individual
containers or clusters of containers (_Pods_) which will run together
and will be assigned to a single container host. It also provides a
means to create and maintain a specified number of replicas of each
pod. Replicas will be placed on different nodes.

### Resource Allocation

Kubernetes allocation mechanism is fairly simple. It does not offer
any way to partition the cluster or to assign pods to sub-sets of the
cluster for resource or policy management purposes. A Kubernetes
cluster is flat and uniform. Other than the constraint that two pod
replicas cannot be placed on the same host, there are no user
accessable controls on the placement of pods or their resource usage.

### Networking

Kubernetes also provides a simple mechanism for networking within the
cluster, the _kube-proxy_.  The kube-proxies can route connections
between containers anywhere within the cluster, but it does not
provide functions like VLANs, VPNs or other more complex SDN (software
defined network) features. Kubernetes can work with _flannel_, an
overlay network service which provides a unique IP address to each
container within the cluster and which can be configured to do more
than the simple routing offered by the kube-proxy service. Flannel too
does not provide complex routing capabilities that are possible with a
proper SDN service.

When running in a commercial cloud service such as Google GCE or
Amazon EC2, Kubernetes does not provide tight integration and control
with the networking services that are available.

### Publication and External Access

Because Kubernetes is concerned mostly with creating and running
containers within the cluster, it does not offer integrated mechanisms
for publishing the services to external users. It can provide
listeners on well known ports on one or more external facing hosts
running special kube-proxy processes, but Kubernetes provides no means
to make applications running in the cluster accessible to clients
outside. These capabilities must be managed manually.

### Network Storage

On the nodes, Kubernetes acts to bridge the boundaries between the
host and the containers. By itself though, it can only make use of the
resources that have been configured on the host. For example,
Kubernetes has the capability to mount NFS filesystems onto the host
and then into new containers, but only if the host has already been
configured to act as an NFS client. [check this. Better example? MAL]

In commercial cloud services Kubernetes offers the ability to reach
native block storage for GCE and EC2/EBS, but it cannot access network
file services such as Gluster or Ceph without prior uniform
configuration on the container host.

### Continuous Integration/Development Hooks

[TBD]

### Installation and Administration

Each host in a fully functional Kubernetes cluster will have a number
of different service processes running. On the master there is an
app-server process, an etcd and probably a kube-proxy and flanneld.
On the nodes there will be a kubelet, a kube-proxy, and a
flanneld. Additional services such as DNS to provide application
publishing and inbound routing must be added and managed by the
cluster admins.

### Summary of Capabilities and Limitations of Kubernetes

[TBD]

## Atomic Enterprise

Atomic Enterprise provides the additional capabilities that fill in
the gaps between the orchestration of containers within the cluster and
access to the resources outside the cluster (client networking,
publishing and discovery, network storage). It also provides the means
to partition the cluster and to create and use placement policies for
containers within those partitions.

[This needs more/clearer introduction]

### Container Deployment - Regions and Zones

Atomic Enterprise adds a couple of features to the cluster which allow
the users and the cluster managers to provide different resource
levels to different users or applications within a single cluster.

### Software Defined Networking - Open vSwitch

[Explain how AE adds networking capabilities both internally and on the boundary of the cluster]

[Contributors?]

### Inbound Access and Publication - HAProxy and integrated SkyDNS

When applications and services run in a containerized environment, the
container management system establishes the connections between the
containers so that they can work as a unit. By itself, this is not
sufficient to make these applications available to users outside the
container service, the actual intended audience. Making hosted
services accessible to users requires two additional related
capabilities:

1. The container service must establish a connection end-point (IP
   address and port) on the network border of the service. 

1. The container service must publish the existence of this end point
   so that external users can find it and connect.

Atomic Enterprise integrates both capabilities and makes them
available to the service developer through the _oc_ CLI interface.

#### Inbound Access to Container Services

[How does AE create routes from the cluster border to the containers?]

[Contributors?]

#### Application Publication and Discovery

[How does AE publish the external access points to hosted services?]

[Contributors?]

### Network Storage - native Gluster and Ceph

While it is common for containerized applications to need persistent
storage (where it is understood that storage within a container lives
only as long as the individual container) and also to need to share
storage (when, for example one set of processes is adding content and
others are consuming it), shared network storage is, properly
"outside" the container management service. That is, it is a resource
which is not directly connected to each container node. Rather, the
file services are managed distinct from the container management
service and are treated as an external resource.

It is possible to statically attach network storage to every node
within a container service cluster, but this is a messy solution. It
introduces fixed host configuration issues.  It also is overkill; most
containers will only use a small fragment of the total available
storage.

Atomic Enterprise has the capability to dynamically mount distributed
network file services into individual containers on demand.

[How does AE allow containers to access and share network storage services]

[Contributors?]

### Installation and Administration - Unified common services

[How does AE simplify installation, monitoring and management of clusters]

[Contributors?]

### Hooks for CI/CD and build processes.

[How does AE assist in keeping hosted containers up to date]

[Contributors?]

## References

- [Docker](http://www.docker.com)

    Docker is a Linux software container system

- [Kubernetes](http://www.github.com/GoogleCloudPlatform/kubernetes)

    Kubernetes is a software container orchestration platform

- [Flannel](https://github.com/coreos/flannel)

    Flannel is a private network underlay system.

## Glossary

- Pod

    A _pod_ is a collection of containers which run together to perform
    some task. They usually share some data or other resource. All of the
    containers in a pod are placed on the same container host.

- Container Host

    A container host is any computer or VM with an
    operating system capable of running containers. In some contexts it
    can mean a specialized distribution, stripped down and tuned for
    running containers.  `
