=====================
 Deploying with Chef
=====================

We use Chef cookbooks to deploy Ceph. See `Managing Cookbooks with Knife`_ for details
on using ``knife``.  For Chef installation instructions, see
`Installing Chef`_.

Clone the Required Cookbooks
----------------------------

To get the cookbooks for Ceph, clone them from git.::

	cd ~/chef-cookbooks
	git clone https://github.com/opscode-cookbooks/apache2.git
	git clone https://github.com/ceph/ceph-cookbooks.git

Install the Cookbooks
---------------------

To install Ceph, you must install the Ceph cookbooks and the Apache cookbooks
(for use with RADOSGW). :: 

	knife cookbook upload apache2 ceph

Configure your Ceph Environment
-------------------------------

The Chef server can support installation of software for multiple environments.
The environment you create for Ceph requires an ``fsid``, the secret for
your monitor(s) if you are running Ceph with ``cephx`` authentication, and
the host name (i.e., short name) for your monitor hosts.

.. tip: Open an empty text file to hold the following values until you create
   your Ceph environment.

For the filesystem ID, use ``uuidgen`` from the ``uuid-runtime`` package to 
generate a unique identifier. :: 

	uuidgen -r

For the monitor(s) secret(s), use ``ceph-authtool`` to generate the secret(s)::

	ceph-authtool /dev/stdout --name=mon. --gen-key  
 
The secret is the value to the right of ``"key ="``, and should look something 
like this:: 

	AQBAMuJPINJgFhAAziXIrLvTvAz4PRo5IK/Log==

To create an environment for Ceph, set a command line editor. For example:: 

	export EDITOR=vim

Then, use ``knife`` to create an environment. :: 

	knife environment create {env-name}
	
For example: 

	knife environment create Ceph

A JSON file will appear. Perform the following steps: 

#. Enter a description for the environment. 
#. In ``"default_attributes": {}``, add ``"ceph" : {}``.
#. Within ``"ceph" : {}``, add ``"monitor-secret":``.
#. Immediately following ``"monitor-secret":`` add the key you generated within quotes, followed by a comma.
#. Within ``"ceph":{}`` and following the ``monitor-secret`` key-value pair, add ``"config": {}``
#. Within ``"config": {}`` add ``"fsid":``.
#. Immediately following ``"fsid":``, add the unique identifier you generated within quotes, followed by a comma.
#. Within ``"config": {}`` and following the ``fsid`` key-value pair, add ``"mon_initial_members":``
#. Immediately following ``"mon_initial_members":``, enter the initial monitor host names.

For example:: 

	"default-attributes" : {
		"ceph": {
			"monitor-secret": "AQBAMuJPINJgFhAAziXIrLvTvAz4PRo5IK/Log==",
			"config": {
				"fsid": "ddca2b02-3ddf-42fb-ba52-0ee1982c6da0",
				"mon_initial_members": "mon-host"
			}
	}

Advanced users (i.e., developers and QA) may also add ``"ceph_branch": "{branch}"``
to ``default-attributes``, replacing ``{branch}`` with the name of the branch you
wish to use (e.g., ``master``). 

Configure the Roles
-------------------

Navigate to the Ceph cookbooks directory. :: 

	cd ~/chef-cookbooks/ceph-cookbooks
	
Create roles for OSDs, monitors, metadata servers, and RADOS Gateways from
their respective role files. ::

	knife role from file ceph/roles/ceph-osd.rb
	knife role from file ceph/roles/ceph-mon.rb
	knife role from file ceph/roles/ceph-mds.rb
	knife role from file ceph/roles/ceph-radosgw.rb

Configure Nodes
---------------

You must configure each node you intend to include in your Ceph cluster. 
Identify nodes for your Ceph cluster. ::

	knife node list
	
.. note: for each host where you installed Chef and executed ``chef-client``, 
   the Chef server should have a minimal node configuration. You can create
   additional nodes with ``knife node create {node-name}``.

For each node you intend to use in your Ceph cluster, configure the node 
as follows:: 

	knife node edit {node-name}

The node configuration should appear in your text editor. Change the 
``chef_environment`` value to ``Ceph`` (or whatever name you set for your
Ceph environment). 

In the ``run_list``, add ``"recipe[ceph::apt]",`` to all nodes as the first
setting, so that Chef can install or update the necessary packages. Then, 
add at least one of:: 

	"role[ceph-mon]"
	"role[ceph-osd]"
	"role[ceph-mds]"
	"role[ceph-radosgw]"

If you add more than one role, separate them with a comma. The following
example adds a node named `mon-host` to the `Ceph` environment and 
runs the ``apt`` recipe followed by the roles ``ceph-mon`` and ``ceph-osd``:: 

	{
  		"chef_environment": "Ceph",
  		"name": "mon-host",
  		"normal": {
    		"tags": [

    		]
  		},
 		 "run_list": [
			"recipe[ceph::apt]",
			"role[ceph-mon]",
			"role[ceph-mds]"
  		]
	}

Prepare OSD Disks
-----------------

For the Ceph 0.48 Argonaut release, install ``gdisk`` and configure the OSD
hard disks for use with Ceph. Replace ``{fsid}`` with the UUID you generated
while using ``uuidgen -r``. 

.. important: This procedure will erase all information in ``/dev/sdb``.

:: 

	sudo apt-get install gdisk
	sudo sgdisk /dev/sdb --zap-all --clear --mbrtogpt --largest-new=1 --change-name=1:'ceph data' --typecode=1:{fsid}

Create a file system and allocate the disk to your cluster. Specify a 
filesystem (e.g., ``ext4``, ``xfs``, ``btrfs``). When you execute 
``ceph-disk-prepare``, remember to replace ``{fsid}`` with the UUID you 
generated while using ``uuidgen -r``::

	sudo mkfs -t ext4 /dev/sdb1
	sudo mount -o user_xattr /dev/sdb1 /mnt
	sudo ceph-disk-prepare --cluster-uuid={fsid} /mnt
	sudo umount /mnt

Finally, simulate a hotplug event. :: 

	sudo udevadm trigger --subsystem-match=block --action=add

.. _Managing Cookbooks with Knife: http://wiki.opscode.com/display/chef/Managing+Cookbooks+With+Knife
.. _Installing Chef: ../../install/chef