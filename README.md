# Project openstack
Manual installation of openstack (ussuri version)


For my undergraduate project, I decided to set up a private cloud based on openstack for the company where I did my internship. So this documentation has been produced. I used only virtual machines and this is only a simulation and should not be confused with a production deployment. 


All the documentation has been made with root rights, all commands starting with # must be executed either by using the sudo command (root) 

---
- - -
****************
## Topology
This installation will be carried out on 03 machines. A controller, a compute (to execute the instances) and a storage machine. I would like to remind that this is a minimal installation in a test environment to simulate the operation of the cloud.

| hostname     | Ram      | vCPU          | interface1     | interface2   | Disque        |  
| :----------- | :------: | ------------: | -------------  | :---------:  | ------------: |
| controller1  | 8 go     | 2             |  192.168.10.210| 10.10.10.194 | 55            |
| compute      | 8go      | 2             |  192.168.10.209| 10.10.10.193 | 25            |
| storage1     | 2 go     | 1             |  192.168.10.212| 10.10.10.196 | 50            |

I leave it up to you to configure the ip addresses on the different machines and also to configure the name resolution between the machines. The operating system used is Ubuntu.  
## Installation 
### Time server 
One of the prerequisites to have a functional OpenStack is to have machines with the same time and for this we will install a time server (ntp or chrony) on the controller and define the other machines as clients. 

On all machines you have to install the chrony package and then you have to replace in /etc/chrony/chrony.conf server 0.ubuntu.pool.ntp.org by ```server controller``` 
on all except the controller which remains unchanged. 

```
#sudo apt install chrony

```
### packages installation 
```
# apt install software-properties-common
# add-apt-repository cloud-archive:stein
# apt update && apt dist-upgrade
# apt install python3-openstackclient
```
### installation of the database on the controller
we get started by maria db 

```
# apt install mariadb-server python-pymysql
```
Let's create and edit the **/etc/mysql/mariadb.conf.d/99-openstack.cnf**  file and then complete the following actions:

Create a [mysqld] section, and set the bind-address key to the management IP address of the controller node to enable access by other nodes via the management network. Set additional keys to enable useful options and the UTF-8 character set:
```
[mysqld]
bind-address = 10.0.0.11

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```
Restart the database service:
```
# service mysql restart
```

### Install and configure components of rabbitmq on controller 
Install the package:
```
# apt install rabbitmq-server
```
Add the openstack user:
```
# rabbitmqctl add_user openstack RABBIT_PASS
```
Permit configuration, write, and read access for the openstack user:
```
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### Install and configure components of memcached on controller 
Install the packages:
```
# apt install memcached python-memcache
```
Edit the ```/etc/memcached.conf```  file and configure the service to use the management IP address of the controller node. This is to enable access by other nodes via the management network:
```
-l 10.0.0.11
 ```
Restart the Memcached service:
```
# service memcached restart
```

### install and configure components of etcd on controller 

```
# apt install etcd
```

Edit the  ```/etc/default/etcd ``` file

```
ETCD_NAME="controller1"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller1=http://192.168.10.210:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.210:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.210:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://192.168.10.210:2379"
```
Enable and restart the etcd service:
```
# systemctl enable etcd
# systemctl restart etcd
```
### Install and configure keystone the identity service 

Use the database access client to connect to the database server as the root user:

```
# mysql
```

Create the keystone database:

```
MariaDB [(none)]> CREATE DATABASE keystone;
```

Grant proper access to the keystone database:

```
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
```

Replace KEYSTONE_DBPASS with a suitable password.
After that ;
Run the following command to install the packages:

```
# apt install keystone
```

Edit the  ```/etc/keystone/keystone.conf ``` file and complete the following actions:

In the [database] section, configure database access:

```
[database]
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
```

Replace KEYSTONE_DBPASS with the password you chose for the database.

Comment out or remove any other connection options in the [database] section.

In the [token] section, configure the Fernet token provider:

```
[token]
provider = fernet
```

Populate the Identity service database:

```
# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

Initialize Fernet key repositories:


```
# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

Bootstrap the Identity service:

```
# keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
  ```

Replace ADMIN_PASS with a suitable password for an administrative user.

Edit the ```/etc/apache2/apache2.conf ``` file and configure the ServerName option to reference the controller node:


```
ServerName controller
```

The ServerName entry will need to be added if it does not already exist.

Restart the Apache service:
```
# service apache2 restart
```

Configure the administrative account by setting the proper environmental variables:
```
$ export OS_USERNAME=admin
$ export OS_PASSWORD=ADMIN_PASS
$ export OS_PROJECT_NAME=admin
$ export OS_USER_DOMAIN_NAME=Default
$ export OS_PROJECT_DOMAIN_NAME=Default
$ export OS_AUTH_URL=http://controller:5000/v3
$ export OS_IDENTITY_API_VERSION=3
```
Create a domain, projects, users, and roles
```
$ openstack domain create --description "An Example Domain" example
```
This guide uses a service project that contains a unique user for each service that you add to your environment. Create the service project:
```
$ openstack project create --domain default \
  --description "Service Project" service
```

Create the myproject project:
```
$ openstack project create --domain default \
  --description "Demo Project" myproject
```

Create the myuser user:
```
$ openstack user create --domain default \
  --password-prompt myuser
```
Create the myrole role:

```
$ openstack role create myrole
```

Add the myrole role to the myproject project and myuser user:

```
$ openstack role add --project myproject --user myuser myrole
```
### Creating OpenStack client scripts
To simplify the connection we will create two scripts for a connection with and without administrator rights

admin-openrc :
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
demo-openrc :
```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=USER_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```
To use one of the scripts to log in, just execute this command :
```
$ . admin-openrc
$ openstack token issue
```

### Install and configure Glance

Use the database access client to connect to the database server as the root user:
```
# mysql
```
Create the glance database:
```
MariaDB [(none)]> CREATE DATABASE glance;
Grant proper access to the glance database:

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
```
Replace GLANCE_DBPASS with a suitable password.

Source the admin credentials to gain access to admin-only CLI commands:
```
$ . admin-openrc
```
To create the service credentials, complete these steps:

Create the glance user:
```
$ openstack user create --domain default --password-prompt glance
```

Add the admin role to the glance user and service project:
```
$ openstack role add --project service --user glance admin
```
Create the glance service entity:
```
$ openstack service create --name glance \
  --description "OpenStack Image" image
```

Create the Image service API endpoints:
```
$ openstack endpoint create --region RegionOne \
  image public http://controller1:9292
```
```
$ openstack endpoint create --region RegionOne \
  image internal http://controller1:9292
```

```
$ openstack endpoint create --region RegionOne \
  image admin http://controller1:9292
```

Install and configure components¶

```
# apt install glance
```

Edit the /etc/glance/glance-api.conf file and complete the following actions:

In the [database] section, configure database access:

```
[database]
# ...
connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```

Replace GLANCE_DBPASS with the password you chose for the Image service database.

In the [keystone_authtoken], [paste_deploy] and [glance_store] sections, configure Identity service access:


```
[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
# ...
flavor = keystone
Replace GLANCE_PASS with the password you chose for the glance user in the Identity service.

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

Populate the Image service database:

```
# su -s /bin/sh -c "glance-manage db_sync" glance
 ```
 
Restart the Image services:


```
# service glance-api restart
```

### Install and configure of other services 
  
 Glance :  For this go to <https://docs.openstack.org/placement/ussuri/install/>

Same thing  for Nova , refere to offical guide  <https://docs.openstack.org/nova/ussuri/install/controller-install-ubuntu.html>

we also need to install and configure :  Horizon (dashboard service) , Neutron (Networking service) and cinder (block storage) 
I used the official openstack guide, so I refer you there <https://docs.openstack.org/ussuri/install/>
I have to notify that you have to adapt the information used (such as ip addresses, machine names) to your needs.   

