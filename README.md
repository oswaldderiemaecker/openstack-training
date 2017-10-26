I am using the Mac for installation and have the VirtualBox installed.

For the Newton openstack setup we must have the three virtual machine ready with atleast below requirement:

* Controller Node: 2 processor, 4 GB memory, and 5 GB storage
* Compute Node: 2 processor, 4 GB memory, and 20 GB storage
* Network Node: 1 processor, 2 GB memory, and 5 GB storage

Each VM has a NAT Network and a Host-Only Adapter set to the same Adapter.

For simplicity we will use the password **rootroot** for all passwords.

# 1 Base Installation

## 1.1 Installation of base system on the VMs

Get [CentOs 7](https://www.centos.org/download/) and install it, configure the network and base settings to suite your
configuration.

## 1.2 Configure Hosts and the Hostname

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

## 1.3 Stop and disable firewalld & NetworkManager Service

**On all Nodes**

```bash
systemctl stop firewalld
systemctl disable firewalld
systemctl stop NetworkManager
systemctl disable NetworkManager
```

Disable SELinux using below command:

```bash
setenforce 0 ; sed -i 's/=enforcing/=disabled/g' /etc/sysconfig/selinux
```

## 1.4 Upgrade the OS and reboot:

```bash
yum update -y ; reboot
```

## 1.5 Verify connectivity

**On the controller Node:**

```bash
ping -c 4 network
ping -c 4 compute
ping -c 4 www.google.com
```

**On the Network Node:**

```bash
ping -c 4 controller
ping -c 4 compute
ping -c 4 www.google.com
```

**On the Compute Node:**

```bash
ping -c 4 controller
ping -c 4 network
ping -c 4 www.google.com
```

## 1.6 Create SSH Access:

**On the controller Node:**

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

## 1.7 Network Time Protocol (NTP) Setup

**On the Controller Node:**

```bash
yum install chrony
```

Edit the /etc/chrony.conf file and configure the server:

```
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst
```

Start the NTP service and configure it to start when the system boots:

```bash
systemctl enable chronyd.service
systemctl start chronyd.service
```

**On Compute and Network node:**

```bash
yum install chrony
```

Edit the /etc/chrony.conf file and configure the server:

```
server controller iburst
```

Start the NTP service and configure it to start when the system boots:

```bash
systemctl enable chronyd.service
systemctl start chronyd.service
```

Verify:

```bash
chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^? controller.example.com        0   6     0     -     +0ns[   +0ns] +/-    0ns
```

## 1.8 Set OpenStack Newton Repository

**On all Nodes install:**

```bash
yum install centos-release-openstack-newton -y
yum update -y
yum install python-openstackclient
```

## 1.9 Install MariaDB

**On Controller node**

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

## 1.10 RabbitMQ message queue Setup

**On Controller node**

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

## 1.11 Memcached setup

**On Controller node**

```bash
yum install memcached python-memcached
```

Start the Memcached service and configure it to start when the system boots:

```bash
systemctl enable memcached.service
systemctl start memcached.service
```

# 2 Services Configuratins

## 2.1.1 Installing Keystone

**On Controller node**

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
unset OS_SERVICE_TOKEN
    export OS_USERNAME=admin
    export OS_PASSWORD=rootroot
    export OS_AUTH_URL=http://192.168.57.102:5000/v3
    export PS1='[\u@\h \W(keystone_admin)]\$ '
    export OS_IDENTITY_API_VERSION=3

export OS_TENANT_NAME=admin
export OS_REGION_NAME=RegionOne
```

## 2.1.2 Create a domain, projects, users, and roles

**On Controller node**

Set the environment:

```bash
. keystonerc_admin
```

Create the service project:

```bash
openstack project create --domain default --description "Service Project" services
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

## 2.1.3 Verify operation of the Identity service

**On Controller node**

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

## 2.2.1 Image (glance) service install and configure

**On Controller node**

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
openstack role add --project services --user glance admin
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
bind_host = 0.0.0.0
bind_port = 9292
workers = 1
image_cache_dir = /var/lib/glance/image-cache
registry_host = 0.0.0.0
debug = False
log_file = /var/log/glance/api.log
log_dir = /var/log/glance

[database]
connection = mysql+pymysql://glance:rootroot@10.0.2.15/glance

[glance_store]
stores = file,http,swift
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
os_region_name=RegionOne

[keystone_authtoken]
auth_uri = http://192.168.57.102:5000/v2.0
auth_type = password
project_name=services
username=glance
password=rootroot
auth_url=http://192.168.57.102:35357

[oslo_policy]
policy_file = /etc/glance/policy.json

[paste_deploy]
flavor = keystone
```

Edit the /etc/glance/glance-registry.conf file and complete the following actions:

```
[DEFAULT]
bind_host = 0.0.0.0
bind_port = 9191
workers = 1
debug = False
log_file = /var/log/glance/registry.log
log_dir = /var/log/glance

[database]
connection = mysql+pymysql://glance:rootroot@10.0.2.15/glance

[keystone_authtoken]
auth_uri = http://192.168.57.102:5000/v2.0
auth_type = password
username=glance
project_name=services
password=rootroot
auth_url=http://192.168.57.102:35357

[oslo_policy]
policy_file = /etc/glance/policy.json

[paste_deploy]
flavor = keystone
```

Create the image cache folder:

```bash
mkdir -p /var/lib/glance/images/
mkdir -p /var/lib/glance/image-cache
chown -R glance:glance /var/lib/glance/images
chown -R glance:glance /var/lib/glance/images-cache
```

Populate the Image service database:

```bash
su -s /bin/sh -c "glance-manage db_sync" glance
```

Start the Image services and configure them to start when the system boots:

```bash
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```

Verify the Glance Service is running:

```bash
systemctl status openstack-glance-api.service openstack-glance-registry.service
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

## 2.2.2 Compute (nova) service install and configure on Controller node

**On Controller node**

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
openstack role add --project services --user nova admin
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
auth_strategy=keystone
use_forwarded_for=False
fping_path=/usr/sbin/fping
rootwrap_config=/etc/nova/rootwrap.conf
allow_resize_to_same_host=False
default_floating_pool=public
force_snat_range=0.0.0.0/0
metadata_host=192.168.57.102
dhcp_domain=novalocal
use_neutron=True
notify_api_faults=False
state_path=/var/lib/nova
scheduler_host_subset_size=1
scheduler_use_baremetal_filters=False
scheduler_available_filters=nova.scheduler.filters.all_filters
scheduler_default_filters=RetryFilter,AvailabilityZoneFilter,RamFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,CoreFilter
scheduler_weight_classes=nova.scheduler.weights.all_weighers
scheduler_host_manager=host_manager
scheduler_driver=filter_scheduler
max_io_ops_per_host=8
max_instances_per_host=50
scheduler_max_attempts=3
report_interval=10
enabled_apis=osapi_compute,metadata
osapi_compute_listen=0.0.0.0
osapi_compute_listen_port=8774
osapi_compute_workers=1
metadata_listen=0.0.0.0
metadata_listen_port=8775
metadata_workers=1
service_down_time=60
vif_plugging_is_fatal=True
vif_plugging_timeout=300
firewall_driver=nova.virt.firewall.NoopFirewallDriver
debug=False
log_dir=/var/log/nova
rpc_backend=rabbit
image_service=nova.image.glance.GlanceImageService
osapi_volume_listen=0.0.0.0

[api_database]
connection=mysql+pymysql://nova:rootroot@10.0.2.15/nova_api

[cinder]
catalog_info=volumev2:cinderv2:publicURL

[conductor]
workers=1

[database]
connection=mysql+pymysql://nova:rootroot@10.0.2.15/nova

[glance]
api_servers=192.168.57.102:9292

[keystone_authtoken]
auth_uri=http://192.168.57.102:5000/
auth_type=password
username=nova
project_name=services
auth_url=http://192.168.57.102:35357
password=rootroot

[libvirt]
vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver

[oslo_concurrency]
lock_path=/var/lib/nova/tmp

[oslo_messaging_rabbit]
kombu_ssl_keyfile=
kombu_ssl_certfile=
kombu_ssl_ca_certs=
rabbit_host=10.0.2.15
rabbit_port=5672
rabbit_use_ssl=False
rabbit_userid=guest
rabbit_password=guest

[oslo_policy]
policy_file=/etc/nova/policy.json

[vnc]
novncproxy_base_url=http://0.0.0.0:6080/vnc_auto.html
novncproxy_host=0.0.0.0
novncproxy_port=6080

[wsgi]
api_paste_config=api-paste.ini
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

## 2.2.3 Compute (nova) service install and configure on Compute Node

**On Compute node**

Set OpenStack Newton Repository on the Compute Node

```bash
yum install centos-release-openstack-newton -y
yum update -y
yum install python-openstackclient
```

Install the packages:

```bash
yum install openstack-nova-compute
```

Edit the /etc/nova/nova.conf file and complete the following actions:

```
[DEFAULT]
auth_strategy=keystone
rootwrap_config=/etc/nova/rootwrap.conf
allow_resize_to_same_host=False
reserved_host_memory_mb=512
heal_instance_info_cache_interval=60
force_snat_range=0.0.0.0/0
metadata_host=192.168.57.102
dhcp_domain=novalocal
use_neutron=True
notify_api_faults=False
state_path=/var/lib/nova
report_interval=10
compute_manager=nova.compute.manager.ComputeManager
service_down_time=60
compute_driver=libvirt.LibvirtDriver
vif_plugging_is_fatal=True
vif_plugging_timeout=300
firewall_driver=nova.virt.firewall.NoopFirewallDriver
force_raw_images=True
debug=False
log_dir=/var/log/nova
rpc_backend=rabbit
image_service=nova.image.glance.GlanceImageService
volume_api_class=nova.volume.cinder.API

[api_database]
connection=mysql+pymysql://nova:rootroot@192.168.57.102/nova_api

[cinder]
catalog_info=volumev2:cinderv2:publicURL

[database]
connection=mysql+pymysql://nova:rootroot@192.168.57.102/nova

[glance]
api_servers=192.168.57.102:9292

[libvirt]
virt_type=qemu
inject_password=False
inject_key=False
inject_partition=-1
live_migration_uri=qemu+tcp://%s/system
cpu_mode=none
vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver

[oslo_concurrency]
lock_path=/var/lib/nova/tmp

[oslo_messaging_rabbit]
kombu_ssl_keyfile=
kombu_ssl_certfile=
kombu_ssl_ca_certs=
rabbit_host=192.168.57.102
rabbit_port=5672
rabbit_use_ssl=False
rabbit_userid=guest
rabbit_password=guest

[vnc]
enabled=True
keymap=en-us
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=compute.example.com
novncproxy_base_url=http://192.168.57.102:6080/vnc_auto.html
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

## 2.3.1 Networking (neutron) service install and setup

**On controller node**

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
openstack role add --project services --user neutron admin
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

## 2.3.2 Configure Neutron the Networking Self-service networks

Install the components:

```bash
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-common
```

Configure the server component

Edit the /etc/neutron/neutron.conf file and complete the following actions:

```bash
[DEFAULT]
bind_host=0.0.0.0
auth_strategy=keystone
core_plugin=neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins=router,metering
allow_overlapping_ips=True
notify_nova_on_port_status_changes=True
notify_nova_on_port_data_changes=True
api_workers=1
rpc_workers=1
router_scheduler_driver=neutron.scheduler.l3_agent_scheduler.ChanceScheduler
l3_ha=False
max_l3_agents_per_router=3
debug=False
log_dir=/var/log/neutron
rpc_backend=rabbit
control_exchange=neutron
nova_url=http://192.168.57.102:8774/v2

[agent]
root_helper=sudo neutron-rootwrap /etc/neutron/rootwrap.conf

[database]
connection=mysql+pymysql://neutron:rootroot@10.0.2.15/neutron

[keystone_authtoken]
auth_uri=http://192.168.57.102:5000/v2.0
auth_type=password
project_name=services
password=rootroot
username=neutron
project_domain_name=Default
user_domain_name=Default
auth_url=http://192.168.57.102:35357

[nova]
region_name=RegionOne
auth_url=http://192.168.57.102:35357
auth_type=password
password=rootroot
project_domain_id=default
project_name=services
tenant_name=services
user_domain_id=default
username=nova

[oslo_concurrency]
lock_path=$state_path/lock

[oslo_messaging_rabbit]
kombu_ssl_keyfile=
kombu_ssl_certfile=
kombu_ssl_ca_certs=
rabbit_host=10.0.2.15
rabbit_port=5672
rabbit_use_ssl=False
rabbit_userid=guest
rabbit_password=guest

[oslo_policy]
policy_file=/etc/neutron/policy.json
```

Configure the Modular Layer 2 (ML2) plug-in:

Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file and complete the following actions:

```bash
[DEFAULT]

[ml2]
type_drivers = vxlan
tenant_network_types = vxlan
mechanism_drivers =openvswitch
path_mtu = 0

[ml2_type_vxlan]
vni_ranges =10:100
vxlan_group = 224.0.0.1

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

Configure the Compute service to use the Networking service:

**On the Controller Node**

Edit the /etc/nova/nova.conf file and perform the following actions:

```bash
[neutron]
url=http://192.168.57.102:9696
region_name=RegionOne
ovs_bridge=br-int
extension_sync_interval=600
service_metadata_proxy=True
metadata_proxy_shared_secret=rootroot
timeout=60
auth_type=v3password
auth_url=http://192.168.57.102:35357/v3
project_name=services
project_domain_name=Default
username=neutron
user_domain_name=Default
password=rootroot
```

The Networking service initialization scripts expect a symbolic link /etc/neutron/plugin.ini pointing to the ML2 plug-in configuration file, /etc/neutron/plugins/ml2/ml2_conf.ini. If this symbolic link does not exist, create it using the following command:

```bash
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

Populate the database:

```bash
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

Restart the Compute API service:

```bash
systemctl restart openstack-nova-api.service
```

Start the Networking services and configure them to start when the system boots.

For both networking options:

```bash
systemctl enable neutron-server.service
```

```bash
systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl restart neutron-server.service
systemctl restart openstack-glance-api.service openstack-glance-registry.service
systemctl restart httpd.service memcached.service
systemctl restart rabbitmq-server.service
```

Verify all is running fine:

```bash
systemctl status openstack-nova-api.service
systemctl status openstack-nova-consoleauth.service
systemctl status openstack-nova-scheduler.service
systemctl status openstack-nova-conductor.service
systemctl status openstack-nova-novncproxy.service
systemctl status neutron-server.service
systemctl status neutron-metadata-agent.service
systemctl status openstack-glance-api.service
systemctl status openstack-glance-registry.service
systemctl status httpd.service memcached.service
systemctl status rabbitmq-server.service
```

## 2.3.2 Configure Neutron on the Compute Node

**On Compute node**

Install the components:

```bash
yum install openstack-neutron
yum install openvswitch
yum install openstack-neutron-openvswitch
```

Edit the /etc/neutron/neutron.conf file and complete the following actions:

```bash
[DEFAULT]
bind_host=0.0.0.0
auth_strategy=keystone
core_plugin=neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins=router,metering
allow_overlapping_ips=True
debug=False
log_dir=/var/log/neutron
rpc_backend=rabbit
control_exchange=neutron

[agent]
root_helper=sudo neutron-rootwrap /etc/neutron/rootwrap.conf

[oslo_concurrency]
lock_path=$state_path/lock

[oslo_messaging_rabbit]
kombu_ssl_keyfile=
kombu_ssl_certfile=
kombu_ssl_ca_certs=
rabbit_host=192.168.57.102
rabbit_port=5672
rabbit_use_ssl=False
rabbit_userid=guest
rabbit_password=guest
```


Add the Neutron configuration into the /etc/nova/nova.conf file on the Compute Node:

```bash
[neutron]
url=http://192.168.57.102:9696
region_name=RegionOne
ovs_bridge=br-int
extension_sync_interval=600
timeout=60
auth_type=v3password
auth_url=http://192.168.57.102:35357/v3
project_name=service
project_domain_name=Default
username=neutron
user_domain_name=Default
password=rootroot
```

Configure the OpenVSwitch agent in /etc/neutron/plugins/ml2/openvswitch_agent.ini:

```bash
[DEFAULT]

[agent]
tunnel_types =vxlan
vxlan_udp_port = 4789
l2_population = False
drop_flows_on_start = False

[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
local_ip = 10.0.2.15

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

Restart all services:

```bash
systemctl restart openvswitch.service
systemctl restart neutron-openvswitch-agent.service
systemctl restart neutron-ovs-cleanup.service
```

Verify all is running fine:

```bash
systemctl status openvswitch.service
systemctl status neutron-openvswitch-agent.service
systemctl status neutron-ovs-cleanup.service
```

Verify OpenVSwitch is installed and Configured correctly:

```bash
ovs-vsctl show
```

Should output:

```
73564bd7-9471-402e-892a-1bd3c518da78
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
    ovs_version: "2.6.1"
```

## 2.3.3 Configure Neutron on the Network Node

**On Network node**

Install the components:

```bash
yum install openstack-neutron
yum install openvswitch
yum install openstack-neutron-openvswitch
```

Edit the /etc/neutron/neutron.conf file and complete the following actions:

```bash
[DEFAULT]
bind_host=0.0.0.0
auth_strategy=keystone
core_plugin=neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins=router,metering
allow_overlapping_ips=True
debug=False
log_dir=/var/log/neutron
rpc_backend=rabbit
control_exchange=neutron

[agent]
root_helper=sudo neutron-rootwrap /etc/neutron/rootwrap.conf

[keystone_authtoken]
auth_uri=http://192.168.57.102:5000/v2.0
auth_type=password
project_name=services
password=rootroot
username=neutron
project_domain_name=Default
user_domain_name=Default
auth_url=http://192.168.57.102:35357

[oslo_concurrency]
lock_path=$state_path/lock

[oslo_messaging_rabbit]
kombu_ssl_keyfile=
kombu_ssl_certfile=
kombu_ssl_ca_certs=
rabbit_host=192.168.57.102
rabbit_port=5672
rabbit_use_ssl=False
rabbit_userid=guest
rabbit_password=guest
```

Configure the OpenVSwitch agent in /etc/neutron/plugins/ml2/openvswitch_agent.ini:

```bash
[DEFAULT]

[agent]
tunnel_types =vxlan
vxlan_udp_port = 4789
l2_population = False
drop_flows_on_start = False

[ovs]
integration_bridge = br-int
tunnel_bridge = br-tun
local_ip = 10.0.2.15

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

Configure l3-agent by editing the /etc/neutron/l3_agent.ini file:

```bash
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
agent_mode = legacy
external_network_bridge =br-ex
debug = False
```

Configure dhcp-agent by editing the /etc/neutron/dhcp_agent.ini file:

```bash
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
resync_interval = 30
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = False
enable_metadata_network = False
debug = False
root_helper=sudo neutron-rootwrap /etc/neutron/rootwrap.conf
state_path=/var/lib/neutron
```

Configure the metadata-agent by editing the /etc/neutron/metadata_agent.ini file:

```bash
[DEFAULT]
nova_metadata_ip = 192.168.57.102
metadata_proxy_shared_secret = rootroot
metadata_workers = 1
debug = False
```

Configure the metadata-agent by editing the /etc/neutron/metering_agent.ini file:

```bash
[DEFAULT]
driver = neutron.services.metering.drivers.noop.noop_driver.NoopMeteringDriver
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
debug = False
```

Restart the services:

```bash
systemctl restart openvswitch.service
systemctl restart neutron-openvswitch-agent.service
systemctl restart neutron-ovs-cleanup.service
systemctl restart neutron-dhcp-agent.service
systemctl restart neutron-l3-agent.service
systemctl restart neutron-metadata-agent.service
```

Verify all is running fine:

```bash
systemctl status openvswitch.service
systemctl status neutron-openvswitch-agent.service
systemctl status neutron-ovs-cleanup.service
systemctl status neutron-dhcp-agent.service
systemctl status neutron-l3-agent.service
systemctl status neutron-metadata-agent.service
```

Verify OpenVSwitch is installed and Configured correctly:

```bash
ovs-vsctl show
```

Should output:

```
d61c5814-4bce-41d6-a966-a4220282f1d1
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "qg-afbd49e9-ee"
            Interface "qg-afbd49e9-ee"
                type: internal
        Port "qg-337673f5-dd"
            Interface "qg-337673f5-dd"
                type: internal
    Bridge br-tun
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.6.1"
```


## 2.4.1 Do the network Configuration:

**On the Controller:**

Create the base networks:

```bash
openstack network create public
openstack subnet create --network public --subnet-range 172.24.4.224/28 public_subnet
openstack network create private
openstack subnet create --network private --subnet-range 10.0.0.0/24 private_subnet
openstack router create private-router
openstack router add subnet private-router private_subnet
```

## 2.5 Install and Configure Cinder

**On Network node**

Install the components:

```bash
yum install openstack-cinder targetcli python-keystone
```



Edit the /etc/cinder/cinder.conf file and complete the following actions:

```bash


## 2.7 Dashboard install and configure

Install the packages:

```bash
yum install openstack-dashboard
```

Edit the /etc/openstack-dashboard/local_settings file and complete the following actions:

```bash
import os
from django.utils.translation import ugettext_lazy as _
from horizon.utils import secret_key
from openstack_dashboard import exceptions
from openstack_dashboard.settings import HORIZON_CONFIG
DEBUG = False
TEMPLATE_DEBUG = DEBUG
WEBROOT = '/dashboard/'
LOGIN_URL = '/dashboard/auth/login/'
LOGOUT_URL = '/dashboard/auth/logout/'
LOGIN_REDIRECT_URL = '/dashboard/'
ALLOWED_HOSTS = ['*', ]
OPENSTACK_API_VERSIONS = {
  'identity': 3,
}
HORIZON_CONFIG = {
    'dashboards': ('project', 'admin', 'settings',),
    'default_dashboard': 'project',
    'user_home': 'openstack_dashboard.views.get_user_home',
    'ajax_queue_limit': 10,
    'auto_fade_alerts': {
        'delay': 3000,
        'fade_duration': 1500,
        'types': ['alert-success', 'alert-info']
    },
    'help_url': "http://docs.openstack.org",
    'exceptions': {'recoverable': exceptions.RECOVERABLE,
                   'not_found': exceptions.NOT_FOUND,
                   'unauthorized': exceptions.UNAUTHORIZED},
}
HORIZON_CONFIG["password_autocomplete"] = "off"
HORIZON_CONFIG["images_panel"] = "legacy"
LOCAL_PATH = os.path.dirname(os.path.abspath(__file__))
SECRET_KEY = '8c725ee43cc84768bda3d611136b2b6c'

CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
    }
}

SESSION_ENGINE = "django.contrib.sessions.backends.cache"
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
OPENSTACK_KEYSTONE_URL = "http://192.168.57.102:5000/v2.0"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"

OPENSTACK_KEYSTONE_BACKEND = {
    'can_edit_domain': True,
    'can_edit_group': True,
    'can_edit_project': True,
    'can_edit_role': True,
    'can_edit_user': True,
    'name': 'native',
}

OPENSTACK_HYPERVISOR_FEATURES = {
    'can_set_mount_point': False,
    'can_set_password': False,
}

OPENSTACK_CINDER_FEATURES = {
    'enable_backup': False,
}

OPENSTACK_NEUTRON_NETWORK = {
    'enable_distributed_router': False,
    'enable_firewall': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_quotas': True,
    'enable_security_group': True,
    'enable_vpn': False,
    'profile_support': None,
}

API_RESULT_LIMIT = 1000
API_RESULT_PAGE_SIZE = 20
TIME_ZONE = "UTC"
POLICY_FILES_PATH = '/etc/openstack-dashboard'
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '%(asctime)s %(process)d %(levelname)s %(name)s '
                      '%(message)s'
        },
        'normal': {
            'format': 'dashboard-%(name)s: %(levelname)s %(message)s'
        },
    },
    'handlers': {
        'null': {
            'level': 'DEBUG',
            'class': 'django.utils.log.NullHandler',
        },
        'console': {

            'level': 'INFO',
            'class': 'logging.StreamHandler',
        },
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': '/var/log/horizon/horizon.log',
            'formatter': 'verbose',
        },
    },
    'loggers': {


        'django.db.backends': {
            'handlers': ['null'],
            'propagate': False,
        },
        'requests': {
            'handlers': ['null'],
            'propagate': False,
        },
        'horizon': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'openstack_dashboard': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'novaclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'cinderclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'keystoneclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'glanceclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'neutronclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'heatclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'ceilometerclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'troveclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'swiftclient': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'openstack_auth': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'nose.plugins.manager': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
        'django': {

            'handlers': ['file'],

            'level': 'INFO',
            'propagate': False,
        },
    }
}
SECURITY_GROUP_RULES = {
    'all_tcp': {
        'name': 'ALL TCP',
        'ip_protocol': 'tcp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_udp': {
        'name': 'ALL UDP',
        'ip_protocol': 'udp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_icmp': {
        'name': 'ALL ICMP',
        'ip_protocol': 'icmp',
        'from_port': '-1',
        'to_port': '-1',
    },
    'ssh': {
        'name': 'SSH',
        'ip_protocol': 'tcp',
        'from_port': '22',
        'to_port': '22',
    },
    'smtp': {
        'name': 'SMTP',
        'ip_protocol': 'tcp',
        'from_port': '25',
        'to_port': '25',
    },
    'dns': {
        'name': 'DNS',
        'ip_protocol': 'tcp',
        'from_port': '53',
        'to_port': '53',
    },
    'http': {
        'name': 'HTTP',
        'ip_protocol': 'tcp',
        'from_port': '80',
        'to_port': '80',
    },
    'pop3': {
        'name': 'POP3',
        'ip_protocol': 'tcp',
        'from_port': '110',
        'to_port': '110',
    },
    'imap': {
        'name': 'IMAP',
        'ip_protocol': 'tcp',
        'from_port': '143',
        'to_port': '143',
    },
    'ldap': {
        'name': 'LDAP',
        'ip_protocol': 'tcp',
        'from_port': '389',
        'to_port': '389',
    },
    'https': {
        'name': 'HTTPS',
        'ip_protocol': 'tcp',
        'from_port': '443',
        'to_port': '443',
    },
    'smtps': {
        'name': 'SMTPS',
        'ip_protocol': 'tcp',
        'from_port': '465',
        'to_port': '465',
    },
    'imaps': {
        'name': 'IMAPS',
        'ip_protocol': 'tcp',
        'from_port': '993',
        'to_port': '993',
    },
    'pop3s': {
        'name': 'POP3S',
        'ip_protocol': 'tcp',
        'from_port': '995',
        'to_port': '995',
    },
    'ms_sql': {
        'name': 'MS SQL',
        'ip_protocol': 'tcp',
        'from_port': '1433',
        'to_port': '1433',
    },
    'mysql': {
        'name': 'MYSQL',
        'ip_protocol': 'tcp',
        'from_port': '3306',
        'to_port': '3306',
    },
    'rdp': {
        'name': 'RDP',
        'ip_protocol': 'tcp',
        'from_port': '3389',
        'to_port': '3389',
    },
}
SESSION_TIMEOUT = 1800
COMPRESS_OFFLINE = True
FILE_UPLOAD_TEMP_DIR = '/var/tmp'
REST_API_REQUIRED_SETTINGS = ['OPENSTACK_HYPERVISOR_FEATURES',
                              'LAUNCH_INSTANCE_DEFAULTS',
                              'OPENSTACK_IMAGE_FORMATS']
```

Set permission on Horizon log:

```bash
chown -R apache:apache /var/log/horizon
```

Finalize the configuration of OpenStack:

Create Flavors:

```bash
openstack flavor create --ram 1024 --vcpus 1 --disk 1 --public t1.tiny
```

# 3 Tips

## 3.1.1 Restart everything

**On the Controller Node:**

```bash
systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl restart neutron-server.service
systemctl restart openstack-glance-api.service openstack-glance-registry.service
systemctl restart httpd.service memcached.service
systemctl restart rabbitmq-server.service
```

Verify all is running fine:

```bash
systemctl status openstack-nova-api.service
systemctl status openstack-nova-consoleauth.service
systemctl status openstack-nova-scheduler.service
systemctl status openstack-nova-conductor.service
systemctl status openstack-nova-novncproxy.service
systemctl status neutron-server.service
systemctl status neutron-metadata-agent.service
systemctl status openstack-glance-api.service
systemctl status openstack-glance-registry.service
systemctl status httpd.service memcached.service
systemctl status rabbitmq-server.service
```

On the Network Node:

```bash
systemctl restart openvswitch.service
systemctl restart neutron-openvswitch-agent.service
systemctl restart neutron-ovs-cleanup.service
systemctl restart neutron-dhcp-agent.service
systemctl restart neutron-l3-agent.service
systemctl restart neutron-metadata-agent.service
```

Verify all is running fine:

```bash
systemctl status openvswitch.service
systemctl status neutron-openvswitch-agent.service
systemctl status neutron-ovs-cleanup.service
systemctl status neutron-dhcp-agent.service
systemctl status neutron-l3-agent.service
systemctl status neutron-metadata-agent.service
```

**On the Compute Node:**

```bash
systemctl restart openvswitch.service
systemctl restart neutron-openvswitch-agent.service
systemctl restart neutron-ovs-cleanup.service
```

Verify all is running fine:

```bash
systemctl status openvswitch.service
systemctl status neutron-openvswitch-agent.service
systemctl status neutron-ovs-cleanup.service
```

OpenStack Ports:

OpenStack Service      Port
Nova-api               8773 (for EC2 API)
                       8774 (for openstack API)
                       8775 (metadata port)
                       3333 (when accessing S3 API)
nova-novncproxy        6080
                       5800/5900 (VNC)
cinder                 8776
glance                 9191 (glance registry)
                       9292 (glance api)
keystone               5000 (public port)
                       35357 (admin port)
http                   80
Mysql                  3306
AMQP                   5672

Update the hours on the Nodes:

```bash
yum install ntpdate
ntpdate -u 0.europe.pool.ntp.org
```

# TODO

* Cinder
* Floating IPs
* ip netns exec
* Swift

