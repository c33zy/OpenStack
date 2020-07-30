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

I leave it up to you to configure the ip addresses on the different machines and also to configure the name resolution between the machines.  
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

Edit the ```/etc/default/etcd ``` file and set the ETCD_INITIAL_CLUSTER, ETCD_INITIAL_ADVERTISE_PEER_URLS, ETCD_ADVERTISE_CLIENT_URLS, ETCD_LISTEN_CLIENT_URLS to the management IP address of the controller node to enable access by other nodes via the management network:

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













Your computer crashed? Try sending a
<kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>Del</kbd>

<foo@bar.com>

    This is code
    So is this
    
    my_array.each do |item|
        puts item
    end
    
    
| Col1         | Col2     | Col3          |
| :----------- | :------: | ------------: |
| Left-aligned | Centered | Right-aligned |
| blah         | blah     | blah          |



