I am using the Mac for installation and have the VirtualBox installed.

For the Newton openstack setup we must have the three virtual machine ready with atleast below requirement:
  
* Controller Node: 2 processor, 4 GB memory, and 5 GB storage
* Compute Node: 1 processor, 2 GB memory, and 10 GB storage
* Network Node: 1 processor, 2 GB memory, and 5 GB storage

Each VM has a NAT Network and a Host-Only Adapter set to the same Adapter.

For simplicity we will use the password rootroot for all passwords.

1) Installation of base system on the VMs

Get CentOs 7(https://www.centos.org/download/) and install it, configure the network and base settings to suite your
configuration.

2) Configure Hosts and the Hostname

On the controller Node:
```bash
echo 'controller' > /etc/hostname
echo '192.168.57.102 controller.example.com controller
192.168.57.100 compute.example.com compute
192.168.57.101 network.example.com network' >> /etc/hosts
```

On the Network Node:
```bash
echo 'network' > /etc/hostname
echo '192.168.57.102 controller.example.com controller
192.168.57.100 compute.example.com compute
192.168.57.101 network.example.com network' >> /etc/hosts
```

On the Compute Node:
```bash
echo 'compute' > /etc/hostname
echo '192.168.57.102 controller.example.com controller
192.168.57.100 compute.example.com compute
192.168.57.101 network.example.com network' >> /etc/hosts
```

3) Upgrade the OS and reboot:

```bash
yum update -y ; reboot
```

4) Verify connectivity

On the controller Node:

```bash
ping -c 4 controller
ping -c 4 network
ping -c 4 compute
```

On the Network Node:

```bash
ping -c 4 controller
ping -c 4 network
ping -c 4 compute
```

On the Compute Node:

```bash
ping -c 4 controller
ping -c 4 network
ping -c 4 compute
```

5) Create SSH Access:

```bash
ssh-keygen
ssh-copy-id root@compute.example.com
ssh-copy-id root@network.example.com
```

Let verify we can connect.

```bash
ssh root@compute.example.com
```

```bash
 ssh root@network.example.com
```

6) Network Time Protocol (NTP) Setup

On the Controller Node:

```bash
yum install chrony
```

Edit the /etc/chrony.conf file and configure the server:

```
server 0.pool.ntp.org
```

Start the NTP service and configure it to start when the system boots:

```bash
systemctl enable chronyd.service
systemctl start chronyd.service
```

On Compute node:

```bash
yum install chrony
```

Edit the /etc/chrony.conf file and configure the server:

```
server controller
```

Start the NTP service and configure it to start when the system boots:

```bash
systemctl enable chronyd.service
systemctl start chronyd.service
```

Do the same on the Network Node.

7) Set OpenStack Newton Repository

```bash
yum install centos-release-openstack-newton -y
yum update -y
yum install python-openstackclient
yum install openstack-selinux
```

8) Install MariaDB

On Controller node

```bash
yum install mariadb mariadb-server python2-PyMySQL
```

Create and edit the /etc/my.cnf.d/openstack.cnf file

```
[mysqld]
bind-address = 10.0.2.15
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

Start the database service and configure it to start when the system boots:

```bash
systemctl enable mariadb.service
systemctl start mariadb.service
```

Secure the database service by running the mysql_secure_installation script.

```bash
mysql_secure_installation
```

8) RabbitMQ message queue Setup

```bash
yum install rabbitmq-server
```

Start the message queue service and configure it to start when the system boots:

```bash
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

Adding the openstack user:

```bash
rabbitmqctl add_user openstack rootroot
```

Permit configuration, write, and read access for the openstack user:

```bash
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

9) Memcached setup

```bash
yum install memcached python-memcached
```

Start the Memcached service and configure it to start when the system boots:

```bash
systemctl enable memcached.service
systemctl start memcached.service
```

10) Installing Keystone

Connect to mysql

```bash
mysql -u root -p
```

And run the following SQL

```
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'rootroot';
```

Install KeyStone:

```bash
yum install openstack-keystone httpd mod_wsgi
```

Edit the /etc/keystone/keystone.conf file and complete with the following:

```
[DEFAULT]
[database]
connection = mysql+pymysql://keystone:rootroot@10.0.2.15/keystone

[token]
provider = fernet
```

Populate the Identity service database:

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

Initialize Fernet key repositories:

```bash
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

Bootstrap the Identity service:

```bash
keystone-manage bootstrap --bootstrap-password rootroot --bootstrap-admin-url http://controller:35357/v3/ \
                          --bootstrap-internal-url http://controller:35357/v3/ \
                          --bootstrap-public-url http://controller:5000/v3/ \
                          --bootstrap-region-id RegionOne
```

Configure the Apache HTTP server:

Edit the /etc/httpd/conf/httpd.conf file and configure the ServerName option to reference the controller node:

```
ServerName controller
```

Create a link to the /usr/share/keystone/wsgi-keystone.conf file:

```bash
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

Start the Apache HTTP service and configure it to start when the system boots:

```bash
systemctl enable httpd.service
systemctl start httpd.service
```

Configure the administrative account by creating a keystonerc_admin file

```
export OS_USERNAME=admin
export OS_PASSWORD=rootroot
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```

11) Create a domain, projects, users, and roles

Set the environment:

```bash
. keystonerc_admin
```

Create the service project:

```bash
openstack project create --domain default --description "Service Project" service
```

Create the demo project:

```bash
openstack project create --domain default --description "Demo Project" demo
```

Create the demo user:

```bash
openstack user create --domain default --password-prompt demo
```

Create the user role:

```bash
openstack role create user
```

Add the user role to the demo project and user:

```bash
openstack role add --project demo --user demo user
```

12) Verify operation of the Identity service

For security reasons, disable the temporary authentication token mechanism:

Edit the /etc/keystone/keystone-paste.ini file and remove admin_token_auth from the [pipeline:public_api], [pipeline:admin_api], and [pipeline:api_v3] sections.

Unset the temporary OS_AUTH_URL and OS_PASSWORD environment variable:

```bash
unset OS_AUTH_URL OS_PASSWORD
```

As the admin user, request an authentication token:

```bash
openstack --os-auth-url http://controller:35357/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
```

As the demo user, request an authentication token:

```bash
openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name demo --os-username demo token issue
```

12) Image (glance) service install and configure

Use the database access client to connect to the database server as the root user:

```bash
mysql -u root -p
```

```
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'rootroot';
```

Create the glance user:

```bash
openstack user create --domain default --password-prompt glance
```

Add the admin role to the glance user and service project:

```bash
openstack role add --project service --user glance admin
```

Create the glance service entity:

```bash
openstack service create --name glance --description "OpenStack Image" image
```

Create the Image service API endpoints:

```bash
openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292
```

Install the packages:

```bash
yum install openstack-glance
```

Edit the /etc/glance/glance-api.conf file and complete the following actions:

```
[DEFAULT]

[database]
connection = mysql+pymysql://glance:rootroot@10.0.2.15/glance

[glance_store]
stores = file,http,swift
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
os_region_name=RegionOne

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = rootroot

[paste_deploy]
flavor = keystone
```

Edit the /etc/glance/glance-registry.conf file and complete the following actions:

```
[DEFAULT]

[database]
connection = mysql+pymysql://glance:rootroot@10.0.2.15/glance
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = rootroot

[paste_deploy]
flavor = keystone
```

Create the image cache folder:

```bash
mkdir /var/lib/glance/images/
```

Populate the Image service database:

su -s /bin/sh -c "glance-manage db_sync" glance

Start the Image services and configure them to start when the system boots:

```bash
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```

Now download the cirros source image:

```bash
sudo yum -y install wget
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2
```

Upload the image to the Image service using the QCOW2 disk format, bare container format, and public visibility so all projects can access it:

```bash
openstack image create "cirros" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare --public
openstack image create "CentOS-7" --file CentOS-7-x86_64-GenericCloud.qcow2 --disk-format qcow2 --container-format bare --public
```

Confirm upload of the image and validate attributes:

```bash
openstack image list
```

13) Compute (nova) service install and configure on Controller node

Before you install and configure the Compute service, you must create databases, service credentials, 
and API endpoints.

Use the database access client to connect to the database server as the root user:

```bash
mysql -u root -p
```

Create the nova_api and nova databases:

```bash
CREATE DATABASE nova_api;
CREATE DATABASE nova;
```

Grant proper access to the databases:

```bash
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'rootroot';
```
Source the admin credentials to gain access to admin-only CLI commands:

```bash
. keystonerc_admin
```

Create the nova user:

```bash
openstack user create --domain default --password-prompt nova
```

Add the admin role to the nova user:

```bash
openstack role add --project service --user nova admin
```

Create the nova service entity:

```bash
openstack service create --name nova --description "OpenStack Compute" compute
```

Create the Compute service API endpoints:

```bash
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1/%\(tenant_id\)s
```

Install the packages:

```bash
yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler
```

Edit the /etc/nova/nova.conf file and complete the following actions:

```bash
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:rootroot@10.0.2.15
auth_strategy = keystone
my_ip = 192.168.57.102
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:rootroot@10.0.2.15/nova_api

[database]
connection = mysql+pymysql://nova:rootroot@10.0.2.15/nova

[glance]
api_servers = http://controller:9292

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = rootroot

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip
```

Populate the Compute databases:

```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage db sync" nova
```

Start the Compute services and configure them to start when the system boots:

```bash
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

14) Compute (nova) service install and configure on Compute Node

Set OpenStack Newton Repository on the Compute Node

```bash
yum install centos-release-openstack-newton -y
yum update -y
yum install python-openstackclient
yum install openstack-selinux
```

Install the packages:

```bash
yum install openstack-nova-compute
```

Edit the /etc/nova/nova.conf file and complete the following actions:

```
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:rootroot@10.0.2.15
auth_strategy = keystone
my_ip = 192.168.57.102
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:rootroot@10.0.2.15/nova_api

[database]
connection = mysql+pymysql://nova:rootroot@10.0.2.15/nova

[glance]
api_servers = http://controller:9292

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = rootroot

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip
```

Start the Compute service including its dependencies and configure them to start automatically when the system boots:

```bash
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

On the Controller Node Verify operation of the Compute service:

```bash
openstack compute service list
```

15) Networking (neutron) service install and setup

On controller node

Use the database access client to connect to the database server as the root user:

```bash
mysql -u root -p
```

Create the neutron database:

```bash
CREATE DATABASE neutron;
```

Grant proper access to the neutron database:

```bash
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'rootroot';
```

Create the neutron user:

```bash
openstack user create --domain default --password-prompt neutron
```

Add the admin role to the neutron user:

```bash
openstack role add --project service --user neutron admin
```

Create the neutron service entity:

```bash
openstack service create --name neutron --description "OpenStack Networking" network
```

Create the Networking service API endpoints:

```bash
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696
```


