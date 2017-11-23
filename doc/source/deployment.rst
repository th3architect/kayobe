==========
Deployment
==========

This section describes usage of Kayobe to install an OpenStack cloud onto a set
of bare metal servers.  We assume access is available to a node which will act
as the hypervisor hosting the seed node in a VM.  We also assume that this seed
hypervisor has access to the bare metal nodes that will form the OpenStack
control plane.  Finally, we assume that the control plane nodes have access to
the bare metal nodes that will form the workload node pool.

Ansible Control Host
====================

Before starting deployment we must bootstrap the Ansible control host.  Tasks
performed here include:

- Install Ansible and role dependencies from Ansible Galaxy.
- Generate an SSH key if necessary and add it to the current user's authorised
  keys.

To bootstrap the Ansible control host::

    (kayobe) $ kayobe control host bootstrap

Physical Network
================

The physical network can be managed by Kayobe, which uses Ansible's network
modules.  Currently Dell Network OS 6 and Dell Network OS 9 switches are
supported but this could easily be extended.  To provision the physical
network::

    (kayobe) $ kayobe physical network configure --group <group> [--enable-discovery]

The ``--group`` argument is used to specify an Ansible group containing
the switches to be configured.

The ``--enable-discovery`` argument enables a one-time configuration of ports
attached to baremetal compute nodes to support hardware discovery via ironic
inspector.

It is possible to limit the switch interfaces that will be configured, either
by interface name or interface description::

    (kayobe) $ kayobe physical network configure --group <group> --interface-limit <interface names>
    (kayobe) $ kayobe physical network configure --group <group> --interface-description-limit <interface descriptions>

The names or descriptions should be separated by commas.  This may be useful
when adding compute nodes to an existing deployment, in order to avoid changing
the configuration interfaces in use by active nodes.

The ``display`` argument will display the candidate switch configuration,
without actually applying it.

Seed Hypervisor
===============

.. note::

   It is not necessary to run the seed services in a VM.  To use an existing
   bare metal host or a VM provisioned outside of Kayobe, this section may be
   skipped.

Host Configuration
------------------

To configure the seed hypervisor's host OS, and the Libvirt/KVM virtualisation
support::

    (kayobe) $ kayobe seed hypervisor host configure

Seed
====

VM Provisioning
---------------

.. note::

   It is not necessary to run the seed services in a VM.  To use an existing
   bare metal host or a VM provisioned outside of Kayobe, this step may be
   skipped.  Ensure that the Ansible inventory contains a host for the seed.

The seed hypervisor should have CentOS and ``libvirt`` installed.  It should
have ``libvirt`` networks configured for all networks that the seed VM needs
access to and a ``libvirt`` storage pool available for the seed VM's volumes.
To provision the seed VM::

    (kayobe) $ kayobe seed vm provision

When this command has completed the seed VM should be active and accessible via
SSH.  Kayobe will update the Ansible inventory with the IP address of the VM.

Host Configuration
------------------

To configure the seed host OS::

    (kayobe) $ kayobe seed host configure

.. note::

   If the seed host uses disks that have been in use in a previous
   installation, it may be necessary to wipe partition and LVM data from those
   disks.  To wipe all disks that are not mounted during host configuration::

       (kayobe) $ kayobe seed host configure --wipe-disks

Building Container Images
-------------------------

.. note::

   It is possible to use prebuilt container images from an image registry such
   as Dockerhub.  In this case, this step can be skipped.

It is possible to use prebuilt container images from an image registry such as
Dockerhub.  In some cases it may be necessary to build images locally either to
apply local image customisation or to use a downstream version of kolla.
Images are built by hosts in the ``container-image-builders`` group, which by
default includes the ``seed``.

To build container images::

    (kayobe) $ kayobe seed container image build

It is possible to build a specific set of images by supplying one or more
image name regular expressions::

    (kayobe) $ kayobe seed container image build bifrost-deploy

In order to push images to a registry after they are built, add the ``--push``
argument.

Deploying Containerised Services
--------------------------------

At this point the seed services need to be deployed on the seed VM.  These
services are deployed in the ``bifrost_deploy`` container.  This command will
also build the Operating System image that will be used to deploy the overcloud
nodes using Disk Image Builder (DIB).

To deploy the seed services in containers::

    (kayobe) $ kayobe seed service deploy

After this command has completed the seed services will be active.

Building Deployment Images
--------------------------

.. note::

   It is possible to use prebuilt deployment images. In this case, this step
   can be skipped.

It is possible to use prebuilt deployment images from the `OpenStack hosted
tarballs <https://tarballs.openstack.org/ironic-python-agent>`_ or another
source.  In some cases it may be necessary to build images locally either to
apply local image customisation or to use a downstream version of Ironic Python
Agent (IPA).  In order to build IPA images, the ``ipa_build_images`` variable
should be set to ``True``.  To build images locally::

    (kayobe) $ kayobe seed deployment image build

Accessing the Seed via SSH (Optional)
-------------------------------------

For SSH access to the seed, first determine the seed's IP address. We can
use the ``kayobe configuration dump`` command to inspect the seed's IP
address::

    (kayobe) $ kayobe configuration dump --host seed --var-name ansible_host

The ``kayobe_ansible_user`` variable determines which user account will be used
by Kayobe when accessing the machine via SSH.  By default this is ``stack``.
Use this user to access the seed::

    $ ssh <kayobe ansible user>@<seed VM IP>

To see the active Docker containers::

    $ docker ps

Leave the seed VM and return to the shell on the control host::

    $ exit

Overcloud
=========

Discovery
---------

.. note::

   If discovery of the overcloud is not possible, a static inventory of servers
   using the bifrost ``servers.yml`` file format may be configured using the
   ``kolla_bifrost_servers`` variable in ``${KAYOBE_CONFIG_PATH}/bifrost.yml``.

Discovery of the overcloud is supported by the ironic inspector service running
in the ``bifrost_deploy`` container on the seed.  The service is configured to
PXE boot unrecognised MAC addresses with an IPA ramdisk for introspection.  If
an introspected node does not exist in the ironic inventory, ironic inspector
will create a new entry for it.

Discovery of the overcloud is triggered by causing the nodes to PXE boot using
a NIC attached to the overcloud provisioning network.  For many servers this
will be the factory default and can be performed by powering them on.

On completion of the discovery process, the overcloud nodes should be
registered with the ironic service running in the seed host's
``bifrost_deploy`` container.  The node inventory can be viewed by executing
the following on the seed::

    $ docker exec -it bifrost_deploy bash
    (bifrost_deploy) $ source env-vars
    (bifrost_deploy) $ ironic node-list

In order to interact with these nodes using Kayobe, run the following command
to add them to the Kayobe and bifrost Ansible inventories::

    (kayobe) $ kayobe overcloud inventory discover

Saving Hardware Introspection Data
----------------------------------

If ironic inspector is in use on the seed host, introspection data will be
stored in the local nginx service.  This data may be saved to the control
host::

    (kayobe) $ kayobe overcloud introspection data save

``--output-dir`` may be used to specify the directory in which introspection
data files will be saved. ``--output-format`` may be used to set the format of
the files.

BIOS and RAID Configuration
---------------------------

.. note::

   BIOS and RAID configuration may require one or more power cycles of the
   hardware to complete the operation.  These will be performed automatically.

Configuration of BIOS settings and RAID volumes is currently performed out of
band as a separate task from hardware provisioning.  To configure the BIOS and
RAID::

    (kayobe) $ kayobe overcloud bios raid configure

After configuring the nodes' RAID volumes it may be necessary to perform
hardware inspection of the nodes to reconfigure the ironic nodes' scheduling
properties and root device hints.  To perform manual hardware inspection::

    (kayobe) $ kayobe overcloud hardware inspect

Provisioning
------------

Provisioning of the overcloud is performed by the ironic service running in the
bifrost container on the seed.  To provision the overcloud nodes::

    (kayobe) $ kayobe overcloud provision

After this command has completed the overcloud nodes should have been
provisioned with an OS image.  The command will wait for the nodes to become
``active`` in ironic and accessible via SSH.

Host Configuration
------------------

To configure the overcloud hosts' OS::

    (kayobe) $ kayobe overcloud host configure

.. note::

   If the controller hosts use disks that have been in use in a previous
   installation, it may be necessary to wipe partition and LVM data from those
   disks.  To wipe all disks that are not mounted during host configuration::

       (kayobe) $ kayobe overcloud host configure --wipe-disks

Building Container Images
-------------------------

.. note::

   It is possible to use prebuilt container images from an image registry such
   as Dockerhub.  In this case, this step can be skipped.

In some cases it may be necessary to build images locally either to apply local
image customisation or to use a downstream version of kolla.  Images are built
by hosts in the ``container-image-builders`` group, which by default includes
the ``seed``. If no seed host is in use, for example in an all-in-one
controller development environment, this group may be modified to cause
containers to be built on the controllers.

To build container images::

    (kayobe) $ kayobe overcloud container image build

It is possible to build a specific set of images by supplying one or more
image name regular expressions::

    (kayobe) $ kayobe overcloud container image build ironic- nova-api

In order to push images to a registry after they are built, add the ``--push``
argument.

Pulling Container Images
------------------------

.. note::

   It is possible to build container images locally avoiding the need for an
   image registry such as Dockerhub.  In this case, this step can be skipped.

In most cases suitable prebuilt kolla images will be available on Dockerhub.
The `stackhpc account <https://hub.docker.com/r/stackhpc/>`_ provides image
repositories suitable for use with kayobe and will be used by default.  To
pull images from the configured image registry::

    (kayobe) $ kayobe overcloud container image pull

Building Deployment Images
--------------------------

.. note::

   It is possible to use prebuilt deployment images. In this case, this step
   can be skipped.

It is possible to use prebuilt deployment images from the `OpenStack hosted
tarballs <https://tarballs.openstack.org/ironic-python-agent>`_ or another
source.  In some cases it may be necessary to build images locally either to
apply local image customisation or to use a downstream version of Ironic Python
Agent (IPA).  In order to build IPA images, the ``ipa_build_images`` variable
should be set to ``True``.  To build images locally::

    (kayobe) $ kayobe overcloud deployment image build

Deploying Containerised Services
--------------------------------

To deploy the overcloud services in containers::

    (kayobe) $ kayobe overcloud service deploy

Once this command has completed the overcloud nodes should have OpenStack
services running in Docker containers.

Interacting with the Control Plane
----------------------------------

Kolla-ansible writes out an environment file that can be used to access the
OpenStack admin endpoints as the admin user::

    $ source ${KOLLA_CONFIG_PATH:-/etc/kolla}/admin-openrc.sh

Kayobe also generates an environment file that can be used to access the
OpenStack public endpoints as the admin user which may be required if the
admin endpoints are not available from the control host::

    $ source ${KOLLA_CONFIG_PATH:-/etc/kolla}/public-openrc.sh

Performing Post-deployment Configuration
----------------------------------------

To perform post deployment configuration of the overcloud services::

    (kayobe) $ source ${KOLLA_CONFIG_PATH:-/etc/kolla}/admin-openrc.sh
    (kayobe) $ kayobe overcloud post configure

This will perform the following tasks:

- Register Ironic Python Agent (IPA) images with glance
- Register introspection rules with ironic inspector
- Register a provisioning network and subnet with neutron
