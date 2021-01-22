Bare metal
==========
JupyterHub can be deployed in bare metal systems. This is the way is was done by the
Gravity Exploration Institute at Cardiff University to make a set of Python 
environments (3.6, 3.7 and 3.8) available to its users.

The main difference with a traditional JupyterHub `installation <https://jupyterhub.readthedocs.io/en/stable/installation-guide-hard.html>`_ 
is the use of conda to install a full environment rather than the recommended
combination of pip/conda.

The main steps are:

#. Install JupuyterHub dependencies:

   * Nodejs
   * npm
   * systemd
   * httpd
   * miniconda

#. Create JupyterHub group for sudospawner
#. Create JupyterHub user
#. Create a sudoers file
#. Create a configuration file
#. Create an anaconda virtual environment file for JupyterHub.
#. Create a sudospawner configuration file
#. Create JupyterHub's systemd service file
#. Create JupyterHub's httpd configuration file
#. Create JupyterHub's static kernels
#. Provision the kernels with the required environments
#. Create a suitable script to start JupyertHub
#. Obtionally create a suitable logo to display in JupyertHub
#. Start the JupyterHub service


There is an ansible script available to try to automatize this process.

Installing Anaconda
-------------------
The first step is making sure that conda>4.8.3 is available in the system. If not we 
do this download and install miniconda to provide it.

tasks/main.yml:

.. literalinclude:: scripts/ansible/playbook/roles/miniconda/tasks/main.yml
  :language: ruby

Where the different variables are defined in *defaults* and *vars*:

defaults/main.yml

.. literalinclude:: scripts/ansible/playbook/roles/miniconda/defaults/main.yml
  :language: ruby

vars/main.yml

.. literalinclude:: scripts/ansible/playbook/roles/miniconda/vars/main.yml
  :language: ruby
