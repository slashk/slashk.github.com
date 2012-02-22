--- 
layout: post
title: "Revisiting OpenStack Architecture: Essex Edition"
category: openstack
---

While it is still [six weeks until OpenStack "Essex" (2012.1) officially is released](http://wiki.openstack.org/EssexReleaseSchedule), release candidates are just around the corner. With this in mind, I thought it would be a good chance to revisit my earlier blog post on [OpenStack Compute ("Nova") architecture](http://ken.pepple.info/openstack/2011/04/22/openstack-nova-architecture/). This time around, instead of detailing the architecture of just a single service, I'll look at all the pieces of the OpenStack project working together.

To level-set everyone's understanding, let's briefly review the OpenStack project components and history. Founded in 2010 by Rackspace and NASA, the project has [released four versions](http://wiki.openstack.org/Releases) and is set to release the fifth ("Essex" or 2012.1) [in April](http://wiki.openstack.org/EssexReleaseSchedule). Originally, it consisted of a trio of "core" services:

* Object Store (["Swift"](http://swift.openstack.org)) provides object storage. It allows you to store or retrieve files (but not mount directories like a fileserver). Several companies provide commercial storage services based on Swift. These include KT, Rackspace (from which Swift originated) and my company   [Internap](http://www.internap.com/flexible-cloud-hosting-solutions/cloud-storage/). In fact, the images for this blog post are being served via the Internap Swift implementation.
* Image (["Glance"](http://glance.openstack.org)) provides a catalog and repository for virtual disk images. These disk images are mostly commonly used in OpenStack Compute. While this service is technically optional, any cloud of size will require it.
* Compute (["Nova"](http://nova.openstack.org/)) provides virtual servers upon demand. Similar to Amazon's EC2 service, it also provides volume services analogous to Elastic Block Services (EBS). [Internap](http://www.internap.com/flexible-cloud-hosting-solutions/enterprise-public-cloud-solutions/open-public-cloud-solutions/) provide a commercial compute service built on Nova and it is used internally at [Mercado Libre](http://openstack.org/user-stories/mercadolibre-inc/) and NASA (where it originated).

The upcoming release promotes two new projects to "core" project status:

* Dashboard (["Horizon"](http://horizon.openstack.org/)) provides a modular web-based user interface for all the OpenStack services.
* Identity (["Keystone"](http://keystone.openstack.org/)) provides authentication and authorization for all the OpenStack services. It also provides a service catalog of services within a particular deployment.

These two projects provide additional infrastructure to support the original three projects.

# Conceptual Architecture

The OpenStack project as a whole is designed to "deliver(ing) a massively scalable cloud operating system." To achieve this, each of the constituent services are designed to work together to provide a complete Infrastructure as a Service (IaaS). This integration is facilitated through public application programming interfaces (APIs) that each service offers (and in turn can consume). While these APIs allow each of the services to use another service, it also allows an implementer to switch out any service as long as they maintain the API. These are (mostly) the same APIs that are available to end users of the cloud. 

Conceptually, you can picture the relationships between the services as so:

<figure>
<img src="http://xip-1077754327103.http.internapcdn.net/xip-1077754327103/blog/nova-concept-int-essex.jpg" alt="OpenStack Essex Conceptual Integration" width="700px" />
<figcaption>OpenStack Essex Conceptual Integration</figcaption></figure>

* Horizon provides a web front end to the other OpenStack services
* Nova stores and retrieves virtual disks ("images") and associated metadata in Glance
* Glance can store the actual virtual disk files in Swift
* All the services (*will eventually*) authenticate with Keystone

This is a stylized and simplified view of the architecture, assuming that the implementer is using all of the services together in the most common configuration. It also only shows the "operator" side of the cloud -- it does not picture how consumers of the cloud may actually use it. For example, many compute users will use object storage heavily (and directly).

# Logical Architecture

As you can imagine, the actual logical architecture is far more complicated than the conceptual architecture shown above. As with any service-oriented architecture, diagrams quickly become "messy" trying to illustrate all the possible combinations of service communications. In the diagram below, I illustrate what I believe will be the most common, "integrated" architecture of an OpenStack-based cloud.

<figure>
<img src="http://xip-1077754327103.http.internapcdn.net/xip-1077754327103/blog/nova-logical-arch-essex.jpg" alt="OpenStack Essex Logical Architecture" width="700px" />
<figcaption>OpenStack Essex Logical Architecture</figcaption></figure>


This picture is consistent with the description above in that:

* End users can interact through a common web interface (Horizon) or directly to each service through their API
* All services authenticate through a common source (facilitated through Keystone)
* Individual services interact with each other through their public APIs (except where privileged administrator commands are necessary)

In the sections below, we'll delve into the architecture for each of the services.  

## Dashboard

Horizon is a modular [Django web application](https://www.djangoproject.com/) that provides an end user and administrator interface to OpenStack services. 

<figure>
<img src="http://xip-1077754327103.http.internapcdn.net/xip-1077754327103/blog/horizon-screenshot.jpg" alt="OpenStack Horizon Screenshot" width="700px" />
<figcaption>OpenStack Horizon Screenshot</figcaption></figure>


As with most web applications, the architecture is fairly simple:

* Horizon is usually deployed via [mod_wsgi](http://code.google.com/p/modwsgi/) in Apache. The code itself is seperated into a reusable python module with most of the logic (interactions with various OpenStack APIs) and presentation (to make it easily customizable for different sites).
* A database (configurable to which one). As it relies mostly on the other services for data, it stores very little data of it's own. 

From a network architecture point of view, this service will need to customer accessible as well as be able to talk to each services public APIs. If you wish to use the administrator functionality (i.e. for other services), it will also need connectivity to their Admin API endpoints (which should be non-customer accessible).

## Compute

Not much has really changed with Nova's architecture. They have added a few new helper services for EC2 compatibility and console services.

* nova-api accepts and responds to end user compute and volume API calls. It supports OpenStack API, Amazon's EC2 API and a special Admin API (for privileged users to perform administrative actions). It also initiates most of the orchestration activities (such as running an instance) as well as enforces some policy (mostly quota checks). In the Essex release, nova-api has been modularized, allowing for implementers to run only specific APIs.
* The nova-compute process is primarily a worker daemon that creates and terminates virtual machine instances via hypervisor's APIs (XenAPI for XenServer/XCP, libvirt for KVM or QEMU, VMwareAPI for VMware, etc.). The process by which it does so is fairly complex but the basics are simple: accept actions from the queue and then perform a series of system commands (like launching a KVM instance) to carry them out while updating state in the database.
* nova-volume manages the creation, attaching and detaching of persistent volumes to compute instances (similar functionality to Amazon’s Elastic Block Storage). It can use volumes from a variety of providers such as iSCSI or AoE.
* The nova-network worker daemon is very similar to nova-compute and nova-volume. It accepts networking tasks from the queue and then performs tasks to manipulate the network (such as setting up bridging interfaces or changing iptables rules).
* The nova-schedule process is conceptually the simplest piece of code in OpenStack Nova: take a virtual machine instance request from the queue and determines where it should run (specifically, which compute server host it should run on). In practice however, I am sure this will grow to be the most complex as it needs to factor in current state of the entire cloud infrastructure and apply complicated algorithm to ensure efficient usage. To that end, nova-schedule implements a pluggable architecture that let’s you choose (or write) your own algorithm for scheduling. Currently, there are several to choose from (simple, chance, etc) and it is a area of hot development for the future releases of OpenStack Nova.
* The queue provides a central hub for passing messages between daemons. This is usually implemented with [RabbitMQ](http://www.rabbitmq.com/) today, but could be any AMPQ message queue (such as [Apache Qpid](http://qpid.apache.org/)).
* The SQL database stores most of the build-time and run-time state for a cloud infrastructure. This includes the instance types that are available for use, instances in use, networks available and projects. Theoretically, OpenStack Nova can support any database supported by SQL-Alchemy but the only databases currently being widely used are sqlite3 (only appropriate for test and development work), MySQL and PostgreSQL.

New to Nova in the intermediate releases are augmented console services. Console services allow end users to access their virtual instance's console through a proxy. This involves a pair of new daemons (nova-console and nova-consoleauth).

Nova interacts with all of the usual suspects: Keystone for authentication, Glance for images and Horizon for web interface. The Glance interacts is interesting, though. The API process can upload and query Glance while nova-compute will download images for use in launching images.

## Object Store

The swift architecture is very distributed to prevent any single point of failure as well as to scale horizontally. It includes the following components: 

* Proxy server accepts incoming requests via the OpenStack Object API or just raw HTTP. It accepts files to upload, modifications to metadata or container creation. In addition, it will also serve files or container listing to web browsers. The proxy server may utilize an optional cache (usually deployed with memcache) to improve performance.
* Account servers manage accounts defined with the object storage service.
* Container servers manages contains a mapping of containers (i.e folders) within the object store service.
• Object servers manages actual objects (i.e. files) on the storage nodes.
* There are also a number of periodic process which run to perform housekeeping tasks on the large data store. The most important of these is the replication services, which ensures consistency and availability through the cluster. Other periodic processes include auditors, updaters and reapers.

Authentication is handled through configurable WSGI middleware (which will usually be Keystone).

## Image Store

The Glance architecture has stayed relatively stable since the Cactus release. The biggest architectural change has been the addition of authentication, which was added in the Diablo release. Just as a quick reminder, Glance has four main parts to it:

* glance-api accepts Image API calls for image discovery, image retrieval and image storage
* glance-registry stores, processes and retrieves metadata about images (size, type, etc.)
* A database to store the image metadata. Like Nova, you can choose your database depending on your preference (but most people use MySQL or SQlite).
* A storage repository for the actual image files. In the diagram, I have shown the most likely configuration (using Swift as the image repository), but this is configurable. In addition to Swift, Glance supports normal filesystems, RADOS block devices, Amazon S3 and HTTP. Be aware that some of these choices are limited to read-only usage.

There are also a number of periodic process which run on Glance to support caching. The most important of these is the replication services, which ensures consistency and availability through the cluster. Other periodic processes include auditors, updaters and reapers.

As you can see from the diagram, Glance serves a central role to the overall IaaS picture. It accepts API requests for images (or image metadata) from end users or Nova components and can store it's disk files in  

## Identity

Keystone provides a single point of integration for OpenStack policy, catalog, token and authentication. 

* Keystone handles API requests as well as providing configurable catalog, policy, token and identity services. 
* Each keystone function has a pluggable backend which allows different ways to use the particular service. Most support standard backends like LDAP or SQL, as well as Key Value Stores (KVS).

Most people will use this as a point of customization for their current authentication services.

## Future Projects

The completes the tour of the OpenStack Essex architecture. However, OpenStack will not stopping there - the following OpenStack release ("Folsom") [will welcome another core service to the fold](http://eavesdrop.openstack.org/meetings/openstack-meeting/2012/openstack-meeting.2012-02-21-19.59.html):

* Network ([Quantum](http://docs.openstack.org/incubation/openstack-network/admin/content/)) provides "network connectivity as a service" between interface devices managed by other Openstack services (most likely Nova).

Although the [release schedule for Folsom](http://www.openstack.org/conference/san-francisco-2012/) is not yet set (probably Fall 2012), I won't wait six months to update the picture for this.