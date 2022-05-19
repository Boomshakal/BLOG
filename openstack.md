# OpenStack

## 必要安装

1. 时间同步
   ```shell
   apt -y install chrony
   vi /etc/chrony/chrony.conf
   pool ntp.aliyun.com        iburst
   systemctl restart chrony
   chronyc sources
   ```

   

2. mariadb 安装
   ```
   apt -y install mariadb-server
   vi /etc/mysql/mariadb.conf.d/50-server.cnf
   
   # line 28 : change
   bind-address = 0.0.0.0
   
   # line 40 : uncomment and change # default value 151 is not enough on Openstack Env
   max_connections = 500
   
   # line 104: confirm default charaset
   # if use 4 bytes UTF-8, specify [utf8mb4]
   character-set-server  = utf8mb4
   collation-server      = utf8mb4_general_ci
   
   systemctl restart mariadb
   ```

   ```shell
   mysql_secure_installation
   # set root password
   Set root password? [Y/n] y
   # remove anonymous users
   Remove anonymous users? [Y/n] y
   # disallow root login remotely
   Disallow root login remotely? [Y/n] y
   # remove test database
   Remove test database and access to it? [Y/n] y
   # reload privilege tables
   Reload privilege tables now? [Y/n] y
   ```

3. 安装 rabbitmq
   ```shell
   apt -y install rabbitmq-server
   # add a user to RabbitMQ
   # set any password for [password]
   rabbitmqctl add_user openstack password
   rabbitmqctl set_permissions openstack ".*" ".*" ".*" 
   ```

4. 安装memcached
   ```shell
   apt -y install memcached
   vi /etc/memcached.conf 
    # line 35: change
   -l 0.0.0.0
   ```

5. 安装其他
   ```shell
   apt -y install software-properties-common python3-pymysql 
   add-apt-repository cloud-archive:yoga 
   apt update && apt -y upgrade
   ```

6. 重启服务
   ```shell
   systemctl restart mariadb rabbitmq-server memcached 
   ```

## 配置keystone

1. Add a User and Database on MariaDB for Keystone.
   ```shell
   mysql
   create database keystone; 
   grant all privileges on keystone.* to keystone@'localhost' identified by 'password'; 
   grant all privileges on keystone.* to keystone@'%' identified by 'password'; 
   flush privileges; 
   exit
   ```

2. 安装Keystone
   ```shell
   apt -y install keystone python3-openstackclient apache2 libapache2-mod-wsgi-py3 python3-oauth2client 
   ```

3. 配置Keystone

   ```shell
   vi /etc/keystone/keystone.conf 
   # line 443 : add to specify Memcache Server
   memcache_servers = 192.168.50.35:11211
   
   # line 604 : change to MariaDB connection info
   connection = mysql+pymysql://keystone:password@192.168.50.35/keystone
   
   # line 2543 : uncomment
   provider = fernet 
   
   su -s /bin/bash keystone -c "keystone-manage db_sync" 
   
   # initialize Fernet key
   keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
   keystone-manage credential_setup --keystone-user keystone --keystone-group keystone 
   
   # define keystone API Host
   export controller=192.168.50.35 
   
   # bootstrap keystone
   # set any password for [adminpassword] section
   keystone-manage bootstrap --bootstrap-password adminpassword \
   --bootstrap-admin-url http://$controller:5000/v3/ \
   --bootstrap-internal-url http://$controller:5000/v3/ \
   --bootstrap-public-url http://$controller:5000/v3/ \
   --bootstrap-region-id RegionOne 
   ```

4. 配置Apache

   ```shell
   vi /etc/apache2/apache2.conf
   # line 70 : add to specify server name
   ServerName dlp.srv.world
   systemctl restart apache2 
   ```

5. 载入环境变量

   ```shell
   vi ~/keystonerc 
   export OS_PROJECT_DOMAIN_NAME=default
   export OS_USER_DOMAIN_NAME=default
   export OS_PROJECT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=adminpassword
   export OS_AUTH_URL=http://192.168.50.35:5000/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   export PS1='\u@\h \W(keystone)\$ '
   
   chmod 600 ~/keystonerc 
   source ~/keystonerc 
   echo "source ~/keystonerc " >> ~/.bashrc 
   ```

   ```shell
   # create [service] project
   openstack project create --domain default --description "Service Project" service
   # confirm settings
   openstack project list 
   ```

## 配置Glance

1. Add users and others for Glance in Keystone.

   ```shell
   openstack user create --domain default --project service --password servicepassword glance
   openstack role add --project service --user glance admin 
   openstack service create --name glance --description "OpenStack Image service" image
   export controller=192.168.50.35
   openstack endpoint create --region RegionOne image public http://$controller:9292 
   openstack endpoint create --region RegionOne image internal http://$controller:9292
   openstack endpoint create --region RegionOne image admin http://$controller:9292
   ```

2. Add a User and Database on MariaDB for Glance

   ```mysql
   mysql 
   create database glance; 
   grant all privileges on glance.* to glance@'localhost' identified by 'password';
   grant all privileges on glance.* to glance@'%' identified by 'password'; 
   flush privileges; 
   exit
   ```

3. 安装Glance

   ```shell
   apt -y install glance 
   ```

4. 配置Glance

   ```shell
   mv /etc/glance/glance-api.conf /etc/glance/glance-api.conf.org 
   vi /etc/glance/glance-api.conf 
   # create new
   
   [DEFAULT]
   bind_host = 0.0.0.0
   
   [glance_store]
   stores = file,http
   default_store = file
   filesystem_store_datadir = /var/lib/glance/images/
   
   [database]
   # MariaDB connection info
   connection = mysql+pymysql://glance:password@192.168.50.35/glance
   
   # keystone auth info
   [keystone_authtoken]
   www_authenticate_uri = http://192.168.50.35:5000
   auth_url = http://192.168.50.35:5000
   memcached_servers = 192.168.50.35:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = glance
   password = servicepassword
   
   [paste_deploy]
   flavor = keystone
   
   chmod 640 /etc/glance/glance-api.conf 	
   chown root:glance /etc/glance/glance-api.conf 
   su -s /bin/bash glance -c "glance-manage db_sync" 
   systemctl restart glance-api 
   systemctl enable glance-api 
   ```

## 添加镜像

1. 添加镜像

   ```shell
   wget http://cloud-images.ubuntu.com/releases/20.04/release/ubuntu-20.04-server-cloudimg-amd64.img 
   modprobe nbd 
   qemu-nbd --connect=/dev/nbd0 ubuntu-20.04-server-cloudimg-amd64.img 
   mount /dev/nbd0p1 /mnt 
   vi /mnt/etc/cloud/cloud.cfg 
   # line 13 : add
   # only the case you'd like to allow SSH password authentication
   ssh_pwauth: true
   # line 101: change
   # only the case if you'd like to allow [ubuntu] user to use SSH password auth
   system_info:
      # This will affect which distro class gets used
      distro: ubuntu
      # Default user name + that default users groups (if added/used)
      default_user:
        name: ubuntu
        lock_passwd: False
        gecos: Ubuntu
   
   umount /mnt 
   qemu-nbd --disconnect /dev/nbd0p1 
   
   # add image to Glance
   openstack image create "Ubuntu2004" --file ubuntu-20.04-server-cloudimg-amd64.img --disk-format qcow2 --container-format bare --public
   
   openstack image list 
   ```

## 配置Nova

1. Add users and others for Nova in Keystone.

```shell
openstack user create --domain default --project service --password servicepassword nova
openstack role add --project service --user nova admin 
openstack user create --domain default --project service --password servicepassword placement
openstack role add --project service --user placement admin 
openstack service create --name nova --description "OpenStack Compute service" compute
openstack service create --name placement --description "OpenStack Compute Placement service" placement
export controller=192.168.50.35
openstack endpoint create --region RegionOne compute public http://$controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://$controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://$controller:8774/v2.1/%\(tenant_id\)s 
openstack endpoint create --region RegionOne placement public http://$controller:8778 
openstack endpoint create --region RegionOne placement internal http://$controller:8778
openstack endpoint create --region RegionOne placement admin http://$controller:8778
```

2. Add a User and Database on MariaDB for Nova.

```mysql
mysql
create database nova;
grant all privileges on nova.* to nova@'localhost' identified by 'password'; 
grant all privileges on nova.* to nova@'%' identified by 'password'; 
create database nova_api; 
grant all privileges on nova_api.* to nova@'localhost' identified by 'password'; 
grant all privileges on nova_api.* to nova@'%' identified by 'password'; 
create database placement;
grant all privileges on placement.* to placement@'localhost' identified by 'password'; 
grant all privileges on placement.* to placement@'%' identified by 'password';
create database nova_cell0; 
grant all privileges on nova_cell0.* to nova@'localhost' identified by 'password';
grant all privileges on nova_cell0.* to nova@'%' identified by 'password';
flush privileges; 
exit
```

3. Install Nova.

   ```shell
   apt -y install nova-api nova-conductor nova-scheduler nova-novncproxy placement-api python3-novaclient 
   ```

4. Configure Nova.

   ```shell
   mv /etc/nova/nova.conf /etc/nova/nova.conf.org 
   vi /etc/nova/nova.conf 
   # create new
   [DEFAULT]
   # define IP address
   my_ip = 192.168.50.35
   state_path = /var/lib/nova
   enabled_apis = osapi_compute,metadata
   log_dir = /var/log/nova
   # RabbitMQ connection info
   transport_url = rabbit://openstack:password@192.168.50.35
   
   [api]
   auth_strategy = keystone
   
   # Glance connection info
   [glance]
   api_servers = http://192.168.50.35:9292
   
   [oslo_concurrency]
   lock_path = $state_path/tmp
   
   # MariaDB connection info
   [api_database]
   connection = mysql+pymysql://nova:password@192.168.50.35/nova_api
   
   [database]
   connection = mysql+pymysql://nova:password@192.168.50.35/nova
   
   # Keystone auth info
   [keystone_authtoken]
   www_authenticate_uri = http://192.168.50.35:5000
   auth_url = http://192.168.50.35:5000
   memcached_servers = 192.168.50.35:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = nova
   password = servicepassword
   
   [placement]
   auth_url = http://192.168.50.35:5000
   os_region_name = RegionOne
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = placement
   password = servicepassword
   
   [wsgi]
   api_paste_config = /etc/nova/api-paste.ini
   
   
   chmod 640 /etc/nova/nova.conf 
   chgrp nova /etc/nova/nova.conf 
   
   mv /etc/placement/placement.conf /etc/placement/placement.conf.org 
   vi /etc/placement/placement.conf 
   # create new
   [DEFAULT]
   debug = false
   
   [api]
   auth_strategy = keystone
   
   [keystone_authtoken]
   www_authenticate_uri = http://192.168.50.35:5000
   auth_url = http://192.168.50.35:5000
   memcached_servers = 192.168.50.35:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = placement
   password = servicepassword
   
   [placement_database]
   connection = mysql+pymysql://placement:password@192.168.50.35/placement
   
   chmod 640 /etc/placement/placement.conf 
   chgrp placement /etc/placement/placement.conf 
   ```

4. Add Data into Database and start Nova services.

   ```shell
   su -s /bin/bash placement -c "placement-manage db sync" 
   su -s /bin/bash nova -c "nova-manage api_db sync" 
   su -s /bin/bash nova -c "nova-manage cell_v2 map_cell0" 
   su -s /bin/bash nova -c "nova-manage db sync" 
   su -s /bin/bash nova -c "nova-manage cell_v2 create_cell --name cell1" 
   systemctl restart apache2 
   
   for service in api conductor scheduler; do
   systemctl restart nova-$service
   done 
   
   # show status
   openstack compute service list 
   ```

   

5. KVM

   ```shell
   apt -y install qemu-kvm libvirt-daemon-system libvirt-daemon virtinst bridge-utils libosinfo-bin libguestfs-tools virt-top 
   ```

   ```shell
   vi /etc/netplan/01-netcfg.yaml
   network:
     ethernets:
       enp1s0:
         dhcp4: no
         # disable existing configuration for ethernet
         #addresses: [192.168.50.35/24]
         #gateway4: 10.0.0.1
         #nameservers:
           #addresses: [192.168.50.1]
         dhcp6: no
     # add configuration for bridge interface
     bridges:
       br0:
         interfaces: [enp1s0]
         dhcp4: no
         addresses: [192.168.50.35/24]
         gateway4: 192.168.50.1
         nameservers:
           addresses: [192.168.50.1]
         parameters:
           stp: false
         dhcp6: no 
     version: 2
   ```

7. Install OpenStack Compute Service (Nova).

   ```shell
   apt -y install nova-compute nova-compute-kvm 
   vi /etc/nova/nova.conf 
   
   # add follows (enable VNC)
   
   [vnc]
   enabled = True
   server_listen = 0.0.0.0
   server_proxyclient_address = 192.168.50.35
   novncproxy_base_url = http://192.168.50.35:6080/vnc_auto.html 
   
   systemctl restart nova-compute nova-novncproxy 
   su -s /bin/bash nova -c "nova-manage cell_v2 discover_hosts" 
   openstack compute service list 
   ```

## 配置Neutron

1. Add user or service for Neutron on Keystone.

```shell
# create [neutron] user in [service] project
openstack user create --domain default --project service --password servicepassword neutron 
# add [neutron] user in [admin] role
openstack role add --project service --user neutron admin 
# create service entry for [neutron]
openstack service create --name neutron --description "OpenStack Networking service" network 
# define Neutron API Host
export controller=192.168.50.35
# create endpoint for [neutron] (public)
openstack endpoint create --region RegionOne network public http://$controller:9696 
# create endpoint for [neutron] (internal)
openstack endpoint create --region RegionOne network internal http://$controller:9696 
# create endpoint for [neutron] (admin)
openstack endpoint create --region RegionOne network admin http://$controller:9696 
```

2. Add a User and Database on MariaDB for Neutron.

   ```mysql
   mysql
   create database neutron_ml2; 
   grant all privileges on neutron_ml2.* to neutron@'localhost' identified by 'password'; 
   grant all privileges on neutron_ml2.* to neutron@'%' identified by 'password'; 
   flush privileges; 
   exit 
   ```

3. Install Neutron services.

   ```shell
   apt -y install neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent python3-neutronclient 
   ```

4. Configure Neutron.

   ```shell
   mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org 
   vi /etc/neutron/neutron.conf 
   
    # create new
   
   [DEFAULT]
   core_plugin = ml2
   service_plugins = router
   auth_strategy = keystone
   state_path = /var/lib/neutron
   dhcp_agent_notification = True
   allow_overlapping_ips = True
   notify_nova_on_port_status_changes = True
   notify_nova_on_port_data_changes = True
   # RabbitMQ connection info
   transport_url = rabbit://openstack:password@192.168.50.35
   
   [agent]
   root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
   
   # Keystone auth info
   [keystone_authtoken]
   www_authenticate_uri = http://192.168.50.35:5000
   auth_url = http://192.168.50.35:5000
   memcached_servers = 192.168.50.35:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = neutron
   password = servicepassword
   
   # MariaDB connection info
   [database]
   connection = mysql+pymysql://neutron:password@192.168.50.35/neutron_ml2
   
   # Nova connection info
   [nova]
   auth_url = http://192.168.50.35:5000
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   region_name = RegionOne
   project_name = service
   username = nova
   password = servicepassword
   
   [oslo_concurrency]
   lock_path = $state_path/tmp
   
   touch /etc/neutron/fwaas_driver.ini 
   chmod 640 /etc/neutron/{neutron.conf,fwaas_driver.ini} 
   chgrp neutron /etc/neutron/{neutron.conf,fwaas_driver.ini} 
   
   vi /etc/neutron/l3_agent.ini 
    # line 21 : add
   interface_driver = linuxbridge
   
   vi /etc/neutron/dhcp_agent.ini
   # line 21 : add
   interface_driver = linuxbridge
   # line 43 : uncomment
   dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
   # line 52 : uncomment and change
   enable_isolated_metadata = true
   
   vi /etc/neutron/metadata_agent.ini
   # line 22 : uncomment and specify Nova API server
   nova_metadata_host = 192.168.50.35
   # line 34 : uncomment and specify any secret key you like
   metadata_proxy_shared_secret = metadata_secret
   # line 312 : add to specify Memcache Server
   memcache_servers = 192.168.50.35:11211
   
   vi /etc/neutron/plugins/ml2/ml2_conf.ini 
   # line 154 : add
   # OK with no value for [tenant_network_types] now (set later if need)
   [ml2]
   type_drivers = flat,vlan,vxlan
   tenant_network_types =
   mechanism_drivers = linuxbridge
   extension_drivers = port_security
   
   vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini 
    # line 225 : add
   [securitygroup]
   enable_security_group = True
   firewall_driver = iptables
   enable_ipset = True
   # line 284 : add own IP address
   local_ip = 192.168.50.35
   
   vi /etc/nova/nova.conf 
   # add follows into [DEFAULT] section
   vif_plugging_is_fatal = True
   vif_plugging_timeout = 300
   
   # add follows to the end : Neutron auth info
   # the value of [metadata_proxy_shared_secret] is the same with the one in [metadata_agent.ini]
   [neutron]
   auth_url = http://192.168.50.35:5000
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   region_name = RegionOne
   project_name = service
   username = neutron
   password = servicepassword
   service_metadata_proxy = True
   metadata_proxy_shared_secret = metadata_secret
   ```

5. Start Neutron services.

   ```shell
   ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini 
   su -s /bin/bash neutron -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head" 
   for service in server l3-agent dhcp-agent metadata-agent linuxbridge-agent; do
   systemctl restart neutron-$service
   systemctl enable neutron-$service
   done 
   systemctl restart nova-api nova-compute 
   # show status
   openstack network agent list 
   ```

6. Configure Neutron Network

   ```shell
   # create a setting file for anonymous interface
   # replace the name [eth1] to your environment
   
   root@dlp ~(keystone)# vi /etc/systemd/network/eth1.network
   
   [Match]
   Name=eth1
   
   [Network]
   LinkLocalAddressing=no
   IPv6AcceptRA=no
   
   ip link set eth1 up 
   vi /etc/neutron/plugins/ml2/ml2_conf.ini 
    # line 206 : add
   
   [ml2_type_flat]
   flat_networks = physnet1
   
   vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini 
    # line 190 : add
   
   [linux_bridge]
   physical_interface_mappings = physnet1:eth1
   
   systemctl restart neutron-linuxbridge-agent 
   ```

7. Create virtual network.

   ```shell
   projectID=$(openstack project list | grep service | awk '{print $2}') 
   openstack network create --project $projectID \
   --share --provider-network-type flat --provider-physical-network physnet1 sharednet1
   
   # create subnet [192.168.50.0/24] in [sharednet1]
   openstack subnet create subnet --network sharednet1 \
   --project $projectID --subnet-range 192.168.50.0/24 \
   --allocation-pool start=192.168.50.200,end=192.168.50.254 \
   --gateway 192.168.50.1 --dns-nameserver 192.168.50.1
   ```

8. confirm setting

   ```shell
   openstack network list 
   openstack subnet list 
   ```

## 添加Openstack 用户

1. Any names are OK you like for user-name or project-name. Also Add flavors which define vCPU or Memory of an instance, too

   ```shell
   # create a project
   openstack project create --domain default --description "Hiroshima Project" hiroshima 
   openstack user create --domain default --project hiroshima --password userpassword serverworld 
   
   # create a role
   openstack role create CloudUser 
   # create a user to the role
   openstack role add --project hiroshima --user serverworld CloudUser
   # create a [flavor]
   openstack flavor create --id 0 --vcpus 1 --ram 2048 --disk 10 m1.small 
   ```

## 创建Instances

1. Login as a common user and create a config for authentication of Keystyone.Create and run an instance.

   ```shell
   li@dlp:~$ vi ~/keystonerc
   export OS_PROJECT_DOMAIN_NAME=default
   export OS_USER_DOMAIN_NAME=default
   export OS_PROJECT_NAME=hiroshima
   export OS_USERNAME=serverworld
   export OS_PASSWORD=userpassword
   export OS_AUTH_URL=http://192.168.50.35:5000/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   export PS1='\u@\h \W(keystone)\$ '
   
   chmod 600 ~/keystonerc 
   source ~/keystonerc 
   echo "source ~/keystonerc " >> ~/.bash_profile 
   
   openstack flavor list 
   openstack image list 
   openstack network list 
   # create a security group for instances
   openstack security group create secgroup01 
   openstack security group list 
   # create a SSH keypair for connecting to instances
   ssh-keygen -q -N "" 
   # add public-key
   openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey 
   openstack keypair list 
   
   netID=$(openstack network list | grep sharednet1 | awk '{ print $2 }') 
   # create and boot an instance
   openstack server create --flavor m1.small --image Ubuntu2004 --security-group secgroup01 --nic net-id=$netID --key-name mykey Ubuntu-2004
   
   openstack server list 
   ```

   

2.  	Configure security settings for the security group you created above to access with SSH and ICMP.

   ```shell
   # permit ICMP
   openstack security group rule create --protocol icmp --ingress secgroup01 
   # permit SSH
   openstack security group rule create --protocol tcp --dst-port 22:22 secgroup01 
   
   openstack security group rule list secgroup01 
   ```

3.  	Login to the instance with SSH. 

   ```shell
   openstack server list 
   virsh list
   ping 192.168.50.249
   ssh ubuntu@192.168.50.249
   exit
   ```

4. control with openstack command

   ```shell
   openstack server list
   openstack server stop Ubuntu-2004
   openstack server start Ubuntu-2004
   openstack server list
   ```

5. VNC console

   ```shell
   openstack server list
   openstack console url show Ubuntu-2004
   ```

## 配置Horizon

1.  	Install Horizon. 

   ```shell
   apt -y install openstack-dashboard 
   ```

2. Configure Horizon.

   ```shell
   vi /etc/openstack-dashboard/local_settings.py 
   
   # line 99 : change Memcache server
   CACHES = {
       'default': {
           'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
           'LOCATION': '192.168.50.35:11211',
       },
   }
   
   # line 113 : add
   SESSION_ENGINE = "django.contrib.sessions.backends.cache" 
   
   # line 126 : set Openstack Host
   # line 127 : comment out and add a line to specify URL of Keystone Host
   OPENSTACK_HOST = "192.168.50.35"
   #OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST
   OPENSTACK_KEYSTONE_URL = "http://192.168.50.35:5000/v3"
   
   # line 131 : set your timezone
   TIME_ZONE = "Asia/Shanghai" 
   
   # add to the end
   OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
   OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
   
   systemctl restart apache2 
   ```

   ```shell
   # this is the optional setting
   # if you allow common users to access to instances details or console on the Dashboard web, set like follows
   vi /etc/nova/policy.json 
   # create new
   # default is [rule:system_admin_api], so only admin users can access to instances details or console
   {
     "os_compute_api:os-extended-server-attributes": "rule:admin_or_owner",
   }
   
   chgrp nova /etc/nova/policy.json 
   chmod 640 /etc/nova/policy.json 
   systemctl restart nova-api 
   ```

3. Access to the URL below with any web browser.

   ```shell
   http://(Dashboard server's hostname or IP address)/horizon/ 
   ```


## 配置Cinder

### Cinder Control Node

1.  Add a User or Endpoints for Cinder to Keystone on Control Node

   ```shell
   # create [cinder] user in [service] project
   openstack user create --domain default --project service --password servicepassword cinder
   # add [cinder] user in [admin] role
   openstack role add --project service --user cinder admin
   # create service entry for [cinder]
   openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
   # define Cinder API Host
   export controller=192.168.50.35
   openstack endpoint create --region RegionOne volumev3 public http://$controller:8776/v3/%\(tenant_id\)s
   # create endpoint for [cinder] (internal)
   openstack endpoint create --region RegionOne volumev3 internal http://$controller:8776/v3/%\(tenant_id\)s
   # create endpoint for [cinder] (admin)
   openstack endpoint create --region RegionOne volumev3 admin http://$controller:8776/v3/%\(tenant_id\)s
   ```

2. Add a User and Database on MariaDB for Cinder.

   ```mysql
   mysql
   create database cinder; 
   grant all privileges on cinder.* to cinder@'localhost' identified by 'password'; 
   grant all privileges on cinder.* to cinder@'%' identified by 'password';
   flush privileges; 
   exit
   ```

3.  Install Cinder Service.

   ```shell
   apt -y install cinder-api cinder-scheduler python3-cinderclient
   mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.org
   vi /etc/cinder/cinder.conf
   
   # create new
   [DEFAULT]
   # define own IP address
   my_ip = 192.168.50.35
   rootwrap_config = /etc/cinder/rootwrap.conf
   api_paste_confg = /etc/cinder/api-paste.ini
   state_path = /var/lib/cinder
   auth_strategy = keystone
   # RabbitMQ connection info
   transport_url = rabbit://openstack:password@192.168.50.35
   enable_v3_api = True
   
   # MariaDB connection info
   [database]
   connection = mysql+pymysql://cinder:password@192.168.50.35/cinder
   
   # Keystone auth info
   [keystone_authtoken]
   www_authenticate_uri = http://192.168.50.35:5000
   auth_url = http://192.168.50.35:5000
   memcached_servers = 192.168.50.35:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = cinder
   password = servicepassword
   
   [oslo_concurrency]
   lock_path = $state_path/tmp
   
   chmod 640 /etc/cinder/cinder.conf
   chgrp cinder /etc/cinder/cinder.conf
   su -s /bin/bash cinder -c "cinder-manage db sync"
   systemctl restart cinder-scheduler
   systemctl enable cinder-scheduler
   
   # set environment variable
   echo "export OS_VOLUME_API_VERSION=3" >> ~/keystonerc
   source ~/keystonerc
   openstack volume service list
   ```

### Cinder Storage Node

1. Install Cinder Volume service.

   ```shell
   apt -y install cinder-volume python3-mysqldb
   ```

2.  Configure Cinder Volume.

   ```shell
   mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.org
   vi /etc/cinder/cinder.conf
   
   # create new
   [DEFAULT]
   # define IP address
   my_ip = 10.0.0.50
   rootwrap_config = /etc/cinder/rootwrap.conf
   api_paste_confg = /etc/cinder/api-paste.ini
   state_path = /var/lib/cinder
   auth_strategy = keystone
   # RabbitMQ connection info
   transport_url = rabbit://openstack:password@192.168.50.35
   enable_v3_api = True
   # Glance connection info
   glance_api_servers = http://192.168.50.35:9292
   # OK with empty value now
   enabled_backends =
   
   # MariaDB connection info
   [database]
   connection = mysql+pymysql://cinder:password@192.168.50.35/cinder
   
   # Keystone auth info
   [keystone_authtoken]
   www_authenticate_uri = http://192.168.50.35:5000
   auth_url = http://192.168.50.35:5000
   memcached_servers = 192.168.50.35:11211
   auth_type = password
   project_domain_name = default
   user_domain_name = default
   project_name = service
   username = cinder
   password = servicepassword
   
   [oslo_concurrency]
   lock_path = $state_path/tmp
   
   chmod 640 /etc/cinder/cinder.conf
   chgrp cinder /etc/cinder/cinder.conf
   systemctl restart cinder-volume
   systemctl enable cinder-volume
   ```

### Cinder Storage (LVM)

1. Create a volume group for Cinder on Storage Node.

   ```shell
   pvcreate /dev/sdb1
   vgcreate -s 32M vg_volume01 /dev/sdb1
   ```

2. Configure Cinder Volume on Storage Node.

   ```shell
   apt -y install targetcli-fb python3-rtslib-fb
   vi /etc/cinder/cinder.conf
   
   # add the value to [enabled_backends] param
   enabled_backends = lvm
   # add to the end
   [lvm]
   target_helper = lioadm
   target_protocol = iscsi
   # IP address of Storage Node
   target_ip_address = 10.0.0.50
   # volume group name created on [1]
   volume_group = vg_volume01
   volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
   volumes_dir = $state_path/volumes
   
   systemctl restart cinder-volume
   ```

3. Configure Nova on Compute Node.

   ```shell
   vi /etc/nova/nova.conf
   # add to the end
   [cinder]
   os_region_name = RegionOne
   
   systemctl restart nova-compute
   ```

4. Login as a common user you'd like to add volumes to own instances.

   ```shell
   echo "export OS_VOLUME_API_VERSION=3" >> ~/keystonerc
   source ~/keystonerc
   openstack volume create --size 10 disk01
   openstack volume list
   ```

5. Attach the virtual disk to an Instance.

   ```shell
   openstack server list
   openstack server add volume Ubuntu-2004 disk01
   
   # the status of attached disk turns [in-use] like follows
   openstack volume list
   
   # detach the disk
   openstack server remove volume Ubuntu-2004 disk01
   ```

   









































































































