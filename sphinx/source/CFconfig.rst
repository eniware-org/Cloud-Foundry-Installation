.. _cf-config:

4. Configure OpenStack
========================

In previous sections we've deployed OpenStack using both Juju and MAAS. The next step is to configure OpenStack for use within a production environment.

The steps te be followed are:

* Setting up the environment variables
* Adding a project
* Virtual network access and Ubuntu cloud image deployment 


.. _cf-env-conf:

4.1. Environment variables
------------------------------

When accessing OpenStack from the command line, specific environment variables need to be set:

* S_AUTH_URL
* OS_USER_DOMAIN_NAME
* OS_USERNAME
* OS_PROJECT_DOMAIN_NAME
* OS_PROJECT_NAME
* OS_PROJECT_NAME
* OS_REGION_NAME
* OS_IDENTITY_API_VERSION
* OS_AUTH_VERSION

The OS_AUTH_URL is the address of the OpenStack Keystone node for authentication. To retrieve this IP addres by Juju use the following command:

.. code:: 

  juju status --format=yaml keystone/0 | grep public-address | awk '{print $2}'

The specific environment variables are included in a file called **openrc**. It is located in ``openstack_bundles/stable/openstack-base`` or you can `download <https://github.com/eniware-org/openstack-bundles/blob/master/stable/shared/openrcv3_project>`_ it. It can easily be sourced (made active).  


The **openrc** file contains the following:

.. code::

   _OS_PARAMS=$(env | awk 'BEGIN {FS="="} /^OS_/ {print $1;}' | paste -sd ' ')
   for param in $_OS_PARAMS; do
       if [ "$param" = "OS_AUTH_PROTOCOL" ]; then continue; fi
       if [ "$param" = "OS_CACERT" ]; then continue; fi
       unset $param
   done
   unset _OS_PARAMS
   
   _keystone_unit=$(juju status keystone --format yaml | \
       awk '/units:$/ {getline; gsub(/:$/, ""); print $1}')
   _keystone_ip=$(juju run --unit ${_keystone_unit} 'unit-get private-address')
   _password=$(juju run --unit ${_keystone_unit} 'leader-get admin_passwd')
   
   export OS_AUTH_URL=${OS_AUTH_PROTOCOL:-http}://${_keystone_ip}:5000/v3
   export OS_USERNAME=admin
   export OS_PASSWORD=${_password}
   export OS_USER_DOMAIN_NAME=admin_domain
   export OS_PROJECT_DOMAIN_NAME=admin_domain
   export OS_PROJECT_NAME=admin
   export OS_REGION_NAME=RegionOne
   export OS_IDENTITY_API_VERSION=3
   # Swift needs this:
   export OS_AUTH_VERSION=3
   # Gnocchi needs this
   export OS_AUTH_TYPE=password


The environment variables can be enabled/sourced with the following command:

.. code:: 

  source openrc

After the **openrc** is created, you can use OpenStack’s Horizon web UI todownload the file, which automatically adjusts the environment variables. You should be loged with the given username. Right click on the user name in the the upper right corner and select from the dropdown list **openrc -v3**

.. note:: If the **openrc** file is manually edited, it is important that all variables are correctly entered

You can check the variables have been set correctly by seeing if your OpenStack endpoints are visible with the ``openstack endpoint list`` command. The output will look something like this:


.. code::

	+----------------------------------+-----------+--------------+--------------+
	| ID                               | Region    | Service Name | Service Type |
	+----------------------------------+-----------+--------------+--------------+
	| 060d704e582b4f9cb432e9ecbf3f679e | RegionOne | cinderv2     | volumev2     |
	| 269fe0ad800741c8b229a0b305d3ee23 | RegionOne | neutron      | network      |
	| 3ee5114e04bb45d99f512216f15f9454 | RegionOne | swift        | object-store |
	| 68bc78eb83a94ac48e5b79893d0d8870 | RegionOne | nova         | compute      |
	| 59c83d8484d54b358f3e4f75a21dda01 | RegionOne | s3           | s3           |
	| bebd70c3f4e84d439aa05600b539095e | RegionOne | keystone     | identity     |
	| 1eb95d4141c6416c8e0d9d7a2eed534f | RegionOne | glance       | image        |
	| 8bd7f4472ced40b39a5b0ecce29df3a0 | RegionOne | cinder       | volume       |
	+----------------------------------+-----------+--------------+--------------+

If the endpoints aren’t visible, it’s likely your environment variables aren’t configured correctly.

.. hint:: As with both MAAS and Juju, most OpenStack operations can be accomplished using either the command line or a web UI.


.. _cf-net-conf: 

4.2. Define an external network
---------------------------------

First step is to define a network called **Pub_Net**. It will use a subnet within the :ref:`range of addresses<install-maas-dhcp1>` reserved in MAAS:

.. code::

 openstack network create Pub_Net --share --external

The output from this command will show the various fields and values for the chosen configuration option. To show the new network ID alongside its name type the command ``openstack network list``:


.. code::
  
	+--------------------------------------+---------+---------+
	| ID                                   | Name    | Subnets |
	+--------------------------------------+---------+---------+
	| fc171d22-d1b0-467d-b6fa-109dfb77787b | Pub_Net |         |
	+--------------------------------------+---------+---------+

The second step is to create a subnet for the network using the various addresses from our MAAS and Juju configuration (192.168.100.3 is the IP address of the MAAS server):

.. code::
  
	openstack subnet create Pub_Subnet --allocation-pool \
	start=192.168.100.150,end=192.168.100.199 --subnet-range 192.168.100.0/24 \
	--no-dhcp --gateway 192.168.100.1 --dns-nameserver 192.168.100.3 \
	--dns-nameserver 8.8.8.8 --network Pub_Net

The output from the previous command provides a comprehensive overview of the new subnet’s configuration:

.. code::

	+-------------------------+--------------------------------------+
	| Field                   | Value                                |
	+-------------------------+--------------------------------------+
	| allocation_pools        | 192.168.100.150-192.168.100.199      |
	| cidr                    | 192.168.100.0/24                     |
	| created_at              | 2017-04-21T13:43:48                  |
	| description             |                                      |
	| dns_nameservers         | 192.168.100.3, 8.8.8.8               |
	| enable_dhcp             | False                                |
	| gateway_ip              | 192.168.100.1                        |
	| host_routes             |                                      |
	| id                      | 563ecd06-bbc3-4c98-b93e              |
	| ip_version              | 4                                    |
	| ipv6_address_mode       | None                                 |
	| ipv6_ra_mode            | None                                 |
	| name                    | Pub_Subnet                           |
	| network_id              | fc171d22-d1b0-467d-b6fa-109dfb77787b |
	| project_id              | 4068710688184af997c1907137d67c76     |
	| revision_number         | None                                 |
	| segment_id              | None                                 |
	| service_types           | None                                 |
	| subnetpool_id           | None                                 |
	| updated_at              | 2017-04-21T13:43:48                  |
	| use_default_subnet_pool | None                                 |
	+-------------------------+--------------------------------------+

 
.. note:: OpenStack has `deprecated <https://docs.openstack.org/python-neutronclient/latest/>`_ the use of the **neutron** command for network configuration, migrating most of its functionality into the Python OpenStack client. Version 2.4.0 or later of this client is needed for the ``subnet create`` command.


.. _cf-cloud-conf:

4.3. Cloud images
--------------------

You need to download an **Ubuntu image** locally in order to be able to аdd it to a **Glance**. Canonical’s Ubuntu cloud images can be found here:

https://cloud-images.ubuntu.com

You could use ``wget`` to download the image of **Ubuntu 18.04 LTS (Bionic)**:

.. code:: 

  wget https://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img

To add this image to Glance use the following command:

.. code:: 
 
	openstack image create --public --min-disk 3 --container-format bare \
	--disk-format qcow2 --property architecture=x86_64 \
	--property hw_disk_bus=virtio --property hw_vif_model=virtio \
	--file bionic-server-cloudimg-amd64.img \
	"bionic x86_64"

Typing ``openstack image list`` you can make sure the image was successfully imported:

.. code::
 
	+--------------------------------------+---------------+--------+
	| ID                                   | Name          | Status |
	+--------------------------------------+---------------+--------+
	| d4244007-5864-4a2d-9cfd-f008ade72df4 | bionic x86_64 | active |
	+--------------------------------------+---------------+--------+

The **Compute > Images** page of **OpenStack’s Horizon web UI** lists many more details about imported images. In particular, note their size as this will limit the minimum root storage size of any OpenStack flavours used to deploy them.

.. figure:: /images/4-horizon_image.png
   :alt: Horizon image details
   :align: center



.. _cf-domain-conf:

4.4. Working with domains and projects
-------------------------------------------

The following is vital part of OpenStack operations:

* **Domains** - abstract resources; a domain is a collection of users and projects that exist within the OpenStack environment.
* **Projects** - organizational units in the cloud to which you can assign users (a project is a group of zero or more users).
* **users** - members of one or more projects. 
* **roles** - define which actions users can perform. You assign roles to user-project pairs.

To create a single domain with a single project and single user for a new deployment, start with the **domain**:

.. code:: 

  openstack domain create MyDomain

To add a **project** to the **domain**:

.. code::
 
  openstack project create --domain MyDomain \
      --description 'First Project' MyProject

To add a **user** and assign that user to the **project**:

.. code::

  openstack user create --domain MyDomain \
      --project-domain MyDomain --project MyProject \
      --password-prompt MyUser

The output to the previous command will be similar to the following:

.. code:: 

	+---------------------+----------------------------------+
	| Field               | Value                            |
	+---------------------+----------------------------------+
	| default_project_id  | 914e59223944433dbf12417ac4cd4031 |
	| domain_id           | 7993528e51344814be2fd53f1f8f82f9 |
	| enabled             | True                             |
	| id                  | e980be28b20b4a2190c41ae478942ab1 |
	| name                | MyUser                           |
	| options             | {}                               |
	| password_expires_at | None                             |
	+---------------------+----------------------------------+


.. hint:: You can create a file to hold the details on the new project and user:
  
  Create the following **myprojectrc** file:
  
  .. code:: 
  
     export OS_AUTH_URL=http://192.168.100.95:5000/v3
     export OS_USER_DOMAIN_NAME=MyDomain
     export OS_USERNAME=MyUser
     export OS_PROJECT_DOMAIN_NAME=MyDomain
     export OS_PROJECT_NAME=MyProject
  
  Source this file’s contents to effectively switch users:
  
  .. code:: 

    source myprojectrc

Every subsequent action will now be performed by **MyUser** user within the new **MyProject** project.


.. cf-vnet-conf

4.5. Create a virtual network
--------------------------------


You need a fixed IP address to access any instances you deploy from OpenStack. In order to assign a fixed IP, you need a project-specific network with a private subnet, and a router to link this network to the **Pub_Net** you created earlier.

To create the new network, enter the following:

.. code:: 
 
  openstack network create MyNetwork

Create a private subnet with the following parameters:

.. code:: 

	openstack subnet create MySubnet --allocation-pool \
	start=10.0.0.10,end=10.0.0.99 --subnet-range 10.0.0.0/24 \
	--gateway 10.0.0.1 --dns-nameserver 192.168.100.3 \
	--dns-nameserver 8.8.8.8 --network MyNetwork

You’ll see verbose output similar to the following:

.. code::

	+-------------------------+--------------------------------------+
	| Field                   | Value                                |
	+-------------------------+--------------------------------------+
	| allocation_pools        | 10.0.0.10-10.0.0.99                  |
	| cidr                    | 10.0.0.0/24                          |
	| created_at              | 2017-04-21T16:46:35                  |
	| description             |                                      |
	| dns_nameservers         | 192.168.100.3, 8.8.8.8               |
	| enable_dhcp             | True                                 |
	| gateway_ip              | 10.0.0.1                             |
	| host_routes             |                                      |
	| id                      | a91a604a-70d6-4688-915e-ed14c7db7ebd |
	| ip_version              | 4                                    |
	| ipv6_address_mode       | None                                 |
	| ipv6_ra_mode            | None                                 |
	| name                    | MySubnet                             |
	| network_id              | 8b0baa43-cb25-4a70-bf41-d4136cbfe16e |
	| project_id              | 1992e606b51b404c9151f8cb464aa420     |
	| revision_number         | None                                 |
	| segment_id              | None                                 |
	| service_types           | None                                 |
	| subnetpool_id           | None                                 |
	| updated_at              | 2017-04-21T16:46:35                  |
	| use_default_subnet_pool | None                                 |
	+-------------------------+--------------------------------------+

The following commands will add the router, connecting this new network to the **Pub_Net**:

.. code:: 

	openstack router create MyRouter
	openstack router set MyRouter --external-gateway Pub_Net
	openstack router add subnet MyRouter MySubnet

Use ``openstack router show MyRouter`` to verify all parameters have been set correctly.

Finally, you can add a floating IP address to your project’s new network:

.. code::

  openstack floating ip create Pub_Net

Details on the address will be shown in the output:

.. code::
  
	+---------------------+--------------------------------------+
	| Field               | Value                                |
	+---------------------+--------------------------------------+
	| created_at          | None                                 |
	| description         |                                      |
	| fixed_ip_address    | None                                 |
	| floating_ip_address | 192.168.100.152                      |
	| floating_network_id | fc171d22-d1b0-467d-b6fa-109dfb77787b |
	| id                  | f9b4193d-4385-4b25-83ed-89ed3358668e |
	| name                | 192.168.100.152                      |
	| port_id             | None                                 |
	| project_id          | 1992e606b51b404c9151f8cb464aa420     |
	| revision_number     | None                                 |
	| router_id           | None                                 |
	| status              | DOWN                                 |
	| updated_at          | None                                 |
	+---------------------+--------------------------------------+

This address will be added to the pool of available floating IP addresses that can be assigned to any new instances you deploy.




.. cf-ssh-conf

4.6. SSH access
--------------------

To create an OpenStack SSH keypair for accessing deployments with SSH, use the following command:

.. code::
  
  openstack keypair create NewKeypair > ~/.ssh/newkeypair.pem

With SSH, it’s imperative that the file has the correct permissions:

.. code:: 

  chmod 600 ~/.ssh/newkeypair.pem

Alternatively, you can import your pre-existing keypair with the following command:

.. code::

  openstack keypair create --public-key ~/.ssh/id_rsa.pub MyKeypair

You can view which keypairs have been added to OpenStack using the ``openstack keypair list`` command, which generates output similar to the following:

.. code::

	+-------------------+-------------------------------------------------+
	| Name              | Fingerprint                                     |
	+-------------------+-------------------------------------------------+
	| MyKeypair         | 1d:35:52:08:55:d5:54:04:a3:e0:23:f0:20:c4:b0:eb |
	| NewKeypair        | 1f:1a:74:a5:cb:87:e1:f3:2e:08:9e:40:dd:dd:7c:c4 |
	+-------------------+-------------------------------------------------+

To permit SSH traffic access to our deployments, we need to define a security group and a corresponding network rule:


.. code:: 

  openstack security group create --description 'Allow SSH' Allow_SSH

The following rule will open TCP port 22 and apply it to the above security group:

.. code::

  openstack security group rule create --proto tcp --dst-port 22 Allow_SSH


  .. cf-cloudinst-conf

4.7. Create a cloud instance
---------------------------------

Before launching our first cloud instance, you’ll need the network ID for the **MyNetwork**. This can be retrieved from the first column of output from the ``openstack network list`` command:

.. code::
  
	+--------------------------------------+-------------+------------------------+
	| ID                                   | Name        | Subnets                |
	+--------------------------------------+-------------+------------------------+
	| fc171d22-d1b0-467d-b6fa-109dfb77787b | Pub_Net     |563ecd06-bbc3-4c98-b93e |
	| 8b0baa43-cb25-4a70-bf41-d4136cbfe16e | MyNetwork   |a91a604a-70d6-4688-915e |
	+--------------------------------------+-------------+------------------------+

Use the **network ID** to replace the example in the following ``server create`` command to deploy a new instance:

.. code::
  
	openstack server create Ubuntu --availability-zone nova \
	--image 'bionic x86_64' --flavor m1.small \
	--key-name NewKeypair --security-group \
	Allow_SSH --nic net-id=8b0baa43-cb25-4a70-bf41-d4136cbfe16e

You can monitor progress with the ``openstack server list`` command by waiting for the server to show a status of **ACTIVE**:

.. code:: 
  
	+--------------------+-----------+--------+--------- ------------+---------------+
	| ID                 | Name      | Status | Networks             | Image Name    |
	+--------------------+-----------+--------+----------------------+---------------+
	| 4a61f2ad-5d89-43a6 | Ubuntu    | ACTIVE | MyNetwork=10.0.0.11  | bionic x86_64 |
	+--------------------+-----------+--------+----------------------+---------------+

All that’s left to do is assign a floating IP to the new server and connect with SSH.

Typing ``openstack floating ip list`` will show the floating IP address you liberated from **Pub_Net** earlier.

.. code:: 

	+----------+---------------------+------------------+------+--------------------+---------+
	| ID       | Floating IP Address | Fixed IP Address | Port | Floating Network   | Project |
	+----------+---------------------+------------------+------+--------------------+---------+
	| f9b4193d | 192.168.100.152     | None             | None | fc171d22-d1b0-467d | 1992e65 |
	+----------+---------------------+------------------+------+--------------------+---------+

The above output shows that the floating IP address is yet to be assigned. Use the following command to assign the IP address to your new instance:


.. code:: 

  openstack server add floating ip Ubuntu 192.168.100.152

You will now be able to connect to your new cloud server using SSH:

.. code:: 
 
  ssh -i ~/.ssh/newkeypair.pem 192.168.100.152


You have now built and successfully deployed a new cloud instance running on OpenStack, taking full advantage of both Juju and MAAS.
