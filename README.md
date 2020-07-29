# Project openstack
Manual installation of openstack (ussuri version)


For my undergraduate project, I decided to set up a private cloud based on openstack for the company where I did my internship. So this documentation has been produced. I used only virtual machines and this is only a simulation and should not be confused with a production deployment. 


All the documentation has been made with root rights, all commands starting with # must be executed either by using the sudo command (root) 

---
- - -
****************
## topology
This installation will be carried out on 03 machines. A controller, a compute (to execute the instances) and a storage machine. I would like to remind that this is a minimal installation in a test environment to simulate the operation of the cloud.

| hostname     | Ram      | vCPU          | interface1     | interface2   | Disque        |  
| :----------- | :------: | ------------: | -------------  | :---------:  | ------------: |
| controller1  | 8 go     | 2             |  192.168.10.210| 10.10.10.194 | 55            |
| compute      | 8go      | 2             |  192.168.10.209| 10.10.10.193 | 25            |
| storage1     | 2 go     | 1             |  192.168.10.212| 10.10.10.196 | 50            |

I leave it up to you to configure the ip addresses on the different machines and also to configure the name resolution between the machines.  
## installation 
### Time server 
One of the prerequisites to have a functional OpenStack is to have machines with the same time and for this we will install a time server (ntp or chrony) on the controller and define the other machines as clients. 

On all machines you have to install the chrony package and then you have to replace in /etc/chrony/chrony.conf server 0.ubuntu.pool.ntp.org by ```server controller``` 
on all except the controller which remains unchanged. 


```
  #sudo apt install chrony
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



