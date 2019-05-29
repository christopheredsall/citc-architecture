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
- Terrafrom runs it's DAG. On the ``mgmt`` VM the OCI
  `User Data <https://docs.cloud.oracle.com/iaas/Content/Compute/References/images.htm?Highlight=init%20userdata>`_ 
  facility copies the `cloud-init <https://cloudinit.readthedocs.io/en/latest/>`_ 
  file ``userdata/bootstrap`` on and it is run
- ``bootstrap`` creates an ansible inventory with the FQDN of the ``mgmt`` node in the group ``management``
- ``bootstrap`` uses ``ansible-pull`` to get the playbook from the GitHub repo.
  Unless the user has overridden it by setting ``ansible_branch`` in
  ``terraform.tfvars`` then it will pull the branch default branch, (currently
  ``3``)

Configure
---------

- The end user waits until the ansible playbook has finished, seen in ``/root/ansible-pull.log``
- They create a yaml file ``~opc/limits.yaml``
- They run the script ``~opc/finish`` which parses the ``limits.yaml`` file and writes the 

Create Compute Nodes
--------------------

Destroy Cluster
---------------



