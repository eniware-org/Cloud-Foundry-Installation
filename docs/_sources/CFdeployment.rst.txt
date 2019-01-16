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
* Created *user* with access to the previously created *project* (ideally you don't want to run as admin).
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

* Make sure you have the :ref:`required flavors<flavours-required>` on OpenStack by enabling the `flavors extension <https://github.com/eniware-org/cf-openstack-validator/tree/master/extensions/flavors>`_ with the `flavors.yml <https://github.com/eniware-org/cf-deployment/blob/master/iaas-support/openstack/flavors.yml>`_ file in this directory. Flavor names need to match those specified in the cloud config.
* If you plan using the `Swift ops file <https://github.com/eniware-org/cf-deployment/blob/master/operations/use-swift-blobstore.yml>`_ to enable Swift as blobstore for the Cloud Controller, you should also run the `Swift extension <https://github.com/eniware-org/cf-openstack-validator/tree/master/extensions/object_storage>`_.


To **start the validation process** type the following command:

.. code::
 
 $ ./validate --stemcell bosh-stemcell-<xxx>-openstack-kvm-ubuntu-trusty-go_agent.tgz --config validator.yml



.. _cf-deploy-terraform:

5.4. Prepare OpenStack environment for BOSH and Cloud Foundry via Terraform 
--------------------------------------------------------------------------------


You can use a **Terraform environment template** to configure your OpenStack project automatically. You will need to create a ``terraform.tfvars`` file with information about the environment. 

.. important:: The terraform scripts will output the OpenStack resource information required for the BOSH manifest. Make sure to treat the created ``terraform.tfstate`` files with care.

.. hint:: Instead of using Terraform, you can prepare an OpenStack environment **manually** as described `here <https://bosh.io/docs/init-openstack/#configuration-of-a-new-openstack-project>`_.


.. _cf-deploy-terraform-inst:

5.4.1. Install Terraform module
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Make sure you have updated package database and installed unzip package:

.. code:: 

 sudo apt-get update       
 sudo apt-get install -y git unzip

To install the **Terraform** module:

.. code:: 
 
  git clone https://github.com/eniware-org/bosh-openstack-environment-templates.git
  wget https://releases.hashicorp.com/terraform/0.11.11/terraform_0.11.11_linux_amd64.zip
  unzip terraform_0.11.11_linux_amd64.zip
  chmod +x terraform
  sudo mv terraform /usr/local/bin/ 



.. _cf-deploy-bosh-env:

5.4.2. OpenStack environment for BOSH
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _cf-deploy-bosh-env-set:

Setup an OpenStack project to install BOSH:
++++++++++++++++++++++++++++++++++++++++++++++

To setup an OpenStack project to install BOSH please use the following `Terraform module <https://github.com/eniware-org/bosh-openstack-environment-templates/tree/master/bosh-init-tf>`_. Adapt ``terraform.tfvars.template`` to your needs.

 1. Create a working folder:
   
 .. code:: 
  
   mkdir tmp

 2. Copy the :ref:`template file<terraform.tfvars-bosh>`:
 
 .. code::
 
   cp bosh-openstack-environment-templates/bosh-init-tf/terraform.tfvars.template tmp/terraform.tfvars
   
 3. Generate a key pair executing the following script:
 
 .. code::
 
   sh bosh-openstack-environment-templates/bosh-init-cf/generate_ssh_keypair.sh
   
 4. Move the generated key pair to your working folder ``tmp``:
 
 .. code::
 
   mv bosh.* tmp/
   
 5. Navigate to the working folder ``tmp``:
 
 .. code::
 
   cd tmp
   
 6. Configure the :ref:`Terraform environment template<terraform.tfvars-bosh>` ``terraform.tfvars``.  
 
 7. Run the following commands:
 
 .. code::
 
   terraform init ../bosh-openstack-environment-templates/bosh-init-tf/
   terraform apply ../bosh-openstack-environment-templates/bosh-init-tf/

 8. Save the ``terraform.tfvars`` and ``terraform.tfstate`` files for **bosh-init-tf**:
 
 .. code::
 
     mv terraform.tfvars bosh_terraform.tfvars
     mv terraform.tfstate bosh_terraform.tfstate


.. _terraform.tfvars-bosh:

Terraform tempalte file configuration for BOSH: 
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

The content of the `terraform tempalte file <https://github.com/eniware-org/bosh-openstack-environment-templates/blob/master/bosh-init-tf/terraform.tfvars.template>`_ ``terraform.tfvars`` for BOSH is as follows:

.. code::

	auth_url = "<auth_url>"
	domain_name = "<domain_name>"
	user_name = "<ostack_user>"
	password = "<ostack_pw>"
	tenant_name = "<ostack_tenant_name>"
	region_name = "<region_name>"
	availability_zone = "<availability_zone>"

	ext_net_name = "<ext_net_name>"
	ext_net_id = "<ext_net_id>"

	# in case your OpenStack needs custom nameservers
	# dns_nameservers = 8.8.8.8

	# Disambiguate keypairs with this suffix
	# keypair_suffix = "<keypair_suffix>"

	# Disambiguate security groups with this suffix
	# security_group_suffix = "<security_group_suffix>"

	# in case of self signed certificate select one of the following options
	# cacert_file = "<path-to-certificate>"
	# insecure = "true"

	
To edit the ``terraform.tfvars`` for BOSH using the described in this documentation scenario:

.. _terraform.tfvars-bosh-example:

.. code::

 nano terraform.tfvars

Enter the following settings:

.. code::

     auth_url="http://192.168.40.228:5000/v3"
     domain_name="cf_domain"
     user_name="eniware"
     password= <your_password>
     tenant_name="cloudfoundry"
     region_name="RegionOne"
     availability_zone="nova"
     ext_net_name="ext_net"
     ext_net_id="db178716-7d8a-444b-854a-685feb5bf7ea"   

* ``auth_url`` is the :ref:`URL of the Keystone service<openstack-test>`, which is ``http://192.168.40.228:5000/v3`` in our case (it can be retrieved by using ``juju status | grep keystone/0`` command).
* The :ref:`created<cf-domain-conf>` *domain* **cf_domain**, with *project* **cloudfondry** and *user* **eniware** are set in the template in the ``domain_name``, ``user_name``, ``password``, and ``tenant_name`` fields.
* ``region_name`` can be retrieved when editing the Neutron config file or from :ref:`here<neutron-region>`.
* The :ref:`defined external network<cf-net-conf>` is set in ``ext_net_name`` filed.
* The ``ext_name_id`` identificator can be retrieved from the OpenStack web UI (go to *Project > Network > Networks*, click on **ext_net** and go to **Overview** tab) or by using the :ref:`command<ext_net-id>` ``openstack network list``.





.. _cf-deploy-cf-env:

5.4.3. OpenStack environment for Cloud Foundry
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


.. _cf-deploy-cf-env-set:

Setup an OpenStack project to install Cloud Foundry:
+++++++++++++++++++++++++++++++++++++++++++++++++++++

To setup the project to install Cloud Foundry please use the following `Terraform module <https://github.com/eniware-org/bosh-openstack-environment-templates/tree/master/cf-deployment-tf>`_. Adapt ``terraform.tfvars.template`` to your needs. Variable ``bosh_router_id`` is output of the previous BOSH terraform module.

 1. Copy the :ref:`template file<terraform.tfvars-cf>` ``terraform.tfvars`` file for **cf-deployment-tf**:
 
 .. code:: 
 
      cp ../bosh-openstack-environment-templates/cf-deployment-tf/terraform.tfvars.template ./terraform.tfvars

 2. Configure the :ref:`Terraform environment template<terraform.tfvars-cf>` ``terraform.tfvars`` for **cf-deployment-tf**:
 
 3. Run the following commands:
 
 .. code::
 
    
    terraform init ../bosh-openstack-environment-templates/cf-deployment-tf/
    terraform apply ../bosh-openstack-environment-templates/cf-deployment-tf/




.. _terraform.tfvars-cf:

Terraform tempalte file configuration for Cloud Foundry: 
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

The content of the `terraform tempalte file <https://github.com/eniware-org/bosh-openstack-environment-templates/blob/master/cf-deployment-tf/terraform.tfvars.template>`_ ``terraform.tfvars`` for Cloud Foundry is as follows:

.. code::

	auth_url = "<auth-url>"
	domain_name = "<domain>"
	user_name = "<user>"
	password = "<password>"
	project_name = "<project-name>"
	region_name = "<region-name>"
	availability_zones = ["<az-1>","<az-2>","<az-3>"]
	ext_net_name = "<external-network-name>"

	# the OpenStack router id which can be used to access the BOSH network
	bosh_router_id = "<bosh-router-id>"

	# in case Openstack has its own DNS servers
	dns_nameservers = ["<dns-server-1>","<dns-server-2>"]

	# does BOSH use a local blobstore? Set to 'false', if your BOSH Director uses e.g. S3 to store its blobs
	use_local_blobstore = "<true or false>" #default is true

	# enable TCP routing setup
	use_tcp_router = "<true or false>" #default is true
	num_tcp_ports = <number> #default is 100, needs to be > 0

	# in case of self signed certificate select one of the following options
	# cacert_file = "<path-to-certificate>"
	# insecure = "true"



To edit the ``terraform.tfvars`` for Cloud Foundry using the described in this documentation scenario:

.. _terraform.tfvars-cf-example:

.. code::

 nano terraform.tfvars

Enter the following settings:

.. code::
 
    
    auth_url="http://192.168.40.228:5000/v3"
    domain_name="cf_domain"
    user_name="eniware"
    password= <your_password>
    tenant_name="cloudfoundry"
    region_name="RegionOne"
    availability_zones = ["nova", "nova", "nova"]
    bosh_router_id = ""
    dns_nameservers = ["8.8.8.8"]
    use_local_blobstore = "true"
    use_tcp_router = "true"
    num_tcp_ports = 100

* ``auth_url``, ``domain_name``, ``user_name``, ``password``, ``tenant_name``, and ``region_name`` are the same as in ``terraform.tfvars`` :ref:`template file for BOSH<terraform.tfvars-bosh-example>`.
* ``bosh_router_id`` can be retrieved from the output of the previous :ref:`terraform script<terraform.tfvars-bosh-example>` for BOSH.







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


