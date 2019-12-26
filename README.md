# OpenStack Installation Tutorial for Ubuntu
## With Open vSwitch Networking
## A Supplementary Guide for Release Newton

Installation of the [OpenStack](https://www.openstack.org/) requires an understanding of the basic networking concepts and ability to configure each component of the OpenStack.
Unfortunately the official [OpenStack Installation Tutorial for Ubuntu](https://docs.openstack.org/newton/install-guide-ubuntu/) by OpenStack is missing important parts to configure many [components](https://en.wikipedia.org/wiki/OpenStack#Components).
Furthermore, some default settings are actually poor choices and leave a lot to be desired.

&nbsp;

## Aim

The aim is to install the **OpenStack Newton** release on three [bare metal servers](https://en.wikipedia.org/wiki/Bare-metal_server) running [Ubuntu 16.04.6 LTS (Xenial Xerus)](http://releases.ubuntu.com/16.04/).

The compute nodes are to be configured for [LVM](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)) for later use as Storage nodes. On each compute node at least 50% of the HDD space must be reserved beforehand for the LVM volume group **cinder-volumes**, see [Install and configure a storage node](https://docs.openstack.org/newton/install-guide-ubuntu/cinder-storage-install.html).

The OpenStack networking component, **Neutron**, can be installed either using [Linux Bridges](https://wiki.linuxfoundation.org/networking/bridge), which is the default, or [Open vSwitch](http://openvswitch.org/). For improved capability here Neutron will be installed on [Open vSwitch Self-service networks](https://docs.openstack.org/newton/networking-guide/deploy-ovs-selfservice.html).

Each server has three network interfaces and can be accessed via [Port Address Translation (PAT)](https://en.wikipedi0.org/wiki/Network_address_translation). For example: to the controller - **ssh west@192.168.115.44 -p 2203**. On the controller's public network there is no DHCP.

The following diagram highlights the current setup with initially assigned IP addresses:

```


    +=============================+
    |                             |
    |          COMPUTE-1          |
    |      public: 10.10.10.101   +-------+
    |     private: 10.10.11.101   +-----+ |
    |  management: 10.10.12.101   +---+ | |
    |     LVM - cinder-volumes    |   | | |
    |                             |   | | |
    +=============================+   | | |          +===+            +=============================+
                                      | | +----------+   |            |                             |
                                      | +------------+ s |            |          CONTROLLER         |
                                      +--------------+ w +------------+      public: 10.10.10.103   |
                                                     | i +------------+     private: 10.10.11.103   |
                                                     | t +------------+  management: 10.10.12.103   |
                                      +--------------+ c |            |      MariaDB + RabbitMQ     |
                                      | +------------+ h |            |         Open vSwitch        |
                                      | | +----------+   |            |                             |
    +=============================+   | | |          +===+            +=============================+
    |                             |   | | |
    |          COMPUTE-2          |   | | |
    |      public: 10.10.10.102   +---+ | |
    |     private: 10.10.11.102   +-----+ |
    |  management: 10.10.12.102   +-------+
    |     LVM - cinder-volumes    |
    |                             |
    +=============================+


```

&nbsp;

## Prerequisites

The version of the [Ubuntu](https://en.wikipedia.org/wiki/Ubuntu_version_history) is important. Experience with **Ubuntu 16.10 Yakkety Yak** empirically proved that a [Long-Term Support (LTS)](https://en.wikipedia.org/wiki/Long_Term_Support) version is required, so the final choice fell on **Ubuntu 16.04 LTS Xenial Xerus**.

[OpenStack releases](https://releases.openstack.org/) follow the alphabetical order. In mid-2017 **Newton** release is seen as mature and reliable, and easy to install required components using Ubuntu repositories.

&nbsp;

## Supplementary Guide

The offical [installation guide](https://docs.openstack.org/newton/install-guide-ubuntu/) gives a brief explanation for each required step.

On this guide the networking is the last installation step, and it installs the alternative Open vSwitch networking alternative.

This guide is meant to be read together with the official guide. It will hopefully help to configure the unexplained configurations.

For each step, the original and modified config files are copied into a directory. See tarball [config_files.tar](./config_files.tar). This tarball consists of three tarball files, and each tarball extracts files into the **etc/** and **usr/** directories.

On each step the official guide asks for:
* some Ubuntu installations of OpenStack components and dependencies via **apt install**,
* DB creation and setting user privileges,
* issuing some OpenStack commands,
* restarting some Ubuntu services.

To ease the burden of installation, whenever a password is asked it is in each and every case **ost2017**, ost is OpenStack Test.

It is important that for later referral, each config file has been copied first with extension **.orig**.

For example:

```
root@ost-ctrl:~# cp -a /etc/glance/glance-api.conf /etc/glance/glance-api.conf.orig
root@ost-ctrl:~# ls /etc/glance/glance-api.conf*
/etc/glance/glance-api.conf  /etc/glance/glance-api.conf.orig
```

&nbsp;

## Environment

### Controller node

#### Configure network interfaces

Preferably the hostname hints on the intended usage, therefore the config file **/etc/hostname** on the controller server has to be changed:

```
$ cat /etc/hostname
ost-ctrl
```

It is important that there is on the **/etc/network/interfaces** file only one gateway configured for the interfaces.

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

auto eno2
iface eno2 inet static
    address 10.10.10.103
    netmask 255.255.255.0
    gateway 10.10.10.1
    dns-nameservers 8.8.8.8

auto eno3
iface eno3 inet static
    address 10.10.11.103
    netmask 255.255.255.0
#   gateway 10.10.11.1

auto eno4
iface eno4 inet static
    address 10.10.12.103
    netmask 255.255.255.0
#   gateway 10.10.12.1
```

#### Configure name resolution

The **/etc/hosts** file must have a mapping for the hostname and all the three servers:

```
127.0.0.1   localhost
127.0.1.1   ost-ctrl

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

10.10.12.101    compute1
10.10.12.102    compute2
10.10.12.103    controller
```

#### Network Time Protocol (NTP)

Copy the config file **/etc/chrony/chrony.conf**:

```
root@ost-ctrl:~# cp -a /etc/chrony/chrony.conf /etc/chrony/chrony.conf.orig
```

Edit: **/etc/chrony/chrony.conf**

```
#pool 2.debian.pool.ntp.org offline iburst
server 0.tr.pool.ntp.org iburst
server 1.tr.pool.ntp.org iburst
server 2.tr.pool.ntp.org iburst
server 3.tr.pool.ntp.org iburst

allow 10.10.12.0/24
```

#### RabbitMQ

[RabbitMQ](https://www.rabbitmq.com/install-debian.html) can be installed using its own APT repository:

```
root@ost-ctrl:~# echo 'deb http://www.rabbitmq.com/debian/ testing main' |
     sudo tee /etc/apt/sources.list.d/rabbitmq.list

root@ost-ctrl:~# wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc |
     sudo apt-key add -

root@ost-ctrl:~# apt update

root@ost-ctrl:~# rabbitmq-plugins enable --offline rabbitmq_management

root@ost-ctrl:~# service rabbitmq-server restart

root@ost-ctrl:~# rabbitmqctl change_password guest ost2017
root@ost-ctrl:~# rabbitmqctl add_user openstack ost2017
root@ost-ctrl:~# rabbitmqctl set_user_tags openstack administrator
root@ost-ctrl:~# rabbitmqctl set_permissions -p / openstack ".*" ".*" ".*"

root@ost-ctrl:~# service rabbitmq-server status
```

Rabbitmq Management plugin is enabled, so that the **openstack** user can login to the http://CONTROLLER-IP-ADDRESS:15672/ admin page.

#### MariaDB

A regular MariaDB version as found in the Ubuntu repositories is sufficient.
If a newer version is wanted, you can check the [MariaDB download](https://downloads.mariadb.org/mariadb/repositories/) pages.

```
root@ost-ctrl:~# apt update

root@ost-ctrl:~# apt install mariadb

root@ost-ctrl:~# service mysql status
```

A new config file, **/etc/mysql/mariadb.conf.d/99-openstack.cnf**, has to be created:

```
[mysqld]
bind-address = 10.10.12.103

default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

You can secure the DB via the built-in app, but omitting strong passwords makes life easier:

```
root@ost-ctrl:~# mysql_secure_installation
```

You can also set a root password the old way:

```
root@ost-ctrl:~# mysqladmin -u root password 'ost2017'
```

A typical MariaDB session is given below:

```
root@ost-ctrl:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 12027
Server version: 10.0.31-MariaDB-0ubuntu0.16.04.2 Ubuntu 16.04

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>  CREATE DATABASE keystone;
MariaDB [(none)]>  GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'ost2017';
MariaDB [(none)]>  GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'ost2017';
MariaDB [(none)]>  FLUSH PRIVILEGES;
MariaDB [(none)]>  EXIT
```

#### Memcached

Copy the config file **/etc/memcached.conf**:

```
root@ost-ctrl:~# cp -a /etc/memcached.conf /etc/memcached.conf.orig
```

Edit: **/etc/chrony/chrony.conf**

```
# Specify which IP address to listen on. The default is to listen on all IP addresses
# This parameter is one of the only security measures that memcached has, so make sure
# it's listening on a firewalled interface.
-l 10.10.12.103
```

&nbsp;

### Compute nodes

#### Configure network interfaces

Preferably the hostname hints on the intended usage, therefore the config file **/etc/hostname** has to be changed on each compute node.

Compute-1:

```
$ cat /etc/hostname
ost-comp-1
```

Compute-2:

```
$ cat /etc/hostname
ost-comp-2
```

It is important that there is on the **/etc/network/interfaces** file only one gateway configured for the interfaces.

Compute-1:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback


auto eno1
iface eno1 inet static
        address 10.10.10.101
        netmask 255.255.255.0
        gateway 10.10.10.1
#   network 10.10.10.0
#   broadcast 10.10.10.255
        dns-nameservers 8.8.8.8

auto eno3
iface eno3 inet static
        address 10.10.11.101
        netmask 255.255.255.0
#       gateway 10.10.11.1

auto eno4
iface eno4 inet static
    address 10.10.12.101
    netmask 255.255.255.0
#   gateway 10.10.12.1
```

Compute-2:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback


auto eno1
iface eno1 inet static
    address 10.10.10.102
    netmask 255.255.255.0
    gateway 10.10.10.1
    dns-nameservers 8.8.8.8

auto eno3
iface eno3 inet static
    address 10.10.11.102
    netmask 255.255.255.0
#   gateway 10.10.11.1

auto eno4
iface eno4 inet static
    address 10.10.12.102
    netmask 255.255.255.0
#   gateway 10.10.12.1
```

&nbsp;

#### Configure name resolution

The **/etc/hosts** file must have a mapping for the hostname and for all the three servers:

Compute-1:

```
127.0.0.1   localhost
127.0.1.1   ost-comp-1

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

10.10.12.101    compute1
10.10.12.102    compute2
10.10.12.103    controller
```

Compute-2:

```
127.0.0.1   localhost
127.0.1.1   ost-comp-2

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

10.10.12.101    compute1
10.10.12.102    compute2
10.10.12.103    controller
```

&nbsp;

### Network Time Protocol (NTP)

Copy the config file **/etc/chrony/chrony.conf** on each compute node.

Compute-1:

```
root@ost-comp-1:~# cp -a /etc/chrony/chrony.conf /etc/chrony/chrony.conf.orig
```

Edit: **/etc/chrony/chrony.conf**

```
#pool 2.debian.pool.ntp.org offline iburst
server controller iburst
```

Compute-2 has an identical configuration.

&nbsp;

## KEYSTONE

Copy the config file **/etc/apache2/apache2.conf**:

```
root@ost-ctrl:~# cp -a /etc/apache2/apache2.conf /etc/apache2/apache2.conf.orig
```

Edit: **/etc/apache2/apache2.conf**

```
#ServerRoot "/etc/apache2"
ServerName controller
```

Copy the config file **/etc/keystone/keystone.conf**:

```
root@ost-ctrl:~# cp -a /etc/keystone/keystone.conf /etc/keystone/keystone.conf.orig
```

Edit: **/etc/keystone/keystone.conf**

```
[database]
connection = mysql+pymysql://keystone:ost2017@controller/keystone
```

Copy the config file **/etc/keystone/keystone-paste.ini**:

```
root@ost-ctrl:~# cp -a /etc/keystone/keystone-paste.ini /etc/keystone/keystone-paste.ini.orig
```

Edit: **/etc/keystone/keystone-paste.ini**

```
[pipeline:public_api]
# The last item in this pipeline must be public_service or an equivalent
# application. It cannot be a filter.
pipeline = cors sizelimit http_proxy_to_wsgi osprofiler url_normalize request_id build_auth_context token_auth json_body ec2_extension public_service

[pipeline:admin_api]
# The last item in this pipeline must be admin_service or an equivalent
# application. It cannot be a filter.
pipeline = cors sizelimit http_proxy_to_wsgi osprofiler url_normalize request_id build_auth_context token_auth json_body ec2_extension s3_extension admin_service

[pipeline:api_v3]
# The last item in this pipeline must be service_v3 or an equivalent
# application. It cannot be a filter.
pipeline = cors sizelimit http_proxy_to_wsgi osprofiler url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension service_v3
```

The following scripts, **admin-openrc** and **demo-openrc**, are used for authentication of the admin and demo projects on the controller:

admin-openrc:

```
root@ost-ctrl:~# cat /root/admin-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ost2017
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

demo-openrc:

```
root@ost-ctrl:~# cat /root/demo-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=ost2017
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

Sourcing the authentication script simplify usage of the OpenStack commands:

```
root@ost-ctrl:~# . admin-openrc
root@ost-ctrl:~#
root@ost-ctrl:~# openstack token issue
+------------+----------------------------------------------------------------------------------------+
| Field      | Value                                                                                  |
+------------+----------------------------------------------------------------------------------------+
| expires    | 2017-10-24 11:47:24+00:00                                                              |
| id         | gAAAAABZ7xo8E4NVW1RC41bMvTJwfvee5G--DyQDNaULuJQYGZfjoci-                               |
|            | JRnD7iMmWFPrRufYXVGkMmszL4_ma6bWbxVRmZlNxHltMec4qmJhk-F                                |
|            | -UmV9wtlUexxSunlPiCMtrb26QHqvsWYOCHfkwfavZE0FfLeC9XU3jRmIwL0S1vn-VWkW7O8               |
| project_id | fa83d80b26174ea3af31bde147f3d4ae                                                       |
| user_id    | 4087c392fdf44657a2e0dba0187fdd95                                                       |
+------------+----------------------------------------------------------------------------------------+
```

Identical scripts are also created on the compute nodes.

&nbsp;

## GLANCE

Copy the config file **/etc/glance/glance-api.conf**:

```
root@ost-ctrl:~# cp -a /etc/glance/glance-api.conf /etc/glance/glance-api.conf.orig
```

Edit: **/etc/glance/glance-api.conf**

```
[database]
connection = mysql+pymysql://glance:ost2017@controller/glance

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = ost2017

[paste_deploy]
flavor = keystone
```

Copy the config file **/etc/glance/glance-registry.conf**:

```
root@ost-ctrl:~# cp -a /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.orig
```

Edit: **/etc/glance/glance-registry.conf**

```
[database]
connection = mysql+pymysql://glance:ost2017@controller/glance

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = ost2017

[paste_deploy]
flavor = keystone
```

To verify the Glance service, or later the Neutron service, use of a simple Linux distribution, like [CirrOS](https://launchpad.net/cirros), is advised:

```
root@ost-ctrl:~# wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```

For daily use, one other Ubuntu image is to be downloaded, a standart Ubuntu 16.04 cloud image:

The images are to be installed either via CLI or the Dashboard into the controller.

```
root@ost-ctrl:~# wget https://cloud-images.ubuntu.com/xenial/20171011/xenial-server-cloudimg-amd64-disk1.img
```

&nbsp;

## NOVA

The OpenStack compute service has been named as **Nova**.
It is installed both on the controller and compute nodes.

On the controller, copy the config file **/etc/nova/nova.conf**:

```
root@ost-ctrl:~# cp -a /etc/nova/nova.conf /etc/nova/nova.conf.orig
```

Edit: **/etc/nova/nova.conf**

```
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
#log-dir=/var/log/nova
state_path=/var/lib/nova
force_dhcp_release=True
verbose=True
ec2_private_dns_show_ip=True
enabled_apis=osapi_compute,metadata
transport_url = rabbit://openstack:ost2017@controller
auth_strategy = keystone
my_ip = 10.10.12.103
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[database]
connection = mysql+pymysql://nova:ost2017@controller/nova

[api_database]
connection = mysql+pymysql://nova:ost2017@controller/nova_api

[libvirt]
use_virtio_for_bridges=True

[wsgi]
api_paste_config=/etc/nova/api-paste.ini

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = ost2017

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = ost2017
service_metadata_proxy = True
metadata_proxy_shared_secret = OST_2017

[cinder]
os_region_name = RegionOne
```

On compute nodes, nearly the same diff, with the proper **my_ip** parameter and no DB lines, will be applied to **/etc/nova/nova.conf**:

```
root@ost-comp-1:~# cp -a /etc/nova/nova.conf /etc/nova/nova.conf.orig
```

Edit: **/etc/nova/nova.conf**

```
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
#log-dir=/var/log/nova
state_path=/var/lib/nova
force_dhcp_release=True
verbose=True
ec2_private_dns_show_ip=True
enabled_apis=osapi_compute,metadata
transport_url = rabbit://openstack:ost2017@controller
auth_strategy = keystone
my_ip = 10.10.12.101
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[database]
#connection=sqlite:////var/lib/nova/nova.sqlite

[api_database]
#connection=sqlite:////var/lib/nova/nova.sqlite

[libvirt]
use_virtio_for_bridges=True

[wsgi]
api_paste_config=/etc/nova/api-paste.ini

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = ost2017

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://192.168.230.88:6083/vnc_auto.html

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = ost2017
service_metadata_proxy = True
metadata_proxy_shared_secret = OST_2017
```

&nbsp;

## HORIZON

The Dashboard, Horizon, is the standart Web interface to OpenStack.

The address of the current installation is [http://192.168.230.88:8083/horizon](http://192.168.230.88:8083/horizon).

&nbsp;

![login](./img/dashboard_login.png "Login page")

&nbsp;

Here, on the "Launch Instance" wizard the **Configuration Drive** option has been found as required to to able to set the public key for later SSH access. So it is enabled as the default choice.

&nbsp;

![config_drive](./img/dashboard_configuration_drive.png "Checkmark for the Configuration Drive")

&nbsp;

Copy the config file **/etc/openstack-dashboard/local_settings.py**:

&nbsp;

```
root@ost-comp-1:~# cp -a /etc/openstack-dashboard/local_settings.py /etc/openstack-dashboard/local_settings.py.orig
```

&nbsp;

Edit: **/etc/openstack-dashboard/local_settings.py**

```
#CACHES = {
#    'default': {
#        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
#        'LOCATION': '127.0.0.1:11211',
#    },
#}

OPENSTACK_HOST = "controller"
#OPENSTACK_KEYSTONE_URL = "http://%s:5000/v2.0" % OPENSTACK_HOST
#OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"

LAUNCH_INSTANCE_DEFAULTS = {
    'config_drive': True
#    'enable_scheduler_hints': True,
#    'disable_image': False,
#    'disable_instance_snapshot': False,
#    'disable_volume': False,
#    'disable_volume_snapshot': False
}

TIME_ZONE = "Europe/Istanbul"

AVAILABLE_THEMES = [
    ('default', 'Default', 'themes/default'),
    ('material', 'Material', 'themes/material'),
]

ALLOWED_HOSTS = ['*', ]

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```

The Ubuntu theme is the default one, although it is made available here it is disabled by the previous config.

Copy the config file **/usr/share/openstack-dashboard/openstack_dashboard/settings.py**:

```
root@ost-comp-1:~# cp /usr/share/openstack-dashboard/openstack_dashboard/settings.py /usr/share/openstack-dashboard/openstack_dashboard/settings.py.orig
```

Edit: **/usr/share/openstack-dashboard/openstack_dashboard/settings.py**

```
AVAILABLE_THEMES = [
    (
        'ubuntu',
        pgettext_lazy('Ubuntu theme', 'Ubuntu'),
        'themes/ubuntu'
    ), (
        'default',
        pgettext_lazy('Default style theme', 'Default'),
        'themes/default'
    ), (
        'material',
        pgettext_lazy("Google's Material Design style theme", "Material"),
        'themes/material'
    ),
]


if not SECRET_KEY:
    if not LOCAL_PATH:
        LOCAL_PATH = os.path.join(os.path.dirname(os.path.abspath(__file__)),
                                  'local')

    from horizon.utils import secret_key
    SECRET_KEY = secret_key.generate_or_read_from_file('/var/lib/openstack-dashboard/secret_key')

```

## CINDER

On the controller, copy the config file **/etc/cinder/cinder.conf**:

```
root@ost-ctrl:~# cp -a /etc/cinder/cinder.conf /etc/cinder/cinder.conf.orig
```

Edit: **/etc/cinder/cinder.conf**

```
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = tgtadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /var/lib/cinder/volumes
transport_url = rabbit://openstack:ost2017@controller
auth_strategy = keystone
my_ip = 10.10.12.103

[database]
connection = mysql+pymysql://cinder:ost2017@controller/cinder

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = ost2017

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

On each compute node, if the physical partition is filled with a logical volume it must be deleted to make space for the OpenStack volumes as required later by new instances.

```
root@ost-comp-1:~# pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda4
  VG Name               cinder-volumes
  PV Size               876.84 GiB / not usable 3.00 MiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              224471
  Free PE               222935
  Allocated PE          1536
  PV UUID               uazhS0-NQ4c-AQUb-D1Gy-Dope-kzVx-jo5K8G

  --- Physical volume ---
  PV Name               /dev/sda2
  VG Name               ost-srv-1-vg
  PV Size               856.81 GiB / not usable 4.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              219343
  Free PE               0
  Allocated PE          219343
  PV UUID               i08Oh1-tv0j-9ci5-72RK-Wa42-jBBK-dXi2pQ

  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               ost-srv-1-vg
  PV Size               127.87 GiB / not usable 4.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              32733
  Free PE               0
  Allocated PE          32733
  PV UUID               CUHViY-4LXG-1pDY-UaRZ-EPKf-uFy0-ecK1Mn

root@ost-comp-1:~# lvdisplay
  --- Logical volume ---
  LV Path                /dev/cinder-volumes/root
  LV Name                root
  VG Name                cinder-volumes
  LV UUID                pyQaVy-32an-Ghha-mdE0-qLXo-vHjC-6lsekh
  LV Write Access        read/write
  LV Creation host, time beta, 2017-10-03 11:01:28 +0300
  LV Status              available
  # open                 0
  LV Size                876.84 GiB
  Current LE             224471
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:2

  --- Logical volume ---
  LV Path                /dev/ost-srv-1-vg/root
  LV Name                root
  VG Name                ost-srv-1-vg
  LV UUID                RtpE9v-e1fJ-3cRO-z9ra-We7B-3OIi-BdqW4z
  LV Write Access        read/write
  LV Creation host, time beta, 2017-10-03 11:22:21 +0300
  LV Status              available
  # open                 1
  LV Size                856.81 GiB
  Current LE             219343
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:0

  --- Logical volume ---
  LV Path                /dev/ost-srv-1-vg/swap_1
  LV Name                swap_1
  VG Name                ost-srv-1-vg
  LV UUID                lP04en-Z0dU-wTNO-urSN-UEYF-7hlG-JEhyMv
  LV Write Access        read/write
  LV Creation host, time beta, 2017-10-03 11:23:09 +0300
  LV Status              available
  # open                 2
  LV Size                127.86 GiB
  Current LE             32733
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           252:1

root@ost-comp-2:~# lvremove cinder-volumes
Do you really want to remove and DISCARD active logical volume root? [y/n]: y
  Logical volume "root" successfully removed
```

On storage nodes, LVM drives must all be filtered together, and other drives must all be rejected.

On compute nodes, copy the config file **/etc/lvm/lvm.conf**:

```
root@ost-comp-1:~# cp -a /etc/lvm/lvm.conf /etc/lvm/lvm.conf.orig
```

Edit: **/etc/lvm/lvm.conf**

```
devices {
        filter = [ "a/sda2/", "a/sda3/", "a/sda4/", "r/.*/"]
}
```

On compute nodes, nearly the same diff, except the **my_ip** parameter, will be applied to **/etc/cinder/cinder.conf**:

```
root@ost-comp-1:~# cp -a /etc/cinder/cinder.conf /etc/cinder/cinder.conf.orig
```

Edit: **/etc/cinder/cinder.conf**

```
root@ost-comp-1:~# cat /etc/cinder/cinder.conf
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = tgtadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /var/lib/cinder/volumes
transport_url = rabbit://openstack:ost2017@controller
auth_strategy = keystone
my_ip = 10.10.12.101
enabled_backends = lvm
glance_api_servers = http://controller:9292

[database]
connection = mysql+pymysql://cinder:ost2017@controller/cinder

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = cinder
password = ost2017

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = tgtadm

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

&nbsp;

## NEUTRON

The regular Neutron installation uses Linux Bridge, and Open vSwitch is preferred, therefore this step was avoided during the installation.

The official page, **Open vSwitch: Self-service networks**, gives only a brief explanation and tersely states that:

> This architecture example augments Open vSwitch: Provider networks to support a nearly limitless quantity of entirely virtual networks.

The final configuration is quite different than the amalgamation of the [Open vSwitch: Provider networks](https://docs.openstack.org/newton/networking-guide/deploy-ovs-provider.html) and [Open vSwitch: Self-service networks](https://docs.openstack.org/newton/networking-guide/deploy-ovs-selfservice.html).


&nbsp;

### Controller

The following packages are required:

```
root@ost-ctrl:~# apt install openvswitch-switch neutron-server neutron-plugin-ml2 neutron-openvswitch-agent neutron-l3-agent neutron-metadata-agent
```

The following six config files has been modified on the controller:

* /etc/sysctl.conf
* /etc/neutron/metadata_agent.ini
* /etc/neutron/neutron.conf
* /etc/neutron/l3_agent.ini
* /etc/neutron/plugins/ml2/openvswitch_agent.ini
* /etc/neutron/plugins/ml2/ml2_conf.ini

Copy the config file **/etc/sysctl.conf**:

```
root@ost-ctrl:~# cp -a /etc/sysctl.conf /etc/sysctl.conf.orig
```

Edit: **/etc/sysctl.conf**

```
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0

net.ipv4.ip_forward=1

net.ipv6.conf.all.forwarding=1
```

Apply the changes:

```
root@ost-ctrl:~# sysctl -p
```

Copy the config file **/etc/neutron/metadata_agent.ini**:

```
root@ost-ctrl:~# cp -a /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.orig
```

Edit: **/etc/neutron/metadata_agent.ini**

```
[DEFAULT]
nova_metadata_ip = controller
metadata_proxy_shared_secret = OST_2017
```

Copy the config file **/etc/neutron/neutron.conf**:

```
root@ost-ctrl:~# cp -a /etc/neutron/neutron.conf /etc/neutron/neutron.conf.orig
```

Edit: **/etc/neutron/neutron.conf**

The **dhcp_agents_per_network** parameter must match the number of DHCP service [provider networks](https://docs.openstack.org/newton/networking-guide/deploy-ovs-provider.html).
> Enable two DHCP agents per network so both compute nodes can provide DHCP service provider networks.

```
[DEFAULT]
#core_plugin = ml2
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
transport_url = rabbit://openstack:ost2017@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
auth_strategy = keystone
dhcp_agents_per_network = 2
interface_driver = openvswitch

[database]
connection = mysql+pymysql://neutron:ost2017@controller/neutron

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = ost2017

[nova]
auth_url = http://controller:35357
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = ost2017
```

Copy the config file **/etc/neutron/l3_agent.ini**:

```
root@ost-ctrl:~# cp -a /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.orig
```

Edit: **/etc/neutron/l3_agent.ini**

```
[DEFAULT]
interface_driver = openvswitch
external_network_bridge =
```

Copy the config file **/etc/neutron/plugins/ml2/openvswitch_agent.ini**:

```
root@ost-ctrl:~# cp -a /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.orig
```

Edit: **/etc/neutron/plugins/ml2/openvswitch_agent.ini**

```
[agent]
tunnel_types = vxlan
l2_population = True
arp_responder = True

[ovs]
bridge_mappings = eno2:br-ex
local_ip = 10.10.11.103

[securitygroup]
enable_security_group = true
enable_ipset = true
firewall_driver = iptables_hybrid
```

Copy the config file **/etc/neutron/plugins/ml2/ml2_conf.ini**:

```
root@ost-ctrl:~# cp -a /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.orig
```

Edit: **/etc/neutron/plugins/ml2/ml2_conf.ini**

```
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = eno2

[ml2_type_vlan]
network_vlan_ranges = eno2:100:199

[ml2_type_vxlan]
vni_ranges = 100:999

[securitygroup]
firewall_driver = iptables_hybrid
enable_ipset = True
```

&nbsp;

### Compute Nodes

The following packages are required, on both compute nodes, of course:

```
root@ost-comp-1:~# apt install neutron-openvswitch-agent neutron-metadata-agent neutron-dhcp-agent
```

The following five config files has been modified on each compute node:

* /etc/neutron/plugins/ml2/openvswitch_agent.ini
* /etc/neutron/neutron.conf
* /etc/neutron/metadata_agent.ini
* /etc/neutron/l3_agent.ini
* /etc/neutron/dhcp_agent.ini

Copy the config file **/etc/neutron/plugins/ml2/openvswitch_agent.ini**:

```
root@ost-comp-1:~# cp -a /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.orig
```

Edit: **/etc/neutron/plugins/ml2/openvswitch_agent.ini**

```
[agent]
tunnel_types = vxlan
l2_population = True
arp_responder = True

[ovs]
local_ip = 10.10.11.101

[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = iptables_hybrid
```

Copy the config file **/etc/neutron/neutron.conf**:

```
root@ost-comp-1:~# cp -a /etc/neutron/neutron.conf /etc/neutron/neutron.conf.orig
```

Edit: **/etc/neutron/neutron.conf**

```
[DEFAULT]
#core_plugin = ml2
# ...
core_plugin = ml2
transport_url = rabbit://openstack:ost2017@controller
auth_strategy = keystone

[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = ost2017
```

Copy the config file **/etc/neutron/metadata_agent.ini**:

```
root@ost-comp-1:~# cp -a /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.orig
```

Edit: **/etc/neutron/metadata_agent.ini**

```
[DEFAULT]
nova_metadata_ip = controller
metadata_proxy_shared_secret = OST_2017
```

Copy the config file **/etc/neutron/l3_agent.ini**:

```
root@ost-comp-1:~# cp -a /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.orig
```

Edit: **/etc/neutron/l3_agent.ini**

```
[DEFAULT]
interface_driver = openvswitch
external_network_bridge =
```

Copy the config file **/etc/neutron/dhcp_agent.ini**:

```
root@ost-comp-1:~# cp -a /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.orig
```

Edit: **/etc/neutron/dhcp_agent.ini**

```
[DEFAULT]
verbose = True
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = False
force_metadata = False
```

&nbsp;

### Controller

Create the Neutron database, **neutron**, and set the privileges:

```
root@ost-ctrl:~# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 12027
Server version: 10.0.31-MariaDB-0ubuntu0.16.04.2 Ubuntu 16.04

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>  CREATE DATABASE neutron;
MariaDB [(none)]>  GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'ost2017';
MariaDB [(none)]>  GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'ost2017';
MariaDB [(none)]>  FLUSH PRIVILEGES;
MariaDB [(none)]>  EXIT
```
Populate the database:

```
# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

&nbsp;

## Prepare Networks

The following interface config sets the bridge:

```
root@ost-ctrl:~# cat /etc/network/interfaces.d/br-ex.cfg
# OpenStack network bridge

auto br-ex
allow-ovs br-ex
iface br-ex inet manual
   ovs_type OVSBridge
   ovs_ports eno2
#   address 10.10.10.103
#   netmask 255.255.255.0
#   gateway 10.10.10.1
#   dns-nameservers 8.8.8.8
```

After initializing the Neutron DB and the confguring the components, services must be restarted on the Controller node and default security rules must be entered which allow ICMP and SSH access to the instances:


```
root@ost-ctrl:~# ip a
root@ost-ctrl:~#
root@ost-ctrl:~# ovs-vsctl show
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-server restart
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-server status
root@ost-ctrl:~#
root@ost-ctrl:~# service openvswitch-switch restart
root@ost-ctrl:~#
root@ost-ctrl:~# service openvswitch-switch status
root@ost-ctrl:~#
root@ost-ctrl:~# ovs-vsctl add-br br-ex
root@ost-ctrl:~# ovs-vsctl add-port br-ex eno2
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-openvswitch-agent restart
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-openvswitch-agent status
root@ost-ctrl:~#
root@ost-ctrl:~# ip a
root@ost-ctrl:~#
root@ost-ctrl:~# ovs-vsctl show
```

The IP of the public interface **eno2** must be assigned to the OVS bridge **br-ex**.

This has been nicely explained on the OVS mail list, see the stream from 2012 [why vm can't ping host eth1](https://mail.openvswitch.org/pipermail/ovs-discuss/2012-June/027298.html).

> Any physical interface attached to a bridge doesn't need IP configuration. That configuration will be ignored, and having any such configuration will generally be confusing and unhelpful.

During this operation, if the controller is accessed via the eno2 interface, 10.10.10.103, the connection will be interrupted.
Therefore it is important to SSH via the private or management interfaces after first SSH'ing to a compute node.

```
root@ost-comp-1:~# ssh west@10.10.12.103
west@10.10.12.103's password:
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-97-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

3 packages can be updated.
0 updates are security updates.


Last login: Tue Oct 24 12:08:37 2017 from 192.168.28.88
west@ost-ctrl:~$ sudo su -
[sudo] password for west:
root@ost-ctrl:~#
root@ost-ctrl:~# ip link set dev br-ex promisc on
root@ost-ctrl:~# ip link set dev br-ex up
root@ost-ctrl:~# ip link set dev br-tun up
root@ost-ctrl:~# ip link set dev br-int up
root@ost-ctrl:~# ip link set dev br-ex state UP
root@ost-ctrl:~# ip link set dev br-tun state UP
root@ost-ctrl:~# ip link set dev br-int state UP
root@ost-ctrl:~#
root@ost-ctrl:~# ovs-vsctl add-port br-ex eno2
root@ost-ctrl:~#
root@ost-ctrl:~# ip address add 10.10.10.103/24 dev br-ex
root@ost-ctrl:~#
root@ost-ctrl:~# ip link set br-ex up
root@ost-ctrl:~#
root@ost-ctrl:~# ip address flush dev eno2
root@ost-ctrl:~#
root@ost-ctrl:~# route
root@ost-ctrl:~#
root@ost-ctrl:~# ip route replace default via 10.10.10.1
root@ost-ctrl:~#
root@ost-ctrl:~# route
```

Security rules must at least allow ICMP and SSH to the instances.

```
root@ost-ctrl:~# openstack security group rule list
root@ost-ctrl:~#
root@ost-ctrl:~# openstack security group rule create --proto icmp default
root@ost-ctrl:~# openstack security group rule create --ethertype IPv6 --proto ipv6-icmp default
root@ost-ctrl:~# openstack security group rule create --proto tcp --dst-port 22 default
root@ost-ctrl:~# openstack security group rule create --ethertype IPv6 --proto tcp --dst-port 22 default
root@ost-ctrl:~#
root@ost-ctrl:~# openstack security group rule list
+--------------------------------------+-------------+-----------+------------+-----------------------+--------------------------------------+
| ID                                   | IP Protocol | IP Range  | Port Range | Remote Security Group | Security Group                       |
+--------------------------------------+-------------+-----------+------------+-----------------------+--------------------------------------+
| 44aaf6cd-201e-43ce-8b8e-06878f959093 | icmp        | 0.0.0.0/0 |            | None                  | b6e51a76-2647-44b5-a7b2-41bf45693708 |
| 78a71b87-5e70-4adf-90dc-5bca1a62d711 | tcp         | None      | 22:22      | None                  | b6e51a76-2647-44b5-a7b2-41bf45693708 |
| 94647bca-5034-464d-9e7c-a3abb0b20ace | tcp         | 0.0.0.0/0 | 22:22      | None                  | b6e51a76-2647-44b5-a7b2-41bf45693708 |
| fe52a253-e03b-42cd-bee1-309337f4401a | ipv6-icmp   | None      |            | None                  | b6e51a76-2647-44b5-a7b2-41bf45693708 |
+--------------------------------------+-------------+-----------+------------+-----------------------+--------------------------------------+
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-l3-agent restart
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-l3-agent status
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-metadata-agent restart
root@ost-ctrl:~#
root@ost-ctrl:~# service neutron-metadata-agent status
root@ost-ctrl:~#
```

Services must be restarted on the compute nodes, too:

```
root@ost-comp-1:~# ovs-vsctl show
root@ost-comp-1:~#
root@ost-comp-1:~# ip netns
root@ost-comp-1:~#
root@ost-comp-1:~# ip a
root@ost-comp-1:~#
root@ost-comp-1:~# route
root@ost-comp-1:~#
root@ost-comp-1:~# service openvswitch-switch restart
root@ost-comp-1:~#
root@ost-comp-1:~# service neutron-metadata-agent restart
root@ost-comp-1:~#
root@ost-comp-1:~# service neutron-openvswitch-agent restart
root@ost-comp-1:~#
root@ost-comp-1:~# service neutron-dhcp-agent restart
root@ost-comp-1:~#
root@ost-comp-1:~# ip a
root@ost-comp-1:~#
root@ost-comp-1:~# ip netns
root@ost-comp-1:~#
root@ost-comp-1:~# ovs-vsctl show
root@ost-comp-1:~#
root@ost-comp-1:~# ip link set dev br-tun up
root@ost-comp-1:~# ip link set dev br-int up
root@ost-comp-1:~# ip link set dev br-tun state UP
root@ost-comp-1:~# ip link set dev br-int state UP
root@ost-comp-1:~#
root@ost-comp-1:~# ovs-vsctl show
root@ost-comp-1:~#
root@ost-comp-1:~# route
```

On the controller node create new networks, the provider for network access, a **flat** network - 10.10.10.0/24, through floating IPs and a private network, a **vxlan** network - 10.40.10.0/24, for assigning an IP to instances:

```
root@ost-ctrl:~# openstack network create --share --provider-physical-network eno2 --provider-network-type flat provider1
root@ost-ctrl:~#
root@ost-ctrl:~# openstack subnet create --subnet-range 10.10.10.0/24 --gateway 10.10.10.1 --network provider1 --allocation-pool start=10.10.10.9,end=10.10.10.99 --dns-nameserver 8.8.8.8 provider1-v4
root@ost-ctrl:~#
root@ost-ctrl:~# openstack subnet create --subnet-range fd00:10:10:10::/64 --gateway fd00:10:10:10::1 --ip-version 6 --ipv6-address-mode slaac --network provider1 --dns-nameserver 2001:4860:4860::8888 provider1-v6
root@ost-ctrl:~#
root@ost-ctrl:~# openstack network set --external provider1
root@ost-ctrl:~#
root@ost-ctrl:~# openstack network create --provider-network-type vxlan selfservice1
root@ost-ctrl:~#
root@ost-ctrl:~# openstack subnet create --subnet-range 10.40.10.0/24 --network selfservice1 --dns-nameserver 8.8.8.8 selfservice1-v4
root@ost-ctrl:~#
root@ost-ctrl:~# openstack subnet create --subnet-range fd00:10:40:10::/64 --ip-version 6 --ipv6-ra-mode slaac --ipv6-address-mode slaac --network selfservice1 --dns-nameserver 2001:4860:4860::8888 selfservice1-v6
root@ost-ctrl:~#
root@ost-ctrl:~# openstack router create router1
root@ost-ctrl:~#
root@ost-ctrl:~# openstack router add subnet router1 selfservice1-v4
root@ost-ctrl:~#
root@ost-ctrl:~# openstack router add subnet router1 selfservice1-v6
root@ost-ctrl:~#
root@ost-ctrl:~# neutron router-gateway-set router1 provider1
root@ost-ctrl:~#
root@ost-ctrl:~# ip netns
root@ost-ctrl:~#
root@ost-ctrl:~# ovs-vsctl show
```

&nbsp;

## Dashboard

First **flavors** must be created for easy resource allocation. Whereas for CirrOS a tiny flavor is good enough, for Ubuntu versions a bigger one is required.

&nbsp;

![flavor_tiny](./img/dashboard_flavor_01.png "Tiny flavor for CirrOS")

&nbsp;

On the Dashboard, the menu option **Project > Access & Security > Key Pairs > Import Key Pair** handles generation of private and public keys.
On the **Import Key Pair** dialog box, the contents of the **cloud.key.pub** is to be pasted into the **Public Key** box.
Usually instances can only be accessed via SSH, so that on the **Launch Instance** wizard setting the public key is mandatory.
Permissions for the private key must be set either to read-write (600) or read-only (400) by the current user.

```
root@ost-ctrl:~# mkdir -p .ssh
root@ost-ctrl:~# cd .ssh
root@ost-ctrl:~/.ssh# ssh-keygen -t rsa -f cloud.key
root@ost-ctrl:~/.ssh# ls
root@ost-ctrl:~/.ssh# cloud.key  cloud.key.pub
root@ost-ctrl:~/.ssh# chmod 400 cloud.key
root@ost-ctrl:~/.ssh# cat cloud.key.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDSn4pNkUdct7zCYHUaNvei+w+iJbyDdoTUMH0wB2gPKeO8tGd2fXVthjNqKOLfnt3Y7EY9jFBct2xJ6cD+alcx48RsvDMqApnViqsl1Tk3yr6Eln7HitX5gOQKOVZzbP/2DT3K1eL6x7o+pFMXRyH+B+/IyHhtOCDDt9Um9GRSih/+8hXO354Adp9erfJShOtMF56sNasVy4tGfbn0G3kRkWVdM1xrg9r+DOQWYlibAB/L0I7IcDgipYbh7BZt/FStEvI6eRkWo39Y7sy1GnD6gbt22MVjfKjYBdDk0MmXyt54AsvMYVoett88bxAyAoH+yFEFi/TJ0g9PyCj+8zon root@ost-ctrl
```

This public key, **cloud.key.pub**, and the corresponding private key, **cloud.key**, can be used for initial SSH access.

The Dashboard Web interface is used to save this public key under a **Key-Pair Name**, such as **cloud-key-pub**. This key will be imported into the Linux instances to be able to SSH without a password. The imported public key will be stored into the user's **~/.ssh/authorized_keys** file.

&nbsp;

![key](./img/dashboard_import_key.png "Import public key")

&nbsp;

After instance initialization, first the instance must be assigned a floating IP and SSH must use the private key of the public key imported to the instance.

```
root@ost-ctrl:~# cd .ssh
root@ost-ctrl:~/.ssh# ssh -i cloud.key ubuntu@10.10.10.11
Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 4.4.0-96-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

14 packages can be updated.
0 updates are security updates.


*** System restart required ***
Last login: Wed Oct 25 10:56:30 2017 from 10.10.10.103
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@u2:~$
```
