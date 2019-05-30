Cluster Lifecycle
=================

Prerequisites
-------------

The end user has a tenancy, configured user, sufficient service limits.

Create Management Node, Network, Filesystem
-------------------------------------------

- The end user clones the `terrafrom repo <https://github.com/ACRC/oci-cluster-terraform>`_
- They copy the provided ``terraform.tfvars.example`` file to
  ``terraform.tfvars`` and edit the credentials, the management shape and the
  availability domain for the management VM and NFS mount target.
- They run ``terrafrom validate``, ``terraform plan``,  ``terraform apply``
- Terraform runs it's DAG creating the resources specified and writes out the
  file ``terrafrom.tfstate``
- Terraform prints out the IPv4 address of the management VM
- On the ``mgmt`` VM the OCI
  `User Data <https://docs.cloud.oracle.com/iaas/Content/Compute/References/images.htm?Highlight=init%20userdata>`_ 
  facility copies the `cloud-init <https://cloudinit.readthedocs.io/en/latest/>`_ 
  file ``userdata/bootstrap`` on and it is run
- ``bootstrap`` creates an ansible inventory with the FQDN of the ``mgmt`` node in the group ``management``
- ``bootstrap`` uses ``ansible-pull`` to get the playbook from the GitHub repo.
  Unless the user has overridden it by setting ``ansible_branch`` in
  ``terraform.tfvars`` then it will pull the branch default branch, (currently
  ``3``)
- The ``management.yml`` ansible playbook is run
- The playbook is split in to sections and ordered so that the user is able to
  log in the management node as soon as possible, even while some of the tasks
  are still running
- Various roles are applied to install amongst other things

  - the 389ds directory server to manager user accounts
  - sssd
  - mysql which is needed for the slurm accounting database
  - slurm 

- Older versions of the ansible playbook pulled the slurm package from Matt's `Fedora copr <https://copr.fedorainfracloud.org/coprs/milliams/citc/>`_ repository
- Newer versions use the `OpenSuSE Build Service <https://build.opensuse.org/project/show/home:Milliams:citc>`_

Configure
---------

- The end user waits until the ansible playbook has finished, seen in ``/root/ansible-pull.log``
- They create a yaml file ``~opc/limits.yaml``
- They run the script ``~opc/finish`` which parses the ``limits.yaml`` file and
  writes the file ``/mnt/shared/etc/slurm/slurm.conf`` on the NFS share which is
  included by ``/etc/slurm/slurm.conf`` and restarts ``slurmctld``.
- They may optionally add users

Create Compute Nodes
--------------------

- When a user submits a compute job slurm checks 
- If a node or nodes need to be stareted Slurm calls ``/usr/local/bin/startnode`` which
  is a python program using the Oracle python API for OCI to start the instance
  using the node name(s) supplied by slurm as the first argument
- ``/usr/local/bin/startnode`` uses a python
  `asyncio Event Loop <https://docs.python.org/3/library/asyncio-eventloop.html>`_ to start the nodes
- ``/usr/local/bin/startnode`` uses a custom `retry strategy
  <https://oracle-cloud-infrastructure-python-sdk.readthedocs.io/en/latest/sdk_behaviors/retries.html>`_
  to catch and recover from API request throtilling "429" return codes.
- The region, compartment, AD and VCN infromation is read from
  ``/etc/citc/startnode.yaml`` on the managemnt node that was populated by ther
  ansible playbook
- Once the instance has started ``citc_oci.py`` waits for the VNIC to become
  available and then informs slurm using ``scontrol`` of the newly started node's
  IPv4 address
- As with the management node the create instance call passes the userdata for cloud-init.
- The compute node user data comes from the file ``/home/slurm/bootstrap.sh`` on the management node
- The compute node ``bootstrap.sh``

  - adds some host SSH keys so that slurm can work
  - installs git
  - creates an ansible inventory with the FQDN of the compute node in the ``compute`` group
  - uses ``ansible-pull`` to grab the ansible repo and run the ``compute.yml`` playbook

- The ``compute.yml`` playbook configures the node as a slurm compute node.
- Once ``slurmd`` has been restarted it checks in with the slurm master (the
  ``mgmt`` node) and jobs get sent to it
 
Node Power Saving
-----------------

- If no job has run on a node for the duration of ``SuspendTime`` seconds in
  ``/mnt/shared/etc/slurm/slurm.conf``, default is 900 seconds, i.e. 15 minutes,
  then slurm calls ``/usr/local/bin/stopnode`` which terminates the instance.

Destroy Cluster
---------------

- Once the end user has finished with their computing they can ``terrafrom
  destory`` from where they checked out the terraform repo.
- in ``compute.tf`` the terraform provisioner ``remote-exec`` will SSH on to
  the management node and call ``/usr/local/bin/stopnode`` on any
  still running compute nodes
- terraform will walk the DAG destroying resources in reverse order.

