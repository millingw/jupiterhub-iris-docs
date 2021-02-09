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

miniconda/tasks/main.yml:

.. literalinclude:: scripts/ansible/playbook/roles/miniconda/tasks/main.yml
  :language: ruby

Where the different variables are defined in *defaults* and *vars*:

miniconda/defaults/main.yml

.. literalinclude:: scripts/ansible/playbook/roles/miniconda/defaults/main.yml
  :language: ruby

miniconda/vars/main.yml

.. literalinclude:: scripts/ansible/playbook/roles/miniconda/vars/main.yml
  :language: ruby

Installing other dependencies
-----------------------------
Depending on the Linux distribution used the way to installed the rest of 
dependencies may vary. This guide has been tested in Centos 7.

In this step we install (in case they are not already available):

* nodejs
* npm
* systemd
* httpd

In the case of nodejs, it might be necessary to first install Node repository. We
can do this by `downloading <https://github.com/nodesource/distributions>`_ the 
installers and running them locally. We can use the following ansible role to perform
the installation:

node/tasks/main.yml

.. literalinclude:: scripts/ansible/playbook/roles/node/tasks/main.yml
  :language: ruby


Authentication
--------------
For demonstration purposes we show how to configure via PAM (Ligo uses Shibboleth for authentication purposes). The authentication method to use can be defined in 
JupyterHub configuration file:

jupyterhub/templates/jupyterhub_config.py.j2

.. literalinclude:: scripts/ansible/playbook/roles/jupyterhub/templates/jupyterhub_config.py.j2

As mentioned, by commenting out the authenticator class jupyterhub falls back to 
using PAM authentication method.

Spawner
-------
Ligo uses a Custom Spawners for JupyterHub (SudoSpawner) to start each single-user 
notebook server. This spawner enables JupyterHub to run without being root, by 
spawning an intermediate process via sudo. This seems like a sensible choice to 
improve system security. In JupyterHub configuration file this is controlled with 
**sudospawner_path**. Besides this, SudoSpawner requires seting up the user that
will actually run the Hub and define which commands is it allowed to execute on 
behalf of users. This is done via a couple of configuration files:

A systemd configuration file for JupyterHub that defines the right user and location
where *jupyterhub* command should be invoked:

jupyterhub/templates/jupyterhub.service.j2

.. literalinclude:: scripts/ansible/playbook/roles/jupyterhub/templates/jupyterhub.service.j2

And a sudoers file that defines the command that JupyterHub's user is allowed to 
execute. Users are allowed to spawn a Jupyter Notebook if they are member of a 
particular group (*LIGO* in Ligo's case):

jupyterhub/templates/jupyterhub.sudofile.j2

.. literalinclude:: scripts/ansible/playbook/roles/jupyterhub/templates/jupyterhub.sudofile.j2


Defaults
--------
It is useful to define default values for some of the parameters used in our 
configuration files. Being stored in a separate file might faciliate to adapt these
templates for different cases.

jupyterhub/defaults/main.yml

.. literalinclude:: scripts/ansible/playbook/roles/jupyterhub/defaults/main.yml

Main tasks
----------
This script runs the main tasks required to deploy our JupyterHub service.

jupyterhub/defaults/main.yml

.. literalinclude:: scripts/ansible/playbook/roles/jupyterhub/tasks/main.yml



Running deployment
------------------
The files above can be run with an ansible playbook

jhub.yml

.. literalinclude:: scripts/ansible/playbook/jhub.yml
