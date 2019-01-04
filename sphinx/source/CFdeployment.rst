.. _cf-deply:

5. Deploying CloudFoundry with BOSH Director on OpenStack
============================================================


.. todo:: Draft: to be deleted after the documentation is ready:

	Deployment of CloudFoundry using the folllowing guide:

	https://github.com/eniware-org/cf-deployment/tree/master/iaas-support/openstack

	General steps:

	1.Configure OpenStack Domain,Project, User, Network for the deployment

	2.Validate the OpenStack configuration using:

	https://github.com/eniware-org/cf-openstack-validator

	3.Setup the OpenStack projects for the BOSH and CloudFoundry installation using TerraForm modules from here:

	https://github.com/eniware-org/bosh-openstack-environment-templates

	4. Install BOSH

	https://bosh.io/docs/init-openstack/#deploy

	5. Prepare and upload cloud-config.yml to BOSH to finilize the cloud configuration

	6. Deploy CloudFoundry



5.1. Prerequisites
--------------------

To be able to proceed, you need to the following:

* :ref:`Working OpenStack environment<install-openstack>` using both Juju and MAAS.
* A user able to create/delete resource in this environment
* `Flavors <https://docs.openstack.org/nova/rocky/user/flavors.html>`_ (an available hardware configuration for a server) with the following names and configuration:

.. list-table::
    :widths: 20 5 7 7 7
    :header-rows: 0
    :stub-columns: 0

    * - **Name**
      - **CPUs**
      - **RAM (MiB)** 
      - **Root Disk (GiB)**
      - **Ephemeral Disk (GiB)**
    * - minimal
      - 1
      - 3840
      - 3
      - 10
    * - small
      - 2
      - 7680
      - 3 
      - 14
    * - small-50GB-ephemeral-disk 
      - 2 
      - 7680 
      - 3 
      - 50
    * - small-highmem 
      - 4 
      - 31232 
      - 3 
      - 10
    * - small-highmem-100GB-ephemeral-disk 
      - 4 
      - 31232 
      - 3 
      - 100
    * - m1.xlarge
      - 8
      - 16384
      - 160
      - 0



5.2. Validate the OpenStack configuration
-------------------------------------------

Before deploying Cloud Foundry, make sure to successfully run the `CF-OpenStack-Validator <https://github.com/cloudfoundry-incubator/cf-openstack-validator>`_ against your project:

* Make sure you have the required flavors on OpenStack by enabling the `flavors extension <https://github.com/cloudfoundry-incubator/cf-openstack-validator/tree/master/extensions/flavors>`_ with the `flavors.yml <https://github.com/eniware-org/cf-deployment/blob/master/iaas-support/openstack/flavors.yml>`_ file in this directory. Flavor names need to match those specified in the cloud config.
* If you plan using the `Swift ops file <https://github.com/eniware-org/cf-deployment/blob/master/operations/use-swift-blobstore.yml>`_ to enable Swift as blobstore for the Cloud Controller, you should also run the `Swift extension <https://github.com/cloudfoundry-incubator/cf-openstack-validator/tree/master/extensions/object_storage>`_.



5.3. Prepare OpenStack resources for BOSH and Cloud Foundry via Terraform 
--------------------------------------------------------------------------------

5.3.1. BOSH
^^^^^^^^^^^^^

To setup an OpenStack project to install BOSH please use the following `Terraform module <https://github.com/cloudfoundry-incubator/bosh-openstack-environment-templates/tree/master/bosh-init-tf>`_. Adapt ``terraform.tfvars.template`` to your needs.


4.3.2. Cloud Foundry
^^^^^^^^^^^^^^^^^^^^^^

To setup the project to install Cloud Foundry please use the following `Terraform module <https://github.com/cloudfoundry-incubator/bosh-openstack-environment-templates/tree/master/cf-deployment-tf>`_. Adapt ``terraform.tfvars.template`` to your needs. Variable ``bosh_router_id`` is output of the previous BOSH terraform module.


5.4. Install BOSH
-------------------

To install the BOSH director please follow the instructions on section :ref:`6. Isntall BOSH<install-bosh>` of this documentation.

For additional information you can visit `bosh.io <https://bosh.io/docs/init-openstack/#deploy>`_.

Make sure the BOSH director is accessible through the BOSH cli, by following the instructions on `bosh.io <https://bosh.io/docs/cli-envs.html>`_. Use this mechanism in all BOSH cli examples in this documentation.


5.5. Cloud Config
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



5.6. Deploy Cloud Foundry
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


