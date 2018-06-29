========================
openstack-service-broker
========================

Builds Ansible Playbook Bundles for use in Automation Broker to expose
OpenStack resources in the Kubernetes Service Catalog.

The project is in the very early stages of development. We will first build a
prototype that demonstrates the concept of managing OpenStack resources through
the Kubernetes Service Catalog, using the http://automationbroker.io/ project
to implement the Open Service Broker API, which in turn uses Ansible playbooks
to drive the underlying services.

* Free software: Apache license
* Source: https://git.openstack.org/cgit/openstack/openstack-service-broker
* Bugs: https://storyboard.openstack.org/#!/project/1038

--------

Join Us
=======

* On the `openstack-dev`_ mailing list (use ``[service-broker]`` in the subject
  line)
* In ``#openstack-service-broker`` on `FreeNode`_.
* Working on the `prototype worklist`_.

.. _openstack-dev: http://lists.openstack.org/pipermail/openstack-dev/
.. _FreeNode: https://freenode.net/
.. _prototype worklist: https://storyboard.openstack.org/#!/worklist/391

FAQ
===

What is the Open Service Broker API?
------------------------------------

The `Open Service Broker API`_ is a standard way to expose external resources
to applications running in a PaaS. It was originally developed in the context
of CloudFoundry, but the same standard was adopted by Kubernetes (and hence
OpenShift) in the form of the `Service Catalog extension`_. (The Service
Catalog in Kubernetes is the component that calls out to a service broker.) So
a single implementation can cover the most popular open-source PaaS offerings.

In many cases, the services take the form of simply a pre-packaged application
that also runs inside the PaaS. But they don't have to be - services can be
anything. Provisioning via the service broker ensures that the services
requested are tied in to the PaaS's orchestration of the application's
lifecycle.

(This is certainly not the be-all and end-all of integration between OpenStack
and containers - we also need ways to tie PaaS-based applications into the
OpenStack's orchestration of a larger group of resources. Some applications may
even use both. But it's an important part of the story.)

What sorts of services would OpenStack expose?
----------------------------------------------

Some example use cases might be:

* The application needs a reliable message queue. Rather than spinning up
  multiple storage-backed containers with anti-affinity policies and dealing
  with the overhead of managing e.g. RabbitMQ, the application requests a Zaqar
  queue from an OpenStack cloud. The overhead of running the queueing service
  is amortised across all of the applications in the cloud. The queue gets
  cleaned up correctly when the application is removed, since it is tied into
  the application definition.
* The application needs a database. Rather than spinning one up in a
  storage-backed container and dealing with the overhead of managing it, the
  application requests a Trove DB from an OpenStack cloud.
* The application includes a service that needs to run on bare metal for
  performance reasons (e.g. could also be a database). The application requests
  a bare-metal server from Nova w/ Ironic for the purpose. (The same applies to
  requesting a VM, but there are alternatives like KubeVirt - which also
  operates through the Service Catalog - available for getting a VM in
  Kubernetes. There are no non-proprietary alternatives for getting a
  bare-metal server.)

`AWS`_, `Azure`_, and `GCP`_ all have service brokers available that support
these and many more services that they provide. I don't know of any reason in
principle not to expose every type of resource that OpenStack provides via a
service broker.

How is this different from cloud-provider-openstack?
----------------------------------------------------

The `Cloud Controller`_ interface in Kubernetes allows Kubernetes itself to
access features of the cloud to provide its service. For example, if k8s needs
persistent storage for a container then it can request that from Cinder through
`cloud-provider-openstack`_. It can also request a load balancer from Octavia
instead of having to start a container running HAProxy to load balance between
multiple instances of an application container (thus enabling use of hardware
load balancers via the cloud's abstraction for them).

In contrast, the Service Catalog interface allows the *application* running on
Kubernetes to access features of the cloud.

What does a service broker look like?
-------------------------------------

A service broker provides an HTTP API with 5 actions:

* List the services provided by the broker
* Create an instance of a resource
* Bind the resource into an instance of the application
* Unbind the resource from an instance of the application
* Delete the resource

The binding step is used for things like providing a set of DB credentials to a
container. You can rotate credentials when replacing a container by revoking
the existing credentials on unbind and creating a new set on bind, without
replacing the entire resource.

Is there an easier way?
-----------------------

Yes! Folks from OpenShift came up with a project called the `Automation
Broker`_. To add support for a service to Automation Broker you just create a
container with an Ansible playbook to handle each of the actions
(create/bind/unbind/delete). This eliminates the need to write another
implementation of the service broker API, and allows us to simply `write
Ansible playbooks instead`_.

(Aside: Heat uses a comparable method to allow users to manage an external
resource using Mistral workflows: the OS::Mistral::ExternalResource resource
type.)

Support for accessing `AWS`_ resources through a service broker is also
implemented using these Ansible Playbook Bundles.

Does this mean maintaining another client interface?
----------------------------------------------------

Maybe not. We already have per-project Python libraries, (deprecated)
per-project CLIs, openstackclient CLIs, openstack-sdk, shade, Heat resource
plugins, and Horizon dashboards. (Mistral actions are generated automatically
from the clients.) Some consolidation is already planned, but it would be great
not to require projects to maintain yet another interface.

One option is to implement a tool that generates a set of playbooks for each of
the resources already exposed (via shade) in the OpenStack Ansible modules.
Then in theory we'd only need to implement the common parts once, and then
every service with support in shade would get this for free. Ideally the same
broker could be used against any OpenStack cloud (so e.g. k8s might be running
in your private cloud, but you may want its service catalog to allow you to
connect to resources in one or more public clouds) - using shade is an
advantage there because it is designed to abstract the differences between
clouds.

Another option might be to write or generate Heat templates for each resource
type we want to expose. Then we'd only need to implement a common way of
creating a Heat stack, and just have a different template for each resource
type. This is the approach taken by the AWS playbook bundles (except with
CloudFormation, obviously). An advantage is that this allows Heat to do any
checking and type conversion required on the input parameters. Heat templates
can also be made to be fairly cloud-independent, mainly because they make it
easier to be explicit about things like ports and subnets than on the command
line, where it's more tempting to allow things to happen in a magical but
cloud-specific way.

I'd prefer to go with the pure-Ansible autogenerated way so we can have support
for everything, but looking at the `GCP`_/`Azure`_/`AWS`_ brokers they have 10,
11 and 17 services respectively, so arguably we could get a comparable number
of features exposed without investing crazy amounts of time if we had to write
templates explicitly.

How would authentication work?
------------------------------

There are two main deployment topologies we need to consider: Kubernetes
deployed by an OpenStack tenant (Magnum-style, though not necessarily using
Magnum) and accessing resources in that tenant's project in the local cloud, or
accessing resources in some remote OpenStack cloud.

We also need to take into account that in the second case, the Kubernetes
cluster may 'belong' to a single cloud tenant (as in the first case) or may be
shared by applications that each need to authenticate to different OpenStack
tenants. (Kubernetes has traditionally assumed the former, but I expect it to
move in the direction of allowing the latter, and it's already fairly common
for OpenShift deployments.)

The way e.g. the `AWS`_ broker works is that you can either use the credentials
provisioned to the VM that k8s is installed on (a 'Role' in AWS parlance - note
that this is completely different to a Keystone Role), or supply credentials to
authenticate to AWS remotely.

OpenStack doesn't yet support per-instance credentials, although we're working
on it. (One thing to keep in mind is that ideally we'll want a way to provide
different permissions to the service broker and cloud-provider-openstack.) An
option in the meantime might be to provide a way to set up credentials as part
of the k8s installation. We'd also need to have a way to specify credentials
manually. Unlike for proprietary clouds, the credentials also need to include
the Keystone auth_url. We should try to reuse openstacksdk's
`clouds.yaml/secure.yaml format`_ if possible.

The OpenShift Ansible Broker works by starting up an Ansible container on k8s
to run a playbook from the bundle, so presumably credentials can be passed as
regular k8s secrets.

In all cases we'll want to encourage users to authenticate using Keystone
`Application Credentials`_.

How would network integration work?
-----------------------------------

`Kuryr`_ allows us to connect application containers in Kubernetes to Neutron
networks in OpenStack. It would be desirable if, when the user requests a VM or
bare-metal server through the service broker, it were possible to choose
between attaching to the same network as Kubernetes pods, or to a different
network.


.. _Open Service Broker API: https://www.openservicebrokerapi.org/
.. _Service Catalog extension: https://kubernetes.io/docs/concepts/service-catalog/
.. _AWS: https://github.com/awslabs/aws-servicebroker#aws-service-broker
.. _Azure: https://github.com/Azure/open-service-broker-azure#open-service-broker-for-azure
.. _GCP: https://github.com/GoogleCloudPlatform/gcp-service-broker#cloud-foundry-service-broker-for-google-cloud-platform
.. _Cloud Controller: https://github.com/kubernetes/community/blob/master/keps/0002-controller-manager.md#remove-cloud-provider-code-from-kubernetes-core
.. _cloud-provider-openstack: https://github.com/kubernetes/cloud-provider-openstack#openstack-cloud-controller-manager
.. _Automation Broker: http://automationbroker.io/
.. _write Ansible playbooks instead: https://docs.openshift.org/latest/apb_devel/index.html
.. _clouds.yaml/secure.yaml format: https://docs.openstack.org/openstacksdk/latest/user/config/configuration.html#config-files
.. _Application Credentials: https://docs.openstack.org/keystone/latest/user/application_credentials.html
.. _Kuryr: https://docs.openstack.org/kuryr/latest/devref/goals_and_use_cases.html
