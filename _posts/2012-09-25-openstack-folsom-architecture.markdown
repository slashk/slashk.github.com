--- 
layout: post
title: "OpenStack Folsom Architecture"
category: openstack
---

*As the Folsom release of OpenStack is due to be released this week, I've taken the time to update my "Intro to OpenStack Architecture 101" for the official documentation. It [merged into the repos yesterday](https://review.openstack.org/#/c/13566/2) and below is an **expanded** version of it.*

##OpenStack Components

There are currently seven core components of OpenStack: Compute, Object Storage,
Identity, Dashboard, Block Storage, Network and Image Service. Let's look at
each in turn:

* Object Store (codenamed "[Swift](http://swift.openstack.org)") allows you to
  store or retrieve files (but not mount directories like a fileserver). Several
  companies provide commercial storage services based on Swift. These include
  KT, Rackspace (from which Swift originated) and Internap. Swift is also used
  internally at many large companies to store their data.
* Image (codenamed "[Glance](http://glance.openstack.org)") provides a catalog
  and repository for virtual disk images. These disk images are mostly commonly
  used in OpenStack Compute. While this service is technically optional, any
  cloud of size will require it.
* Compute (codenamed "[Nova](http://nova.openstack.org/)") provides virtual
  servers upon demand. [Rackspace](http://www.rackspace.com) and
  [HP](https://www.hpcloud.com/) provide commercial compute services built on
  Nova and it is used internally at companies like Mercado Libre and NASA (where
  it originated).
* Dashboard (codenamed "[Horizon](http://horizon.openstack.org/)") provides a
  modular web-based user interface for all the OpenStack services. With this web
  GUI, you can perform most operations on your cloud like launching an instance,
  assigning IP addresses and setting access controls.
* Identity (codenamed "[Keystone](http://keystone.openstack.org/)") provides
  authentication and authorization for all the OpenStack services. It also
  provides a service catalog of services within a particular OpenStack
  cloud.
* Network (codenamed "[Quantum](http://quantum.openstack.org/)") provides
  "network connectivity as a service" between interface devices managed by other
  OpenStack services (most likely Nova). The service works by allowing users to
  create their own networks and then attach interfaces to them. Quantum has a
  pluggable architecture to support many popular networking vendors and
  technologies. **Quantum is new in the Folsom release**.
* Block Storage (codenamed "Cinder") provides persistent block storage to guest
  VMs. This project was born from code originally in Nova (the `nova-volume`
  service described below). Please note that this is block storage (or volumes)
  not filesystems like NFS or CIFS share. **Cinder is new for the Folsom
    release**.

In addition to these core projects, there are also a number of "incubation" projects that are being considered for future inclusion in the OpenStack core.

## Comparison Against AWS

Although all of the OpenStack services are made to stand on their own, many users want some form of AWS compatibility for tools, APIs or just familiarity:

* Nova is conceptually similar to EC2. In fact, there is are a variety of ways to provide EC2 API compatibility to it.
* Swift is conceptually similar to S3. It implements a subset of the S3 API in WSGI middleware.
* Glance provides many of the same features as Amazon's AMI catalog.
* Cinder provides block services similar to EBS.


##Conceptual Architecture

The OpenStack project as a whole is designed to "deliver(ing) a massively
scalable cloud operating system." To achieve this, each of the constituent
services are designed to work together to provide a complete Infrastructure as a
Service (IaaS). This integration is facilitated through public application
programming interfaces (APIs) that each service offers (and in turn can
consume). While these APIs allow each of the services to use another service, it
also allows an implementer to switch out any service as long as they maintain
the API. These are (mostly) the same APIs that are available to end users of the
cloud.

Conceptually, you can picture the relationships between the services as so:

[<img src="http://26a0ff8ca8ba32139f7d-db711c577a50b6bdc946ea71aaca027d.r97.cf1.rackcdn.com/openstack-conceptual-arch-folsom.jpg" alt="OpenStack Folsom Conceptual Architecture" width="700px" />](http://26a0ff8ca8ba32139f7d-db711c577a50b6bdc946ea71aaca027d.r97.cf1.rackcdn.com/openstack-conceptual-arch-folsom.jpg)

* Dashboard ("Horizon") provides a web front end to the other OpenStack services
* Compute ("Nova") stores and retrieves virtual disks ("images") and associated
  metadata in Image ("Glance")
* Network ("Quantum") provides virtual networking for Compute.
* Block Storage ("Cinder") provides storage volumes for Compute.
* Image ("Glance") can store the actual virtual disk files in the Object
  Store("Swift")
* All the services authenticate with Identity ("Keystone")

This is a stylized and simplified view of the architecture, assuming that the
implementer is using all of the services together in the most common
configuration. It also only shows the "operator" side of the cloud -- it does
not picture how consumers of the cloud may actually use it. For example, many
users will access object storage heavily (and directly).

##Logical Architecture

As you can imagine, the logical architecture is far more complicated than the
conceptual architecture shown above. As with any service-oriented architecture,
diagrams quickly become "messy" trying to illustrate all the possible
combinations of service communications. The diagram below, illustrates the most
common architecture of an OpenStack-based cloud. However, as OpenStack supports
a wide variety of technologies, it does not represent the only architecture
possible.


[<img src="http://26a0ff8ca8ba32139f7d-db711c577a50b6bdc946ea71aaca027d.r97.cf1.rackcdn.com/openstack-logical-arch-folsom.jpg" alt="OpenStack Folsom Logical Architecture" width="700px" />](http://26a0ff8ca8ba32139f7d-db711c577a50b6bdc946ea71aaca027d.r97.cf1.rackcdn.com/openstack-logical-arch-folsom.jpg)


This picture is consistent with the conceptual architecture above in that:

* End users can interact through a common web interface (Horizon) or directly to
  each service through their API
* All services authenticate through a common source (facilitated through
  Keystone)
* Individual services interact with each other through their public APIs (except
  where privileged administrator commands are necessary)

In the sections below, we'll delve into the architecture for each of the
services.


###Dashboard

Horizon is a modular [Django web application](https://www.djangoproject.com/)
that provides an end user and administrator interface to OpenStack services.


<img src="http://xip-1077754327103.http.internapcdn.net/xip-1077754327103/blog/horizon-screenshot.jpg" alt="OpenStack Horizon Screenshot" width="700px" />


As with most web applications, the architecture is fairly simple:

* Horizon is usually deployed via [mod_wsgi](http://code.google.com/p/modwsgi/)
  in Apache. The code itself is separated into a reusable python module with
  most of the logic (interactions with various OpenStack APIs) and presentation
  (to make it easily customizable for different sites).
* A database (configurable as to which one). As it relies mostly on the other
  services for data, it stores very little data of its own.

From a network architecture point of view, this service will need to be customer
accessible as well as be able to talk to each service's public APIs. If you wish
to use the administrator functionality (i.e. for other services), it will also
need connectivity to their Admin API endpoints (which **should not** be customer
accessible).

###Compute

Nova is the most complicated and distributed component of OpenStack. A large
number of processes cooperate to turn end user API requests into running virtual
machines. Below is a list of these processes and their functions:

* `nova-api` accepts and responds to end user compute API calls. It supports
  OpenStack Compute API, Amazon's EC2 API and a special Admin API (for
  privileged users to perform administrative actions). It also initiates most of
  the orchestration activities (such as running an instance) as well as enforces
  some policy (mostly quota checks).
* The `nova-compute` process is primarily a worker daemon that creates and
  terminates virtual machine instances via hypervisor's APIs (XenAPI for
  XenServer/XCP, libvirt for KVM or QEMU, VMwareAPI for VMware, etc.). The
  process by which it does so is fairly complex but the basics are simple:
  accept actions from the queue and then perform a series of system commands
  (like launching a KVM instance) to carry them out while updating state in the
  database.
* `nova-volume` manages the creation, attaching and detaching of persistent
  volumes to compute instances (similar functionality to Amazon’s Elastic Block
  Storage). It can use volumes from a variety of providers such as iSCSI or
  [Rados Block Device in Ceph](http://ceph.newdream.net). A new OpenStack
  projects, Cinder, will eventually replace `nova-volume` functionality. In the
  Folsom release, `nova-volume` and the Block Storage service will have similar
  functionality.
* The `nova-network` worker daemon is very similar to `nova-compute` and
  `nova-volume`. It accepts networking tasks from the queue and then performs
  tasks to manipulate the network (such as setting up bridging interfaces or
  changing iptables rules). This functionality is being migrated to Quantum, a
  separate OpenStack service. In the Folsom release, much of the functionality
  will be duplicated between `nova-network` and Quantum.
* The `nova-schedule` process is conceptually the simplest piece of code in
  OpenStack Nova: take a virtual machine instance request from the queue and
  determines where it should run (specifically, which compute server host it
  should run on).
* The queue provides a central hub for passing messages between daemons. This is
  usually implemented with [RabbitMQ](http://www.rabbitmq.com/) today, but could
  be any AMPQ message queue (such as [Apache Qpid](http://qpid.apache.org/)).
  New to the Folsom release is support for [Zero MQ](http://www.zeromq.org/)
  (*note: I've only included this so that Eric Windisch won't be hounding me
  mercilessly about it's omission*).
* The SQL database stores most of the build-time and run-time state for a cloud
  infrastructure. This includes the instance types that are available for use,
  instances in use, networks available and projects. Theoretically, OpenStack
  Nova can support any database supported by SQL-Alchemy but the only databases
  currently being widely used are sqlite3 (only appropriate for test and
  development work), MySQL and PostgreSQL.
* Nova also provides console services to allow end users to access their virtual
  instance's console through a proxy. This involves several daemons
  (`nova-console`, `nova-vncproxy` and `nova-consoleauth`).


Nova interacts with many other OpenStack services: Keystone for authentication,
Glance for images and Horizon for web interface. The Glance interactions are
central. The API process can upload and query Glance while `nova-compute` will
download images for use in launching images.

###Object Store

The swift architecture is very distributed to prevent any single point of
failure as well as to scale horizontally. It includes the following components:


* Proxy server (`swift-proxy-server`) accepts incoming requests via
  the OpenStack Object API or just raw HTTP. It accepts files to upload,
  modifications to metadata or container creation. In addition, it will also
  serve files or container listing to web browsers. The proxy server may utilize
  an optional cache (usually deployed with memcache) to improve performance.
* Account servers manage accounts defined with the object storage service.
* Container servers manage a mapping of containers (i.e folders) within the
  object store service.
* Object servers manage actual objects (i.e. files) on the storage nodes.


There are also a number of periodic process which run to perform housekeeping
tasks on the large data store. The most important of these is the replication
services, which ensures consistency and availability through the cluster. Other
periodic processes include auditors, updaters and reapers.

The object store can also serve static web pages and objects via HTTP. In fact, the diagrams in this blog post are being served out of Rackspace Cloud's Swift service.

Authentication is handled through configurable WSGI middleware (which will
usually be Keystone).

###Image Store

The Glance architecture has stayed relatively stable since the Cactus release.
The biggest architectural change has been the addition of authentication, which
was added in the Diablo release. Just as a quick reminder, Glance has four main
parts to it:

* `glance-api` accepts Image API calls for image discovery, image retrieval and
  image storage.
* `glance-registry` stores, processes and retrieves metadata about images (size,
  type, etc.).
* A database to store the image metadata. Like Nova, you can choose your
  database depending on your preference (but most people use MySQL or SQlite).
* A storage repository for the actual image files. In the diagram above, Swift
  is shown as the image repository, but this is configurable. In addition to
  Swift, Glance supports normal filesystems, RADOS block devices, Amazon S3 and
  HTTP. Be aware that some of these choices are limited to read-only usage.

There are also a number of periodic process which run on Glance to support
caching. The most important of these is the replication services, which ensures
consistency and availability through the cluster. Other periodic processes
include auditors, updaters and reapers.

As you can see from the diagram in the Conceptual Architecture section, Glance
serves a central role to the overall IaaS picture. It accepts API requests for
images (or image metadata) from end users or Nova components and can store its
disk files in the object storage service, Swift.

###Identity

Keystone provides a single point of integration for OpenStack policy, catalog,
token and authentication.

* `keystone` handles API requests as well as providing configurable
  catalog, policy, token and identity services.
* Each Keystone function has a pluggable backend which allows different ways to
  use the particular service. Most support standard backends like LDAP or SQL,
  as well as Key Value Stores (KVS).

Most people will use this as a point of customization for their current
authentication services.

###Network

Quantum provides "network connectivity as a service" between interface devices
managed by other OpenStack services (most likely Nova). The service works by
allowing users to create their own networks and then attach interfaces to them.
Like many of the OpenStack services, Quantum is highly configurable due to it's
plug-in architecture. These plug-ins accommodate different networking equipment
and software. As such, the architecture and deployment can vary dramatically. In
the above architecture, a simple Linux networking plug-in is shown.
        
* `quantum-server` accepts API requests and then routes them to the
  appropriate quantum plugin for action.
* Quantum plugins and agents perform the actual actions such as plugging and
  unplugging ports, creating networks or subnets and IP addressing. These
  plugins and agents differ depending on the vendor and technologies used in the
  particular cloud. Quantum ships with plugins and agents for: Cisco virtual and
  physical switches, Nicira NVP product, NEC OpenFlow products, Open vSwitch,
  Linux bridging and the Ryu Network Operating System. The common agents are L3
  (layer 3), DHCP (dynamic host IP addressing) and the specific plug-in agent. 
* Most Quantum installations will also make use of a messaging queue to route
  information between the `quantum-server` and various agents as well as a
  database to store networking state for particular plugins.

Quantum will interact mainly with Nova, where it will provide networks and
connectivity for its instances.

###Block Storage

Cinder separates out the persistent block storage functionality that was
previously part of Openstack Compute (in the form of `nova-volume`) into it's own
service. The OpenStack Block Storage API allows for manipulation of volumes,
volume types (similar to compute flavors) and volume snapshots.

* `cinder-api` accepts API requests and routes them to cinder-volume
  for action.
* `cinder-volume` acts upon the requests by reading or writing to the
  Cinder database to maintain state, interacting with other processes (like
  `cinder-scheduler`) through a message queue and directly upon block
  storage providing hardware or software. It can interact with a variety of
  storage providers through a driver architecture. Currently, there are drivers
  for IBM, SolidFire, NetApp, Nexenta, Zadara, linux iSCSI and other storage
  providers.
* Much like `nova-scheduler`, the `cinder-scheduler` daemon picks the optimal
  block storage provider node to create the volume on.
* Cinder deployments will also make use of a messaging queue to route
  information between the cinder processes as well as a database to store volume
  state.

Like Quantum, Cinder will mainly interact with Nova, providing volumes for its
instances.


## Future Projects

A number of projects are in the work for future versions of OpenStack. They include:

* **Ceilometer** is a metering project. The project offers metering information
  and the ability to code more ways to know what has happened on an OpenStack
  cloud. While it provides metering, it is not a billing project. A full billing
  solution requires metering, rating, and billing. Metering lets you know what
  actions have taken place, rating enables pricing and line items, and billing
  gathers the line items to create a bill to send to the consumer and collect
  payment. Ceilometer is available as a preview.
* **Heat** provides a REST API to orchestrate multiple cloud applications
  implementing standards such as AWS CloudFormation.

