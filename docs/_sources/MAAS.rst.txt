
.. _install-maas:

1. Install MAAS
================

`MAAS <https://maas.io>`_ (Metal As A Service) - provides the management of physical servers like virtual machines in the cloud.
MAAS can work at any scale, from a test deployment using a handful of machines to thousands of machines deployed across multiple regions.

The typical MAAS environment includes as a framework for deployment the following:

* A Region controller interacts with and controls the wider environment for a region.
* One or more Rack controllers manage locally grouped hardware, usually within a data centre rack.
* Multiple Nodes are individual machines managed by the Rack controller, and ultimately, the Region controller.
* Complex Networking topologies can be modelled and implemented by MAAS, from a single fabric to multiple zones and many overlapping spaces.


.. note::
	See `Concepts and Terms <https://docs.maas.io/2.1/en/intro-concepts>`_ in the MAAS documentation for clarification on the terminology used within MAAS.

	
	
.. _maas-requirements:

1.1. Requirements
------------------

The minimum requirements for the machines that run MAAS vary widely depending on local implementation and usage. The minimum requirements for the machines that run MAAS are considered in the `MAAS documentation <https://docs.maas.io/2.2/en/#minimum-requirements>`_.

The hardware that will be used for the purpose of this documentation is based on the following specifications:

* 1 x MAAS Rack with Region controller: 8GB RAM, 2 CPUs, 1 NIC, 40GB storage
* 1 x Juju node: 4GB RAM, 2 CPUs, 1 NIC, 40GB storage
* 4 x OpenStack cloud nodes: 8GB RAM, 2 CPUs, 2 NICs, 80GB storage

Your hardware could differ considerably from the above and both MAAS and Juju will easily adapt. The Juju node could operate perfectly adequately with half the RAM (this would need to be defined as a bootstrap constraint) and adding more nodes will obviously improve performance.

.. note::
	It will be used the web UI whenever possible. However it can also be used `CLI <https://docs.ubuntu.com/maas/2.2/en/manage-cli>`_ and the `API <https://docs.ubuntu.com/maas/2.2/en/api>`_.

	
.. _maas-installation:

1.2. Installation
------------------

First, you need to have fresh install of `Ubuntu Server 16.04 TLS <http://releases.ubuntu.com/16.04/>`_ on the machine that will be hosting both the MAAS Rack and Region Controllers.

The configuration of the network is depends on your own infrastructure (see the `Ubuntu Server Network Configuration <https://help.ubuntu.com/lts/serverguide/network-configuration.html>`_ documentation for further details on modifying your network configuration). 

To update the package database and install MAAS, issue the following commands:

.. code::
	
	sudo apt update
	sudo apt install maas

Before MAAS can be configured an administrator account need to be created:

.. code::
	
	sudo maas createadmin

An ussername, password and email address should be filled in.
After that you need to specify if you want to import an SSH key. MAAS uses the public SSH key of a user to manage and secure access to deployed nodes. If you want to skip this, press ``Enter``. In the next step you can do this from the web UI.


.. _maas-onboarding:

1.3. On-boarding
-----------------

MAAS is now running without being configured. You can check this by pointing your browser to **http://<your.maas.ip>:5240/MAAS/**.
Now you sign in with the login credentials, and the web interface will launch the ‘Welcome to MAAS’ on-boarding page:

.. _install-maas-welcome:

.. figure:: /images/1-install-maas_welcome.png
   :alt: Welcome to MAAS

   
   
.. _maas-connectivity:   
   
1.4. Connectivity and images
-----------------------------
There are two steps left necessary for MAAS to get up and running. Unless you have specific requirements, most of these options can be left at their default values:

* Connectivity: important services that default to being outside of your network. These include package archives and the DNS forwarder.
* Ubuntu: for deployed nodes, MAAS needs to import the versions and image architectures. Specify 16.04 LTS as well as 14.04 LTS to add additional image.

.. _install-maas-images:

.. figure:: /images/1-install-maas_images.png
   :alt: Ubuntu images

  
  
.. _maas-ssh:
   
1.5. SSH key
------------

You have several options for importing your public SSH key(s). One is to import the key from Launchpad or Github by entering your user ID for these services. Another option is to add a local public key file, usually ``HOME/ssh/id.rsa.pub`` by selecting **Upload** button and placing the content in the box that appears. Click Import to complete the setting.

.. _install-maas-sshkeys:

.. figure:: /images/1-install-maas_sshkeys.png
   :alt: SSH key import



Adding SSH keys completes this initial MAAS configuration. Click Go to the dashboard to move to the MAAS dashboard and the device discovery process


You can generate a local SSH public/private key pair from the Linux account you are using for managing MAAS. When asked for a passphrase, leave it blank.

.. code::
	
	ssh-keygen –t rsa

This completes the initial setup of MAAS. Press **Go** button to the dashboard to go to the device discovery process.



.. _install-maas-networking:

1.6. Networking
-----------------
   
By default, MAAS will monitor local network traffic and report any devices it discovers on the **Device discovery** page of the web UI. This page is basic and is the first one to load after finishing installation.   

.. _install-maas-discovery:

.. figure:: /images/1-install-maas_discovery.png
   :alt: Device discovery

Before taking the configuration further, you need to tell MAAS about your network and how y’d like connections to be configured.

These options are managed from the **Subnets** page of the web UI. The subnets page defaults to listing connections by fabric and MAAS creates one fabric per physical NIC on the MAAS server. Once you are set up a machine with a single NIC, a single fabric will be be listed linked to the external subnet.

To the subnet that will going to manage the nodes, you need to add DHCP. This is done by selecting **untagged** VLAN the subnet to the right of **fabric-0**.

The page that appears will be labelled something similar to **Default VLAN in fabric-0**. From here, click the **Take action** button at the top right and select **Provide DHCP**. A new pane will appear that allows you to specify the start and end IP addresses for the DHCP range. Select **Provide DHCP** to accept the default values. The VLAN summary should now show DHCP as **Enabled**.

.. _install-maas-dhcp:

.. figure:: /images/1-install-maas_dhcp.png
   :alt: Provide DHCP

   
   
.. _install-maas-Ubuntu-images:   
   
1.7. Images
-------------

You have already downloaded the images you need as part of the on-boarding process, but it’s worth checking that both the images you requested are available. To do this, select the **Images** page from the top menu of the web UI.

The **Images** page allows you to download new images, use a custom source for images, and check on the status of any images currently downloaded. These appear at the bottom, and both 16.04 LTS and 14.04 LTS should be listed with a status of **Synced**.

.. _install-maas-imagestatus:

.. figure:: /images/1-install-maas_imagestatus.png
   :alt: Image status



.. _install-maas-nodes:
   
1.8. Adding nodes
-------------------

MAAS is now ready to accept new nodes. To do this, first ensure your four cloud nodes and single Juju node are set to boot from a PXE image. Now simply power them on. MAAS will add these new nodes automatically by taking the following steps:

* Detect each new node on the network
* Probe and log each node’s hardware (using an ephemeral boot image)
* Add each node to the **Nodes** page with a status of **New**

While it is not the most appropriate way, at this stage it is advisable to include each node individually in order to trace each one strictly.

In order to fully manage a deployment, MAAS needs to be able power cycle each node. This is why MAAS will attempt to power each node off during the discovery phase. If your hardware does not power off, it’s likely that it’s not using an IPMI based BMC and you will need to edit a node’s power configuration to enable MAAS to control its power. See the `MAAS documentation <https://docs.maas.io/2.2/en/nodes-power-types>`_ for more information on power types, including a `table <https://docs.maas.io/2.2/en/nodes-power-types#bmc-driver-support>`_ showing a feature comparison for the supported BMC drivers.

To edit a node’s power configuration, click on the arbitrary name your machine has been given in the **Nodes** page. This will open the configuration page for that specific machine. **Power** is the second section from the top.

Use the drop-down **Power type** menu to open the configuration options for your node’s specific power configuration and enter any further details that the configuration may require.
   
.. _install-maas-power:

.. figure:: /images/1-install-maas_power.png
   :alt: Power configuration  

When you make the necessary changes, click **Save changes**. The machine can now be turned off from the **Take option** menu in the top right.   
   


.. _install-maas-commission-nodes:

1.9. Commission nodes
-----------------------

From the **Nodes** page, select all the check boxes for all the machines in a **New** state and use the **Take action** menu to select **Commission**. After a few minutes, successfully commissioned nodes will change their status to **Ready**. The CPU cores, RAM, number of drives and storage fields should now correctly reflect the hardware on each node.

For more information on the different states and actions for a node, see `Node actions <https://docs.maas.io/2.1/en/intro-concepts#node-actions>`_ in the MAAS documentation.

You are now almost at the stage where you can let Juju do its thing. But before you take that next step, you are going to rename and tag the newly added nodes so that you can instruct Juju which machines to use for which purpose.

To change the name of a node, select it from the **Nodes** page and use the editable name field in the top right. All nodes will automatically be suffixed with **.maas**. Click on **Save** to save the change.

Tags are normally used to identify nodes with specific hardware, such GPUs for GPU-accelerated CUDA processing. This allows Juju to target these capabilities when deploying applications that may use them. But they can also be used for organisational and management purposes. This is how you are going to use them, by adding a **compute** tag to the four cloud nodes and a juju tag to the node that will act as the Juju controller.

Tags are added from the **Machine summary** section of the same individual node page we used to rename a node. Click **Edit** on this section and look for **Tags**. A tag is added by entering a name for the tag in the empty field and clicking **Save changes**.   
   
.. _install-maas-tags:

.. figure:: /images/1-install-maas_tags.png
   :alt: Adding tags 

A common picture of the state of the nodes that have already been added to the MAAS. You can see the names, tags, and hardware information on each node:   

+-------------------+------------+--------+-------+---------+-----------+
| Node name         | Tag(s)     | CPU(s) | RAM	  | Drives  |  Storage  |
+===================+============+========+=======+=========+===========+
| os-compute01.maas | compute    | 2      |  6.0  |   3     |    85.9   |
+-------------------+------------+--------+-------+---------+-----------+
| os-compute02.maas | compute    | 2      |  6.0  |   3     |    85.9   |
+-------------------+------------+--------+-------+---------+-----------+
| os-compute03.maas | compute    | 2      |  6.0  |   3     |    85.9   |
+-------------------+------------+--------+-------+---------+-----------+   
| os-compute04.maas | compute    | 2      |  6.0  |   3     |    85.9   |
+-------------------+------------+--------+-------+---------+-----------+   
| os-juju01.maas    | juju       | 2      |  4.0  |   1     |    42.9   |
+-------------------+------------+--------+-------+---------+-----------+   




.. _install-maas-next:
   
1.10. Next steps
-----------------

Everything is now configured and ready for our next step. This will involve deploying the Juju controller onto its own node. From there, you will be using Juju and MAAS together to deploy OpenStack into the four remaining cloud nodes.   
   
   

