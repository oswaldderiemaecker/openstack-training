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
bind-address = 192.168.57.102
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
admin_token = 602c191552204c67bcc8e7d120b8b20d
debug = False
log_dir = /var/log/keystone
rpc_backend = rabbit
public_port=5000
admin_bind_host=0.0.0.0
public_bind_host=0.0.0.0
admin_port=35357

[catalog]
template_file = /etc/keystone/default_catalog.templates
driver = sql

[credential]
key_repository = /etc/keystone/credential-keys

[database]
connection = mysql+pymysql://keystone_admin:rootroot@10.0.2.15/keystone

[eventlet_server]
public_workers=2
admin_workers=2

[fernet_tokens]
key_repository = /etc/keystone/fernet-keys

[token]
expiration = 3600
provider = keystone.token.providers.uuid.Provider
driver = sql
revoke_by_id = True

[ssl]
enable=False
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
keystone-manage bootstrap --bootstrap-password rootroot –bootstrap-admin-url http://controller:35357/v3/ \
                          --bootstrap-internal-url http://controller:35357/v3/ \
                          --bootstrap-public-url http://controller:5000/v3/ \
                          --bootstrap-region-id RegionOne
```

Configure the Apache HTTP server:

Copy the ./keystone/httpd/conf/httpd.conf to /etc/httpd/conf/httpd.conf file.

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
export OS_AUTH_URL=http://192.168.57.102:5000/v2.0
export PS1='[\u@\h \W(keystone_admin)]\$ '

export OS_TENANT_NAME=admin
export OS_REGION_NAME=RegionOne
```

11) Create a domain, projects, users, and roles

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
