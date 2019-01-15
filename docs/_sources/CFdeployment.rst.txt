.. _cf-deploy:

5. Deploying CloudFoundry with BOSH Director on OpenStack
============================================================


.. todo:: Draft: to be deleted after the documentation is ready:

  
  1. Configure OpenStack Domain,Project, User, Network for the deployment - Done in Section :ref:`"4. Configure OpenStack"<cf-config>`
  
  2. Validate the OpenStack configuration using:
  https://github.com/eniware-org/cf-openstack-validator : 
  
  * cf-openstack-validator installation done in section :ref:`5.2. CF-OpenStack-Validator installation<cf-deploy-cfval-install>`
  
  3. Setup the OpenStack projects for the BOSH and CloudFoundry installation using TerraForm modules from here:
  https://github.com/eniware-org/bosh-openstack-environment-templates
  
  4. Install BOSH:
  https://bosh.io/docs/init-openstack/#deploy
  
  5. Prepare and upload ``cloud-config.yml`` to BOSH to finilize the cloud configuration.
  
  6. Deploy CloudFoundry.


In :ref:`previous section<cf-config>` we've configured OpenStack for use within a production environment. Various types of :ref:`clients<cf-clients-install>` were installed for different OpenStack operations. We have set :ref:`environment variables<cf-env-conf>`, :ref:`external network<cf-net-conf>`, :ref:`flavors<cf-domain-flavors>`, :ref:`domain, project and user<cf-domain-conf>`.
	
In this section we'll create an environment that consists of a BOSH Director and Cloudfoudry deployment that it orchestrates.
	
	


.. _cf-deploy-prereq:

5.1. Prerequisites
--------------------

To be able to proceed, you need to the following:

* :ref:`Working OpenStack environment<install-openstack>` using both Juju and MAAS.
* A :ref:`user<cf-domain-conf>` able to create/delete resource in this environment.
* :ref:`Flavors<cf-domain-flavors>` with properly configured names and settings.




.. _cf-deploy-cfval-install:

5.2. CF-OpenStack-Validator installation
------------------------------------------

`CF OpenStack Validator is an extension <https://github.com/eniware-org/cf-openstack-validator/tree/master/extensions/object_storage>`_  that verifies whether the OpenStack installation is ready to run BOSH and install Cloud Foundry. The intended place to run the validator is a VM within your OpenStack.


.. _cf-deploy-cfval-pre:

5.2.1. Prerequisites for CF-OpenStack-Validator
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

OpenStack configuration requirements are as follows:

* :ref:`Keystone<openstack-deploy>` v.2/v.3 Juju charm installed.
* Created :ref:`OpenStack project<cf-domain-conf>`.
* Created user with access to the previously created project (ideally you don't want to run as admin).
* Created network - connect the network with a router to your external network.
* Allocated a floating IP.
* Allowed ssh access in the *default* security group - create a key pair by executing:

 .. code::
  
     $ ssh-keygen -t rsa -b 4096 -N "" -f cf-validator.rsa_id

 Upload the generated public key to OpenStack as **cf-validator**.

* A public :ref:`image<cf-cloud-conf>` available in **Glance**.

The validator runs on Linux. Please ensure that the following packages are installed on your system:

* `ruby <https://packages.ubuntu.com/bionic/ruby>`_ 2.4.x or newer
* `make <https://packages.ubuntu.com/bionic/make>`_
* `gcc <https://packages.ubuntu.com/bionic/gcc>`_
* `zlib1g-dev <https://packages.ubuntu.com/bionic/zlib1g-dev>`_
* `libssl-dev <https://packages.ubuntu.com/bionic/libssl-dev>`_
* `ssh <https://packages.ubuntu.com/bionic/ssh>`_


.. _cf-deploy-cfval-command:

5.2.2. Installation of CF-OpenStack-Validator
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To clone the CF-OpenStack-Validator repository:

.. code:: 

   git clone https://github.com/eniware-org/cf-openstack-validator

Navigate to the **cf-openstack-validator** folder:

.. code:: 

   cd cf-openstack-validator

Copy the :ref:`generated private key<cf-deploy-cfval-pre>` into the **cf-openstack-validator** folder.

Copy ``validator.template.yml`` to ``validator.yml`` and replace occurrences of **<replace-me>** with appropriate values (see :ref:`prerequisites<cf-deploy-cfval-pre>`):

 * If using Keystone v.3, ensure there are values for *domain* and *project*.
 * If using Keystone v.2, remove *domain* and *project*, and ensure there is a value for *tenant*. Also use the Keystone v.2 URL as *auth_url*.

.. code:: 

 $ cp validator.template.yml validator.yml


Download a **stemcell** from `OpenStack stemcells bosh.io <https://bosh.io/stemcells/bosh-openstack-kvm-ubuntu-trusty-go_agent>`_:

.. code:: 

 $ wget --content-disposition https://bosh.io/d/stemcells/bosh-openstack-kvm-ubuntu-trusty-go_agent

Install the following dependencies:

.. code:: 
  
 $ gem install bundler
 $ bundle install


To start the validation process type the following command:

.. code::
 
 $ ./validate --stemcell bosh-stemcell-<xxx>-openstack-kvm-ubuntu-trusty-go_agent.tgz --config validator.yml


.. _cf-deploy-cfval-conf:

5.2.3. Additional configurations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

* **CPI:**
 
 Validator downloads **CPI** release from the URL specified in the validator configuration. You can override this by specifying the ``--cpi-release`` command line option with the path to a CPI release tarball.

 If you already have a CPI compiled, you can specify the path to the executable in the environment variable ``OPENSTACK_CPI_BIN``. This is used when no CPI release is specified on the command line. It overrides the setting in the validator configuration file.

* **Extensions:**
 
 You can extend the validator with custom tests. For a detailed description and examples, please have a look at the `extension documentation <https://github.com/eniware-org/cf-openstack-validator/blob/master/docs/extensions.md>`_.

 The `eniware-org repository <https://github.com/eniware-org/cf-openstack-validator/tree/master/extensions>`_ already contains some extensions. Each extension has its own documentation which can be found in the corresponding extension folder.

To learn about available cf-validator options run the following command: 

.. code:: 
 
 $ ./validate --help


You can find more additional OpenStack related configuration options for possible solutions `here <https://github.com/eniware-org/cf-openstack-validator/blob/master/docs/openstack_configurations.md>`_.




.. _cf-deploy-cfval:

5.3. Validate the OpenStack configuration
-------------------------------------------

Before deploying Cloud Foundry, make sure to successfully run the `CF-OpenStack-Validator <https://github.com/eniware-org/cf-openstack-validator>`_ against your project:

* Make sure you have the required flavors on OpenStack by enabling the `flavors extension <https://github.com/eniware-org/cf-openstack-validator/tree/master/extensions/flavors>`_ with the `flavors.yml <https://github.com/eniware-org/cf-deployment/blob/master/iaas-support/openstack/flavors.yml>`_ file in this directory. Flavor names need to match those specified in the cloud config.
* If you plan using the `Swift ops file <https://github.com/eniware-org/cf-deployment/blob/master/operations/use-swift-blobstore.yml>`_ to enable Swift as blobstore for the Cloud Controller, you should also run the `Swift extension <https://github.com/eniware-org/cf-openstack-validator/tree/master/extensions/object_storage>`_.




.. _cf-deploy-terraform:

5.4. Prepare OpenStack resources for BOSH and Cloud Foundry via Terraform 
--------------------------------------------------------------------------------

5.4.1. BOSH
^^^^^^^^^^^^^

To setup an OpenStack project to install BOSH please use the following `Terraform module <https://github.com/cloudfoundry-incubator/bosh-openstack-environment-templates/tree/master/bosh-init-tf>`_. Adapt ``terraform.tfvars.template`` to your needs.


5.4.2. Cloud Foundry
^^^^^^^^^^^^^^^^^^^^^^

To setup the project to install Cloud Foundry please use the following `Terraform module <https://github.com/cloudfoundry-incubator/bosh-openstack-environment-templates/tree/master/cf-deployment-tf>`_. Adapt ``terraform.tfvars.template`` to your needs. Variable ``bosh_router_id`` is output of the previous BOSH terraform module.



.. _cf-deploy-bosh:

5.5. Install BOSH
-------------------

To install the BOSH director please follow the instructions on section :ref:`6. Isntall BOSH<install-bosh>` of this documentation.

For additional information you can visit `bosh.io <https://bosh.io/docs/init-openstack/#deploy>`_.

Make sure the BOSH director is accessible through the BOSH cli, by following the instructions on `bosh.io <https://bosh.io/docs/cli-envs.html>`_. Use this mechanism in all BOSH cli examples in this documentation.



.. _cf-deploy-cloudconf:

5.6. Cloud Config
--------------------

After the BOSH director has been installed, you can prepare and upload a cloud config based on the `cloud-config.yml <https://github.com/eniware-org/cf-deployment/blob/master/iaas-support/openstack/cloud-config.yml>`_ file.

Take the variables and outputs from the Terraform run of ``cf-deployment-tf`` to finalize the cloud config.

Use the following command to upload the cloud config.


.. code:: 
  
  bosh update-cloud-config \
       -v availability_zone1="<az-1>" \
       -v availability_zone2="<az-2>" \
       -v availability_zone3="<az-3>" \
       -v network_id1="<cf-network-id-1>" \
       -v network_id2="<cf-network-id-2>" \
       -v network_id3="<cf-network-id-3>" \
       cf-deployment/iaas-support/openstack/cloud-config.yml



   
.. _cf-deploy-cf:

5.7. Deploy Cloud Foundry
-----------------------------

To deploy Cloud Foundry run the following command filling in the necessary variables. system_domain is the user facing domain name of your Cloud Foundry installation.


.. code:: 

  bosh -d cf deploy cf-deployment/cf-deployment.yml \
       -o cf-deployment/operations/use-compiled-releases.yml \
       -o cf-deployment/operations/openstack.yml \
       -v system_domain="<system-domain>"

**With Swift as Blobstore**

* Create four containers in Swift, which are used to store the artifacts for buildpacks, app-packages, droplets, and additional resources, respectively. The container names need to be passed in as variables in the below command snippet
* Set a `Temporary URL Key <https://docs.openstack.org/swift/latest/api/temporary_url_middleware.html#secret-keys>`_ for your Swift account

Add the following lines to the deploy cmd:

.. code:: 

  -o cf-deployment/operations/use-swift-blobstore.yml \
  -v auth_url="<auth-url>" \
  -v openstack_project="<project-name>" \
  -v openstack_domain="<domain>" \
  -v openstack_username="<user>" \
  -v openstack_password="<password>" \
  -v openstack_temp_url_key="<temp-url-key>" \
  -v app_package_directory_key="<app-package-directory-key>" \
  -v buildpack_directory_key="<buildpack-directory-key>" \
  -v droplet_directory_key="<droplet-directory-key>" \
  -v resource_directory_key="<resource-directory-key>" \


