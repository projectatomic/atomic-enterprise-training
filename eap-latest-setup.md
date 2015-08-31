<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Atomic Enterprise Platform Early Access Program](#atomic-enterprise-platform-early-access-program)
  - [Architecture and Requirements](#architecture-and-requirements)
    - [Architecture](#architecture)
    - [Requirements](#requirements)
  - [Setting Up the Environment](#setting-up-the-environment)
    - [Assumptions](#assumptions)
    - [Hostnames](#hostnames)
    - [The router and wildcard DNS](#the-router-and-wildcard-dns)
    - [Git](#git)
    - [Preparing Each VM](#preparing-each-vm)
    - [Ansible preparation](#ansible-preparation)
        - [Ansible preparation: Installing Ansible client and the playbooks](#ansible-preparation-installing-ansible-client-and-the-playbooks)
        - [Ansible preparation: SSH](#ansible-preparation-ssh)
        - [Ansible: inventory](#ansible-inventory)
    - [Run the installer (on the master VM)](#run-the-installer-on-the-master-vm)
  - [Launch your very first pod](#launch-your-very-first-pod)
  - [Regions and Zones](#regions-and-zones)
    - [Scheduler and Defaults](#scheduler-and-defaults)
    - [The NodeSelector](#the-nodeselector)
    - [Customizing the Scheduler Configuration](#customizing-the-scheduler-configuration)
    - [Node Labels](#node-labels)
  - [Useful Logs](#useful-logs)
  - [Auth and Projects](#auth-and-projects)
    - [Configuring htpasswd Authentication](#configuring-htpasswd-authentication)
    - [A Project for Everything](#a-project-for-everything)
  - [Your First Application](#your-first-application)
    - [Resources](#resources)
    - [Applying Quota to Projects](#applying-quota-to-projects)
    - [Applying Limit Ranges to Projects](#applying-limit-ranges-to-projects)
    - [Login](#login)
    - [Grab the Training Repo Again](#grab-the-training-repo-again)
    - [The Hello World Definition JSON](#the-hello-world-definition-json)
    - [Run the Pod](#run-the-pod)
        - [Inspecting the Docker state](#inspecting-the-docker-state)
    - [Delete the Pod](#delete-the-pod)
    - [Quota Enforcement](#quota-enforcement)
  - [Services](#services)
  - [Optional: Routing](#optional-routing)
    - [Creating a Wildcard Certificate](#creating-a-wildcard-certificate)
    - [Creating the Router](#creating-the-router)
    - [Router Placement By Region](#router-placement-by-region)
    - [Viewing Router Stats](#viewing-router-stats)
  - [The Complete Pod-Service-Route](#the-complete-pod-service-route)
    - [Creating the Definition](#creating-the-definition)
    - [Project Status](#project-status)
    - [Verifying the Service](#verifying-the-service)
    - [Verifying the Routing](#verifying-the-routing)
  - [Project Administration](#project-administration)
    - [Deleting a Project](#deleting-a-project)
  - [The registry, imagestreams, and more](#the-registry-imagestreams-and-more)
  - [Conclusion](#conclusion)
- [APPENDIX - Docker Storage Setup](#appendix---docker-storage-setup)
- [APPENDIX - DNSMasq setup](#appendix---dnsmasq-setup)
    - [Verifying DNSMasq](#verifying-dnsmasq)
- [APPENDIX - Import/Export of Docker Images (Disconnected Use)](#appendix---importexport-of-docker-images-disconnected-use)
- [APPENDIX - Cleaning Up](#appendix---cleaning-up)
- [APPENDIX - Troubleshooting](#appendix---troubleshooting)
- [APPENDIX - Infrastructure Log Aggregation](#appendix---infrastructure-log-aggregation)
  - [Enable Remote Logging on Master](#enable-remote-logging-on-master)
  - [Enable logging to /var/log/openshift](#enable-logging-to-varlogopenshift)
  - [Configure nodes to send atomic logs to your master](#configure-nodes-to-send-atomic-logs-to-your-master)
  - [Optionally Log Each Node to a unique directory](#optionally-log-each-node-to-a-unique-directory)
- [APPENDIX - Working with HTTP Proxies](#appendix---working-with-http-proxies)
  - [Importing ImageStreams](#importing-imagestreams)
  - [Setting Environment Variables in Pods](#setting-environment-variables-in-pods)
  - [Proxying Docker Pull](#proxying-docker-pull)
  - [Future Considerations](#future-considerations)
- [APPENDIX - Installing in IaaS Clouds](#appendix---installing-in-iaas-clouds)
  - [Generic Cloud Install](#generic-cloud-install)
  - [Automated AWS Install With Ansible](#automated-aws-install-with-ansible)
- [APPENDIX - Linux, Mac, and Windows clients](#appendix---linux-mac-and-windows-clients)
  - [Downloading The Clients](#downloading-the-clients)
  - [Log In To Your Atomic Environment](#log-in-to-your-atomic-environment)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Atomic Enterprise Platform Early Access Program

This guide focuses on Atomic Enterprise Platform.  However, there is
ongoing work to share more work with OpenShift v3, and thus you might
find newer instructions in the [OpenShift
version](https://github.com/openshift/training) of this guide.

## Architecture and Requirements
### Architecture
The documented architecture for the early access testing is pretty simple. There are
three systems:

* Master + Node
* Node
* Node

The master is the scheduler/orchestrator and the API endpoint for all commands.
This is similar to OpenShift V2's "broker". We are also running the node software on the
master.

The "node" hosts user applications. You will learn much more about
the inner workings of Atomic throughout the rest of the document.

### Requirements

You should begin this tutorial with a basic grasp of how to provision
multiple Red Hat Enterprise Linux instances on at least one chosen
physical/virtual platform.  That platform could be local
libvirt/virt-manager on your laptop, a private OpenStack instance, Red
Hat Satellite with bare metal PXE booting, or it could be EC2 or
another public cloud, etc.

For the deployment target, you should know at least:

 - How to set up authentication with ssh (e.g. in public cloud, via cloud-init most commonly, on bare metal, with kickstart or manually in the Anaconda GUI)
 - How to manage storage and networking (particularly firewalls)

Each machine should have 4+ GB of memory, 20+ GB of disk space, and
the following configuration:

* Red Hat Enterprise Linux >=7.1 (Note: 7.1 kernel is required for openvswitch)
* If using Anaconda, the "Minimal" installation option (the default for the KVM and AWS images etc. is fine)

**Note:** At the current time, Atomic Enterprise Platform is not
supported for deployment on Red Hat Enterprise Linux Atomic Host.

The majority of storage requirements are related to Docker, etcd, and a docker
image repository.  The etcd state (like all service state by default) lives in `/var/lib/etcd/`,
so be sure to allocate storage for it. Leave space somewhere on the master to store
images and metadata for the docker registry. In later examples we will use `/mnt/registry`.
For Docker, the currently recommended storage configuration is "direct LVM" using
[docker-storage-setup](https://github.com/projectatomic/docker-storage-setup/).

Please see [APPENDIX - Docker Storage Setup](#appendix---docker-storage-setup)
for information on setting up storage for docker.

[//]: # (TODO: what is a correct subscription name???)

As part of signing up for the beta program, you should have received an
evaluation subscription. This subscription gave you access to the beta software.
You will need to use subscription manager to both register your VMs, and attach
them to the *Atomic Enterprise High Touch Beta* subscription.

## Setting Up the Environment
### Assumptions
In most cases you will see references to "example.com" and other FQDNs related
to it. If you choose not to use "example.com" in your configuration, that is
fine, but remember that you will have to adjust files and actions accordingly.

### Hostnames

- All of your VMs must be able to access one another by each other's hostname.

  In almost all cases, when referencing VMs you must use hostnames and the
  hostnames that you use must match the output of `hostname -f` on each of your
  nodes. Forward DNS resolution of hostnames is an **absolute requirement**. This
  training document assumes the following configuration:

  * ae-master.example.com (master+node)
  * ae-node1.example.com
  * ae-node2.example.com

  We do our best to point out where you will need to change things if your
  hostnames do not match.

### The router and wildcard DNS

- Atomic Enterprise comes with a "router" component for external
  access to the cluster, using HAProxy.  This is optional, but it is
  very common to want external systems to be able to access a cluster.

  If you do not have administrative access to an existing DNS system
  (e.g. Route 53 in AWS, or a private DNS infrastructure), you will
  need to set up your own DNS server in the beta testing environment.
  Documentation is provided on DNSMasq in an appendix, [APPENDIX -
  DNSMasq setup](#appendix---dnsmasq-setup)

  Remember that NetworkManager may make changes to your DNS
  configuration/resolver/etc. You will need to properly configure your interfaces'
  DNS settings and/or configure NetworkManager appropriately.

  More information on NetworkManager can be found in this comment:

    https://github.com/openshift/training/issues/193#issuecomment-105693742

  If you choose to use the router, you will need to have a wildcard
  for a DNS zone resolve, ultimately, to the IP address of the
  router. For this training, we will ensure that the router will end
  up on the server that is running the master. Go ahead and create a
  wildcard DNS entry for "cloudapps" (or something similar), with a
  low TTL, that points to the public IP address of your master.

  For example:

    *.cloudapps.example.com. 300 IN  A 192.168.133.2

  It is possible to use dnsmasq inside of your beta environment to handle these
  duties. See the [appendix on dnsmasq](#appendix---dnsmasq-setup) if you can't
  easily manipulate your existing DNS environment.

### Git
You will either need internet access or read and write access to an internal
http-based git server where you will duplicate the public code repositories used
in the labs.

### Preparing Each VM
Once your VMs are built and you have verified DNS and network connectivity you
should:

[//]: # (TODO: What are the right channels??)
[//]: # (TODO: Identify what is needed from rhel-server-7-ose-beta-rpms and what will be AE's equivalent)

1. Configure yum / subscription manager as follows:
    ```
    subscription-manager register --auto-attach
    subscription-manager repos --disable="*"
    subscription-manager repos \
      --enable="rhel-7-server-rpms" \
      --enable="rhel-7-server-extras-rpms" \
      --enable="rhel-7-server-ose-3.0-rpms"
    ```

1. Install rpms missing from minimal we are likely to need.
    ```
    yum -y install deltarpm wget vim-enhanced net-tools bind-utils tmux git docker
    ```

1. Install our atomic enterprise repo
    ```
    curl -o /etc/yum.repos.d/atomic-enterprise.repo http://mirror.ops.rhcloud.com/atomic/mirror/.atomic-enterprise-early-1/atomic-enterprise.repo
    ```

1. Make sure docker storage is configured correctly before starting docker!

    See [APPENDIX - Docker Storage Setup](#appendix---docker-storage-setup)

1. To speed things up later you can grab docker images needed later (Optional)

    Make sure you completed the storage setup first! See [APPENDIX - Docker
    Storage Setup](#appendix---docker-storage-setup)

    ```
    systemctl start docker
    docker pull registry.access.redhat.com/openshift3/ose-haproxy-router
    docker pull registry.access.redhat.com/openshift3/ose-deployer
    docker pull registry.access.redhat.com/openshift3/ose-pod
    docker pull registry.access.redhat.com/openshift3/ose-docker-registry
    ```
    It may be advisable to pull the following Docker images as well, since they are
    used during the various labs:
    ```
    docker pull docker.io/atomicenterprise/hello-atomic
    ```

### Ansible preparation

Currently, the vast majority of the heavy lifting for an Atomic
Enterprise installation is currently implemented in an Ansible code
base.  Ansible is available in the
[EPEL](https://fedoraproject.org/wiki/EPEL) repository.

In order to proceed, you must have functional ssh access to each node
from a host with Ansible installed.  It will be significantly more
pleasant if you have SSH public key authentication, rather than having
to repeatedly type passwords.

If you already have the knowledge and ability to run Ansible from your
workstation, or a separate server with the SSH private keys or agent
forwarding, you can skip to the next section.

The following two steps detail how to prepare for Ansible from the
master node (or you could choose to use a separate server).

##### Ansible preparation: Installing Ansible client and the playbooks

1. Install EPEL repo and then ansible

    ```
    yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
    sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo
    yum -y --enablerepo=epel install ansible
    ```

1. Clone the Atomic Enterprise Ansible repo:

    ```
    git clone https://github.com/projectatomic/atomic-enterprise-ansible.git
    ```

##### Ansible preparation: SSH

1.  Ensure that this works:

    ```
    ssh ae-node1.example.com
    ```

    If for example you used Anaconda to install on bare metal or virtual machines,
    you may only have password authentication set up.  You can use the `ssh-copy-id`
    program:

    ```
    for node in ae-master.example.com ae-node1.example.com ae-node2.example.com; do
        ssh-copy-id ${node}
    done
    ```

##### Ansible: inventory

1. Edit the byo (bring your own) inventory file to include your hosts

    ```
    cd atomic-enterprise-ansible
    vi inventory/byo/hosts
    ```

    Replace `[masters]` and `[nodes]` sections with following content or modify
    them according to your DNS environment.

    ```
    [masters]
    ae-master.example.com

    [nodes]
    ae-master.example.com openshift_node_labels="{'region': 'infra', 'zone': 'default'}"
    ae-node1.example.com openshift_node_labels="{'region': 'primary', 'zone': 'east'}"
    ae-node2.example.com openshift_node_labels="{'region': 'primary', 'zone': 'west'}"
    ```

    - For now do not worry much about the information after
      `openshift_node_labels=`. But do no omit it entirely.

### Run the installer (on the master VM)

1. Run ansible to set up the cluster
    ```
    ansible-playbook -i inventory/byo/hosts playbooks/byo/config.yml
    ```

1. Do not move along unless this worked! Success looks (something) like this:

    ```
    -------------------------------------------------------------------------------
    PLAY RECAP ********************************************************************
    ae-master.example.com      : ok=95   changed=42   unreachable=0    failed=0
    ae-node1.example.com       : ok=18   changed=22   unreachable=0    failed=0
    ae-node2.example.com       : ok=18   changed=22   unreachable=0    failed=0
    localhost                  : ok=5    changed=0    unreachable=0    failed=0
    -------------------------------------------------------------------------------
    ```

1. Run oc get nodes (your cluster should be running!)
    ```
    oc get nodes
    ```

    You should see something like:
    ```
    -----------------------------------------------------------------------------
    NAME                    LABELS                                         STATUS
    ae-master.example.com   kubernetes.io/hostname=ae-master.example.com   Ready
    ae-node1.example.com    kubernetes.io/hostname=ae-node1.example.com    Ready
    ae-node2.example.com    kubernetes.io/hostname=ae-node2.example.com    Ready
    -----------------------------------------------------------------------------
    ```

<!--
[//]: # (TODO: remove this once the issue is resolved and update above `get nodes` output)

1. There's a [bug](https://github.com/openshift/openshift-ansible/issues/305)
   in current ansible installer preventing labels to be set. For now, let's set
   them manually:
    ```
    oc label nodes ae-master.example.com region=infra zone=default
    oc label nodes ae-node1.example.com  region=primary zone=east
    oc label nodes ae-node2.example.com  region=primary zone=west
    ```
    You should see them assigned in the output of the next `oc get nodes`.
-->

## Launch your very first pod

We will launch a pod, see that it starts and then delete it. More about pods,
services, scheduling, authentication and all sorts of other information follows.

1. Clone the atomic-enterprise-training repository.

   You should perform this operation on the **master** host as root, which
   will automatically have the credentials to access the cluster.

   This repository has materials used later in later exercises; we
   call the directory `training` for short.

   ```
   git clone https://github.com/projectatomic/atomic-enterprise-training.git training
   ```

1. Launch your first pod
    ```
    oc create -f /root/training/eap-latest/hello-pod.json
    ```

1. Verify the pod started
    ```
    oc get pods
    ```

    While it's starting, you should see:
    ```
    --------------------------------------------------
    NAME           READY     REASON    RESTARTS   AGE
    hello-atomic   0/1       Pending   0          4s
    --------------------------------------------------
    ```
    Keep running `oc get pods` until the pod is in state `Running`, this can
    take roughly a minute and involves downloading a docker image so time can
    vary depending on network speed:
    ```
    --------------------------------------------------
    NAME           READY     REASON    RESTARTS   AGE
    hello-atomic   1/1       Running   0          1m
    --------------------------------------------------
    ```

1. Get extended information about the pod
    ```
    oc describe pods hello-atomic
    ```
    The output should look something like:

    **Note:** Take notice of the IP field below:
    ```
    ----------------------------------------------------------------
    Name:                    hello-atomic
    Image(s):                atomicenterprise/hello-atomic:latest
    Host:                    ae-master.example.com/192.168.122.154
    Labels:                  name=hello-atomic
    Status:                  Running
    IP:                      10.1.0.2
    Replication Controllers: <none>
    ...
    ----------------------------------------------------------------
    ```

1. Access new pod
    ```
    # execute this on the node which hosts the pod (Host:)
    curl http://$IP_FROM_ABOVE:8080/
    ```

1. Delete the new pod
    ```
    oc delete pod hello-atomic
    ```

## Regions and Zones

There was also some information about "regions" and "zones" in the hosts file.
Let's talk about those concepts now.

If you are familiar with OpenShift 2, in that system, "regions" and
"zones" enable organizations to provide some topologies for
application resiliency. Apps would be spread throughout the zones in a
region and, depending on the way you configured OpenShift 2, you could
make different regions accessible to users.

Atomic Enterprise (and OpenShift v3) are different, being based on an
entirely new underlying Kubernetes infrastructure.  Kubernetes doesn't
actually care about topology - it is topology agnostic.  On top of
that, Atomic Enterprise provides advanced controls for implementing
whatever topologies you can dream up, leveraging filtering and
affinity rules to ensure that parts of applications (pods) are either
grouped together or spread apart.

For the purposes of a simple example, we'll be sticking with the
"regions" and "zones" theme. But, as you go through these examples,
think about what other complex topologies you could implement. Perhaps
"secure" and "insecure" hosts, or other topologies.

First, we need to talk about the "scheduler" and its default configuration.

### Scheduler and Defaults
The "scheduler" is essentially the Atomic master. Any time a pod needs to be
created (instantiated) somewhere, the master needs to figure out where to do
this. This is called "scheduling". The default configuration for the scheduler
looks like the following JSON (although this is embedded in the Origin code
and you won't find this in a file):

    {
      "predicates" : [
        {"name" : "PodFitsResources"},
        {"name" : "MatchNodeSelector"},
        {"name" : "HostName"},
        {"name" : "PodFitsPorts"},
        {"name" : "NoDiskConflict"}
      ],
      "priorities" : [
        {"name" : "LeastRequestedPriority", "weight" : 1},
        {"name" : "ServiceSpreadingPriority", "weight" : 1}
      ]
    }

When the scheduler tries to make a decision about pod placement, first it goes
through "predicates", which essentially filter out the possible nodes we can
choose. Note that, depending on your predicate configuration, you might end up
with no possible nodes to choose. This is totally OK (although generally not
desired).

These default options are documented in the link above, but the quick overview
is:

* Place pod on a node that has enough resources for it (duh)
* Place pod on a node that doesn't have a port conflict (duh)
* Place pod on a node that doesn't have a storage conflict (duh)

And some more obscure ones:

* Place pod on a node whose `NodeSelector` matches
* Place pod on a node whose hostname matches the `Host` attribute value

The next thing is, of the available nodes after the filters are applied, how do
we select the "best" one. This is where "priorities" come in. Long story short,
the various priority functions each get a score, multiplied by the weight, and
the node with the highest score is selected to host the pod.

Again, the defaults are:

* Choose the node that is "least requested" (the least busy)
* Spread services around - minimize the number of pods in the same service on
    the same node

And, for an extremely detailed explanation about what these various
configuration flags are doing, check out:

[//]: # (TODO: docs.openshift.org -> ???)

    http://docs.openshift.org/latest/admin_guide/scheduler.html

In a small environment, these defaults are pretty sane. Let's look at one of the
important predicates (filters) before we move on to "regions" and "zones".

### The NodeSelector
`NodeSelector` is a part of the Pod data model. And, if we think back to our pod
definition, there was a "label", which is just a key:value pair. In the case of
a `NodeSelector`, our labels (key:value pairs) are used to help us try to find
nodes that match, assuming that:

* The scheduler is configured to MatchNodeSelector
* The end user creating the pod knows which labels are out there

But this use case is also pretty simplistic. It doesn't really allow for a
topology, and there's not a lot of logic behind it. Also, if I specify a
NodeSelector label when using MatchNodeSelector and there are no matching nodes,
my workload will never get scheduled. Bummer.

How can we make this more intelligent? We'll finally use "regions" and "zones".

### Customizing the Scheduler Configuration

The Ansible installer is configured to understand "regions" and "zones" as a
matter of convenience. However, for the master (scheduler) to actually do
something with them requires changing from the default configuration Take a look
at `/etc/origin/master/master-config.yaml` and find the line with `schedulerConfigFile`.

You should see:

    schedulerConfigFile: "/etc/origin/master/scheduler.json"

Then, take a look at `/etc/origin/master/scheduler.json`. It will have the
following content:

    {
      "predicates" : [
        {"name" : "PodFitsResources"},
        {"name" : "PodFitsPorts"},
        {"name" : "NoDiskConflict"},
        {"name" : "Region", "argument" : {"serviceAffinity" : { "labels" : ["region"]}}}
      ],
      "priorities" : [
        {"name" : "LeastRequestedPriority", "weight" : 1},
        {"name" : "ServiceSpreadingPriority", "weight" : 1},
        {"name" : "Zone", "weight" : 2, "argument" : {"serviceAntiAffinity" : { "label" : "zone" }}}
      ]
    }

To quickly review the above (this explanation sort of assumes that you read the
scheduler documentation, but it's not critically important):

* Filter out nodes that don't fit the resources, don't have the ports, or have
    disk conflicts
* If the pod specifies a label with the key "region", filter nodes by the value.

So, if we have the following nodes and the following labels:

* Node 1 -- "region":"infra"
* Node 2 -- "region":"primary"
* Node 3 -- "region":"primary"

If we try to schedule a pod that has a `NodeSelector` of "region":"primary",
then only Node 1 and Node 2 would be considered.

OK, that takes care of the "region" part. What about the "zone" part?

Our priorities tell us to:

* Score the least-busy node higher
* Score any nodes who don't already have a pod in this service higher
* Score any nodes whose zone label's value **does not** match higher

Why do we score a zone that **doesn't** match higher? Note that the definition
for the Zone priority is a `serviceAntiAffinity` -- anti affinity. In this case,
our anti affinity rule helps to ensure that we try to get nodes from *different*
zones to take our pod.

If we consider that our "primary" region might be a certain datacenter, and that
each "zone" in that datacenter might be on its own power system with its own
dedicated networking, this would ensure that, within the datacenter, pods of an
application would be spread across power/network segments.

The documentation link has some more complicated examples. The topological
possibilities are endless!

### Node Labels
The assignments of "regions" and "zones" at the node-level are handled by labels
on the nodes. You can look at how the labels were implemented by doing:

    oc get nodes
    NAME                    LABELS                                                                   STATUS
    ae-master.example.com   kubernetes.io/hostname=ae-master.example.com,region=infra,zone=default   Ready
    ae-node1.example.com    kubernetes.io/hostname=ae-node1.example.com,region=primary,zone=east     Ready
    ae-node2.example.com    kubernetes.io/hostname=ae-node2.example.com,region=primary,zone=west     Ready

At this point we have a running AE environment across three hosts, with
one master and three nodes, divided up into two regions -- "*infra*structure"
and "primary".

From here we will start to deploy "applications" and other resources into
AE.

## Useful Logs
RHEL 7 uses `systemd` and `journal`. As such, looking at logs is not a matter of
`/var/log/messages` any longer. You will need to use `journalctl`.

Since we are running all of the components in higher loglevels, it is suggested
that you use your terminal emulator to set up windows for each process. If you
are familiar with the Ruby Gem, `tmuxinator`, there is a config file in the
training repository. Otherwise, you should run each of the following in its own
window:

[//]: # (TODO: check service nodes)

    journalctl -f -u atomic-enterprise-master
    journalctl -f -u atomic-enterprise-node

**Note:** You will want to do this on the other nodes, but you won't need the
"-master" service. You may also wish to watch the Docker logs, too.

**Note:** There is an appendix on configuring [Log
Aggregation](#appendix---infrastructure-log-aggregation)

## Auth and Projects
### Configuring htpasswd Authentication
Atomic Enterprise supports a number of mechanisms for authentication. The simplest
use case for our testing purposes is `htpasswd`-based authentication.

In the "real world" your developers would likely be using the AE tools on
their own machines (e.g. `oc`). For the Early Access training, we
will create user accounts for two non-privileged users of AE, *joe* and
*alice*, on the **master**. This is done for convenience and because we'll be using
`htpasswd` for authentication.

    useradd joe
    useradd alice

To start, we will need the `htpasswd` binary, which is made available by
installing:

    yum -y install httpd-tools

From there, we can create a password for our users, Joe and Alice:

    touch /etc/origin/passwd
    htpasswd -b /etc/origin/passwd joe redhat
    htpasswd -b /etc/origin/passwd alice redhat

[//]: # (TODO: fix the /etc/origin/master.yaml path)

The Atomic Enterprise configuration is kept in a YAML file which currently lives at
`/etc/origin/master/master-config.yaml`. Ansible was configured to edit
the `oauthConfig`'s `identityProviders` stanza so that it looks like the following:

    identityProviders:
    - challenge: true
      login: true
      name: htpasswd_auth
      provider:
        apiVersion: v1
        file: /etc/origin/passwd
        kind: HTPasswdPasswordIdentityProvider

More information on these configuration settings (and other identity providers) can be found here:

[//]: # (TODO: Will we have something like docs.automic.org ?)

    http://docs.openshift.org/latest/admin_guide/configuring_authentication.html#HTPasswdPasswordIdentityProvider

### A Project for Everything
Atomic Enterprise (AE) has a concept of "projects" to contain a number of
different resources:
services and their pods, builds and so on. They are somewhat similar to
"namespaces" in OpenShift v2. We'll explore what this means in more details
throughout the rest of the labs. Let's create a project for our first
application.

We also need to understand a little bit about users and administration. The
default configuration for CLI operations currently is to be the `master-admin`
user, which is allowed to create projects. We can use the "admin"
Atomic command to create a project, and assign an administrative user to it:

    oadm new-project demo --display-name="Atomic Enterprise Demo" \
    --description="This is the first demo project with Atomic Enterprise" \
    --admin=joe

This command creates a project:
* with the id `demo`
* with a display name
* with a description
* with an administrative user `joe` who can login with the password defined by
    htpasswd

Future use of command line statements will have to reference this project in
order for things to land in the right place.

## Your First Application
At this point you essentially have a sufficiently-functional AE
environment. It is now time to create the classic "Hello World" application
using some sample code.  But, first, some housekeeping.

Also, don't forget, the materials for these labs are in your
`~/training/eap-latest` folder.

### Resources
There are a number of different resource types in AE, and, essentially,
going through the motions of creating/destroying apps, scaling, building and
etc. all ends up manipulating AE and Kubernetes resources under the
covers. Resources can have quotas enforced against them, so let's take a moment
to look at some example JSON for project resource quota might look like:

[//]: # (TODO: check, what will be correct version of api)


    {
      "apiVersion": "v1",
      "kind": "ResourceQuota",
      "metadata": {
        "name": "test-quota"
      },
      "spec": {
        "hard": {
          "memory": "512Mi",
          "cpu": "200m",
          "pods": "3",
          "services": "3",
          "replicationcontrollers": "3",
          "resourcequotas": "1"
        }
      }
    }

The above quota (simply called *test-quota*) defines limits for several
resources. In other words, within a project, users cannot "do stuff" that will
cause these resource limits to be exceeded. Since quota is enforced at the
project level, it is up to the users to allocate resources (more specifically,
memory and CPU) to their pods/containers. AE will soon provide sensible
defaults.

* Memory

    The memory figure is in bytes, but various other suffixes are supported (eg:
    Mi (mebibytes), Gi (gibibytes), etc.

* CPU

    CPU is a little tricky to understand. The unit of measure is actually a
    "Kubernetes Compute Unit" (KCU, or "kookoo"). The KCU is a "normalized" unit
    that should be roughly equivalent to a single hyperthreaded CPU core.
    Fractional assignment is allowed. For fractional assignment, the
    **m**illicore may be used (eg: 200m = 0.2 KCU)

More details on CPU will come later.

We will get into a description of what pods, services and replication
controllers are over the next few labs. Lastly, we can ignore "resourcequotas",
as it is a bit of a trick so that Kubernetes doesn't accidentally try to apply
two quotas to the same namespace.

### Applying Quota to Projects
At this point we have created our "demo" project, so let's apply the quota above
to it.

```
    cd training/eap-latest
    oc create -f quota.json --namespace=demo
```

If you want to see that it was created:

```
    oc get -n demo quota
    NAME
    test-quota
```

And if you want to verify limits or examine usage:

    oc describe quota test-quota -n demo
    Name:                   test-quota
    Resource                Used    Hard
    --------                ----    ----
    cpu                     0m      200m
    memory                  0       512Mi
    pods                    0       3
    replicationcontrollers  0       3
    resourcequotas          1       1
    services                0       3

**Note:** Once creating the quota, it can take a few moments for it to be fully
processed. If you get blank output from the `get` or `describe` commands, wait a
few moments and try again.

### Applying Limit Ranges to Projects
In order for quotas to be effective you need to also create Limit Ranges
which set the maximum, minimum, and default allocations of memory and cpu at
both a pod and container level. Without default values for containers projects
with quotas will fail because the deployer and other infrastructure pods are
unbounded and therefore forbidden.

Still in the `training/eap-latest` directory:
```
    oc create -f limits.json --namespace=demo
```

Review your limit ranges

    oc describe limitranges limits -n demo
    Name:           limits
    Type            Resource        Min     Max     Default
    ----            --------        ---     ---     ---
    Pod             memory          5Mi     750Mi   -
    Pod             cpu             10m     500m    -
    Container       cpu             10m     500m    100m
    Container       memory          5Mi     750Mi   100Mi

### Login
Since we have taken the time to create the *joe* user as well as a project for
him, we can log into a terminal as *joe* and then set up the command line
tooling.

Open a terminal as `joe`:

    # su - joe

Then, execute:

    oc login -u joe \
    --certificate-authority=/etc/origin/master/ca.crt \
    --server=https://ae-master.example.com:8443

Atomic Enterprise, by default, is using a self-signed SSL certificate, so we must point
our tool at the CA file.

The `login` process created a file called `config` in the `~/.kube/config`
folder. Take a look at it, and you'll see something like the following:

[//]: # (TODO: /var/lib/openshift/openshift.local.certificates -> ???)

```
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: ../../../../etc/origin/master/ca.crt
        server: https://ae-master.example.com:8443
      name: ae-master-example-com-8443
    contexts:
    - context:
        cluster: ae-master-example-com-8443
        namespace: demo
        user: joe/ae-master-example-com:8443
      name: demo/ae-master-example-com:8443/joe
    current-context: demo/ae-master-example-com:8443/joe
    kind: Config
    preferences: {}
    users:
    - name: joe/ae-master-example-com:8443
      user:
        token: ZmQwMjBiZjUtYWE3OC00OWE1LWJmZTYtM2M2OTY2OWM0ZGIw
```

This configuration file has an authorization token, some information about where
our server lives, our project, etc.

**Note:** See the [troubleshooting guide](#appendix---troubleshooting) for
details on how to fetch a new token once this one expires.  The installer sets
the default token lifetime to 4 hours.

### Grab the Training Repo Again
Since Joe and Alice can't access the training folder in root's home directory,
go ahead and grab it inside Joe's home folder:

    cd
    git clone https://github.com/projectatomic/atomic-enterprise-training.git training
    cd ~/training/eap-latest

### The Hello World Definition JSON
In the `eap-latest` training folder, you can see the contents of our pod definition by
using `cat`:

```
    cat hello-pod.json
    {
      "kind": "Pod",
      "apiVersion": "v1",
      "metadata": {
        "name": "hello-atomic",
        "creationTimestamp": null,
        "labels": {
          "name": "hello-atomic"
        }
      },
      ...
```

A *pod* is a *group* of Docker containers that are always scheduled
together on the same node, and can access each other's exposed ports.
If you are familiar with OpenShift V2 terminology, it is somewhat
similar to a *gear*.  Reality is more complex, and we will learn more
about the terms as we explore AE further.

### Run the Pod
As `joe`, to create the pod from our JSON file, execute the following:

    oc create -f hello-pod.json

Remember, we've "logged in" to AE and our project, so this will create
the pod inside of it. The command should display the ID of the pod:

    pods/hello-atomic

Issue a `get pods` to see overview of what was defined:

    oc get pods
    NAME           READY     REASON    RESTARTS   AGE
    hello-atomic   1/1       Running   0          7s

You may want to know more about the `hello-atomic` pod:

    oc describe pod hello-atomic
    Name:                           hello-atomic
    Image(s):                       atomicenterprise/hello-atomic:v0.5.2.2
    Host:                           os-node2.example.com/192.168.133.4
    Labels:                         name=hello-atomic
    Status:                         Running
    IP:                             10.1.1.5
    Replication Controllers:        <none>
    Containers:
      hello-atomic:
        Image:              atomicenterprise/hello-atomic:v0.5.2.2
        State:              Running
          Started:          Wed, 08 Jul 2015 17:27:33 +0200
        Ready:              True
        Restart Count:      0
    ...

However, we have not yet created a *service* for this pod, and thus it
won't be exported by the system.  But if we had other applications running,
this pod could talk to them.

Note that this hello-atomic pod is distinct from the one you launched
earlier as an administrator.

You should again be able to access it via:

```
    curl http://$IP_FROM_ABOVE:8080/
```

##### Inspecting the Docker state

Docker is the container infrastructure underlying Kubernetes.  Let's
take a moment to look at what is going on at that level.

On the node where the pod is running (`HOST`), look at the list of
Docker containers with `docker ps` (in a `root` terminal) to see the
bound ports.  We should see an `openshift3/ose-pod` container bound to
36061 on the host and bound to 8080 on the container, along with
several other `ose-pod` containers.

```
# docker ps
abd9061cf2fd        atomicenterprise/hello-atomic   "/hello-atomic"     19 minutes ago      Up 19 minutes                           k8s_hello-atomic.65804d4d_hello-atomic_demo_71c467dc-4db3-11e5-a843-fa163e472414_96731d91   
e8af42b67175        aos3/aos-pod:v3.0.1.100         "/pod"              19 minutes ago      Up 19 minutes                           k8s_POD.2d6df2fa_hello-atomic_demo_71c467dc-4db3-11e5-a843-fa163e472414_b47f195f
```

The `aos3/aos-pod` container exists because of the way network
namespacing works in Kubernetes. For the sake of simplicity, think of
the container as nothing more than a way for the host OS to get an
interface created for the corresponding pod to be able to receive
traffic. Deeper understanding of networking in AE is outside the scope
of this material.

### Delete the Pod
As `joe`, go ahead and delete this pod so that you don't get confused in later examples:

    oc delete pod hello-atomic

Take a moment to think about what this pod exercise really did -- it referenced
an arbitrary Docker image, made sure to fetch it (if it wasn't present), and
then ran it. This could have just as easily been an application from an ISV
available in a registry or something already written and built in-house.

This is really powerful. We will explore using "arbitrary" Docker images later.

### Quota Enforcement
Since we know we can run a pod directly, we'll go through a simple quota
enforcement exercise. The `hello-quota` JSON will attempt to create four
instances of the "hello-atomic" pod. It will fail when it tries to create the
fourth, because the quota on this project limits us to three total pods.

Go ahead and use `oc create` and you will see the following:

    oc create -f hello-quota.json
    pods/hello-atomic-1
    pods/hello-atomic-2
    pods/hello-atomic-3
    Error from server: Pod "hello-atomic-4" is forbidden: Limited to 3 pods

Let's delete these pods quickly. As `joe` again:

    oc delete pod --all

**Note:** You can delete most resources using "--all" but there is *no sanity
check*. Be careful.

## Services

From the [Kubernetes
documentation](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md):

    A Kubernetes service is an abstraction which defines a logical set of pods and a
    policy by which to access them - sometimes called a micro-service. The goal of
    services is to provide a bridge for non-Kubernetes-native applications to access
    backends without the need to write code that is specific to Kubernetes. A
    service offers clients an IP and port pair which, when accessed, redirects to
    the appropriate backends. The set of pods targeted is determined by a label
    selector.

If you think back to the simple pod we created earlier, there was a "label":

      "labels": {
        "name": "hello-atomic"
      },

Now, let's look at a *service* definition:

```
$ cat hello-service.json	
{
    "kind": "Service",
    "apiVersion": "v1",
    "metadata": {
        "name": "hello-atomic-service"
    },
    "spec": {
        "selector": {
            "name": "hello-atomic"
        },
        "ports": [
            {
		"protocol": "TCP",
		"port": 8080
            }
        ]
    }
}
```

The *service* has a `selector` element. In this case, it is a key:value pair of
`name:hello-atomic`. If you looked at the output of `oc get pods` on your
master, you saw that the `hello-atomic` pod has a label:

    name=hello-atomic

The definition of the *service* tells Kubernetes that any pods with the label
"name=hello-atomic" are associated, and should have traffic distributed
amongst them. In other words, the service itself is the "connection to the
network", so to speak, or the input point to reach all of the pods.

By default, the security policy for non-cluster administrators does
not allow binding to specific ports on the host.  Let's make the
hello-atomic app exposed across the cluster with the above service.

We first recreate the pod, as the quota demo deleted earlier versions.

```
    oc create -f hello-pod.json
    oc create -f hello-service.json
    oc describe services/hello-atomic-service
```

Note the `Endpoints: 10.1.0.6:8080` entry.  You should be able to
access the `hello-atomic` pod from *any* node within the cluster using
`curl`:

```
$ curl http://10.1.0.6:8080
Hello Atomic!
```

## Optional: Routing

Services are a way for pods inside your cluster to talk to each other.
As mentioned above, if you want *external* access, Atomic Enterprise
comes with a "router" component.  This is based on the robust HAProxy
project, and can perform things like common SSL terminaton.

In a simplification of the process, the `aos3/aos3-haproxy-router`
container we will create below is a pre-configured instance of HAProxy
as well as some of the AE framework. The Atomic instance running in
this container watches for route resources on the Atomic master.

Here is an example route resource JSON definition:

    {
      "kind": "Route",
      "apiVersion": "v1",
      "metadata": {
        "name": "hello-atomic-route"
      },
      "spec": {
        "host": "hello-atomic.cloudapps.example.com",
        "to": {
          "name": "hello-atomic-service"
        },
        "tls": {
          "termination": "edge"
        }
      }
    }

When the `oc` command is used to create this route, a new instance of a route
*resource* is created inside AE's data store. This route resource is
affiliated with a service.

The HAProxy/Router is watching for changes in route resources. When a new route
is detected, an HAProxy pool is created. When a change in a route is detected,
the pool is updated.

This HAProxy pool ultimately contains all pods that are in a service. Which
service? The service that corresponds to the `serviceName` directive that you
see above.

You'll notice that the definition above specifies TLS edge termination. This
means that the router should provide this route via HTTPS. Because we provided
no certificate info, the router will provide the default SSL certificate when
the user connects. Because this is edge termination, user connections to the
router will be SSL encrypted but the connection between the router and the pods
is unencrypted.

It is possible to utilize various TLS termination mechanisms, and more details
is provided in the router documentation:

    http://docs.openshift.org/latest/architecture/core_objects/routing.html#securing-routes

We'll see this edge termination in action shortly.

### Creating a Wildcard Certificate

In order to serve a valid certificate for secure access to applications in our
cloud domain, we will need to create a key and wildcard certificate that the
router will use by default for any routes that do not specify a key/cert of their
own. Atomic supplies a command for creating a key/cert signed by the AE's
CA which we will use.

On the master, as `root`:

    CA=/etc/origin/master
    oadm create-server-cert --signer-cert=$CA/ca.crt \
          --signer-key=$CA/ca.key --signer-serial=$CA/ca.serial.txt \
          --hostnames='*.cloudapps.example.com' \
          --cert=cloudapps.crt --key=cloudapps.key

Now we need to combine `cloudapps.crt` and `cloudapps.key` with the CA into
a single PEM format file that the router needs in the next step.

    cat cloudapps.crt cloudapps.key $CA/ca.crt > cloudapps.router.pem

Make sure you remember where you put this PEM file.

### Creating the Router

The router is the ingress point for all traffic destined for AE
services. It currently supports only HTTP(S) traffic (and "any"
TLS-enabled traffic via SNI). While it is called a "router", it is essentially a
proxy.

The `aos3/aos-haproxy-router` container listens on the host network
interface unlike most containers that listen only on private IPs. The
router proxies external requests for route names to the IPs of actual
pods identified by the service associated with the route.

AE's admin command set enables you to deploy router pods automatically.

Follow these instructions from the OpenShift documentation:

      https://docs.openshift.org/latest/admin_guide/install/deploy_router.html#haproxy-router

Let's check the pods:

    oc get pods

In the output, you should see the router READY state change to `1/1` after a
few moments (it may take up to a few minutes):

    oc get pods
    NAME              READY     REASON    RESTARTS   AGE
    router-1-deploy   1/1       Running   0          21s

In the above router creation command (`oadm router...`) we also specified
`--selector`. This flag causes a `nodeSelector` to be placed on all of the pods
created. If you think back to our "regions" and "zones" conversation, the
AE environment is currently configured with an *infra*structure region
called "infra". This `--selector` argument asks AE:

*Please place all of these router pods in the infra region*.

### Router Placement By Region
In the very beginning of the documentation, we indicated that a wildcard DNS
entry is required and should point at the master. When the router receives a
request for an FQDN that it knows about, it will proxy the request to a pod for
a service. But, for that FQDN request to actually reach the router, the FQDN has
to resolve to whatever the host is where the router is running. Remember, the
router is bound to ports 80 and 443 on the *host* interface. Since our wildcard
DNS entry points to the public IP address of the master, the `--selector` flag
used above ensures that the router is placed on our master as it's the only node
with the label `region=infra`.

For a true HA implementation, one would want multiple "infra" nodes and
multiple, clustered router instances. We will describe this later.

### Viewing Router Stats
Haproxy provides a stats page that's visible on port 1936 of your router host.
Currently the stats page is password protected with a static password, this
password will be generated using a template parameter in the future, for now the
password is `cEVu2hUb` and the username is `admin`.

To make this accessible publicly, you will need to open this port on your master:

    iptables -I OS_FIREWALL_ALLOW -p tcp -m tcp --dport 1936 -j ACCEPT

You will also want to add this rule to `/etc/sysconfig/iptables` as well to keep it
across reboots. However, don't restart the iptables service, as this would destroy
docker networking. Use the `iptables` command to change rules on a live system.

Feel free to not open this port if you don't want to make this accessible, or if
you only want it accessible via port forwarding, etc.

**Note**: Unlike OpenShift v2 this router is not specific to a given project, as
such it's really intended to be viewed by cluster admins rather than project
admins.

Ensure that port 1936 is accessible and visit:

    http://admin:cEVu2hUb@ae-master.example.com:1936 

to view your router stats.

## The Complete Pod-Service-Route
With a router now available, let's take a look at an entire
Pod-Service-Route definition template and put all the pieces together.

Don't forget -- the materials are in `~/training/eap-latest`.

### Creating the Definition

Let's look at `test-complete.json`, which is a complete definition for
a pod with a corresponding service and a corresponding route. It also
includes a deployment configuration.

```
cat test-complete.json
{
  "kind": "List",
  "apiVersion": "v1",
  "metadata": {
    "name": "hello-service-complete-example"
  },
  "items": [
    {
...
```

In the JSON above:

* There is a pod whose containers have the label `name=hello-atomic-label` and the nodeSelector `region=primary`
* There is a service:
  * with the id `hello-atomic-service`
  * with the selector `name=hello-atomic`
* There is a route:
  * with the FQDN `hello-atomic.cloudapps.example.com`
  * with the `spec` `to` `name=hello-atomic-service`

If we work from the route down to the pod:

* The route for `hello-atomic.cloudapps.example.com` has an HAProxy pool
* The pool is for any pods in the service whose ID is `hello-atomic-service`,
    via the `serviceName` directive of the route.
* The service `hello-atomic-service` includes every pod who has a label
    `name=hello-atomic`
* There is a single pod with a single container that has the label
    `name=hello-atomic`

If you are not using the `example.com` domain you will need to edit the route
portion of `test-complete.json` to match your DNS environment.

**Logged in as `joe`,** go ahead and use `oc` to create everything:

    oc create -f test-complete.json

You should see something like the following:

    services/hello-atomic-service
    routes/hello-atomic-route
    pods/hello-atomic

You can verify this with other `oc` commands:

    oc get pods

    oc get services

    oc get routes

**Note:** May need to force resize:

    https://github.com/openshift/origin/issues/2939

### Project Status
AE provides a handy tool, `oc status`, to give you a summary of
common resources existing in the current project:

    oc status
    In project Atomic Enterprise Demo (demo)

    service hello-atomic-service (172.30.196.23:27017 -> 8080)
      hello-atomic deploys docker.io/openshift/hello-atomic:v0.5.2.2
        #1 deployed 3 minutes ago - 1 pod

    To see more information about a Service or DeploymentConfig, use 'oc describe service <name>' or 'oc describe dc <name>'.
    You can use 'oc get all' to see lists of each of the types described above.

`oc status` does not yet show bare pods or routes.

### Verifying the Service
Services are not externally accessible without a route being defined, because
they always listen on "local" IP addresses (eg: 172.x.x.x). However, if you have
access to the AE environment, you can still test a service.

    oc get services
    NAME                   LABELS    SELECTOR                  IP              PORT(S)
    hello-atomic-service   <none>    name=hello-atomic-label   172.30.17.229   27017/TCP

We can see that the service has been defined based on the JSON we used earlier.
If the output of `oc get pods` shows that our pod is running, we can try to
access the service:

    curl `oc get services | grep hello-atomic | awk '{print $4":"$5}' | sed -e 's/\/.*//'`
    Hello Atomic!

This is a good sign! It means that, if the router is working, we should be able
to access the service via the route.

### Verifying the Routing
Verifying the routing is a little complicated, but not terribly so. Since we
specified that the router should land in the "infra" region, we know that its
Docker container is on the master. Log in there as `root`.

We can use `oc exec` to get a bash interactive shell inside the running
router container. The following command will do that for us:

    oc exec -it -p $(oc get pods | grep router | awk '{print $1}' | head -n 1) /bin/bash

You are now in a bash session *inside* the container running the router.

Since we are using HAProxy as the router, we can cat the `routes.json` file:

    cat /var/lib/containers/router/routes.json

If you see some content that looks like:

    "demo/hello-atomic-service": {
      "Name": "demo/hello-atomic-service",
      "EndpointTable": {
        "10.1.0.9:8080": {
          "ID": "10.1.0.9:8080",
          "IP": "10.1.0.9",
          "Port": "8080"
        }
      },
      "ServiceAliasConfigs": {
        "demo-hello-atomic-route": {
          "Host": "hello-atomic.cloudapps.example.com",
          "Path": "",
          "TLSTermination": "edge",
          "Certificates": {
            "hello-atomic.cloudapps.example.com": {
              "ID": "demo-hello-atomic-route",
              "Contents": "",
              "PrivateKey": ""
            }
          },
          "Status": "saved"
        }
      }
    }

You know that "it" worked -- the router watcher detected the creation of the
route in AE and added the corresponding configuration to HAProxy.

Go ahead and `exit` from the container.

    [root@router-1-2yefi /]# exit
    exit

You can reach the route securely and check that it is using the right certificate:

    curl --cacert /etc/origin/master/ca.crt \
             https://hello-atomic.cloudapps.example.com
    Hello Atomic!

And:

    openssl s_client -connect hello-atomic.cloudapps.example.com:443 \
                       -CAfile /etc/origin/master/ca.crt
    CONNECTED(00000003)
    depth=1 CN = openshift-signer@1430768237
    verify return:1
    depth=0 CN = *.cloudapps.example.com
    verify return:1
    [...]

Since we used AE's CA to create the wildcard SSL certificate, and since
that CA is not "installed" in our system, we need to point our tools at that CA
certificate in order to validate the SSL certificate presented to us by the
router. With a CA or all certificates signed by a trusted authority, it would
not be necessary to specify the CA everywhere.

## Project Administration
When we created the `demo` project, `joe` was made a project administrator. As
an example of an administrative function, if `joe` now wants to let `alice` look
at his project, with his project administrator rights he can add her using the
`oadm policy` command:

    [joe]$ oadm policy add-role-to-user view alice

**Note:** `oadm` will act, by default, on whatever project the user has
selected. If you recall earlier, when we logged in as `joe` we ended up in the
`demo` project. We'll see how to switch projects later.

Go back to root, then open a new terminal window as the `alice` user:

```
    su - alice
```

[//]: # (TODO: fix ca path)
[//]: # (TODO: fix "Authentication required ... XXX" text)

and login to Atomic Enterprise:

    oc login -u alice \
    --certificate-authority=/etc/origin/master/ca.crt \
    --server=https://ae-master.example.com:8443

    Authentication required for https://ae-master.example.com:8443 (openshift)
    Password:  <redhat>
    Login successful.

    Using project "demo"

`alice` has no projects of her own yet (she is not an administrator of
anything), so she is automatically configured to look at the `demo` project
since she has access to it. She has "view" access, so `oc status` and `oc get
pods` and so forth should show her the same thing as `joe`:

    [alice]$ oc get pods
    NAME           READY     REASON    RESTARTS   AGE
    hello-atomic   1/1       Running   0          4s

However, she cannot make changes:

    [alice]$ oc delete pod hello-atomic
    Error from server: User "alice" cannot delete pods in project "demo"

`joe` could also give `alice` the role of `edit`, which gives her access
to do nearly anything in the project except adjust access.

    [joe]$ oadm policy add-role-to-user edit alice

Now she can delete that pod if she wants, but she can not add access for
another user or upgrade her own access. To allow that, `joe` could give
`alice` the role of `admin`, which gives her the same access as himself.

    [joe]$ oadm policy add-role-to-user admin alice

There is no "owner" of a project, and projects can certainly be created
without any administrator. `alice` or `joe` can remove the `admin`
role (or all roles) from each other or themselves at any time without
affecting the existing project.

    [joe]$ oadm policy remove-user joe

Check `oadm policy help` for a list of available commands to modify
project permissions. Atomic Enterprise RBAC is extremely flexible. The roles
mentioned here are simply defaults - they can be adjusted (per-project
and per-resource if needed), more can be added, groups can be given
access, etc. Check the documentation for more details:

* http://docs.openshift.org/latest/dev_guide/authorization.html
* https://github.com/openshift/origin/blob/master/docs/proposals/policy.md

Of course, there be dragons. The basic roles should suffice for most uses.

**Note:** There is a bug that actually prevents the remove-user from removing
the user:

https://github.com/openshift/origin/issues/2785

It appears to be fixed but may not have made Early Access release.

### Deleting a Project
Since we are done with this "demo" project, and since the `alice` user is a
project administrator, let's go ahead and delete the project. This should also
end up deleting all the pods, and other resources, too.

As the `alice` user:

    oc delete project demo

If you switch to the `root` user and issue `oc get project` you will see that
the demo project's status is "Terminating". If you do an `oc get pod -n demo`
you may see the pods, still. It takes about 60 seconds for the project deletion
cleanup routine to finish.

Once the project disappears from `oc get project`, doing `oc get pod -n demo`
should return no results.

## The registry, imagestreams, and more

Atomic Enterprise comes with an integrated Docker registry, with
support for "ImageStreams" which record which supports tracking and
reacting to new base images.  In OpenShift, ImageStreams are integrated
with the S2I image builder.

For more information on setting up the built-in registry, see [registry-and-imagestreams](registry-and-imagestreams.md).

## Conclusion
This concludes the Early Access Program training. Look for more example
applications to come!

# APPENDIX - Docker Storage Setup
**IMPORTANT:** The default docker storage configuration uses loopback devices
and is not appropriate for production. Red Hat considers the dm.thinpooldev
storage option to be the only appropriate configuration for production use.

To configure the storage for Docker, you'll need to first install Docker.
```
yum -y install docker
```

In order to use dm.thinpooldev you must have an LVM thinpool available, the
[docker-storage-setup](https://github.com/projectatomic/docker-storage-setup/)
package will assist you in configuring LVM. However you must provision your
host to fit one of these three scenarios :

*  Root filesystem on LVM with free space remaining on the volume group. Run
`docker-storage-setup` with no additional configuration, it will allocate the
remaining space for the thinpool.

*  A dedicated LVM volume group where you'd like to create your thinpool

        cat <<EOF > /etc/sysconfig/docker-storage-setup
        VG=docker-vg
        SETUP_LVM_THIN_POOL=yes
        EOF
        docker-storage-setup

*  A dedicated block device, which will be used to create a volume group and thinpool

        cat <<EOF > /etc/sysconfig/docker-storage-setup
        DEVS=/dev/vdc
        VG=docker-vg
        SETUP_LVM_THIN_POOL=yes
        EOF
        docker-storage-setup

Once complete you should have a thinpool named `docker-pool` and docker should
be configured to use it in `/etc/sysconfig/docker-storage`.

    # lvs
    LV                  VG        Attr       LSize  Pool Origin Data%  Meta% Move Log Cpy%Sync Convert
    docker-pool         docker-vg twi-a-tz-- 48.95g             0.00   0.44

    # cat /etc/sysconfig/docker-storage
    DOCKER_STORAGE_OPTIONS=--storage-opt dm.fs=xfs --storage-opt dm.thinpooldev=/dev/mapper/openshift--vg-docker--pool

**Note:** If you had previously used docker with loopback storage you should
clean out `/var/lib/docker` This is a destructive operation and will delete all
images and containers on the host.

    systemctl stop docker
    rm -rf /var/lib/docker/*
    systemctl start docker

# APPENDIX - DNSMasq setup
[dnsmasq.conf](./eap-latest/dnsmasq.conf) file and a sample [hosts](./eap-latest/hosts)
file. If you do not have the ability to manipulate DNS in your
environment, or just want a quick and dirty way to set up DNS, you can
install dnsmasq on one of your nodes. Do **not** install DNSMasq on
your master. OpenShift now has an internal DNS service provided by
Go's "SkyDNS" that is used for internal service communication.

    yum -y install dnsmasq

Copy your current `/etc/resolv.conf` to a new file such as
`/etc/resolv.conf.upstream`.  Ensure you *only* have an upstream
resolver there (eg: Google DNS @ `8.8.8.8`), not the address of your
dnsmasq server.

Enable and start the dnsmasq service:

    systemctl enable dnsmasq; systemctl start dnsmasq

You will need to ensure the following, or fix the following:

* Your IP addresses match the entries in `/etc/hosts`
* Your hostnames for your machines match the entries in `/etc/hosts`
* Your `cloudapps` domain points to the correct node ip in `dnsmasq.conf`
* Each of your systems has the same `/etc/hosts` file
* Your master and nodes `/etc/resolv.conf` points to the IP address of the node
  running DNSMasq as the first nameserver
* Your dnsmasq instance uses the `resolv-file` option to point to `/etc/resolv.conf.upstream` only.
* That you also open port 53 (TCP and UDP) to allow DNS queries to hit the node

Following this setup for dnsmasq will ensure that your wildcard domain works,
that your hosts in the `example.com` domain resolve, that any other DNS requests
resolve via your configured local/remote nameservers, and that DNS resolution
works inside of all of your containers. Don't forget to start and enable the
`dnsmasq` service.

### Verifying DNSMasq

You can query the local DNS on the master using `dig` (provided by the
`bind-utils` package) to make sure it returns the correct records:

    dig ae-master.example.com

    ...
    ;; ANSWER SECTION:
    ae-master.example.com. 0  IN  A 192.168.133.2
    ...

The returned IP should be the public interface's IP on the master. Repeat for
your nodes. To verify the wildcard entry, simply dig an arbitrary domain in the
wildcard space:

    dig foo.cloudapps.example.com

    ...
    ;; ANSWER SECTION:
    foo.cloudapps.example.com 0 IN A 192.168.133.2
    ...
    
[//]: # (TODO: LDAP basic auth service requires STI - find a way around it)

# APPENDIX - Import/Export of Docker Images (Disconnected Use)
Docker supports import/save of Images via tarball. These instructions are
general and may not be 100% accurate for the current release. You can do
something like the following on your connected machine:

[//]: # (TODO: change image names?)

    docker pull registry.access.redhat.com/openshift3/ose-haproxy-router
    docker pull registry.access.redhat.com/openshift3/ose-deployer
    docker pull registry.access.redhat.com/openshift3/ose-pod
    docker pull registry.access.redhat.com/openshift3/ose-docker-registry
    docker pull atomicenterprise/hello-atomic

This will fetch all of the images. You can then save them to a tarball:

[//]: # (TODO: change image names?)

    docker save -o beta4-images.tar \
    registry.access.redhat.com/openshift3/ose-haproxy-router \
    registry.access.redhat.com/openshift3/ose-deployer \
    registry.access.redhat.com/openshift3/ose-pod \
    registry.access.redhat.com/openshift3/ose-docker-registry \
    atomicenterprise/hello-atomic

**Note: On an SSD-equipped system this took ~2 min and uses 1.8GB of disk
space**

Sneakernet that tarball to your disconnected machines, and then simply load the
tarball:

    docker load -i beta1-images.tar

**Note: On an SSD-equipped system this took ~4 min**

# APPENDIX - Cleaning Up
Figuring out everything that you have deployed is a little bit of a bear right
now. The following command will show you just about everything you might need to
delete. Be sure to change your context across all the namespaces and the
master-admin to find everything:

    for resource in build buildconfig images imagestream deploymentconfig \
    route replicationcontroller service pod; do echo -e "Resource: $resource"; \
    oc get $resource; echo -e "\n\n"; done

Deleting a project with `oc delete project` should delete all of its resources,
but you may need help finding things in the default project (where
infrastructure items are). Deleting the default project is not recommended.

# APPENDIX - Troubleshooting

An experimental diagnostics command is in progress for Atomic Enterprise.
Once merged it should be available as `origin ex diagnostics`. There may
be out-of-band updated versions of diagnostics under
[Luke Meyer's release page](https://github.com/sosiouxme/origin/releases).
Running this may save you some time by pointing you in the right direction
for common issues. This is very much still under development however.

Common problems

* All of a sudden authentication seems broken for non-admin users.  Whenever I run oc commands I see output such as:

        F0310 14:59:59.219087   30319 get.go:164] request
        [&{Method:GET URL:https://ae-master.example.com:8443/api/v1beta1/pods?namespace=demo
        Proto:HTTP/1.1 ProtoMajor:1 ProtoMinor:1 Header:map[] Body:<nil> ContentLength:0 TransferEncoding:[]
        Close:false Host:ae-master.example.com:8443 Form:map[] PostForm:map[]
        MultipartForm:<nil> Trailer:map[] RemoteAddr: RequestURI: TLS:<nil>}]
        failed (401) 401 Unauthorized: Unauthorized

    In most cases if admin (certificate) auth is still working this means the
    token is invalid.  Soon there will be more polish in the oc tooling to
    handle this edge case automatically but for now the simplist thing to do is
    to recreate the client config.

[//]: # (TODO: ~/.config/openshift, ca paths)


        # The login command creates a .kubeconfig file in the CWD.
        # But we need it to exist in ~/.kube
        cd ~/.kube

        # If a stale token exists it will prevent the beta4 login command from working
        rm .kubeconfig

        oc login \
        --certificate-authority=/etc/origin/master/ca.crt \
        --cluster=master --server=https://ae-master.example.com:8443 \
        --namespace=[INSERT NAMESPACE HERE]

* When using an "oc" command like "oc get pods" I see a "certificate signed by
    unknown authority error":

        F0212 16:15:52.195372   13995 create.go:79] Post
        https://ae-master.example.net:8443/api/v1beta1/pods?namespace=default:
        x509: certificate signed by unknown authority

    Check the value of $KUBECONFIG:

        echo $kubeconfig

    If you don't see anything, you may have changed your `.bash_profile` but
    have not yet sourced it. Make sure that you followed the step of adding
    `$KUBECONFIG`'s export to your `.bash_profile` and then source it:

        source ~/.bash_profile

* When issuing a `curl` to my service, I see `curl: (56) Recv failure:
    Connection reset by peer`

    It can take as long as 90 seconds for the service URL to start working.
    There is some internal house cleaning that occurs inside Kubernetes
    regarding the endpoint maps.

    If you look at the log for the node, you might see some messages about
    looking at endpoint maps and not finding an endpoint for the service.

    To find out if the endpoints have been updated you can run:

    `oc describe service $name_of_service` and check the value of `Endpoints:`

# APPENDIX - Infrastructure Log Aggregation

[//]: # (TODO: /var/log/openshift -> ???)

Given the distributed nature of Atomic Enterprise you may find it beneficial to
aggregate logs from your AE infrastructure services. By default, AE
services log to the systemd journal and rsyslog persists those log messages to
`/var/log/messages`. We'll reconfigure rsyslog to write these entries to
`/var/log/openshift` and configure the master host to accept log data from the
other hosts.

## Enable Remote Logging on Master
Uncomment the following lines in your master's `/etc/rsyslog.conf` to enable
remote logging services.

    $ModLoad imtcp
    $InputTCPServerRun 514

Restart rsyslog

    systemctl restart rsyslog

[//]: # (TODO: /var/log/openshift -> ???)

## Enable logging to /var/log/openshift
On your master update the filters in `/etc/rsyslog.conf` to divert openshift logs to `/var/log/openshift`

    # Log openshift processes to /var/log/openshift
    :programname, contains, "openshift"                     /var/log/openshift

    # Log anything (except mail) of level info or higher.
    # Don't log private authentication messages!
    # Don't log openshift processes to /var/log/messages either
    :programname, contains, "openshift" ~
    *.info;mail.none;authpriv.none;cron.none                /var/log/messages

Restart rsyslog

    systemctl restart rsyslog

## Configure nodes to send atomic logs to your master
On your other hosts send openshift logs to your master by adding this line to
`/etc/rsyslog.conf`

    :programname, contains, "openshift" @@ae-master.example.com

Restart rsyslog

    systemctl restart rsyslog

[//]: # (TODO: /var/log/openshift -> ???)

Now all your openshift related logs will end up in `/var/log/openshift` on your
master.

## Optionally Log Each Node to a unique directory
You can also configure rsyslog to store logs in a different location
based on the source host. On your master, add these lines immediately prior to
`$InputTCPServerRun 514`

    $template TmplMsg, "/var/log/remote/%HOSTNAME%/%PROGRAMNAME:::secpath-replace%.log"
    $RuleSet remote1
    authpriv.*   ?TmplAuth
    *.info;mail.none;authpriv.none;cron.none   ?TmplMsg
    $RuleSet RSYSLOG_DefaultRuleset   #End the rule set by switching back to the default rule set
    $InputTCPServerBindRuleset remote1  #Define a new input and bind it to the "remote1" rule set

Restart rsyslog

    systemctl restart rsyslog

Now logs from remote hosts will go to `/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log`

See these documentation sources for additional rsyslog configuration information

    https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/System_Administrators_Guide/s1-basic_configuration_of_rsyslog.html
    http://www.rsyslog.com/doc/v7-stable/configuration/filters.html

# APPENDIX - Working with HTTP Proxies

In many production environments direct access to the web is not allowed.  In
these situations there is typically an HTTP(S) proxy available.  Configuring
AE deployments to use these proxies is as simple as setting
standard environment variables.  The trick is knowing where to place them.

## Importing ImageStreams

Since the importer is on the Master we need to make the configuration change
there.  The easiest way to do that is to add environment variables `NO_PROXY`,
`HTTP_PROXY`, and `HTTPS_PROXY` to `/etc/sysconfig/atomic-enterprise-master` then restart
your master.

~~~
HTTP_PROXY=http://USERNAME:PASSWORD@10.0.1.1:8080/
HTTPS_PROXY=https://USERNAME:PASSWORD@10.0.0.1:8080/
NO_PROXY=master.example.com
~~~

It's important that the Master doesn't use the proxy to access itself so make
sure it's listed in the `NO_PROXY` value.

Now restart the Service:
~~~
systemctl restart atomic-enterprise-master
~~~

If you had previously imported ImageStreams without the proxy configuration to can re-run the process as follows:

[//]: # (TODO: openshift namespace ??)

~~~
oc delete imagestreams -n openshift --all
oc create -f image-streams.json -n openshift
~~~

## Setting Environment Variables in Pods

It's not only at build time that proxies are required.  Many applications will
need them too.  In previous examples we used environment variables in
`DeploymentConfig`s to pass in database connection information.  The same can
be done for configuring a `Pod`'s proxy at runtime:

[//]: # (TODO: apiVersion)

    {
      "apiVersion": "v1beta1",
      "kind": "DeploymentConfig",
      "metadata": {
        "name": "frontend"
      },
      "template": {
        "controllerTemplate": {
          "podTemplate": {
            "desiredState": {
              "manifest": {
                "containers": [
                  {
                    "env": [
                      {
                        "name": "HTTP_PROXY",
                        "value": "http://USER:PASSWORD@IPADDR:PORT"
                      },
    ...

## Proxying Docker Pull

This is yet another case where it may be necessary to tunnel traffic through a
proxy.  In this case you can edit `/etc/sysconfig/docker` and add the variables
in shell format:

    NO_PROXY=mycompany.com
    HTTP_PROXY=http://USER:PASSWORD@IPADDR:PORT
    HTTPS_PROXY=https://USER:PASSWORD@IPADDR:PORT

## Future Considerations

We're working to have a single place that administrators can set proxies for
all network traffic.

# APPENDIX - Installing in IaaS Clouds
This appendix contains two "versions" of installation instructions. One is for
"generic" clouds, where the installer does not provision any resources on the
actual cloud (eg: it does not stand up VMs or configure security groups).
Another is specifically for AWS, which can take your API credentials and
configure the entire AWS environment, too.

## Generic Cloud Install

[//]: # (TODO: is OSEv3 some key that needs to be changed?)

    [OSEv3:children]
    masters
    nodes
    
    [OSEv3:vars]
    deployment_type=enterprise
    
    # The default user for the image used
    ansible_ssh_user=ec2-user
    
    # host group for masters
    # The entries should be either the publicly accessible dns name for the host
    # or the publicly accessible IP address of the host.
    [masters]
    ec2-52-6-179-239.compute-1.amazonaws.com
    
    # host group for nodes
    [nodes]
    ec2-52-6-179-239.compute-1.amazonaws.com openshift_node_labels="{'region': 'infra', 'zone': 'default'}" #The master
    ec2-52-4-251-128.compute-1.amazonaws.com openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
    ... <additional node hosts go here> ...

**Testing the Auto-detected Values:**
Run the openshift_facts playbook:

[//]: # (TODO: fix the facts file name)

    cd ~/atomic-enterprise-ansible
    ansible-playbook playbooks/byo/openshift_facts.yml

The output will be similar to:

[//]: # (TODO: result["ansible_facts"]["openshift"])
[//]: # (TODO: use_openshift_sdn ??)

    ok: [10.3.9.45] => {
        "result": {
            "ansible_facts": {
                "openshift": {
                    "common": {
                        "hostname": "ip-172-31-8-89.ec2.internal",
                        "ip": "172.31.8.89",
                        "public_hostname": "ec2-52-6-179-239.compute-1.amazonaws.com",
                        "public_ip": "52.6.179.239",
                        "use_openshift_sdn": true
                    },
                    "provider": {
                      ... <snip> ...
                    }
                }
            },
            "changed": false,
            "invocation": {
                "module_args": "",
                "module_name": "openshift_facts"
            }
        }
    }
    ...

Next, we'll need to override the detected defaults if they are not what we expect them to be

[//]: # (TODO: take care of openshift* variables)

- hostname
  * Should resolve to the internal ip from the instances themselves.
  * openshift_hostname will override.
* ip
  * Should be the internal ip of the instance.
  * openshift_ip will override.
* public hostname
  * Should resolve to the external ip from hosts outside of the cloud
  * provider openshift_public_hostname will override.
* public_ip
  * Should be the externally accessible ip associated with the instance
  * openshift_public_ip will override

To override the the defaults, you can set the variables in your inventory. For example, if using AWS and managing dns externally, you can override the host public hostname as follows:

    [masters]
    ec2-52-6-179-239.compute-1.amazonaws.com openshift_public_hostname=ae-master.public.example.com

**Running ansible:**

    ansible ~/atomic-enterprise-ansible/playbooks/byo/config.yml

## Automated AWS Install With Ansible

**Requirements:**
- ansible-1.8.x
- python-boto

**Assumptions Made:**
- The user's ec2 credentials have the following permissions:
  - Create instances
  - Create EBS volumes
  - Create and modify security groups
    - The following security groups will be created:
      - openshift-v3-training-master
      - openshift-v3-training-node
  - Create and update route53 record sets
- The ec2 region selected is using ec2 classic or has a default vpc and subnets configured.
  - When using a vpc, the default subnets are expected to be configured for auto-assigning a public ip as well.
- If providing a different ami id using the EC2_AMI_ID, it is a cloud-init enabled RHEL-7 image.

**Setup (Modifying the Values Appropriately):**

    export AWS_ACCESS_KEY_ID=MY_ACCESS_KEY
    export AWS_SECRET_ACCESS_KEY=MY_SECRET_ACCESS_KEY
    export EC2_REGION=us-east-1
    export EC2_AMI_ID=ami-12663b7a
    export EC2_KEYPAIR=MY_KEYPAIR_NAME
    export RHN_USERNAME=MY_RHN_USERNAME
    export RHN_PASSWORD=MY_RHN_PASSWORD
    export ROUTE_53_WILDCARD_ZONE=cloudapps.example.com
    export ROUTE_53_HOST_ZONE=example.com

**Clone the atomic-enterprise-ansible repo and configure helpful symlinks:**
    ansible-playbook clone_and_setup_repo.yml

**Configuring the Hosts:**

    ansible-playbook -i inventory/aws/hosts openshift_setup.yml

**Accessing the Hosts:**
Each host will be created with an 'openshift' user that has passwordless sudo configured.

# APPENDIX - Linux, Mac, and Windows clients

The Atomic Enterprise client `oc` is available for Linux, Mac OSX, and Windows. You
can use these clients to perform all tasks in this documentation that make use
of the `oc` command.

## Downloading The Clients

[//]: # (TODO: make this relevant to Atomic Enterprise)

Visit [Download Red Hat OpenShift Enterprise Beta](https://access.redhat.com/downloads/content/289/ver=/rhel---7/0.5.2.2/x86_64/product-downloads)
to download the Beta4 clients. You will need to sign into Customer Portal using
an account that includes the OpenShift Enterprise High Touch Beta entitlements.

## Log In To Your Atomic Environment

You will need to log into your environment using `oc login` as you have
elsewhere. If you have access to the CA certificate you can pass it to `oc` with
the --certificate-authority flag or otherwise import the CA into your host's
certificate authority. If you do not import or specify the CA you will be
prompted to accept an untrusted certificate which is not recommended.

[//]: # (TODO: fix ca cert path)
[//]: # (TODO: update the text with correct output)

The CA is created on your master in `/var/lib/openshift/openshift.local.certificates/ca/cert.crt`

    C:\Users\test\Downloads> oc --certificate-authority="cert.crt"
    OpenShift server [[https://localhost:8443]]: https://ae-master.example.com:8443
    Authentication required for https://ae-master.example.com:8443 (openshift)
    Username: joe
    Password:
    Login successful.

    Using project "demo"

On Mac OSX and Linux you will need to make the file executable

   chmod +x oc

