# Atomic Enterprise Platform
Based on the world’s leading enterprise Linux, Red Hat Atomic Enterprise Platform provides the foundation for production scale container deployments, utilizing the same core enabling technologies as Red Hat [OpenShift Enterprise 3](https://www.openshift.com/products/origin), including [Docker](https://www.docker.com/) as a Linux container format, and [Kubernetes](http://kubernetes.io/) for container orchestration. 

For more information [read the press release](http://www.redhat.com/en/about/press-releases/red-hat-unveils-red-hat-atomic-enterprise-platform-production-deployment-secure-certified-linux-containers-scale).

This guide focuses on Atomic Enterprise Platform.  However, there is
ongoing work to share more work with OpenShift v3, and thus you might
find newer instructions in the [OpenShift
version](https://github.com/openshift/training) of this guide.

## Differences from Openshift v3
The following features are not provided with Atomic Enterprise, but are with OpenShift v3:
- STI (Source-To-Image builder)
- UI (web console)
- Docker Builder

Atomic Enterprise will be introduced with a series of early access releases.
This document provides an overview of the training materials for each release
and what that release covers.

**Work in progress**

## Early Access Program based on Openshift v3
The following is an overview of the Atomic Enterprise (AE) features covered in
Early Access Program training:
- Installation and configuration of AE and atomic-sdn
- Adding nodes to the master
- Extensive command line use
- Basic HTTP/S (only) routing. 
- Work with supplied example applications
- User authentication
- Project quotas (display)
- Complex / Tiered app deployment
- Arbitrary docker image deployment
- Ansible-based installer (non-interactive)
- Improvements and enhancements in the CLI
- "Regions" and "Zones"
- Quota enforcement
- Templates from the console
- Beginnings of user/team management
- Internal service DNS
- Expanded documentation/explanations
- Integrated sdn into OpenShift
  - No separate openshift-sdn-node and openshift-sdn-master packages or services
    You should remove openshift-sdn-master and openshift-sdn-node packages or
    preferably reprovision your environment when installing Beta 4
  - Openvswitch based implementation provided via 'openshift-sdn-ovs' plugin
    package and configuring `networkPluginName: redhat/openshift-ovs-subnet` on
    master and nodes. Ansible does this for you by default.
[//]: # (- Web console and project basics)
[//]: # (- Rollback / Activate)
[//]: # (- "integration" / webhooks)

[The specific documentation is here.](eap-latest-setup.md)
