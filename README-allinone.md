For the Newton openstack setup we must have one virtual machine ready with at least below requirement:

* Controller Node: 2 processor, 8 GB memory, and 16 GB storage

For simplicity we will use the password **rootroot** for all passwords.

It is recommended to make snapshots often so you can go back if any goes wrong.

# 1 Base Installation

## 1.1 Installation of base system on the VMs

Get [CentOs 7](https://www.centos.org/download/) and install it, configure the network and base settings to suite your
configuration.

## 1.2 Configure Hosts and the Hostname

Get your IP address:

```bash
ip addr list
```

On the controller Node:
```bash
echo '192.168.178.93 controller.example.com controller
192.168.178.93 compute.example.com compute
192.168.178.93 network.example.com network' >> /etc/hosts
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
getenforce
```

## 1.4 Upgrade the OS and reboot:

```bash
yum update -y ; reboot
```

## 1.5 Verify connectivity

**On the controller Node:**

```bash
ping -c 4 www.google.com
```

## 1.6 Network Time Protocol (NTP) Setup

**On the Controller Node:**

```bash
yum install chrony -y
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

Update the hour:

```bash
yum install ntpdate -y
ntpdate -u 0.europe.pool.ntp.org
```

## 1.7 Set OpenStack Newton Repository

**On all Nodes install:**

```bash
yum install centos-release-openstack-pike -y
yum update -y
yum install python-openstackclient openstack-selinux -y
```

## 1.8 Install MariaDB

**On Controller node**

```bash
yum install mariadb mariadb-server python2-PyMySQL -y
```

Create and edit the /etc/my.cnf.d/openstack.cnf file

```
[mysqld]
bind-address = controller.example.com
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
systemctl status mariadb.service
```

Secure the database service by running the mysql_secure_installation script.

```bash
mysql_secure_installation
```

## 1.10 RabbitMQ message queue Setup

**On Controller node**

```bash
yum install rabbitmq-server -y
```

Start the message queue service and configure it to start when the system boots:

```bash
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
systemctl status rabbitmq-server.service
```

Adding the openstack user:

```bash
rabbitmqctl add_user openstack rootroot
```

Permit configuration, write, and read access for the openstack user:

```bash
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## 1.10 Memcached setup

**On Controller node**

```bash
yum install memcached python-memcached -y
```

Start the Memcached service and configure it to start when the system boots:

```bash
systemctl enable memcached.service
systemctl start memcached.service
systemctl status memcached.service
```

# 2 Services Configurations

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
yum install openstack-keystone httpd mod_wsgi -y
```

Edit the /etc/keystone/keystone.conf file and replace with the following:

```
[DEFAULT]
[database]
connection = mysql+pymysql://keystone:rootroot@controller.example.com/keystone

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
keystone-manage bootstrap --bootstrap-password rootroot --bootstrap-admin-url http://controller.example.com:35357/v3/ \
                          --bootstrap-internal-url http://controller.example.com:35357/v3/ \
                          --bootstrap-public-url http://controller.example.com:5000/v3/ \
                          --bootstrap-region-id RegionOne
```

Configure the Apache HTTP server:

Edit the /etc/httpd/conf/httpd.conf file and configure the ServerName option to reference the controller node IP:

```
ServerName 192.168.178.93
```

Create a link to the /usr/share/keystone/wsgi-keystone.conf file:

```bash
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
```

Disable SELinux again using below command:

```bash
setenforce 0 ; sed -i 's/=enforcing/=disabled/g' /etc/sysconfig/selinux
getenforce
```

Start the Apache HTTP service and configure it to start when the system boots:

```bash
systemctl enable httpd.service
systemctl start httpd.service
systemctl status httpd.service
```

Configure the administrative account by creating a keystonerc_admin file

```
unset OS_SERVICE_TOKEN
    export OS_USERNAME=admin
    export OS_PASSWORD=rootroot
    export OS_AUTH_URL=http://controller.example.com:5000/v3
    export PS1='[\u@\h \W(keystone_admin)]\$ '

export OS_TENANT_NAME=admin
export OS_REGION_NAME=RegionOne
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
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

Unset the temporary OS_AUTH_URL and OS_PASSWORD environment variable:

```bash
unset OS_AUTH_URL OS_PASSWORD
```

As the admin user, request an authentication token:

```bash
openstack --os-auth-url http://controller.example.com:35357/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
```

As the demo user, request an authentication token:

```bash
openstack --os-auth-url http://controller.example.com:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name demo --os-username demo token issue
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
openstack endpoint create --region RegionOne image public http://controller.example.com:9292
openstack endpoint create --region RegionOne image internal http://controller.example.com:9292
openstack endpoint create --region RegionOne image admin http://controller.example.com:9292
```

Install the packages:

```bash
yum install openstack-glance -y
```

Edit the /etc/glance/glance-api.conf file and replace with the following actions:

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
connection = mysql+pymysql://glance:rootroot@controller.example.com/glance

[glance_store]
stores = file,http,swift
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
os_region_name=RegionOne

[keystone_authtoken]
auth_uri = http://controller.example.com:5000/v2.0
auth_type = password
project_name=services
username=glance
password=rootroot
auth_url=http://controller.example.com:35357

[oslo_policy]
policy_file = /etc/glance/policy.json

[paste_deploy]
flavor = keystone
```

Edit the /etc/glance/glance-registry.conf file and replace with the following actions:

```
[DEFAULT]
bind_host = 0.0.0.0
bind_port = 9191
workers = 1
debug = False
log_file = /var/log/glance/registry.log
log_dir = /var/log/glance

[database]
connection = mysql+pymysql://glance:rootroot@controller.example.com/glance

[keystone_authtoken]
auth_uri = http://controller.example.com:5000/v2.0
auth_type = password
username=glance
project_name=services
password=rootroot
auth_url=http://controller.example.com:35357

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
chown -R glance:glance /var/lib/glance/image-cache
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
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img  
```

Upload the image to the Image service using the QCOW2 disk format, bare container format, and public visibility so all projects can access it:

```bash
openstack image create "cirros" --file cirros-0.3.5-x86_64-disk.img --disk-format qcow2 --container-format bare --public
```

Confirm upload of the image and validate attributes:

```bash
openstack image list
```

## 2.2.2 Cinder service install and configure on Controller node

**On Controller node**

Use the database access client to connect to the database server as the root user:

```bash
mysql -u root -p
```

```
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'rootroot';
```

Create the cinder user:

```bash
openstack user create --domain default --password-prompt cinder
```

Add the admin role to the glance user and service project:

```bash
openstack role add --project services --user cinder admin
```

Create the glance service entity:

```bash
openstack service create --name cinder --description "OpenStack Block Storage" volume
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2

```

Create the Image service API endpoints:

```bash
openstack endpoint create --region RegionOne volume public http://controller.example.com:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume internal http://controller.example.com:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volume admin http://controller.example.com:8776/v1/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 public http://controller.example.com:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 internal http://controller.example.com:8776/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev2 admin http://controller.example.com:8776/v2/%\(tenant_id\)s
```

Install the packages:

```bash
yum install openstack-cinder targetcli -y
```

Edit the /etc/cinder/cinder.conf file and replace with the following actions:

```bash
[DEFAULT]
enable_v3_api = True
storage_availability_zone = nova
default_availability_zone = nova
default_volume_type = iscsi
enabled_backends = lvm
osapi_volume_listen = 0.0.0.0
osapi_volume_workers = 1
nova_catalog_info = compute:nova:publicURL
nova_catalog_admin_info = compute:nova:adminURL
debug = False
log_dir = /var/log/cinder
rpc_backend = rabbit
control_exchange = openstack
api_paste_config = /etc/cinder/api-paste.ini
glance_host=controller.example.com

[database]
connection = mysql+pymysql://cinder:rootroot@controller.example.com/cinder

[keystone_authtoken]
auth_uri = http://controller.example.com:5000
auth_type = password
username=cinder
auth_url=http://controller.example.com:35357
project_name=services
password=rootroot

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp

[oslo_messaging_rabbit]
kombu_ssl_keyfile =
kombu_ssl_certfile =
kombu_ssl_ca_certs =
rabbit_host = controller.example.com
rabbit_port = 5672
rabbit_use_ssl = False
rabbit_userid = guest
rabbit_password = guest

[oslo_policy]
policy_file = /etc/cinder/policy.json

[lvm]
iscsi_helper=lioadm
iscsi_ip_address=192.168.178.93
volume_driver=cinder.volume.drivers.lvm.LVMVolumeDriver
volumes_dir=/var/lib/cinder/volumes
volume_backend_name=lvm
```

**Change the iscsi_ip_address to your controller IP:**

```bash
[lvm]
...
iscsi_ip_address=192.168.178.93
...
```

Populate the Compute databases:

```bash
su -s /bin/sh -c "cinder-manage db sync" cinder
```

Create the volume folder:

```bash
mkdir -p /var/lib/cinder/volumes
```

Lets create the LVM Volume.

Verify the disk is attached:

```bash
fdisk -l
Disk /dev/sdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

Create the LVM volume (using loop for testing purpose, should use lvm partition):

```bash
dd if=/dev/zero of=/var/lib/cinder/cinder-volumes bs=1G count=4
losetup /dev/loop0 /var/lib/cinder/cinder-volumes
pvcreate /dev/loop0
vgcreate "cinder-volumes" /dev/loop0
vgdisplay
```

Create a systemd unit file /usr/lib/systemd/system/openstack-losetup.service to mount our loop device:

```bash
[Unit]
    Description=Setup cinder-volume loop device
    DefaultDependencies=false
    Before=openstack-cinder-volume.service
    After=local-fs.target

    [Service]
    Type=oneshot
    ExecStart=/usr/bin/sh -c '/usr/sbin/losetup -j /var/lib/cinder/cinder-volumes | /usr/bin/grep /var/lib/cinder/cinder-volumes || /usr/sbin/losetup -f /var/lib/cinder/cinder-volumes'
    ExecStop=/usr/bin/sh -c '/usr/sbin/losetup -j /var/lib/cinder/cinder-volumes | /usr/bin/cut -d : -f 1 | /usr/bin/xargs /usr/sbin/losetup -d'
    TimeoutSec=60
    RemainAfterExit=yes

    [Install]
    RequiredBy=openstack-cinder-volume.service
```

Enable the service at boot:

```bash
ln -s /usr/lib/systemd/system/openstack-losetup.service /etc/systemd/system/multi-user.target.wants/openstack-losetup.service
```

Restart the service:

```bash
systemctl enable openstack-cinder-api.service
systemctl enable openstack-cinder-scheduler.service 
systemctl enable openstack-cinder-volume.service 
systemctl enable openstack-cinder-backup.service
systemctl restart openstack-losetup.service
systemctl restart openstack-cinder-api.service
systemctl restart openstack-cinder-scheduler.service 
systemctl restart openstack-cinder-volume.service 
systemctl restart openstack-cinder-backup.service
```

Verify the service:

```bash
systemctl status openstack-cinder-api.service
systemctl status openstack-cinder-scheduler.service 
systemctl status openstack-cinder-volume.service 
systemctl status openstack-cinder-backup.service
```

## 2.2.2 Compute (nova) service install and configure

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
CREATE DATABASE nova_cell0;
```

Grant proper access to the databases:

```bash
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'rootroot';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'rootroot';
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

Create the Nova Compute service API endpoints:

```bash
openstack endpoint create --region RegionOne compute public http://controller.example.com:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://controller.example.com:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://controller.example.com:8774/v2.1/%\(tenant_id\)s
```

Create the nova placement user:

```bash
openstack user create --domain default --password-prompt placement
```

Add the admin role to the nova placement user:

```bash
openstack role add --project services --user placement admin
```

Create the nova placement service entity:

```bash
openstack service create --name placement --description "Placement API" placement
```

Create the Nova Placement service API endpoints:

```bash
openstack endpoint create --region RegionOne placement public http://controller.example.com:8778
openstack endpoint create --region RegionOne placement internal http://controller.example.com:8778
openstack endpoint create --region RegionOne placement admin http://controller.example.com:8778
```

Install the packages:

```bash
yum install openstack-nova-api openstack-nova-conductor openstack-nova-console openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-compute openstack-nova-placement-api openstack-nova-migration -y
```

Edit the /etc/nova/nova.conf file and replace with the following actions:

```bash
[DEFAULT]
my_ip = 192.168.178.93
rootwrap_config=/etc/nova/rootwrap.conf
compute_driver=libvirt.LibvirtDriver
allow_resize_to_same_host=True
vif_plugging_is_fatal=True
vif_plugging_timeout=300
force_raw_images=True
reserved_host_memory_mb=512
cpu_allocation_ratio=16.0
ram_allocation_ratio=1.5
heal_instance_info_cache_interval=60
force_snat_range=0.0.0.0/0
metadata_host=controller.example.com
dhcp_domain=novalocal
use_neutron = True
firewall_driver=nova.virt.firewall.NoopFirewallDriver
state_path=/var/lib/nova
report_interval=10
service_down_time=60
enabled_apis=osapi_compute,metadata
osapi_compute_listen=0.0.0.0
osapi_compute_listen_port=8774
osapi_compute_workers=2
metadata_listen=0.0.0.0
metadata_listen_port=8775
metadata_workers=2
debug=False
log_dir=/var/log/nova
transport_url=rabbit://guest:guest@controller.example.com:5672/
image_service=nova.image.glance.GlanceImageService
osapi_volume_listen=0.0.0.0
volume_api_class=nova.volume.cinder.API

[api]
auth_strategy=keystone
use_forwarded_for=False
fping_path=/usr/sbin/fping

[api_database]
connection=mysql+pymysql://nova:rootroot@controller.example.com/nova_api

[cinder]
catalog_info=volumev2:cinderv2:publicURL

[conductor]
workers=2

[database]
connection=mysql+pymysql://nova:rootroot@controller.example.com/nova

[filter_scheduler]
host_subset_size=1
max_io_ops_per_host=8
max_instances_per_host=50
available_filters=nova.scheduler.filters.all_filters
enabled_filters=RetryFilter,AvailabilityZoneFilter,RamFilter,DiskFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,CoreFilter
use_baremetal_filters=False
weight_classes=nova.scheduler.weights.all_weighers

[glance]
api_servers=controller.example.com:9292

[keystone_authtoken]
auth_uri=http://controller.example.com:5000/
auth_type=password
auth_url=http://controller.example.com:35357
username=nova
password=rootroot
project_name=services

[libvirt]
virt_type=qemu
inject_password=False
inject_key=False
inject_partition=-1
live_migration_uri=qemu+ssh://nova_migration@%s/system?keyfile=/etc/nova/migration/identity
cpu_mode=none
vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver

[notifications]
notify_api_faults=False

[oslo_concurrency]
lock_path=/var/lib/nova/tmp

[oslo_messaging_rabbit]
ssl=False

[oslo_policy]
policy_file=/etc/nova/policy.json

[placement]
os_region_name=RegionOne
auth_type=password
auth_url=http://controller.example.com:35357/v3
project_name=services
project_domain_name=Default
username=placement
user_domain_name=Default
password=rootroot

[scheduler]
host_manager=host_manager
driver=filter_scheduler
max_attempts=3

[vnc]
enabled=True
keymap=en-us
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=$my_ip
novncproxy_base_url=http://$my_ip:6080/vnc_auto.html
novncproxy_host=0.0.0.0
novncproxy_port=6080

[wsgi]
api_paste_config=api-paste.ini

[placement_database]
connection=mysql+pymysql://nova_placement:rootroot@controller.example.com/nova_placement
```

**Ensure you changed the my_ip variable with your local IP:**

```bash
[DEFAULT]
my_ip = 172.31.27.1
...
```

Populate the Compute databases:

```bash
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
su -s /bin/sh -c "nova-manage db sync" nova
```

Configure Nova Placement:

```bash
cd /etc/httpd/conf.d
vi 00-nova-placement-api.conf
Listen 8778

<VirtualHost *:8778>
  WSGIProcessGroup nova-placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
  WSGIDaemonProcess nova-placement-api processes=3 threads=1 user=nova group=nova
  WSGIScriptAlias / /usr/bin/nova-placement-api

  <IfVersion >= 2.4>
    ErrorLogFormat "%M"
  </IfVersion>
  ErrorLog /var/log/nova/nova-placement-api.log
  #SSLEngine On
  #SSLCertificateFile ...
  #SSLCertificateKeyFile ...
<Directory /usr/bin>
      Require all granted
</Directory>
</VirtualHost>

Alias /nova-placement-api /usr/bin/nova-placement-api
<Location /nova-placement-api>
  SetHandler wsgi-script
  Options +ExecCGI
  WSGIProcessGroup nova-placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
</Location>
```

Change the owner of the nova placement api:

```bash
chown nova:nova /usr/bin/nova-placement-api
chown nova:nova /var/log/nova/nova-placement-api.log
```

Restart apache:

```bash
systemctl restart httpd.service
```

Verify nova cell0 and cell1 are registered correctly:

```bash
nova-manage cell_v2 list_cells
```

Discover Compute Host:

```bash
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

Start the Compute services and configure them to start when the system boots:

```bash
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl status openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
```

Start the Compute service including its dependencies and configure them to start automatically when the system boots:

```bash
systemctl enable libvirtd.service 
systemctl enable openstack-nova-compute.service
systemctl restart libvirtd.service 
systemctl restart openstack-nova-compute.service
systemctl status libvirtd.service 
systemctl status openstack-nova-compute.service
```

On the Controller Node Verify operation of the Compute service:

```bash
openstack compute service list
+----+------------------+------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host       | Zone     | Status  | State | Updated At                 |
+----+------------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-conductor   | controller | internal | enabled | up    | 2017-11-04T10:10:18.000000 |
|  2 | nova-consoleauth | controller | internal | enabled | up    | 2017-11-04T10:10:17.000000 |
|  3 | nova-scheduler   | controller | internal | enabled | up    | 2017-11-04T10:10:17.000000 |
|  6 | nova-compute     | controller | nova     | enabled | up    | 2017-11-04T10:10:20.000000 |
+----+------------------+------------+----------+---------+-------+----------------------------+
```

## 2.3.1 Networking (neutron) service install and setup

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
openstack endpoint create --region RegionOne network public http://controller.example.com:9696
openstack endpoint create --region RegionOne network internal http://controller.example.com:9696
openstack endpoint create --region RegionOne network admin http://controller.example.com:9696
```

## 2.3.2 Configure Neutron the Networking Self-service networks

Install the components:

```bash
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-common -y
```

Configure the server component

Edit the /etc/neutron/neutron.conf file and replace with the following actions:

```bash
[DEFAULT]
bind_host=0.0.0.0
auth_strategy=keystone
core_plugin=neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins=router,metering
allow_overlapping_ips=True
notify_nova_on_port_status_changes=True
notify_nova_on_port_data_changes=True
api_workers=2
rpc_workers=2
router_scheduler_driver=neutron.scheduler.l3_agent_scheduler.ChanceScheduler
l3_ha=False
max_l3_agents_per_router=3
debug=False
log_dir=/var/log/neutron
transport_url=rabbit://guest:guest@controller.example.com:5672/
control_exchange=neutron

[agent]
root_helper=sudo neutron-rootwrap /etc/neutron/rootwrap.conf

[database]
connection=mysql+pymysql://neutron:rootroot@controller.example.com/neutron

[keystone_authtoken]
auth_uri=http://controller.example.com:5000/
auth_type=password
auth_url=http://controller.example.com:35357
username=neutron
password=rootroot
user_domain_name=Default
project_name=services
project_domain_name=Default

[nova]
region_name=RegionOne
auth_url=http://controller.example.com:35357
auth_type=password
password=rootroot
project_domain_id=default
project_domain_name=Default
project_name=services
tenant_name=services
user_domain_id=default
user_domain_name=Default
username=nova

[oslo_concurrency]
lock_path=$state_path/lock

[oslo_messaging_rabbit]
ssl=False

[oslo_policy]
policy_file=/etc/neutron/policy.json
```

Configure the Modular Layer 2 (ML2) plug-in:

Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file and complete the following actions:

```bash
[DEFAULT]

[ml2]
type_drivers=vxlan,flat
tenant_network_types=vxlan
mechanism_drivers=openvswitch
extension_drivers=port_security
path_mtu=0

[ml2_type_flat]
flat_networks=*

[ml2_type_vxlan]
vni_ranges=10:100
vxlan_group=224.0.0.1

[securitygroup]
firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group=True
```

Configure the Compute service to use the Networking service:

Edit the /etc/nova/nova.conf file and add our neutron service information:

```bash
[neutron]
url=http://controller.example.com:9696
region_name=RegionOne
ovs_bridge=br-int
default_floating_pool=nova
extension_sync_interval=600
service_metadata_proxy=True
metadata_proxy_shared_secret=a44139447afa46ae
timeout=60
auth_type=v3password
auth_url=http://controller.example.com:35357/v3
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

Configure OpenVSwitch:

```bash
yum install openvswitch -y
yum install openstack-neutron-openvswitch -y
```

Configure the OpenVSwitch agent in /etc/neutron/plugins/ml2/openvswitch_agent.ini:

```bash
[DEFAULT]

[agent]
tunnel_types=vxlan
vxlan_udp_port=4789
l2_population=False
drop_flows_on_start=False

[ovs]
integration_bridge=br-int
tunnel_bridge=br-tun
local_ip=172.31.52.18
bridge_mappings=extnet:br-ex

[securitygroup]
firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

**Ensure to set the local_ip to your Node IP:**

```
[ovs]
...
local_ip=172.31.52.18
...
```

Restart all services:

```bash
systemctl enable openvswitch.service
systemctl enable neutron-openvswitch-agent.service
systemctl enable neutron-ovs-cleanup.service
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
c7b7a559-9c84-42a0-b742-02f894b7a039
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-int
        Controller "tcp:127.0.0.1:6633"
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "2.7.2"
```

Configure l3-agent by replacing the /etc/neutron/l3_agent.ini file with:

```bash
[DEFAULT]
interface_driver=neutron.agent.linux.interface.OVSInterfaceDriver
agent_mode=legacy
debug=False
```

Configure the OpenVSwitch br-ex Bridge

```bash
systemctl restart openvswitch.service
ovs-vsctl --may-exist add-br br-ex
```

Configure dhcp-agent by replacing the /etc/neutron/dhcp_agent.ini file with:

```bash
[DEFAULT]
interface_driver=neutron.agent.linux.interface.OVSInterfaceDriver
resync_interval=30
enable_isolated_metadata=False
enable_metadata_network=False
debug=False
state_path=/var/lib/neutron
root_helper=sudo neutron-rootwrap /etc/neutron/rootwrap.conf
```

Configure the metadata-agent by editing the /etc/neutron/metadata_agent.ini file:

```bash
[DEFAULT]
metadata_proxy_shared_secret=a44139447afa46ae
metadata_workers=2
debug=False
nova_metadata_ip=172.31.52.18
```

**Ensure you update the nova_metadata_ip with your local IPs:**

```bash
[DEFAULT]
...
nova_metadata_ip=172.31.52.18
```

Enable the services:

```bash
systemctl enable openstack-nova-compute.service
systemctl enable libvirtd.service openstack-nova-compute.service

systemctl enable openvswitch.service
systemctl enable neutron-openvswitch-agent.service
systemctl enable neutron-ovs-cleanup.service
systemctl enable neutron-dhcp-agent.service
systemctl enable neutron-l3-agent.service
systemctl enable neutron-metadata-agent.service
```

Restart the services:

```bash
systemctl restart openstack-nova-compute.service
systemctl restart libvirtd.service openstack-nova-compute.service

systemctl restart openvswitch.service
systemctl restart neutron-openvswitch-agent.service
systemctl restart neutron-ovs-cleanup.service
systemctl restart neutron-dhcp-agent.service
systemctl restart neutron-l3-agent.service
systemctl restart neutron-metadata-agent.service
```

Verify all is running fine:

```bash
systemctl status openstack-nova-compute.service
systemctl status libvirtd.service openstack-nova-compute.service

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
c7b7a559-9c84-42a0-b742-02f894b7a039
    Manager "ptcp:6640:127.0.0.1"
        is_connected: true
    Bridge br-ex
        Controller "tcp:127.0.0.1:6633"
            is_connected: true
        fail_mode: secure
        Port br-ex
            Interface br-ex
                type: internal
        Port phy-br-ex
            Interface phy-br-ex
                type: patch
                options: {peer=int-br-ex}
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
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port int-br-ex
            Interface int-br-ex
                type: patch
                options: {peer=phy-br-ex}
    ovs_version: "2.7.2"
```

Ensure all network agent are running:

```bash
openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 09ce770b-8509-428d-88c5-5e24a7d2bbb0 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 48ac34bc-aa8d-49e3-b2f5-ab6dc8d95f16 | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| ec7c1424-41e1-4713-9881-42be0ab779de | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| f0bdd034-5546-48d1-a422-ea1682f74f6b | Open vSwitch agent | controller | None              | :-)   | UP    | neutron-openvswitch-agent |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

Configure the br-ex interface:

```bash
cd /etc/sysconfig/network-scripts/
```

```bash
vi ifcfg-br-ex

ONBOOT=yes
USERCTL=no
DEVICE=br-ex
NAME=br-ex
DEVICETYPE=ovs
IPADDR=172.31.54.21
NETMASK=255.255.255.0
GATEWAY=172.31.54.1
TYPE=OVSBridge
OVSDHCPINTERFACES=eth0
OVS_EXTRA="set bridge br-ex other-config:hwaddr=12:48:64:6e:a6:c6 fail_mode=standalone"
```

Ensure you replaced the IP address with eth0 IP.

```bash
IPADDR=172.31.54.21
NETMASK=255.255.255.0
GATEWAY=172.31.54.1
```

As well as the Mac address:

```bash
OVS_EXTRA="set bridge br-ex other-config:hwaddr=12:2e:14:32:c6:3e fail_mode=standalone"
```

Restart the network:

```bash
systemctl restart network
```

Verify br-ex is bridge to eth0:

```bash
ip addr list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP qlen 1000
    link/ether 12:2e:14:32:c6:3e brd ff:ff:ff:ff:ff:ff
    inet 172.31.52.18/20 brd 172.31.63.255 scope global dynamic eth0
       valid_lft 3586sec preferred_lft 3586sec
    inet6 fe80::102e:14ff:fe32:c63e/64 scope link
       valid_lft forever preferred_lft forever
3: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether d6:70:96:99:b5:e8 brd ff:ff:ff:ff:ff:ff
4: br-tun: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether ea:66:13:c6:76:4a brd ff:ff:ff:ff:ff:ff
5: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether ba:74:52:f3:ae:42 brd ff:ff:ff:ff:ff:ff
6: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN qlen 1000
    link/ether 12:2e:14:32:c6:3e brd ff:ff:ff:ff:ff:ff
    inet 172.31.52.18/24 brd 172.31.52.255 scope global br-ex
       valid_lft forever preferred_lft forever
    inet6 fe80::102e:14ff:fe32:c63e/64 scope link
       valid_lft forever preferred_lft forever
```

Ensure all services and agents are running fine:

```bash
. keystonerc_admin
openstack compute service list
+----+------------------+------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host       | Zone     | Status  | State | Updated At                 |
+----+------------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-conductor   | controller | internal | enabled | up    | 2017-11-04T11:09:50.000000 |
|  2 | nova-consoleauth | controller | internal | enabled | up    | 2017-11-04T11:09:40.000000 |
|  3 | nova-scheduler   | controller | internal | enabled | up    | 2017-11-04T11:09:41.000000 |
|  6 | nova-compute     | controller | nova     | enabled | up    | 2017-11-04T11:09:46.000000 |
+----+------------------+------------+----------+---------+-------+----------------------------+

openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 09ce770b-8509-428d-88c5-5e24a7d2bbb0 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 48ac34bc-aa8d-49e3-b2f5-ab6dc8d95f16 | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| ec7c1424-41e1-4713-9881-42be0ab779de | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| f0bdd034-5546-48d1-a422-ea1682f74f6b | Open vSwitch agent | controller | None              | :-)   | UP    | neutron-openvswitch-agent |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+

openstack hypervisor list
+----+-----------------------------+-----------------+--------------+-------+
| ID | Hypervisor Hostname         | Hypervisor Type | Host IP      | State |
+----+-----------------------------+-----------------+--------------+-------+
|  2 | ip-172-31-27-1.ec2.internal | QEMU            | 172.31.27.1  | up    |
+----+-----------------------------+-----------------+--------------+-------+
```

## 2.7 Dashboard install and configure

Install the packages:

```bash
yum install openstack-dashboard -y
```

Edit the /etc/openstack-dashboard/local_settings file and replace with the following actions:

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
SECRET_KEY = '409cc64b19ad44aea45fcef49746fabe'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',


        'LOCATION': '127.0.0.1:11211',


    }
}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
OPENSTACK_KEYSTONE_URL = "http://controller.example.com:5000/v3"
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
OPENSTACK_HEAT_STACK = {
    'enable_user_pass': True
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
HORIZON_CONFIG["disallow_iframe_embed"] = True
```

Remove the openstack-dashboard.conf file:

```
cd /etc/httpd/conf.d
rm openstack-dashboard.conf
```

Configure the VirtualHost:

```bash
vi 15-horizon_vhost.conf

<VirtualHost *:80>
  ServerName 34.238.85.43

  DocumentRoot "/var/www/"

  Alias /dashboard/static "/usr/share/openstack-dashboard/static"

  <Directory "/var/www/">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride None
    Require all granted
  </Directory>

  <Directory /usr/share/openstack-dashboard/openstack_dashboard/wsgi>
    Options All
    AllowOverride All
    Require all granted
  </Directory>

  <Directory /usr/share/openstack-dashboard/static>
    Options All
    AllowOverride All
    Require all granted
  </Directory>

  ErrorLog "/var/log/httpd/horizon_error.log"
  ServerSignature Off
  CustomLog "/var/log/httpd/horizon_access.log" combined

  RedirectMatch permanent  ^/$ /dashboard

  WSGIApplicationGroup %{GLOBAL}
  WSGIDaemonProcess apache group=apache processes=3 threads=10 user=apache
  WSGIProcessGroup apache
  WSGIScriptAlias /dashboard "/usr/share/openstack-dashboard/openstack_dashboard/wsgi/django.wsgi"
</VirtualHost>
```

Ensure to set your Public IP address:

```bash
<VirtualHost *:80>
  ServerName 34.238.85.43
  ...
```

Restart Apache:

```bash
systemctl restart httpd.service
```

Set permission on Horizon log:

```bash
chown -R apache:apache /var/log/horizon
```

## 2.8 Finalize the configuration of OpenStack

Create Flavors:

```bash
openstack flavor create --id 0 --ram 512   --vcpus 1 --disk 1  m1.tiny
openstack flavor create --id 1 --ram 1024  --vcpus 1 --disk 1  m1.small
openstack flavor create --id 2 --ram 2048  --vcpus 1 --disk 1  m1.large
```
## 2.8.1 Do the network Configuration:

Create Public Floating Network (All Tenants)

This is the virtual network that OpenStack will bridge to the outside world. You will assign public IPs to your instances from this network.

```bash
. keystonerc_admin
neutron net-create public --router:external=True --provider:network_type=vxlan --provider:segmentation_id=96
neutron subnet-create --name public_subnet --disable-dhcp --allocation-pool start=192.168.178.100,end=192.168.178.150 public 192.168.178.0/24
```

Ensure to update the IPs for the allocation-pool and netmask with your local IPs.

Setup Tenant Network/Subnet

This is the private network your instances will attach to. Instances will be issued IPs from this private IP subnet.

```bash
. keystonerc_demo
neutron net-create private
neutron subnet-create --name private_subnet --dns-nameserver 8.8.8.8 --dns-nameserver 8.8.4.4 --allocation-pool start=10.0.30.10,end=10.0.30.254 private 10.0.30.0/24
```

Create an External Router to Attach to floating IP Network

This router will attach to your private subnet and route to the public network, which is where your floating IPs are located.

```bash
neutron router-create extrouter
neutron router-gateway-set extrouter public
neutron router-interface-add extrouter private_subnet
```

Restart all the services:

Ensure all services are enabled:

```bash
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl enable neutron-server.service
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl enable httpd.service memcached.service
systemctl enable rabbitmq-server.service
systemctl enable openstack-losetup.service
systemctl enable openstack-cinder-api.service
systemctl enable openstack-cinder-scheduler.service 
systemctl enable openstack-cinder-volume.service 
systemctl enable openstack-cinder-backup.service
```

Restart the services:

```bash
systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl restart neutron-server.service
systemctl restart openstack-glance-api.service openstack-glance-registry.service
systemctl restart httpd.service memcached.service
systemctl restart rabbitmq-server.service
systemctl restart openstack-losetup.service
systemctl restart openstack-cinder-api.service
systemctl restart openstack-cinder-scheduler.service 
systemctl restart openstack-cinder-volume.service 
systemctl restart openstack-cinder-backup.service
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
systemctl status openstack-losetup.service
systemctl status openstack-cinder-api.service
systemctl status openstack-cinder-scheduler.service 
systemctl status openstack-cinder-volume.service 
systemctl status openstack-cinder-backup.service
```

**Neutron:**

Ensure the services are enabled:

```bash
systemctl enable openvswitch.service
systemctl enable neutron-openvswitch-agent.service
systemctl enable neutron-ovs-cleanup.service
systemctl enable neutron-dhcp-agent.service
systemctl enable neutron-l3-agent.service
systemctl enable neutron-metadata-agent.service
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

**Nova Compute:**

Ensure the services are enabled:

```bash
systemctl enable openstack-nova-compute.service
systemctl enable libvirtd.service openstack-nova-compute.service
```

Restart the services:

```bash
systemctl restart openstack-nova-compute.service
systemctl restart libvirtd.service openstack-nova-compute.service
```

Verify all is running fine:

```bash
systemctl status openstack-nova-compute.service
systemctl status libvirtd.service openstack-nova-compute.service
```

Checks on the controller:

```bash
openstack compute service list
+----+------------------+------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host       | Zone     | Status  | State | Updated At                 |
+----+------------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-consoleauth | controller | internal | enabled | up    | 2017-10-26T17:21:04.000000 |
|  2 | nova-conductor   | controller | internal | enabled | up    | 2017-10-26T17:21:09.000000 |
|  3 | nova-scheduler   | controller | internal | enabled | up    | 2017-10-26T17:21:04.000000 |
|  6 | nova-compute     | compute    | nova     | enabled | up    | 2017-10-26T17:21:03.000000 |
+----+------------------+------------+----------+---------+-------+----------------------------+

openstack network agent list
+---------------------+--------------------+---------+-------------------+-------+-------+-----------------------+
| ID                  | Agent Type         | Host    | Availability Zone | Alive | State | Binary                |
+---------------------+--------------------+---------+-------------------+-------+-------+-----------------------+
| 002a66db-4c2a-4b7f- | DHCP agent         | network | nova              | True  | UP    | neutron-dhcp-agent    |
| be0c-3fafa02b0605   |                    |         |                   |       |       |                       |
| 1f516dea-104f-43b9  | Open vSwitch agent | compute | None              | True  | UP    | neutron-openvswitch-  |
| -bc7c-8e1611ffd7a5  |                    |         |                   |       |       | agent                 |
| 38b2b976-c4bb-4b48- | Open vSwitch agent | network | None              | True  | UP    | neutron-openvswitch-  |
| 8a33-232de0608e0d   |                    |         |                   |       |       | agent                 |
| 67807e3d-3833-4a67- | L3 agent           | network | nova              | True  | UP    | neutron-l3-agent      |
| 9086-a7cb23f81f00   |                    |         |                   |       |       |                       |
| 7d2e2b57-eda7-470e- | Metadata agent     | network | None              | True  | UP    | neutron-metadata-     |
| a6e4-020a55600d3a   |                    |         |                   |       |       | agent                 |
+---------------------+--------------------+---------+-------------------+-------+-------+-----------------------+

openstack hypervisor list
+----+-----------------------------+-----------------+--------------+-------+
| ID | Hypervisor Hostname         | Hypervisor Type | Host IP      | State |
+----+-----------------------------+-----------------+--------------+-------+
|  2 | ip-172-31-27-1.ec2.internal | QEMU            | 172.31.27.1  | up    |
+----+-----------------------------+-----------------+--------------+-------+
```

Networking:

```bash
openstack network list --external
+--------------------------------------+--------+--------------------------------------+
| ID                                   | Name   | Subnets                              |
+--------------------------------------+--------+--------------------------------------+
| 13eaf423-1901-4176-a184-e69a48f87586 | public | 89203467-67c0-42e4-ba55-f64387ea5ad4 |
+--------------------------------------+--------+--------------------------------------+

openstack network list
+--------------------------------------+---------+--------------------------------------+
| ID                                   | Name    | Subnets                              |
+--------------------------------------+---------+--------------------------------------+
| 13eaf423-1901-4176-a184-e69a48f87586 | public  | 89203467-67c0-42e4-ba55-f64387ea5ad4 |
| d1617aa6-1645-4a11-bbfe-dbf0e299f6c7 | private | f11e2036-7191-492d-a677-89a1ed647185 |
+--------------------------------------+---------+--------------------------------------+

openstack router list
+--------------------------------------+-----------+--------+-------+-------------+-------+----------------------------------+
| ID                                   | Name      | Status | State | Distributed | HA    | Project                          |
+--------------------------------------+-----------+--------+-------+-------------+-------+----------------------------------+
| 1c531ca9-3fbc-4e6a-94c0-1d61ccda3cd6 | extrouter | ACTIVE | UP    | False       | False | 133d12787b254e4e8422c1db84700c65 |
+--------------------------------------+-----------+--------+-------+-------------+-------+----------------------------------+

openstack router show extrouter
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field                   | Value                                                                                                                                                                                     |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| admin_state_up          | UP                                                                                                                                                                                        |
| availability_zone_hints |                                                                                                                                                                                           |
| availability_zones      | nova                                                                                                                                                                                      |
| created_at              | 2017-11-04T22:31:07Z                                                                                                                                                                      |
| description             |                                                                                                                                                                                           |
| distributed             | False                                                                                                                                                                                     |
| external_gateway_info   | {"network_id": "849eb0f2-4a9f-4127-b7d9-0a01e6759e35", "enable_snat": true, "external_fixed_ips": [{"subnet_id": "7a7b25e5-1a02-42d9-ab73-a454a5abc9fa", "ip_address": "172.31.18.104"}]} |
| flavor_id               | None                                                                                                                                                                                      |
| ha                      | False                                                                                                                                                                                     |
| id                      | 1c531ca9-3fbc-4e6a-94c0-1d61ccda3cd6                                                                                                                                                      |
| name                    | extrouter                                                                                                                                                                                 |
| project_id              | 133d12787b254e4e8422c1db84700c65                                                                                                                                                          |
| revision_number         | 3                                                                                                                                                                                         |
| routes                  |                                                                                                                                                                                           |
| status                  | ACTIVE                                                                                                                                                                                    |
| tags                    |                                                                                                                                                                                           |
| updated_at              | 2017-11-04T22:31:14Z                                                                                                                                                                      |
+-------------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Block Storage:

```bash
openstack volume service list
+------------------+----------------+------+---------+-------+----------------------------+
| Binary           | Host           | Zone | Status  | State | Updated At                 |
+------------------+----------------+------+---------+-------+----------------------------+
| cinder-backup    | controller     | nova | enabled | up    | 2017-10-28T13:12:12.000000 |
| cinder-scheduler | controller     | nova | enabled | up    | 2017-10-28T13:12:53.000000 |
| cinder-volume    | controller@lvm | nova | enabled | up    | 2017-10-28T13:12:49.000000 |
+------------------+----------------+------+---------+-------+----------------------------+
```

Nova Status:

```bash
nova-status upgrade check
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


