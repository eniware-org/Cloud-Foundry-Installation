.. _install-bosh:

6. Install BOSH
================

.. figure:: /images/6-bosh-logo-full.png
   :alt: BOSH logo

`BOSH <https://bosh.io>`_ is a project that unifies release engineering, deployment, and lifecycle management of small and large-scale cloud software. BOSH can provision and deploy software over hundreds of VMs. It also performs monitoring, failure recovery, and software updates with zero-to-minimal downtime.

While BOSH was developed to deploy Cloud Foundry PaaS, it can also be used to deploy almost any other software (Hadoop, for instance). BOSH is particularly well-suited for large distributed systems. In addition, BOSH supports multiple Infrastructure as a Service (IaaS) providers like VMware vSphere, Google Cloud Platform, Amazon Web Services EC2, Microsoft Azure, and OpenStack. There is a Cloud Provider Interface (CPI) that enables users to extend BOSH to support additional IaaS providers such as Apache CloudStack and VirtualBox.


.. _install-bosh-get-started:

6.1. Getting Started
----------------------
The bosh `CLI <https://bosh.io/docs/cli-v2/>`_ is the command line tool used for interacting with all things BOSH. Release binaries are available on `GitHub <https://github.com/cloudfoundry/bosh-cli/releases>`_. See `Installation <https://bosh.io/docs/cli-v2-install/>`_ for more details on how to download and install.



.. _install-bosh-cli:

6.2. Installing the BOSH CLI
------------------------------

Choose your preferred installation method below to get the latest version of bosh.

Using the binary directly
^^^^^^^^^^^^^^^^^^^^^^^^^^

To install the BOSH binary directly:

1) Navigate to the `BOSH CLI GitHub release page <https://github.com/cloudfoundry/bosh-cli/releases>`_ and choose the correct download for your operating system.
2) Make the bosh binary executable and move the binary to your **PATH**:

   .. code::
   
      $ chmod +x ./bosh
      $ sudo mv ./bosh /usr/local/bin/bosh

3) You should now be able to use bosh. Verify by querying the CLI for its version:

   .. code::
    
      $ bosh -v
      version 5.3.1-8366c6fd-2018-09-25T18:25:51Z
      
      Succeeded

Using Homebrew on macOS
^^^^^^^^^^^^^^^^^^^^^^^^

If you are on macOS with `Homebrew <https://brew.sh>`_, you can install using the `Cloud Foundry tap <https://github.com/cloudfoundry/homebrew-tap>`_.

1) Use **brew** to install **bosh-cli**:

   .. code::
        
       $ brew install cloudfoundry/tap/bosh-cli

2) You should now be able to use bosh. Verify by querying the CLI for its version:

   .. code::
       
       $ bosh -v
       version 5.3.1-8366c6fd-2018-09-25T18:25:51Z
       
       Succeeded

   .. note:: We currently do not publish BOSH CLI via apt or yum repositories.

   
   
.. _install-bosh-add-dependences:

6.3. Additional Dependencies
------------------------------   

When you are using bosh to bootstrap BOSH or other standalone VMs, you will need a few extra dependencies installed on your local system.

.. note:: If you will not be using create-env and delete-env commands, you can skip this section.


Ubuntu Trusty
^^^^^^^^^^^^^^
If you are running on Ubuntu Trusty, ensure the following packages are installed on your system:

.. code::
   
   $ sudo apt-get install -y build-essential zlibc zlib1g-dev ruby ruby-dev openssl libxslt-dev libxml2-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3
   
   wget http://archive.ubuntu.com/ubuntu/pool/main/r/readline6/libreadline6_6.3-8ubuntu2_amd64.deb
   wget http://archive.ubuntu.com/ubuntu/pool/main/r/readline6/libreadline6-dev_6.3-8ubuntu2_amd64.deb
   sudo dpkg -i libreadline6_6.3-8ubuntu2_amd64.deb
   sudo dpkg -i libreadline6-dev_6.3-8ubuntu2_amd64.deb
   



.. _install-bosh-quick-start:

6.4. Quick Start
----------------

The easiest ways to get started with BOSH is by running on your local workstation with `VirtualBox <https://www.virtualbox.org>`_. If you are interested in bringing up a director in another environment, like `Google Cloud Platform <https://cloud.google.com/>`_, choose your IaaS from the navigation for more detailed instructions.


Prerequisites
^^^^^^^^^^^^^^^

Before trying to deploy the Director, make sure you have satisfied the following requirements:

1. For best performance, ensure you have at least 8GB RAM and 50GB of free disk space.
2. Install the bosh :ref:`CLI <install-bosh-cli>` and its :ref:`additional dependencies <install-bosh-add-dependences>`.
3. Install VirtualBox.


Install
^^^^^^^^
First, create a workspace for our virtualbox environment. This directory will keep some state and configuration files that we will need.

.. code::
   
   $ mkdir -p ~/bosh-env/virtualbox
   $ cd ~/bosh-env/virtualbox

Next, we'll use `bosh-deployment <https://github.com/cloudfoundry/bosh-deployment>`_, the recommended installation method, to bootstrap our director.

.. code::
   
   $ git clone https://github.com/cloudfoundry/bosh-deployment.git

Now, we can run the ``virtualbox/create-env.sh`` script to create our test director and configure the environment with some defaults.

.. code::
   
   $ ./bosh-deployment/virtualbox/create-env.sh

During the bootstrap process, you will see a few stages:

* Creating BOSH Director - dependencies are downloaded, the VM is created, and BOSH is installed, configured, and started.
* Adding Network Routes - a route to the virtual network is added to ensure you will be able to connect to BOSH-managed VMs.
* Generating ``.envrc`` - a settings file is generated so you can easily connect to the environment later.
* Configuring Environment Alias - an alias is added for the bosh command so you can reference the environment as vbox.
* Updating Cloud Config - default settings are applied to the Director so you easily deploy software later.

After a few moments, BOSH should be started. To verify, first load your connection settings, and then run your first bosh command where you should see similar output.

.. code::
   
   $ source .envrc
   $ bosh -e vbox env
   Using environment '192.168.50.6' as client 'admin'
   
   Name      bosh-lite
   UUID      7ce65259-471a-424b-88cb-9d3cee85db2c
   Version   265.2.0 (00000000)
   CPI       warden_cpi
   User      admin

Congratulations - BOSH is running! Now you're ready to deploy.

.. note:: *Troubleshooting*
   If you run into any trouble, please continue to the VirtualBox Troubleshooting section.

   
Deploy
^^^^^^^

Run through quick steps below or follow `deploy workflow <https://bosh.io/docs/basic-workflow/>`_ that goes through the same steps but with more explanation.

1. Update cloud config

  .. code::
    
    $ bosh -e vbox update-cloud-config bosh-deployment/warden/cloud-config.yml

2. Upload stemcell

  .. code::
    
    $ bosh -e vbox upload-stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent?v=3468.17 \ --sha1 1dad6d85d6e132810439daba7ca05694cec208ab

3. Deploy example deployment

  .. code::
    
    $ bosh -e vbox -d zookeeper deploy <(wget -O- https://raw.githubusercontent.com/cppforlife/zookeeper-release/master/manifests/zookeeper.yml)

4. Run Zookeeper smoke tests

  .. code::
    
      $ bosh -e vbox -d zookeeper run-errand smoke-tests


	  
Clean up
^^^^^^^^^

The test director can be deleted using the ``virtualbox/delete-env.sh`` script.

.. code::
   
   $ ./bosh-deployment/virtualbox/delete-env.sh


   
   
.. _install-bosh-init-openstack:  
   
6.5. Initialize New Environment on OpenStack
------------------------------------------------

`TODO <https://bosh.io/docs/init-openstack/>`_







