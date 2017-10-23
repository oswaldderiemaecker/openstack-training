I am using the Mac for installation and have the VirtualBox installed.

For the Newton openstack setup we must have the three virtual machine ready with atleast below requirement:
  
* Controller Node: 2 processor, 4 GB memory, and 5 GB storage
* Compute Node: 1 processor, 2 GB memory, and 10 GB storage
* Network Node: 1 processor, 2 GB memory, and 5 GB storage

Each VM has a NAT Network and a Host-Only Adapter set to the same Adapter.

For simplicity we will use the password rootroot for all passwords.

# 1) Installation of base system on the VMs

Get CentOs 7(https://www.centos.org/download/) and install it, configure the network and base settings to suite your
configuration.

# 2) Configure Hosts and the Hostname

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

# 3) Upgrade the OS and reboot:

```bash
yum update -y ; reboot
```

# 4) Verify connectivity

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

# 5) Create SSH Access:

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

# 6) Network Time Protocol (NTP) Setup

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

# 7) Set OpenStack Newton Repository

```bash
yum install centos-release-openstack-newton -y
yum update -y
yum install python-openstackclient
yum install openstack-selinux
```

# 8) Install MariaDB

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

# 9) RabbitMQ message queue Setup

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

# 10) Memcached setup

```bash
yum install memcached python-memcached
```

Start the Memcached service and configure it to start when the system boots:

```bash
systemctl enable memcached.service
systemctl start memcached.service
```

# 11) Installing Keystone

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

# 12) Create a domain, projects, users, and roles

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

# 13) Verify operation of the Identity service

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

# 14) Image (glance) service install and configure

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

# 15) Compute (nova) service install and configure on Controller node

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

# 16) Compute (nova) service install and configure on Compute Node

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

# 17) Networking (neutron) service install and setup

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

# 18) Configure the Networking Self-service networks

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
api_workers=2
rpc_workers=2
router_scheduler_driver=neutron.scheduler.l3_agent_scheduler.ChanceScheduler
l3_ha=False
max_l3_agents_per_router=3
debug=False
log_dir=/var/log/neutron
rpc_backend=rabbit
control_exchange=neutron
#nova_url=http://192.168.57.102:8774/v2
transport_url = rabbit://openstack:rootroot@10.0.2.15

[agent]
root_helper=sudo neutron-rootwrap /etc/neutron/rootwrap.conf

[database]
connection=mysql+pymysql://neutron:rootroot@10.0.2.15/neutron

[keystone_authtoken]
auth_uri=http://controller:5000
auth_url=http://controller:35357
auth_type=password
project_name=services
username=neutron
project_domain_name=Default
user_domain_name=Default
password=rootroot

[nova]
region_name=RegionOne
auth_url=http://controller:35357
auth_type=password
password=rootroot
project_domain_id=default
project_name=services
tenant_name=services
user_domain_id=default
username=nova

[oslo_concurrency]
lock_path=$state_path/lock
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

Edit the /etc/nova/nova.conf file and perform the following actions:

[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = rootroot

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
systemctl enable neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```

# 19) Configure Neutron on the Compute Node

Install the components:

```bash
yum install openstack-neutron
yum install openvswitch
yum install openstack-neutron-openvswitch
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

Start the services:

```bash
systemctl restart openvswitch.service
systemctl restart neutron-openvswitch-agent.service
systemctl restart neutron-ovs-cleanup.service
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

On the Controller:

Create the base networks:

```bash
neutron net-create public
neutron subnet-create --name public_subnet public 172.24.4.224/28
neutron net-create private
neutron subnet-create --name private_subnet private 10.0.0.0/24
neutron router-create private-router
neutron router-interface-add private-router private_subnet
```

# 20) Dashboard install and configure

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

openstack flavor create --ram 1024 --vcpus 1 --disk 1 --public t1.tiny

systemctl restart neutron-server.service neutron-dhcp-agent.service neutron-metadata-agent.service neutron-l3-agent

# 21) Restart everything

On the Controller Node:

```bash
systemctl restart openstack-nova-api.service openstack-nova-consoleauth.service openstack-nova-scheduler.service openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl start neutron-server.service neutron-linuxbridge-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
systemctl restart httpd.service memcached.service
```

Verify all is running fine:

```bash
systemctl status openstack-nova-api.service
systemctl status openstack-nova-consoleauth.service
systemctl status openstack-nova-scheduler.service
systemctl status openstack-nova-conductor.service
systemctl status openstack-nova-novncproxy.service
systemctl status neutron-server.service
systemctl status neutron-dhcp-agent.service
systemctl status neutron-metadata-agent.service
systemctl status openstack-glance-api.service
systemctl status openstack-glance-registry.service
systemctl status httpd.service memcached.service
```

On the Compute Node:

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
