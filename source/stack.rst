Software Stack
==============

.. code::
   
   +--------------------------+
   |                          |
   |    User's Applications   |
   |                          |
   +--------------------------+
   |                          |
   |    Ansible               |
   |                          |
   +--------------------------+
   |                          |
   |    Terraform             |
   |                          |
   +--------------------------+
   |                          |
   |    Cloud Infrastructure  |
   |                          |
   +--------------------------+

`Terraform <https://www.terraform.io/>`_ is used as a layer to abstract out the
cloud infrastructure which could in principle be any public cloud or an on
premise solution. Currently only the Oracle Cloud Infrastructure is supported.
Implementing support for another cloud infrastructure would entail rewriting
some of the files at the terrafrom layer.

The `Ansible <https://www.ansible.com/>`_ layer then depends on resources built by terraform:

- A network that can reach the interenet with security rules that allow servers
  created in the network to talk to one another 
- A VM called ``mgmt``
- An NFS server called ``fileserver`` exporting a share called ``/shared``

It configures the system to be able to run `Slurm
<https://slurm.schedmd.com/documentation.html>`_ and using the `Cloud
<https://slurm.schedmd.com/elastic_computing.html>`_ node type spin up and down
nodes on demand.

