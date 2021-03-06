OpenStack Virtual Baremetal
===========================

A collection of tools to enable the use of OpenStack instances as baremetal
for testing deployment systems.

How-To
------

Instructions for patching the host cloud, setting up the base environment,
and deploying a virtual baremetal Heat stack.

.. warning:: This process requires patches and configuration settings that
             may not be appropriate for production clouds.

Patching the Host Cloud
^^^^^^^^^^^^^^^^^^^^^^^

Apply the Nova pxe boot patch file in the ``patches`` directory to the host
cloud Nova.  Examples:

RDO Kilo::

    sudo cp patches/kilo/nova-pxe-boot.patch /usr/lib/python2.7/site-packages/nova
    cd /usr/lib/python2.7/site-packages/nova
    sudo patch -p1 < nova-pxe-boot.patch

Devstack:

   .. note:: The patch may not apply cleanly against master Nova
             code.  If/when that happens, the patch will need to
             be applied manually.

   ::

      cp patches/kilo/nova-pxe-boot.patch /opt/stack/nova
      cd /opt/stack/nova
      patch -p1 < nova-pxe-boot.patch

Configuring the Host Cloud
^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Neutron must be configured to use the NoopFirewallDriver.  Edit
   ``/etc/neutron/plugins/ml2/ml2_conf.ini`` and set the option
   ``firewall_driver`` in the ``[securitygroup]`` section as follows::

       firewall_driver = neutron.agent.firewall.NoopFirewallDriver

#. In Liberty and later versions, arp spoofing must be disabled.  Edit
   ``/etc/neutron/plugins/ml2/ml2_conf.ini`` and set the option
   ``prevent_arp_spoofing`` in the ``[agent]`` section as follows::

        prevent_arp_spoofing = False

#. The Nova option ``force_config_drive`` must _not_ be set.

#. (Optional) It can be helpful to set::

        shutdown_timeout=15

   in nova.conf as well.  This causes Nova to wait less time when shutting
   down an instance gracefully, and since graceful shutdown will never
   happen with the baremetal instances booted from an empty image it speeds
   things up a bit.

#. ``/etc/nova/nova.conf`` needs ``allow_duplicate_networks=true`` in the
   ``[neutron]`` section so that the BMC instance can have multiple nics on the
   same network.

#. Restart ``nova-compute`` and ``neutron-openvswitch-agent`` to apply the
   changes above.

Preparing the Host Cloud Environment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Source an rc file that will provide admin credentials for the host cloud.

#. Create an empty base image for the baremetal instances::

    qemu-img create -f qcow2 empty.qcow2 40G
    glance image-create --name empty --disk-format qcow2 --container-format bare < empty.qcow2

#. Upload a CentOS 7 image for use as the base BMC instance::

    wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1503.qcow2

    glance image-create --name CentOS-7-x86_64-GenericCloud-1503 --disk-format qcow2 --container-format bare < CentOS-7-x86_64-GenericCloud-1503.qcow2

#. Create recommended flavors::

    nova flavor-create baremetal auto 4096 50 2
    nova flavor-create bmc auto 512 20 1

   These flavors can be customized if desired.  For large environments
   with many baremetal instances it may be wise to give the bmc flavor
   more memory.  A 512 MB BMC will run out of memory around 20 baremetal
   instances.

#. Source an rc file that will provide user credentials for the host cloud.

#. Create provisioning network.

   .. note:: The CIDR used for the subnet does not matter.
             Standard tenant and external networks are also needed to
             provide floating ip access to the undercloud and bmc instances

   .. warning:: Do not enable DHCP on this network.  Addresses will be
                assigned by the undercloud Neutron.

   ::

      neutron net-create provision
      neutron subnet-create --name provision --no-gateway --disable-dhcp provision 192.0.2.0/24

#. Create "public" network.

   .. note:: The CIDR used for the subnet does not matter.
             This can be used as the network for the public API endpoints
             on the overcloud, but it does not have to be accessible
             externally.  Only the undercloud VM will need to have access
             to this network.

   .. warning:: Do not enable DHCP on this network.  Doing so may cause
                conflicts between the host cloud metadata service and the
                undercloud metadata service.  Overcloud nodes will be
                assigned addresses on this network by the undercloud Neutron.

   ::

       neutron net-create public
       neutron subnet-create --name public --no-gateway --disable-dhcp public 10.0.0.0/24

#. Add a Nova keypair to be injected into instances::

    nova keypair-add --pub-key ~/.ssh/id_rsa.pub default

Create the baremetal Heat stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Copy the example env file and edit it to reflect the host environment::

    cp templates/env.yaml.example env.yaml
    vi env.yaml

#. Deploy the stack::

    bin/deploy-stack

#. Wait for Heat stack to complete:

   .. note:: The BMC instance does post-deployment configuration that can
             take a while to complete, so the Heat stack completing does
             not necessarily mean the environment is entirely ready for
             use.  To determine whether the BMC is finished starting up,
             run ``nova console-log bmc``.  The BMC service outputs a
             message like "Managing instance [uuid]" when it is fully
             configured.  There should be one of these messages for each
             baremetal instance.

   ::

      heat stack-show baremetal

#. Boot a VM to serve as the undercloud::

    nova boot undercloud --flavor m1.large --image centos7 --nic net-id=[tenant net uuid] --nic net-id=[provisioning net uuid]
    neutron floatingip-create [external net uuid]
    neutron port-list
    neutron floatingip-associate [floatingip uuid] [undercloud instance port id]

#. Build a nodes.json file that can be imported into Ironic::

    bin/build-nodes-json
    scp nodes.json centos@[undercloud floating ip]:~/instackenv.json

   .. note:: ``build-nodes-json`` also outputs a file named ``bmc_bm_pairs``
             that lists which BMC address corresponds to a given baremetal
             instance.

#. The undercloud vm can now be used with something like RDO Manager
   to do a baremetal-style deployment to the virtual baremetal instances
   deployed previously.
