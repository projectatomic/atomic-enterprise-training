<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Atomic Enterprise Platform Early Access Program](#atomic-enterprise-platform-early-access-program)
  - [Architecture and Requirements](#architecture-and-requirements)
    - [Architecture](#architecture)
    - [Requirements](#requirements)
  - [Setting Up the Environment](#setting-up-the-environment)
    - [Use a Terminal Window Manager](#use-a-terminal-window-manager)
    - [DNS](#dns)
    - [Assumptions](#assumptions)
    - [Git](#git)
    - [Preparing Each VM](#preparing-each-vm)
    - [Grab Docker Images (Optional, Recommended)](#grab-docker-images-optional-recommended)
    - [Clone the Training Repository](#clone-the-training-repository)
  - [Ansible-based Installer](#ansible-based-installer)
    - [Install Ansible](#install-ansible)
    - [Generate SSH Keys](#generate-ssh-keys)
    - [Distribute SSH Keys](#distribute-ssh-keys)
    - [Clone the Ansible Repository](#clone-the-ansible-repository)
    - [Configure Ansible](#configure-ansible)
    - [Modify Hosts](#modify-hosts)
    - [Run the Ansible Installer](#run-the-ansible-installer)
    - [Add Development Users](#add-development-users)
  - [Useful Logs](#useful-logs)
  - [Auth and Projects](#auth-and-projects)
    - [Configuring htpasswd Authentication](#configuring-htpasswd-authentication)
    - [A Project for Everything](#a-project-for-everything)
  - [Your First Application](#your-first-application)
    - ["Resources"](#resources)
    - [Applying Quota to Projects](#applying-quota-to-projects)
    - [Login](#login)
    - [Grab the Training Repo Again](#grab-the-training-repo-again)
    - [The Hello World Definition JSON](#the-hello-world-definition-json)
    - [Run the Pod](#run-the-pod)
    - [Extra Credit](#extra-credit)
    - [Delete the Pod](#delete-the-pod)
    - [Quota Enforcement](#quota-enforcement)
  - [Adding Nodes](#adding-nodes)
    - [Modifying the Ansible Configuration](#modifying-the-ansible-configuration)
  - [Regions and Zones](#regions-and-zones)
    - [Scheduler and Defaults](#scheduler-and-defaults)
    - [The NodeSelector](#the-nodeselector)
    - [Customizing the Scheduler Configuration](#customizing-the-scheduler-configuration)
    - [Restart the Master](#restart-the-master)
    - [Label Your Nodes](#label-your-nodes)
  - [Services](#services)
  - [Routing](#routing)
    - [Creating the Router](#creating-the-router)
    - [Router Placement By Region](#router-placement-by-region)
  - [The Complete Pod-Service-Route](#the-complete-pod-service-route)
    - [Creating the Definition](#creating-the-definition)
    - [Project Status](#project-status)
    - [Verifying the Service](#verifying-the-service)
    - [Verifying the Routing](#verifying-the-routing)
  - [Project Administration](#project-administration)
    - [Deleting a Project](#deleting-a-project)
  - [The Registry](#the-registry)
    - [Registry Placement By Region (optional)](#registry-placement-by-region-optional)
  - [Conclusion](#conclusion)
- [APPENDIX - DNSMasq setup](#appendix---dnsmasq-setup)
    - [Verifying DNSMasq](#verifying-dnsmasq)
- [APPENDIX - Import/Export of Docker Images (Disconnected Use)](#appendix---importexport-of-docker-images-disconnected-use)
- [APPENDIX - Cleaning Up](#appendix---cleaning-up)
- [APPENDIX - Pretty Output](#appendix---pretty-output)
- [APPENDIX - Troubleshooting](#appendix---troubleshooting)
- [APPENDIX - Infrastructure Log Aggregation](#appendix---infrastructure-log-aggregation)
  - [Enable Remote Logging on Master](#enable-remote-logging-on-master)
  - [Enable logging to /var/log/openshift](#enable-logging-to-varlogopenshift)
  - [Configure nodes to send Atomic logs to your master](#configure-nodes-to-send-atomic-logs-to-your-master)
  - [Optionally Log Each Node to a unique directory](#optionally-log-each-node-to-a-unique-directory)
- [APPENDIX - Working with HTTP Proxies](#appendix---working-with-http-proxies)
  - [Importing ImageStreams](#importing-imagestreams)
  - [Setting Environment Variables in Pods](#setting-environment-variables-in-pods)
  - [Proxying Docker Pull](#proxying-docker-pull)
  - [Future Considerations](#future-considerations)
- [APPENDIX - Installing in IaaS Clouds](#appendix---installing-in-iaas-clouds)
  - [Generic Cloud Install](#generic-cloud-install)
    - [An Example Hosts File (/etc/ansible/hosts)](#an-example-hosts-file-etcansiblehosts)
    - [Testing the Auto-detected Values](#testing-the-auto-detected-values)
  - [Automated AWS Install With Ansible](#automated-aws-install-with-ansible)
    - [Requirements:](#requirements-1)
    - [Assumptions Made:](#assumptions-made)
    - [Setup (Modifying the Values Appropriately):](#setup-modifying-the-values-appropriately)
    - [Configuring the Hosts:](#configuring-the-hosts)
    - [Accessing the Hosts:](#accessing-the-hosts)
- [APPENDIX - Linux, Mac, and Windows clients](#appendix---linux-mac-and-windows-clients)
  - [Downloading The Clients](#downloading-the-clients)
  - [Log In To Your Atomic Environment](#log-in-to-your-atomic-environment)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Atomic Enterprise Platform Early Access Program
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

The "node" hosts user applications. The main difference is that "gears" have
been replaced with Docker container instances. You will learn much more about
the inner workings of Atomic throughout the rest of the document.

### Requirements
Each of the virtual machines should have 4+ GB of memory, 20+ GB of disk space,
and the following configuration:

* RHEL 7.1 or RHEL AH 7.1 (Note: 7.1 kernel is required for openvswitch)
* "Minimal" installation option
* NetworkManager **disabled**

The majority of storage requirements are related to Docker and etcd (the data
store). Both of their contents live in /var, so it is recommended that the
majority of the storage be allocated to /var.

[//]: # (TODO: what is a correct subscription name???)
As part of signing up for the beta program, you should have received an
evaluation subscription. This subscription gave you access to the beta software.
You will need to use subscription manager to both register your VMs, and attach
them to the *Atomic Enterprise High Touch Beta* subscription.

All of your VMs should be on the same logical network and be able to access one
another.

Forward DNS resolution of hostnames is an **absolute requirement**. This
training document assumes the following configuration:

* ae-master.example.com (master+node)
* ae-node1.example.com
* ae-node2.example.com

If you cannot create real forward resolving DNS entries in your DNS system, you
will need to set up your own DNS server in the beta testing environment.
Documentation is provided on DNSMasq in an appendix, [APPENDIX - DNSMasq
setup](#appendix---dnsmasq-setup)

## Setting Up the Environment
### Use a Terminal Window Manager
We **strongly** recommend that you use some kind of terminal window manager
(Screen, Tmux).

### DNS
You will need to have a wildcard for a DNS zone resolve, ultimately, to the IP
address of the OpenShift router. For this training, we will ensure that the
router will end up on the OpenShift server that is running the master. Go
ahead and create a wildcard DNS entry for "cloudapps" (or something similar),
with a low TTL, that points to the public IP address of your master.

For example:

    *.cloudapps.example.com. 300 IN  A 192.168.133.2

In almost all cases, when referencing VMs you must use hostnames and the
hostnames that you use must match the output of `hostname -f` on each of your
nodes. By extension, you must at least have all hostname/ip mappings in
/etc/hosts files or forward DNS should work.

It is possible to use dnsmasq inside of your beta environment to handle these
duties. See the [appendix on dnsmasq](#appendix---dnsmasq-setup) if you can't
easily manipulate your existing DNS environment.

### Assumptions
In most cases you will see references to "example.com" and other FQDNs related
to it. If you choose not to use "example.com" in your configuration, that is
fine, but remember that you will have to adjust files and actions accordingly.

### Git
You will either need internet access or read and write access to an internal
http-based git server where you will duplicate the public code repositories used
in the labs.

### Preparing Each VM
Once your VMs are built and you have verified DNS and network connectivity you
can:

[//]: # (TODO: What are the right channels??)
[//]: # (TODO: Identify what is needed from rhel-server-7-ose-beta-rpms and what will be AE's equivalent)

* Configure yum / subscription manager as follows:

        subscription-manager repos --disable="*"
        subscription-manager repos \
        --enable="rhel-7-server-rpms" \
        --enable="rhel-7-server-extras-rpms" \
        --enable="rhel-7-server-optional-rpms" \
        --enable="rhel-server-7-ose-beta-rpms"

    **Note:** You will have to register/attach your system first.
    *rhel-server-7-ose-beta-rpms* is not a typo.  The name will change at GA to be
    consistent with the RHEL channel names.

* Import the GPG key for beta:

        rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-beta

On **each** VM:

1. Install deltarpm to make package updates a little faster:

        yum -y install deltarpm

1. Remove NetworkManager:

        yum -y remove NetworkManager*

1. Install missing packages:

        yum -y install wget vim-enhanced net-tools bind-utils tmux git

1. Update:

        yum -y update

### Grab Docker Images (Optional, Recommended)
**If you want** to pre-fetch Docker images to make the first few things in your
environment happen **faster**, you'll need to first install Docker:

    yum -y install docker

Make sure that you are running at least `docker-1.6.0-6.el7.x86_64`.

You'll need to add `--insecure-registry 0.0.0.0/0` to your
`/etc/sysconfig/docker` `OPTIONS`. Then:

    systemctl start docker

On all of your systems, grab the following docker images:

    docker pull registry.access.redhat.com/openshift3_beta/ose-haproxy-router:v0.4.3.2
    docker pull registry.access.redhat.com/openshift3_beta/ose-deployer:v0.4.3.2
    docker pull registry.access.redhat.com/openshift3_beta/ose-pod:v0.4.3.2
    docker pull registry.access.redhat.com/openshift3_beta/ose-docker-registry:v0.4.3.2

It may be advisable to pull the following Docker images as well, since they are
used during the various labs:

[//]: # (TODO: check whether all of these are needed for this tutorial)

    docker pull registry.access.redhat.com/openshift3_beta/ruby-20-rhel7
    docker pull registry.access.redhat.com/openshift3_beta/mysql-55-rhel7
    docker pull openshift/hello-openshift:v0.4.3
    docker pull openshift/ruby-20-centos7

**Note:** If you built your VM for a previous beta version and at some point
used an older version of Docker, you need to *reinstall* or *remove+install*
Docker after removing `/etc/sysconfig/docker`. The options in the config file
changed and RPM will not overwrite your existing file if you just do a "yum
update".

    yum -y remove docker
    rm /etc/sysconfig/docker*
    yum -y install docker

### Clone the Training Repository
On your master, it makes sense to clone the training git repository:

[//]: # (TODO: fix the git url)

    cd
    git clone https://github.com/projectatomic/atomic-enterprise-training.git training

**REMINDER**
Almost all of the files for this training are in the training folder you just
cloned.

[//]: # (TODO: check that ansible intaller will be ready)

## Ansible-based Installer
The installer uses Ansible. Eventually there will be an interactive text-based
CLI installer that leverages Ansible under the covers. For now, we have to
invoke Ansible manually.

### Install Ansible
Ansible currently comes from the EPEL repository.

Install EPEL:

    yum -y install \
    http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm

Disable EPEL so that it is not accidentally used later:

    sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo

There's currently a bug in the latest Ansible version, so we need to use a
slightly older one. Install the packages for Ansible:

    yum -y --enablerepo=epel install ansible

### Generate SSH Keys
Because of the way Ansible works, SSH key distribution is required. First,
generate an SSH key on your master, where we will run Ansible:

    ssh-keygen

Do *not* use a password.

### Distribute SSH Keys
An easy way to distribute your SSH keys is by using a `bash` loop:

    for host in ae-master.example.com ae-node1.example.com \
    ae-node2.example.com; do ssh-copy-id -i ~/.ssh/id_rsa.pub \
    $host; done

Remember, if your FQDNs are different, you would have to modify the loop
accordingly.

### Clone the Ansible Repository
The configuration files for the Ansible installer are currently available on
Github. Clone the repository:

[//]: # (TODO: provide ansible repo/branch and path)

    cd
    git clone https://github.com/detiber/openshift-ansible.git -b v3-beta3
    cd ~/openshift-ansible

### Configure Ansible
Copy the staged Ansible configuration files to `/etc/ansible`:

    /bin/cp -r ~/training/eap-beta3/ansible/* /etc/ansible/

### Modify Hosts
If you are not using the "example.com" domain and the training example
hostnames, modify `/etc/ansible/hosts` accordingly. Do not adjust the commented
lines (`#`) at this time.

### Run the Ansible Installer
Now we can simply run the Ansible installer:

[//]: # (TODO: fix the path)

    ansible-playbook ~/openshift-ansible/playbooks/byo/config.yml

If you looked at the Ansible hosts file, note that our master
(ae-master.example.com) was present in both the `master` and the `node`
section.

Effectively, Ansible is going to install and configure both the master and node
software on `ae-master.example.com`. Later, we will modify the Ansible
configuration to add the extra nodes.

### Add Development Users

In the "real world" your developers would likely be using the Atomic tools on
their own machines (e.g. `oc`). For the early access training, we
will create user accounts for two non-privileged users of Atomic, *joe* and
*alice*, on the master. This is done for convenience and because we'll be using
`htpasswd` for authentication.

    useradd joe
    useradd alice

We will come back to these users later.

## Useful Logs
RHEL 7 uses `systemd` and `journal`. As such, looking at logs is not a matter of
`/var/log/messages` any longer. You will need to use `journalctl`.

Since we are running all of the components in higher loglevels, it is suggested
that you use your terminal emulator to set up windows for each process. If you
are familiar with the Ruby Gem, `tmuxinator`, there is a config file in the
training repository. Otherwise, you should run each of the following in its own
window:

[//]: # (TODO: check service nodes)

    journalctl -f -u atomic-master
    journalctl -f -u atomic-node
    journalctl -f -u atomic-sdn-master
    journalctl -f -u atomic-sdn-node

**Note:** You will want to do this on the other nodes, but you won't need the
"-master" service. You may also wish to watch the Docker logs, too.

**Note:** There is an appendix on configuring [Log
Aggregation](#appendix---infrastructure-log-aggregation)

## Auth and Projects
### Configuring htpasswd Authentication
Atomic Enterprise supports a number of mechanisms for authentication. The simplest
use case for our testing purposes is `htpasswd`-based authentication.

To start, we will need the `htpasswd` binary, which is made available by
installing:

    yum -y install httpd-tools

From there, we can create a password for our users, Joe and Alice:

    touch /etc/atomic-passwd
    htpasswd -b /etc/atomic-passwd joe redhat
    htpasswd -b /etc/atomic-passwd alice redhat

[//]: # (TODO: fix the /etc/openshift/master.yaml path)

The Atomic Enterprise configuration is kept in a YAML file which currently lives at
`/etc/openshift/master.yaml`. We need to edit the `oauthConfig`'s
`identityProviders` stanza so that it looks like the following:

    identityProviders:
    - challenge: true
      login: true
      name: apache_auth
      provider:
        apiVersion: v1
        file: /etc/atomic-passwd
        kind: HTPasswdPasswordIdentityProvider

More information on these configuration settings can be found here:

[//]: # (TODO: Will we have something like docs.automic.org ?)

    http://docs.openshift.org/latest/admin_guide/configuring_authentication.html#HTPasswdPasswordIdentityProvider

If you're feeling lazy, use your friend `sed`:

[//]: # (TODO: fix the /etc/openshift/master.yaml path)

    sed -i -e 's/name: anypassword/name: apache_auth/' \
    -e 's/kind: AllowAllPasswordIdentityProvider/kind: HTPasswdPasswordIdentityProvider/' \
    -e '/kind: HTPasswdPasswordIdentityProvider/i \      file: \/etc\/atomic-passwd' \
    /etc/openshift/master.yaml

Restart `atomic-master`:

    systemctl restart atomic-master

### A Project for Everything
Atomic Enterprise (AE) has a concept of "projects" to contain a number of
different resources: services and their pods, builds and so on. They are
somewhat similar to "namespaces" in OpenShift v2. We'll explore what this means
in more details throughout the rest of the labs. Let's create a project for our
first application.

We also need to understand a little bit about users and administration. The
default configuration for CLI operations currently is to be the `master-admin`
user, which is allowed to create projects. We can use the "admin"
AE command to create a project, and assign an administrative user to it:

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
`~/training/eap-beta3` folder.

### "Resources"
There are a number of different resource types in AE, and, essentially,
going through the motions of creating/destroying apps, scaling, building and
etc. all ends up manipulating AE and Kubernetes resources under the
covers. Resources can have quotas enforced against them, so let's take a moment
to look at some example JSON for project resource quota might look like:

[//]: # (TODO: check, what will be correct version of api)


    {
      "apiVersion": "v1beta3",
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
to it. Still in a `root` terminal in the `training/eap-beta3` folder:

    oc create -f quota.json --namespace=demo

If you want to see that it was created:

    oc get -n demo quota
    NAME
    test-quota

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

### Login
Since we have taken the time to create the *joe* user as well as a project for
him, we can log into a terminal as *joe* and then set up the command line
tooling.

Open a terminal as `joe`:

    # su - joe

Then, execute:

[//]: # (TODO: /var/lib/openshift/openshift.local.certificates -> ???)

    oc login -u joe \
    --certificate-authority=/var/lib/openshift/openshift.local.certificates/ca/cert.crt \
    --server=https://ae-master.example.com:8443

Atomic Enterprise, by default, is using a self-signed SSL certificate, so we must point
our tool at the CA file.

The `login` process created a file called `.config` in the `~/.config/openshift`
folder. Take a look at it, and you'll see something like the following:

[//]: # (TODO: /var/lib/openshift/openshift.local.certificates -> ???)

    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /var/lib/openshift/openshift.local.certificates/ca/cert.crt
        server: https://ae-master.example.com:8443
      name: ae-master-example-com-8443
    contexts:
    - context:
        cluster: ae-master-example-com-8443
        namespace: demo
        user: joe
      name: demo
    current-context: demo
    kind: Config
    preferences: {}
    users:
    - name: joe
      user:
        token: ZmQwMjBiZjUtYWE3OC00OWE1LWJmZTYtM2M2OTY2OWM0ZGIw

This configuration file has an authorization token, some information about where
our server lives, our project, etc.

**Note:** See the [troubleshooting guide](#appendix---troubleshooting) for
details on how to fetch a new token once this one expires.  The installer sets
the default token lifetime to 4 hours.

### Grab the Training Repo Again
Since Joe and Alice can't access the training folder in root's home directory,
go ahead and grab it inside Joe's home folder:

[//]: # (TODO: set this to the right repository)

    cd
    git clone https://github.com/projectatomic/atomic-enterprise-training.git training
    cd ~/training/eap-beta3

### The Hello World Definition JSON
In the `eap-beta3` training folder, you can see the contents of our pod definition by using
`cat`:

[//]: # (TODO: make a new hello-openshift image and correct "image" attr below)

    cat hello-pod.json
    {
      "id": "hello-atomic",
      "kind": "Pod",
      "apiVersion":"v1beta2",
      "labels": {
        "name": "hello-atomic"
      },
      "desiredState": {
        "manifest": {
          "version": "v1beta1",
          "id": "hello-atomic",
          "containers": [{
            "name": "hello-atomic",
            "image": "openshift/hello-openshift:v0.4.3",
            "ports": [{
              "hostPort": 6061,
              "containerPort": 8080
            }]
          }]
        }
      }
    }

In the simplest sense, a *pod* is an application or an instance of something. If
you are familiar with OpenShift V2 terminology, it is similar to a *gear*.
Reality is more complex, and we will learn more about the terms as we explore
AE further.

### Run the Pod
To create the pod from our JSON file, execute the following:

    oc create -f hello-pod.json

Remember, we've "logged in" to AE and our project, so this will create
the pod inside of it. The command should display the ID of the pod:

    pods/hello-atomic

Issue a `get pods` to see the details of how it was defined:

[//]: # (TODO: get the right name of hello-openshift image)

    oc get pods
    POD           IP         CONTAINER(S)  IMAGE(S)                           HOST                                  LABELS              STATUS    CREATED
    hello-atomic  10.1.0.6   hello-atomic  openshift/hello-openshift:v0.4.3   ae-master.example.com/192.168.133.2   name=hello-atomic   Running   10 seconds

[//]: # (TODO: openshift3_beta/ -> ???)

Look at the list of Docker containers with `docker ps` (in a `root` terminal) to
see the bound ports.  We should see an `openshift3_beta/ose-pod` container bound
to 6061 on the host and bound to 8080 on the container, along with several other
`ose-pod` containers.

[//]: # (TODO: correct names, images and container IDs)

    CONTAINER ID        IMAGE                              COMMAND              CREATED             STATUS              PORTS                    NAMES
    ded86f750698        openshift/hello-openshift:v0.4.3   "/hello-openshift"   7 minutes ago       Up 7 minutes                                 k8s_hello-openshift.9ac8152d_hello-openshift_demo_18d03b48-0089-11e5-98b9-525400616fe9_c43c7d54
    405d63115a60        openshift3_beta/ose-pod:v0.4.3.2   "/pod"               7 minutes ago       Up 7 minutes        0.0.0.0:6061->8080/tcp   k8s_POD.a01602bc_hello-openshift_demo_18d03b48-0089-11e5-98b9-525400616fe9_dffebcf1     

[//]: # (TODO: openshift3_beta/ -> ???)

The `openshift3_beta/ose-pod` container exists because of the way network
namespacing works in Kubernetes. For the sake of simplicity, think of the
container as nothing more than a way for the host OS to get an interface created
for the corresponding pod to be able to receive traffic. Deeper understanding of
networking in AE is outside the scope of this material.

To verify that the app is working, you can issue a curl to the app's port:

    curl http://localhost:6061
    Hello OpenShift!

Hooray!

### Extra Credit
If you try to curl the pod IP and port, you get "connection refused". See if you
can figure out why.

### Delete the Pod
Go ahead and delete this pod so that you don't get confused in later examples. Don't forget to
do this as the ```joe``` user:

    oc delete pod hello-atomic

Take a moment to think about what this pod exercise really did -- it referenced
an arbitrary Docker image, made sure to fetch it (if it wasn't present), and
then ran it. This could have just as easily been an application from an ISV
available in a registry or something already written and built in-house.

This is really powerful. We will explore using "arbitrary" docker images later.

### Quota Enforcement
Since we know we can run a pod directly, we'll go through a simple quota
enforcement exercise. The `hello-quota` JSON will attempt to create four
instances of the "hello-atomic" pod. It will fail when it tries to create the
fourth, because the quota on this project limits us to three total pods.

Go ahead and use `oc create` and you will see the following:

    oc create -f hello-quota.json
    pods/1-hello-atomic
    pods/2-hello-atomic
    pods/3-hello-atomic
    Error: pods "4-hello-atomic" is forbidden: Limited to 3 pods

Let's delete these pods quickly. As `joe` again:

    oc delete pod --all

**Note:** You can delete most resources using "--all" but there is *no sanity
check*. Be careful.

## Adding Nodes
We are getting ready to build out our complete environment and add more
infrastructure. We will begin by adding our other two nodes.

It is extremely easy to add nodes to an existing AE environment. Return
to a `root` terminal on your master.

### Modifying the Ansible Configuration
On your master, edit the `/etc/ansible/hosts` file and uncomment the nodes, or
add them as appropriate for your DNS/hostnames.

Then, run the ansible playbook again:

[//]: # (TODO: openshift-snsible -> ???)

    ansible-playbook ~/openshift-ansible/playbooks/byo/config.yml

Once the installer is finished, you can check the status of your environment
(nodes) with `oc get nodes`. You'll see something like:

    NAME                      LABELS        STATUS
    ae-master.example.com   Schedulable   <none>    Ready
    ae-node1.example.com    Schedulable   <none>    Ready
    ae-node2.example.com    Schedulable   <none>    Ready

## Regions and Zones
Now that we have a larger AE environment, let's examine more complicated
application and deployment paradigms. If you think you're about to learn how to
configure regions and zones in AEP, you're only partially correct.

In OpenShift 2, we introduced the specific concepts of "regions" and "zones" to
enable organizations to provide some topologies for application resiliency. Apps
would be spread throughout the zones in a region and, depending on the way you
configured OpenShift, you could make different regions accessible to users.

The reason that you're only "partially" correct in your assumption is that, for
Atomic Enterprise and OpenShift v3, Kubernetes doesn't actually care about your
topology. In other words, AE is "topology agnostic". In fact, AE provides
advanced controls for implementing whatever topologies you can dream up,
leveraging filtering and affinity rules to ensure that parts of applications
(pods) are either grouped together or spread apart.

For the purposes of a simple example, we'll be sticking with the "regions" and
"zones" theme. But, as you go through these examples, think about what other
complex topologies you could implement.

First, we need to talk about the "scheduler" and its default configuration.

### Scheduler and Defaults
The "scheduler" is essentially the Atomic master. Any time a pod needs to be
created (instantiated) somewhere, the master needs to figure out where to do
this. This is called "scheduling". The default configuration for the scheduler
looks like the following JSON (although this is embedded in the AE code and you
won't find this in a file):

    {
      "predicates" : [
        {"name" : "PodFitsResources"},
        {"name" : "MatchNodeSelector"},
        {"name" : "HostName"},
        {"name" : "PodFitsPorts"},
        {"name" : "NoDiskConflict"}
      ],"priorities" : [
        {"name" : "LeastRequestedPriority", "weight" : 1},
        {"name" : "ServiceSpreadingPriority", "weight" : 1}
      ]
    }

When the scheduler tries to make a decision about pod placement, first it goes
through "predicates", which essentially filter out the possible nodes we can
choose. Note that, depending on your predicate configuration, you might end up
with no possible nodes to choose. This is totally OK (although generally not
desired).

These default options are documented in the link below, but the quick overview
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
[//]: # (TODO: /etc/openshift/master.yaml -> ???)

The first step is to edit the Atomic master's configuration to tell it to
look for a specific scheduler config file. As `root` edit
`/etc/openshift/master.yaml` and find the line with `schedulerConfigFile`.
Change it to:

[//]: # (TODO: /etc/openshift/scheduler.json -> ???)

    schedulerConfigFile: "/etc/openshift/scheduler.json"

Then, create `/etc/openshift/scheduler.json` from the training materials:

[//]: # (TODO: /etc/openshift/ -> ???)

    /bin/cp -r ~/training/eap-beta3/scheduler.json /etc/openshift/

It will have the following content:

    {
      "predicates" : [
        {"name" : "PodFitsResources"},
        {"name" : "PodFitsPorts"},
        {"name" : "NoDiskConflict"},
        {"name" : "Region", "argument" : {"serviceAffinity" : { "labels" : ["region"]}}}
      ],"priorities" : [
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

* Node 1 -- "region":"primary"
* Node 2 -- "region":"primary"
* Node 3 -- "region":"infra"

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

The documentation link has some more complicated examples. The topoligical
possibilities are endless!

### Restart the Master
Go ahead and restart the master. This will make the new scheduler take effect.
As `root` on your master:

[//]: # (TODO: check the correct names of services)

    systemctl restart atomic-master

### Label Your Nodes
Just before configuring the scheduler, we added more nodes. If you perform the
following as the `root` user:

    oc get node -o json | sed -e '/"resourceVersion"/d' > ~/nodes.json

You will have the JSON output of the definition of all of your nodes. Go ahead and
edit this file. Add the following to the beginning of the `"metadata": {}`
block for your "master" node inside the files `"items"` list:

    "labels" : {
      "region" : "infra",
      "zone" : "NA"
    },

So the end result should look like (note, indentation is not significant in JSON):

[//]: # (TODO: find out correct ipVersion)

    {
        "kind": "List",
        "apiVersion": "v1beta3",
        "items": [
            {
                "kind": "Node",
                "apiVersion": "v1beta3",
                "metadata": {
                    "labels" : {
                      "region" : "infra",
                      "zone" : "NA"
                    },
                    "name": "ae-master.example.com",
                    [...]


For your node1, add the following:

    "labels" : {
      "region" : "primary",
      "zone" : "east"
    },

For your node2, add the following:

    "labels" : {
      "region" : "primary",
      "zone" : "west"
    },

Then, as `root` update your nodes using the following:

    oc update node -f ~/nodes.json

Note: At release the user should not need to edit JSON like this; the
installer should be able to configure nodes initially with desired labels,
and there should be better tools for changing them afterward.

Note: If you end up getting an error while attempting to update the nodes, review your json. Ensure that there are commas in the previous element to your added label sections.

Check the results to ensure the labels were applied:

    oc get nodes

    NAME                     LABELS                     STATUS
    ae-master.example.com    region=infra,zone=NA       Ready
    ae-node1.example.com     region=primary,zone=east   Ready
    ae-node2.example.com     region=primary,zone=west   Ready

Now there is one final step that is necessary due to a [caching
bug](https://github.com/openshift/origin/issues/1727#issuecomment-94518311)
which is not fixed for Early Access. Each node needs to be restarted with:

[//]: # (TODO: check the service name)

    systemctl restart atomic-node

## Services
From the [Kubernetes
documentation](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/services.md):

    A Kubernetes service is an abstraction which defines a logical set of pods and a
    policy by which to access them - sometimes called a micro-service. The goal of
    services is to provide a bridge for non-Kubernetes-native applications to access
    backends without the need to write code that is specific to Kubernetes. A
    service offers clients an IP and port pair which, when accessed, redirects to
    the appropriate backends. The set of pods targetted is determined by a label
    selector.

If you think back to the simple pod we created earlier, there was a "label":

      "labels": {
        "name": "hello-atomic"
      },

Now, let's look at a *service* definition:

[//]: # (TODO: check the apiVersion)

    {
      "id": "hello-atomic-service",
      "kind": "Service",
      "apiVersion": "v1beta1",
      "port": 27017,
      "selector": {
        "name": "hello-atomic"
      }
    }

The *service* has a `selector` element. In this case, it is a key:value pair of
`name:hello-atomic`. If you looked at the output of `oc get pods` on your
master, you saw that the `hello-atomic` pod has a label:

    name=hello-atomic

The definition of the *service* tells Kubernetes that any pods with the label
"name=hello-atomic" are associated, and should have traffic distributed
amongst them. In other words, the service itself is the "connection to the
network", so to speak, or the input point to reach all of the pods. Generally
speaking, pod containers should not bind directly to ports on the host. We'll
see more about this later.

But, to really be useful, we want to make our application accessible via a FQDN,
and that is where the routing tier comes in.

## Routing

[//]: # (TODO: correct openshift3_beta/)

The AE routing tier is how FQDN-destined traffic enters the Atomic
environment so that it can ultimately reach pods. In a simplification of the
process, the `openshift3_beta/ose-haproxy-router` container we will create below
is a pre-configured instance of HAProxy as well as some of the AE
framework. The Atomic instance running in this container watches for route
resources on the Atomic master.

Here is an example route resource JSON definition:

[//]: # (TODO: check the apiVersion)

    {
      "kind": "Route",
      "apiVersion": "v1beta1",
      "metadata": {
        "name": "hello-atomic-route"
      },
      "id": "hello-atomic-route",
      "host": "hello-atomic.cloudapps.example.com",
      "serviceName": "hello-atomic-service"
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

### Creating the Router
The router is the ingress point for all traffic destined for AE
services. It currently supports only HTTP(S) traffic (and "any"
TLS-enabled traffic via SNI). While it is called a "router", it is essentially a
proxy.

[//]: # (TODO: rename openshift3_beta/)

The `openshift3_beta/ose-haproxy-router` container listens on the host network
interface unlike most containers that listen only on private IPs. The router
proxies external requests for route names to the IPs of actual pods identified
by the service associated with the route.


[//]: # (TODO: AE -> Origin ???)

AE's admin command set enables you to deploy router pods automatically.
As the `root` user, try running it with no options and you should see the note
that a router is needed:

    oadm router
    F0223 11:50:57.985423    2610 router.go:143] Router "router" does not exist
    (no service). Pass --create to install.

So, go ahead and do what it says:

    oadm router --create
    F0223 11:51:19.350154    2617 router.go:148] You must specify a .kubeconfig
    file path containing credentials for connecting the router to the master
    with --credentials

Just about every form of communication with AE components is secured by
SSL and uses various certificates and authentication methods. Even though we set
up our `.kubeconfig` for the root user, `oadm router` is asking us what
credentials the *router* should use to communicate. We also need to specify the
router image, since the tooling defaults to upstream/origin:

[//]: # (TODO: /var/lib/openshift/openshift.local.certiciates/openshift-router)
[//]: # (TODO: registry.access.redhat.com/openshift3_beta/ose-${component}:${version})

    oadm router --create \
    --credentials=/var/lib/openshift/openshift.local.certificates/openshift-router/.kubeconfig \
    --images='registry.access.redhat.com/openshift3_beta/ose-${component}:${version}'

If this works, you'll see some output:

    services/router
    deploymentConfigs/router

Let's check the pods with the following:

    oc get pods | awk '{print $1"\t"$3"\t"$5"\t"$7"\n"}' | column -t

In the output, you should see the router pod status change to "running" after a
few moments (it may take up to a few minutes):

    POD                   CONTAINER(S)  HOST                                 STATUS
    deploy-router-1f99mb  deployment    ae-master.example.com/192.168.133.2  Succeeded
    router-1-ats7z        router        ae-node2.example.com/192.168.133.4   Running

Note: You may or may not see the deploy pod, depending on when you run this
command. Also the router may not end up on the master.

### Router Placement By Region
In the very beginning of the documentation, we indicated that a wildcard DNS
entry is required and should point at the master. When the router receives a
request for an FQDN that it knows about, it will proxy the request to a pod for
a service. But, for that FQDN request to actually reach the router, the FQDN has
to resolve to whatever the host is where the router is running. Remember, the
router is bound to ports 80 and 443 on the *host* interface. Since our wildcard
DNS entry points to the public IP address of the master, we need to ensure that
the router runs *on* the master.

Remember how we set up regions and zones earlier? In our setup we labeled the
master with the "infra" region. Without specifying a region or a zone in our
environment, the router pod had an equal chance of ending up on any node, but we
can ensure that it always and only lands in the "infra" region (thus, on the
master) using a NodeSelector.

To do this, we will modify the `deploymentConfig` for the router. If you recall,
when we created the router we saw both a `deploymentConfig` and `service`
resource.

We have not discussed DeploymentConfigs (or even Deployments) yet. The brief
summary is that a DeploymentConfig defines not only the pods (and containers)
but also how many pods should be created and also transitioning from one pod
definition to another.  We'll learn a little bit more about deployment
configurations later.  For now, as `root`, we will use `oc edit` to manipulate
the router DeploymentConfig and modify the router's pod definition to add a
NodeSelector, so that router pods will be placed where we want them.  Whew!

    oc edit deploymentConfigs/router

`oc edit` will bring up the default system editor (vi) with a YAML
representation of the resource, in this case the router's `deploymentConfig`.
You could also edit it as JSON or use a different editor; see `oc edit --help`.

Note: In future releases, you will be able to supply NodeSelector and other
labels at creation time rather than editing the object after the fact.

We will specify our NodeSelector within the `podTemplate:` block that
defines the pods to create. It is easiest to just place it right after
that line, like this: (indentation *is* significant in YAML)

    [...]
    template:
      controllerTemplate:
        podTemplate:
          nodeSelector:
            region: infra
          desiredState:
            manifest:
    [...]

Once you save this file and exit the editor, the DeploymentConfig will be
updated in AE's data store and a new router deployment will be created
based on the new definition.  It will take at least a few seconds for this to
happen (possibly longer if the router image has not been pulled to the master
yet).  Watch `oc get pods` until the router pod has been recreated and assigned
to the master host.

For a true HA implementation, one would want multiple "infra" nodes and
multiple, clustered router instances.

## The Complete Pod-Service-Route
With a router now available, let's take a look at an entire
Pod-Service-Route definition template and put all the pieces together.

Don't forget -- the materials are in `~/training/eap-beta3`.

### Creating the Definition
The following is a complete definition for a pod with a corresponding service
and a corresponding route. It also includes a deployment configuration.

[//]: # (TODO: check the apiVersion)
[//]: # (TODO: cnahge items[3]['triggers']["from"]["name"])
[//]: # (TODO: cnahge items[3]['template']["controllerTemplate"]["podTemplate"]["desiredState"]["manifest"]["containers"]["image"])

    {
      "metadata":{
        "name":"hello-service-pod-meta"
      },
      "kind":"Config",
      "apiVersion":"v1beta1",
      "creationTimestamp":"2014-09-18T18:28:38-04:00",
      "items":[
        {
          "id": "hello-atomic-service",
          "kind": "Service",
          "apiVersion": "v1beta1",
          "port": 27017,
          "containerPort": 8080,
          "selector": {
            "name": "hello-atomic"
          }
        },
        {
          "kind": "Route",
          "apiVersion": "v1beta1",
          "metadata": {
            "name": "hello-atomic-route"
          },
          "id": "hello-atomic-route",
          "host": "hello-atomic.cloudapps.example.com",
          "serviceName": "hello-atomic-service"
        },
        {
          "apiVersion": "v1beta1",
          "kind": "ImageStream",
          "metadata": {
            "name": "hello-atomic"
          }
        },
        {
            "kind": "DeploymentConfig",
            "apiVersion": "v1beta1",
            "metadata": {
                "name": "hello-atomic"
            },
            "triggers": [
                {
                  "imageChangeParams": {
                    "automatic": true,
                    "containerNames": [
                      "hello-atomic"
                    ],
                    "from": {
                      "name": "hello-openshift"
                    },
                    "tag": "latest"
                  },
                  "type": "ImageChange"
                },
                {
                  "type": "ConfigChange"
                }
            ],
            "template": {
                "strategy": {
                    "type": "Recreate"
                },
                "controllerTemplate": {
                    "replicas": 1,
                    "replicaSelector": {
                        "name": "hello-atomic"
                    },
                    "podTemplate": {
                        "desiredState": {
                            "manifest": {
                                "version": "v1beta2",
                                "id": "",
                                "volumes": null,
                                "containers": [
                                    {
                                        "name": "hello-atomic",
                                        "image": "openshift/hello-openshift:v0.4.3",
                                        "ports": [
                                            {
                                                "containerPort": 8080,
                                                "protocol": "TCP"
                                            }
                                        ],
                                        "resources": {},
                                        "livenessProbe": {
                                            "tcpSocket": {
                                                "port": 8080
                                            },
                                            "timeoutSeconds": 1,
                                            "initialDelaySeconds": 10
                                        },
                                        "terminationMessagePath": "/dev/termination-log",
                                        "imagePullPolicy": "PullIfNotPresent",
                                        "capabilities": {}
                                    }
                                ],
                                "restartPolicy": {
                                    "always": {}
                                },
                                "dnsPolicy": "ClusterFirst"
                            }
                        },
                        "nodeSelector": {
                          "region": "primary"
                        },
                        "labels": {
                            "name": "hello-atomic"
                        }
                    }
                }
            },
            "latestVersion": 1
        }
      ]
    }

In the JSON above:

* There is a pod that has the label `name=hello-atomic` and the nodeSelector `region=primary`
* There is a service:
  * with the id `hello-atomic-service`
  * with the selector `name=hello-atomic`
* There is a route:
  * with the FQDN `hello-atomic.cloudapps.example.com`
  * with the `serviceName` directive `hello-atomic-service`

If we work from the route down to the pod:

* The route for `hello-atomic.cloudapps.example.com` has an HAProxy pool
* The pool is for any pods in the service whose ID is `hello-atomic-service`,
    via the `serviceName` directive of the route.
* The service `hello-atomic-service` includes every pod who has a label
    `name=hello-atomic`
* There is a single pod with a single container that has the label
    `name=hello-atomic`

**Logged in as `joe`,** edit `test-complete.json` and change the `host` stanza for
the route to have the correct domain, matching the DNS configuration for your
environment. Once this is done, go ahead and use `oc` to apply it:

    oc create -f test-complete.json

 You should see something like the following:

[//]: # (TODO: check the name of imageStream)

    services/hello-atomic-service
    routes/hello-atomic-route
    imageStreams/openshift/hello-atomic
    deploymentConfigs/hello-atomic

You can verify this with other `oc` commands:

    oc get pods

    oc get services

    oc get routes

### Project Status
AE provides a handy tool, `oc status`, to give you a summary of
common resources existing in the current project:

[//]: # (TODO: check the image name - hello-openshift:latest)

    oc status
    In project Atomic Enterprise Demo (demo)

    service hello-atomic-service (172.30.17.237:27017 -> 8080)
      hello-atomic deploys hello-openshift:latest
        #1 deployed about a minute ago

    To see more information about a service or deployment config, use 'oc describe service <name>' or 'oc describe dc <name>'.
    You can use 'oc get pods,svc,dc,bc,builds' to see lists of each of the types described above.

`oc status` does not yet show bare pods or routes. The output will be
more interesting when we get to builds and deployments.

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

[//]: # (TODO: fix the text "Hello OpenShift" after fixing the image)


    curl `oc get services | grep hello-atomic | awk '{print $4":"$5}' | sed -e 's/\/.*//'`
    Hello OpenShift!

This is a good sign! It means that, if the router is working, we should be able
to access the service via the route.

### Verifying the Routing
Verifying the routing is a little complicated, but not terribly so. Since we
specified that the router should land in the "infra" region, we know that its
Docker container is on the master.

We ultimately want the PID of the container running the router so that we can go
"inside" it. On the master system, as the `root` user, issue the following to
get the PID of the router:

    docker inspect --format {{.State.Pid}}   \
      `docker ps | grep haproxy-router | awk '{print $1}'`
    2239

The output will be a PID -- in this case, the PID is `2239`. We can use
`nsenter` to jump inside that container:

    nsenter -m -u -n -i -p -t 2239
    [root@mainrouter /]#

You are now in a bash session *inside* the container running the router.

Since we are using HAProxy as the router, we can cat the `routes.json` file:

    cat /var/lib/containers/router/routes.json

If you see some content that looks like:

    "demo/hello-atomic-service": {
        "Name": "demo/hello-atomic-service",
        "EndpointTable": {
          "10.1.2.2:8080": {
            "ID": "10.1.2.2:8080",
            "IP": "10.1.2.2",
            "Port": "8080"
          }
        },
        "ServiceAliasConfigs": {
          "hello-atomic.cloudapps.example.com-": {
            "Host": "hello-atomic.cloudapps.example.com",
            "Path": "",
            "TLSTermination": "",
            "Certificates": null
          }
        }
      }

You know that "it" worked -- the router watcher detected the creation of the
route in AE and added the corresponding configuration to HAProxy.

Go ahead and `exit` from the container, and then curl your fancy,
publicly-accessible Atomic application!

[//]: # (TODO: fix the text)


    [root@mainrouter /]# exit
    logout
    # curl http://hello-atomic.cloudapps.example.com
    Hello OpenShift!

Hooray!

If your machine is capable of resolving the wildcard DNS, you should also be
able to view this in your web browser:

    http://hello-atomic.cloudapps.example.com

## Project Administration
When we created the `demo` project, `joe` was made a project administrator. As
an example of an administrative function, if `joe` now wants to let `alice` look
at his project, with his project administrator rights he can add her using the
`oadm policy` command:

    [joe]$ oadm policy add-role-to-user view alice

**Note:** `oadm` will act, by default, on whatever project the user has
selected. If you recall earlier, when we logged in as `joe` we ended up in the
`demo` project. We'll see how to switch projects later.

Open a new terminal window as the `alice` user and the login to AE:

[//]: # (TODO: fix ca path)
[//]: # (TODO: fix "Authentication required ... XXX" text)


    oc login -u alice \
    --certificate-authority=/var/lib/openshift/openshift.local.certificates/ca/cert.crt \
    --server=https://ae-master.example.com:8443

    Authentication required for https://ae-master.example.com:8443 (openshift)
    Password:  <redhat>
    Login successful.

    Using project "demo"

`alice` has no projects of her own yet (she is not an administrator of
anything), so she is automatically configured to look at the `demo` project
since she has access to it. She has "view" access, so `oc status` and `oc get
pods` and so forth should show her the same thing as `joe`. However, she cannot
make changes:

[//]: # (TODO: fix image names)

    [alice]$ oc get pods
    POD                       IP      CONTAINER(S)   IMAGE(S)
    hello-atomic-1-zdgmt   10.1.2.4   hello-atomic   openshift/hello-openshift
    [alice]$ oc delete pod hello-atomic-1-zdgmt
    Error from server: "/api/v1beta1/pods/hello-atomic-1-zdgmt?namespace=demo" is forbidden because alice cannot delete on pods with name "hello-atomic-1-zdgmt" in demo

`joe` could also give `alice` the role of `edit`, which gives her access to all
activities except for project administration.

    [joe]$ oadm policy add-role-to-user edit alice

Now she can delete that pod if she wants, but she can not add access for
another user or upgrade her own access. To allow that, `joe` could give
`alice` the role of `admin`, which gives her the same access as himself.

    [joe]$ oadm policy add-role-to-user admin alice

There is no "owner" of a project, and projects can be created without any
administrator. `alice` or `joe` can remove the `admin` role (or all roles) from
each other, or themselves, at any time without affecting the existing project.

    [joe]$ oadm policy remove-user joe

Check `oadm policy help` for a list of available commands to modify
project permissions. OpenShift RBAC is extremely flexible. The roles
mentioned here are simply defaults - they can be adjusted (per-project
and per-resource if needed), more can be added, groups can be given
access, etc. Check the documentation for more details:

[//]: # (TODO: docs.openshift.org -> docs.atomic.org ??)

* http://docs.openshift.org/latest/dev_guide/authorization.html
* https://github.com/openshift/origin/blob/master/docs/proposals/policy.md

Of course, there be dragons. The basic roles should suffice for most uses.

### Deleting a Project
Since we are done with this "demo" project, and since the `alice` user is a
project administrator, let's go ahead and delete the project. This should also
end up deleting all the pods, and other resources, too.

As the `alice` user:

    oc delete project demo

If you quickly go to the web console and return to the top page, you'll see a
warning icon that will pop-up a hover tip saying the project is marked for
deletion.

If you switch to the `root` user and issue `oc get project` you will see that
the demo project's status is "Terminating". If you do an `oc get pod -n demo`
you may see the pods, still. It takes about 60 seconds for the project deletion
cleanup routine to finish.

Once the project disappears from `oc get project`, doing `oc get pod -n demo`
should return no results.

Note: As of the Early Access, a user with the `edit` role can actually delete the project.
[This will be fixed](https://github.com/openshift/origin/issues/1885).


## The Registry
[//]: # (TODO: What is it for in AE??)
[//]: # (TODO: Write some example how to use it since we don't have STI)

Atomic Enterprise provides a Docker registry that administrators may run inside
the Atomic environment that will manage images "locally". Let's take a moment
to set that up.

`oadm` again comes to our rescue with a handy installer for the
registry. As the `root` user, run the following:

[//]: # (TODO: fix the ca path)
[//]: # (TODO: fix the image path)

    oadm registry --create \
    --credentials=/var/lib/openshift/openshift.local.certificates/openshift-registry/.kubeconfig \
    --images='registry.access.redhat.com/openshift3_beta/ose-${component}:${version}'

You'll get output like:

    services/docker-registry
    deploymentConfigs/docker-registry

You can use `oc get pods`, `oc get services`, and `oc get deploymentconfig`
to see what happened. This would also be a good time to try out `oc status`
as root:

[//]: # (TODO: fix image paths)

    oc status

    In project default

    service docker-registry (172.30.17.196:5000 -> 5000)
      docker-registry deploys registry.access.redhat.com/openshift3_beta/ose-docker-registry:v0.4.3.2
        #1 deployed about a minute ago

    service kubernetes (172.30.17.2:443 -> 443)

    service kubernetes-ro (172.30.17.1:80 -> 80)

    service router (172.30.17.129:80 -> 80)
      router deploys registry.access.redhat.com/openshift3_beta/ose-haproxy-router:v0.4.3.2
        #2 deployed 8 minutes ago
        #1 deployed 7 minutes ago

The project we have been working in when using the `root` user is called
"default". This is a special project that always exists (you can delete it, but
AE will re-create it) and that the administrative user uses by default.
One interesting feature of `oc status` is that it lists recent deployments.
When we created the router and adjusted it, that adjustment resulted in a second
deployment. We will talk more about deployments when we get into builds.

Anyway, ultimately you will have a Docker registry that is being hosted by AE
and that is running on one of your nodes.

To quickly test your Docker registry, you can do the following:

    curl `oc get services | grep registry | awk '{print $4":"$5}' | sed -e 's/\/.*//'`

And you should see:

    "docker-registry server (dev) (v0.9.0)"

If you get "connection reset by peer" you may have to wait a few more moments
after the pod is running for the service proxy to update the endpoints necessary
to fulfill your request. You can check if your service has finished updating its
endpoints with:

    oc describe service docker-registry

And you will eventually see something like:

    Name:                   docker-registry
    Labels:                 docker-registry=default
    Selector:               docker-registry=default
    IP:                     172.30.17.64
    Port:                   <unnamed>       5000/TCP
    Endpoints:              10.1.0.5:5000
    Session Affinity:       None
    No events.

Once there is an endpoint listed, the curl should work.

### Registry Placement By Region (optional)
In the beta environment, as architected, there is no real need for the registry
to land on any particular node. However, for consistency, you might want to keep
AE "infrastructure" components on the master's node. We can use our
previously-defined "infra" region for this purpose.

To do this, edit the created DeploymentConfig definition with `osc edit`:

    oc edit dc docker-registry

As before, specify your NodeSelector within the `podTemplate:` block that
defines the pods to create. It is easiest to just place it right after
that line, like this: (indentation *is* significant in YAML)

    [...]
    template:
      controllerTemplate:
        podTemplate:
          nodeSelector:
            region: infra
          desiredState:
            manifest:
    [...]

Once you save this file and exit, the DeploymentConfig will be updated and
a new registry deployment will soon be created with the new definition.

If you are going to move the registry, do it now or don't do it all. As
dedicated storage volumes did not make the Early Access drop, restarting the
registry pod will result in an empty registry -- all the images will be lost.
This will be a Very.Bad.Thing.

[//]: # (TODO: Missing template section)
[//]: # (TODO: Are templates instantiable without UI?)
[//]: # (TODO: update the wiring example so that it doesn't require STI)
[//]: # (TODO: update the rollback example so that it doens't require STI)
[//]: # (TODO: can EAP example be modified to not require STI??)

## Conclusion
This concludes the Early Access training. Look for more example applications to come!

# APPENDIX - DNSMasq setup
In this training repository is a sample `dnsmasq.conf` file and a sample `hosts`
file. If you do not have the ability to manipulate DNS in your environment, or
just want a quick and dirty way to set up DNS, you can install dnsmasq on one of
your nodes. Do **not** install DNSMasq on your master. Atomic Enterprise now has an
internal DNS service provided by Go's "SkyDNS" that is used for internal service
communication, which will be explored more in future training materials.

    yum -y install dnsmasq

Replace `/etc/dnsmasq.conf` with the one from this repository, and replace
`/etc/hosts` with the `hosts` file from this repository.

Enable and start the dnsmasq service:

    systemctl enable dnsmasq; systemctl start dnsmasq

You will need to ensure the following, or fix the following:

* Your IP addresses match the entries in `/etc/hosts`
* Your hostnames for your machines match the entries in `/etc/hosts`
* Your `cloudapps` domain points to the correct node ip in `dnsmasq.conf`
* Each of your systems has the same `/etc/hosts` file
* The first `nameserver` in `/etc/resolv.conf` on the node running dnsmasq should be 127.0.0.1 and the second nameserver should be your corporate or upstream DNS resolver (eg: Google DNS @ 8.8.8.8); alternatively put upstream resolver as `server=8.8.8.8` in `/etc/dnsmasq.conf`
* the other nodes' and master's `/etc/resolv.conf` points to the IP address of the node
  running DNSMasq as the first nameserver
* That you also open port 53 (UDP) to allow DNS queries to hit the node

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

    docker pull registry.access.redhat.com/openshift3_beta/ose-haproxy-router:v0.4.3.2
    docker pull registry.access.redhat.com/openshift3_beta/ose-deployer:v0.4.3.2
    docker pull registry.access.redhat.com/openshift3_beta/ose-pod:v0.4.3.2
    docker pull registry.access.redhat.com/openshift3_beta/ose-docker-registry:v0.4.3.2
    docker pull registry.access.redhat.com/openshift3_beta/ruby-20-rhel7
    docker pull registry.access.redhat.com/openshift3_beta/mysql-55-rhel7
    docker pull openshift/hello-openshift

This will fetch all of the images. You can then save them to a tarball:

[//]: # (TODO: change image names?)

    docker save -o beta3-images.tar \
    registry.access.redhat.com/openshift3_beta/ose-haproxy-router:v0.4.3.2 \
    registry.access.redhat.com/openshift3_beta/ose-deployer:v0.4.3.2 \
    registry.access.redhat.com/openshift3_beta/ose-pod:v0.4.3.2 \
    registry.access.redhat.com/openshift3_beta/ose-docker-registry:v0.4.3.2 \
    registry.access.redhat.com/openshift3_beta/ruby-20-rhel7 \
    registry.access.redhat.com/openshift3_beta/mysql-55-rhel7 \
    openshift/hello-openshift

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

# APPENDIX - Pretty Output
If the output of `oc get pods` is a little too busy, you can use the following
to limit some of what it returns:

    oc get pods | awk '{print $1"\t"$3"\t"$5"\t"$7"\n"}' | column -t

# APPENDIX - Troubleshooting

An experimental diagnostics command is in progress for Atomic Enterprise, to hopefully
be included in the origin binary for the next release. For now, you can download
the one for Early Access under [Luke Meyer's release page](https://github.com/sosiouxme/origin/releases).
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


        # If a stale token exists it will prevent the login command from working
        rm ~/.config/openshift/.config

        oc login \
        --certificate-authority=/var/lib/openshift/openshift.local.certificates/ca/cert.crt \
        --cluster=master --server=https://ae-master.example.com:8443 \
        --namespace=[INSERT NAMESPACE HERE]

* When using an "oc" command like "oc get pods" I see a "certificate signed by
    unknown authority error":

        F0212 16:15:52.195372   13995 create.go:79] Post
        https://ae-master.example.net:8443/api/v1beta1/pods?namespace=default:
        x509: certificate signed by unknown authority

    This generally means you do not have a client config file at all, as it should
    supply the certificate authority for validating the master. You could also
    have the wrong CA in your client config. You should probably regenerate
    your client config as in the previous suggestion.

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
aggregate logs from your AE infastructure services. By default, AE
services log to the systemd journal and rsyslog persists those log messages to
`/var/log/messages`. We''ll reconfigure rsyslog to write these entries to
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

## Configure nodes to send Atomic logs to your master
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
OpenShift builds and deployments to use these proxies is as simple as setting
standard environment variables.  The trick is knowing where to place them.

## Importing ImageStreams

Since the importer is on the Master we need to make the configuration change
there.  The easiest way to do that is to add environment variables `NO_PROXY`,
`HTTP_PROXY`, and `HTTPS_PROXY` to `/etc/sysconfig/atomic-master` then restart
your master.

~~~
HTTP_PROXY=http://USERNAME:PASSWORD@10.0.1.1:8080/
HTTPS_PROXY=https://USERNAME:PASSWORD@10.0.0.1:8080/
NO_PROXY=master.example.com
~~~

It's important that the master doesn't use the proxy to access itself so make
sure it's listed in the `NO_PROXY` value.

Now restart the Service:
~~~
systemctl restart atomic-master
~~~

If you had previously imported ImageStreams without the proxy configuration to can re-run the process as follows:

[//]: # (TODO: openshift namespace ??)

~~~
osc delete imagestreams -n openshift --all
osc create -f image-streams.json -n openshift
~~~

## Setting Environment Variables in Pods

It's not only at build time that proxies are required.  Many applications will
need them too.  In previous examples we used environment variables in
`DeploymentConfig`s to pass in database connection information.  The same can
be done for configuring a `Pod`'s proxy at runtime:

[//]: # (TODO: apiVersion)

~~~
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
~~~

## Proxying Docker Pull

This is yet another case where it may be necessary to tunnel traffic through a
proxy.  In this case you can edit `/etc/sysconfig/docker` and add the variables
in shell format:

~~~
NO_PROXY=mycompany.com
HTTP_PROXY=http://USER:PASSWORD@IPADDR:PORT
HTTPS_PROXY=https://USER:PASSWORD@IPADDR:PORT
~~~

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

### An Example Hosts File (/etc/ansible/hosts)

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
    ec2-52-6-179-239.compute-1.amazonaws.com #The master
    ... <additional node hosts go here> ...

### Testing the Auto-detected Values

[//]: # (TODO: make a new atomic-enterprise repo and move ansible configs there)

Run the openshift_facts playbook:

    cd ~/openshift-ansible
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

To override the the defaults, you can set the variables in your inventory. For
example, if using AWS and managing dns externally, you can override the host
public hostname as follows:

    [masters]
    ec2-52-6-179-239.compute-1.amazonaws.com openshift_public_hostname=ae-master.public.example.com

Running ansible:

    ansible-playbook ~/openshift-ansible/playbooks/byo/config.yml

## Automated AWS Install With Ansible

### Requirements:
- ansible-1.8.x
- python-boto

### Assumptions Made:

[//]: # (TODO: openshift security groups??)

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

### Setup (Modifying the Values Appropriately):

    export AWS_ACCESS_KEY_ID=MY_ACCESS_KEY
    export AWS_SECRET_ACCESS_KEY=MY_SECRET_ACCESS_KEY
    export EC2_REGION=us-east-1
    export EC2_AMI_ID=ami-12663b7a
    export EC2_KEYPAIR=MY_KEYPAIR_NAME
    export RHN_USERNAME=MY_RHN_USERNAME
    export RHN_PASSWORD=MY_RHN_PASSWORD
    export ROUTE_53_WILDCARD_ZONE=cloudapps.example.com
    export ROUTE_53_HOST_ZONE=example.com

### Configuring the Hosts:

    ansible-playbook -i inventory/aws/hosts openshift_setup.yml

### Accessing the Hosts:
[//]: # (TODO: change openshift user)

Each host will be created with an 'openshift' user that has passwordless sudo configured.

# APPENDIX - Linux, Mac, and Windows clients

The Atomic Enterprise client `oc` is available for Linux, Mac OSX, and Windows. You
can use these clients to perform all tasks in this documentation that make use
of the `oc` command.

## Downloading The Clients

[//]: # (TODO: make this relevant to Atomic Enterprise)

Visit [Download Red Hat OpenShift Enterprise Beta](https://access.redhat.com/downloads/content/289/ver=/rhel---7/0.4.3.2/x86_64/product-downloads)
to download the Beta3 clients. You will need to sign into Customer Portal using
an account that includes the OpenShift Enterprise High Touch Beta entitlements.

**Note**: Certain versions of Internet Explorer will save the Windows
client without the .exe extension. Please rename the file to `oc.exe`.

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

In the future users will be able to download clients directly from the Atomic Enterprise
console rather than needing to visit Customer Portal.
