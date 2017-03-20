========================================
Playbook for non-disrupting Ceph upgrade
========================================

Synopsis
--------

This document describes how to upgrade a Ceph cluster deployed by Fuel_.
The playbook works fine with clusters deployed by `ceph-ansible`_ and
possibly other tools, however you might need to adjust ``ansible.cfg``.

* `Supported Ceph versions`_
* `Preparations`_
* `Preflight checks`_
* `Upgrade ceph cluster`_
* `Restart VMs`_


Supported Ceph versions
-----------------------

The playbook works with Hammer and Jewel Ceph releases, *provided that both
the currently running and target Ceph versions have no bugs hindering the upgrade*.
In reality even the patchlevel releases of the same branch happen to cause
upgrade issues, see `bug #17386`_, `bug #18582`_. It's a good idea to
perform an upgrade of a test cluster before upgrading a production one using
the same versions of Ceph (both ``target`` and ``current`` versions).

The following upgrades are known to work:

* From 0.94.6 (Hammer) to 0.94.9 and 0.94.10 patched by Mirantis
* From 0.94.9 patched by Mirantis to 10.2.6 (Jewel), both upstream and
  Mirantis' version
* From 0.94.10 patched by Mirantis to 10.2.6 (Jewel), both upstream and
  Mirantis' version

The following upgrades are known to *NOT* work:

* From 0.94.6 (Hammer) to upstream 0.94.7 and newer Hammer point releases
* From 0.94.7 and newer Hammer point releases to upstream Jewel 10.2.5 and
  earlier Jewel point releases

The above lists are (very) *incomplete*. Please use a test cluster to find
out if the upgrade causes any problems.


Preparations
------------

Please note that the upgrade procedure does not disrupt clients (such as VMs
using rbd images).

* Install ansible 2.2.x on the master node

  - Enable EPEL repository::

      wget -N https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
      yum install epel-release-latest-7.noarch.rpm

  - Install ``ansible`` package::

      yum install ansible

* Clone this repository (on the master node)::

    git clone https://github.com/asheplyakov/mos9-ceph-upgrade.git

* Prepare inventory file ``hosts`` which lists all monitor, OSD, and client
  nodes using the ``scripts/make_fuel_inventory.sh`` script.
  Use the ``hosts.sample`` example to check if the generated inventory file
  makes sense. Note: inventory *must* use hostnames (either short or fully
  qualified), listing just an IP address of a node won't work (and can break
  the cluster).


Preflight checks
----------------

* Check if ansible is able to communicate to the cluster::

    ansible -i hosts all -m ping

* Check if the cluster is OK, ``ceph -s`` should report ``HEALTH_OK``,
  all PGs should be active+clean, if not, you should fix any problems
  before upgrading.

* Check if `apt-get`` can access Ubuntu and MOS APT repositories
  (either directly or via a proxy) on monitor, OSD, and client nodes.

* (For QA only) In order to generate a test load (so bugs in ceph and upgrade
  scenario can be exposed) run on the master node::

    ansible-playbook -i hosts make_test_load.yml


Upgrade ceph cluster
----------------------

* On the master node run::

    ansible-playbook -i hosts site.yml


Restart VMs
-----------

Although the upgrading servers (monitors and OSDs) according to the above
instructions does not disrupt clients, it's necessary to restart qemu
processes serving VMs (and other services which use ceph) In order to apply
client libraries (``librados2.so`` and ``librbd1.so``) fixes. Note that
the VMs (and daemons) which haven't been restarted will keep using the old
(buggy) version of rbd client library (``librbd1.so``) and might hit
the `rbd cache data corruption`_ bug.

.. _rbd cache data corruption: http://tracker.ceph.com/issues/17545
.. _Fuel: https://wiki.openstack.org/wiki/Fuel
.. _ceph-ansible: https://github.com/ceph/ceph-ansible
.. _bug #17386: http://tracker.ceph.com/issues/17386
.. _bug #18582: http://tracker.ceph.com/issues/18582
