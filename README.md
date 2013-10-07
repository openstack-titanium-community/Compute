Compute
=======
###Description

This guideline follows installation and configuration of cookbook OpenStack compute utilising chef-client and chef-server in a 3 node cluster. All of the OpenStack cookbooks can be found from https://github.com/stackforge. Also, list of questions and bugs regarding these cookbooks can be found from the respective site https://launchpad.net/openstack-chef. Below is a list of other useful web sites.

Cookbook OpenStack Compute<br>https://github.com/stackforge/cookbook-openstack-compute.git

Nova Guidelines<br>http://docs.openstack.org/trunk/openstack-compute/install/yum/content/ch_installing-openstack-overview.html

Nova Commands<br>http://docs.openstack.org/user-guide/content/novaclient_commands.html

###Requirements

One node with Chef Server 0.10.0 or higher required. Three nodes with Linux environment, these will be used to deploy the recipes for cookbook OpenStack compute. One node with chef-client installed with the chef repository and local cookbooks.

###Dependencies

The following dependencies are required for cookbook OpenStack compute. 

* openstack common
* openstack identity
* openstack-image
* openstack-network
* MySQL
* RabbitMQ

###MySQL

Database port 3306 is required to be opened for openstack services to communicate with the database.
On the Ubuntu environment this can be done by two ways detailed below:
```bash
sudo ufw disable (disables the firewall)
sudo ufw allow 3306 (open up the specific port)
```
Second configuration includes changing the MySQL bind ip address this allows remotes connection to the database.

1. Open file /etc/mysql/my.cnf
2. Edit bind-address with IP address
3. Restart MySQL /etc/init.d/mysqld restart

###Create Database Nova
The following steps create a database table ‘nova’ and provide database privileges.

1.	Log in to MySQL and create a database called 'nova'.
```bash
mysql –h host –u user –p
CREATE DATABASE nova;
```

2. Create	a encrypted password for nova database and grant privileges.
```html
select password('nova');
+-------------------------------------------+
| password('nova')                          |
+-------------------------------------------+
| *0BE3B501084D35F4C66DD3AC4569EAE5EA738212 |
+-------------------------------------------+
GRANT ALL PRIVILEGES ON *.* TO 'nova'@'%' IDENTIFIED BY PASSWORD '*0BE3B501084D35F4C66DD3AC4569EAE5EA738212' 
WITH GRANT OPTION;
FLUSH PRIVILEGES;
```
3. Verify  MySQL connection
```bash
telnet 10.125.0.11 3306
mysql -u nova -h 10.125.0.11 –p
```
###RabbitMQ
Host IP and port number is required from the RabbitMQ messaging service as well as username and password if databag is not being used.

###OpenStack Compute Configuration
use git to get clone of cookbook openstack compute
```bash
git clone http://github.com/stackforge/cookbook-openstack-compute.git
```
Edit file default.rb located at openstack-image/attributes/default.rb and add configuration to set this deployment in a developer mode, this will allow username and passwords to be default i.e. username nova and password nova.
```bash
default["openstack"]["developer_mode"] = true
```
Include the MySQL database IP address
```bash
default["openstack"]["db"]["compute"]["host"] = "10.49.117.250"
```
Add Keystone service endpoints, used for authorisation and authentication.
```bash
default['openstack']['endpoints']['identity-admin']['host'] = "10.49.117.243"
default['openstack']['endpoints']['identity-api']['host'] = "10.49.117.243"
```
Add Glance service endpoint
```bash
default['openstack']['endpoints']['image-registry']['host'] = "10.49.117.243"
default['openstack']['endpoints']['image-api']['host'] = "10.49.117.243"
```
Add Nova service endpoint
```bash
default['openstack']['endpoints']['compute-api']['host'] = node['network']['interfaces']['eth0']['addresses'].to_hash.select {|addr, info| info["family"] == "inet"}.flatten.first
default['openstack']['endpoints']['compute-ec2-api']['host'] = node['network']['interfaces']['eth0']['addresses'].to_hash.select {|addr, info| info["family"] == "inet"}.flatten.first
default['openstack']['endpoints']['compute-ec2-admin']['host'] = node['network']['interfaces']['eth0']['addresses'].to_hash.select {|addr, info| info["family"] == "inet"}.flatten.first
default['openstack']['endpoints']['compute-xvpvnc']['host'] = node['network']['interfaces']['eth0']['addresses'].to_hash.select {|addr, info| info["family"] == "inet"}.flatten.first
default['openstack']['endpoints']['compute-novnc']['host'] = node['network']['interfaces']['eth0']['addresses'].to_hash.select {|addr, info| info["family"] == "inet"}.flatten.first
```
Add RabbitMQ credentials
```bash
default["openstack"]["compute"]["rabbit"]["username"] = "guest"
default["openstack"]["compute"]["rabbit"]["host"] = "10.49.117.250"
```
Add Network configuration
```bash
default["openstack"]["compute"]["networks"] = [
  {
    "label" => "public",
    "ipv4_cidr" => "10.49.117.0/24",
    "num_networks" => "1",
    "network_size" => "255",
    "bridge" => "br100",
    "bridge_dev" => "eth0",
    "dns1" => "8.8.8.8",
    "dns2" => "8.8.4.4",
    "multi_host" => 'T'
  },
  {
    "label" => "private",
    "ipv4_cidr" => "192.168.200.0/24",
    "num_networks" => "1",
    "network_size" => "255",
    "bridge" => "br200",
    "bridge_dev" => "eth2",
    "dns1" => "8.8.8.8",
    "dns2" => "8.8.4.4",
"multi_host" => 'T'
  }
]
```
Amend the compute bind addresses, to the external network interface so the IP can be accessible.
```bash
default["openstack"]["compute"]["xvpvnc_proxy"]["bind_interface"] = "eth0"
default["openstack"]["compute"]["novnc_proxy"]["bind_interface"] = "eth0"
default["openstack"]["compute"]["libvirt"]["bind_interface"] = "eth0"
```

###Recipe Change

Amendment has been added to the nova-common recipe (/recipes/nova-common.rb), as it fails to pick up admin tenant name and user credentials consistently, this has been replaced from keystone to node.

change
```bash
ksadmin_tenant_name = keystone["openstack"]["identity"]["admin_tenant_name"]
ksadmin_user = keystone["openstack"]["identity"]["admin_user"]
```
to 
```bash
ksadmin_tenant_name = node["openstack"]["identity"]["admin_tenant_name"]
ksadmin_user = node["openstack"]["identity"]["admin_user"]
```
In addtion to this change the below keystone credentials attributes have been added to the attribute file (attributes/default.rb).
```bash
default["openstack"]["identity"]["admin_user"] = "admin"
default["openstack"]["identity"]["admin_tenant_name"] = "admin"
```

###Compute Cluster Deployment
Upload the amended cookbook openstack-compute to the chef server.
```bash
sudo knife cookbook import openstack-compute
```
Create a role for which can be tied to multiple nodes. The role in turn will include a run list or recipesn to apply.
```bash
knife role create nova_base
```
Amend the created role to include a run list of openstack compute recipes.
```bash
knife role edit nova_base
```
```bash
{
  "name": "nova_base",
  "description": "base configuration for openstack compute",
  "json_class": "Chef::Role",
  "default_attributes": {
  },
  "override_attributes": {
  },
  "chef_type": "role",
"run_list": [
    "recipe[apt]",
    "recipe[mysql::client]",
    "recipe[openstack-common]",
    "recipe[openstack-compute::api-ec2]",
    "recipe[openstack-compute::api-metadata]",
    "recipe[openstack-compute::api-os-compute]",
    "recipe[openstack-compute::compute]",
    "recipe[openstack-compute::libvirt]",
    "recipe[openstack-compute::identity_registration]",
    "recipe[openstack-compute::network]",
    "recipe[openstack-compute::nova-cert]",
    "recipe[openstack-compute::nova-common]",
    "recipe[openstack-compute::nova-setup]",
    "recipe[openstack-compute::scheduler]",
    "recipe[openstack-compute::vncproxy]",
    "recipe[openstack-compute::conductor]"
  ],
  "env_run_lists": {
  }
}
```
Deploy our cluster with the 3 nodes with chef using the role created.
```bash
knife bootstrap 10.49.117.241 -x username -P password -r 'role[nova_base]' --sudo
knife bootstrap 10.49.117.242 -x username -P password -r 'role[nova_base]' --sudo
knife bootstrap 10.49.117.243 -x username -P password -r 'role[nova_base]' –sudo
```
###Verification
A successful message will be displayed on completion of the bootstrap. The new nodes will be displayed in the chef server UI with all the recipes applied. The nova.conf located under /etc/nova per node will contain details of the openstack service endpoints, database configuration and RabbitMQ. To test openstack compute has been configured and installed correctly nova commands can be run to test the environment.


