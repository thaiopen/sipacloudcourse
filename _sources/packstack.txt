========================
Openstack with Packstack
========================

packstack
=========

vagrant
-------
สร้าง directory ชื่อ openstack และภายในมี Vagrantfile ดังนี้

.. literalinclude::  _source/Vagrantfile3

Download complete file :download:`Vagrantfile3 <./_source/Vagrantfile3>`

เข้าไปยัง controller
::
    mkdir ~/packstack
    cd packstack
    wget https://thaiopen.github.io/sipacloudcourse/_downloads/Vagrantfile3
    mv Vagrantfile3 Vagrantfile

    vagrant up --no-parallel
    vagrant ssh controller

    #test ping to compute, selinux, network
    ping compute
    getenforce
    systemctl is-active NetworkManager
    systemctl is-active firewalld

Disk prepare for cinder
-----------------------
เตรียม disk ให้กับ cinder ด้วยการสร้าง volume group ชื่อว่า  cinder-volumes
::

    sudo su -
    fdisk -l
    ...
    Disk /dev/vdb: 21.5 GB, 21474836480 bytes, 41943040 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    ...

    ##use /dev/vdb
    pvcreate /dev/vdb
    vgcreate cinder-volumes /dev/vdb

Install Packstack 2 way
***********************
การติดตั้ง Openstack ด้วย packstack เป็นการติดตั้งบน Redhat, CentOS7, Fedora โดยมีเบื้องหลังการ
ติดตั้งโดยการใช้ puppet module สามารถติดตั้ง packstack ได้ 2
Method1
-------
ติดตั้งผ่าน repository::

    sudo su -
    yum install -y openstack-packstack

Method2
-------
ติดตั้งผ่าน source code (https://github.com/openstack/packstack)::

    sudo su -
    yum install git -y
    git clone git://github.com/openstack/packstack.git

    ##Follow activity
    cd packstack
    git log
    git checkout -b mystack

    #install python dependency
    yum install python-pip python-devel -y
    yum groupinstall "Development Tools" -y
    yum install libffi-devel openssl-devel

    #install
    python setup.py install
    ...
    Using /usr/lib/python2.7/site-packages
    Finished processing dependencies for packstack===8.0.0.0rc1.dev114.gae579f6

    #copy module packstack ไปยัง puppet
    ls
    cp -r packstack/puppet/modules/packstack /usr/share/openstack-puppet/modules

หลังจากติดตั้ง packstack ทั้งสองวิธีแล้ว จะมี คำสั่ง ``packstack`` สำหรับการติดตั้ง openstack
โดยจะสร้าง answerfile มาแล้วทำการแก้ไข::

    #go back to /root
    cd ~

    ## generate answer file with
    packstack --gen-answer-file "answer-$(date +%b-%d-%y).txt"
    ls  answer-*.txt

    ## packstack จะใช้ ip ของ eth0 เป็น ip ของ Management ip ของ openstack แต่เราจะใช้
    ## ip ของ eth1 แทน
    ## check ip in answerfile

    grep HOSTS answer-Jul-21-16.txt
    CONFIG_COMPUTE_HOSTS=192.168.121.9
    CONFIG_NETWORK_HOSTS=192.168.121.9

    ## ดูค่า ip ของ eth0, eth1

    ip -4 a show eth0 | awk '/inet/ {print $2;}'
    192.168.121.9/24
    ip -4 a show eth1 | awk '/inet/ {print $2;}'
    10.0.0.10/24

    ## การแก้ไขด้วยการใช้คำสั่ง ``sed``

    sed -i.orig s/192.168.121.9/10.0.0.10/g  answer-Jul-21-16.txt

Edit Packstack Config
*********************
ไฟล์ answerfile นี้ สามารถแก้ไข และ run ซ้ำได้ แต่ห้าม generate ใหม่
::

	  ## ตัวอย่าง
    grep -n ADMIN_PW  answerfile001.txt
    vim  answerfile001.txt +(line no)

    CONFIG_KEYSTONE_ADMIN_PW=password
    CONFIG_LBAAS_INSTALL=y
    CONFIG_NEUTRON_METERING_AGENT_INSTALL=y
    CONFIG_NEUTRON_FWAAS=y

    CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vlan
    CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=vlan
    CONFIG_NEUTRON_ML2_VLAN_RANGES=physnet2:1:1000

    CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=physnet2:br-eth2
    CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-ex:eth0,br-eth2:eth2
    CONFIG_HEAT_INSTALL=y

    CONFIG_HEAT_CFN_INSTALL=y
    CONFIG_TROVE_INSTALL=y
    CONFIG_HORIZON_SSL=y
    CONFIG_PROVISION_DEMO=n

การแก้ไขค่าจะใช้ crudini เป็นตัวช่วย::

    answerfile=answer-Jul-21-16.txt
    crudini --set $answerfile general CONFIG_KEYSTONE_ADMIN_PW password
    crudini --set $answerfile general CONFIG_LBAAS_INSTALL y
    crudini --set $answerfile general CONFIG_NEUTRON_METERING_AGENT_INSTALL y
    crudini --set $answerfile general CONFIG_NEUTRON_FWAAS y

    crudini --set $answerfile general CONFIG_NEUTRON_ML2_TYPE_DRIVERS vlan,vxlan,gre,flat,local
    crudini --set $answerfile general CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES local,vlan,gre,vxlan

    crudini --set $answerfile general CONFIG_NEUTRON_ML2_VLAN_RANGES physnet2:1:1000

    crudini --set $answerfile general CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS ext-net:br-ex,physnet2:br-eth2
    crudini --set $answerfile general CONFIG_NEUTRON_OVS_BRIDGE_IFACES br-ex:eth0,br-eth2:eth2

    crudini --set $answerfile general CONFIG_HEAT_INSTALL y
    crudini --set $answerfile general CONFIG_TROVE_INSTALL y

    crudini --set $answerfile general CONFIG_HEAT_CFN_INSTALL y
    crudini --set $answerfile general CONFIG_HORIZON_SSL y
    crudini --set $answerfile general CONFIG_PROVISION_DEMO n
    crudini --set $answerfile general CONFIG_CINDER_VOLUMES_CREATE n

Install openstack puppet module
-------------------------------
::

    export GEM_HOME=/tmp/somedir
    gem install r10k
    ## go to packstacksource
    cd ~/packstack
    /tmp/somedir/bin/r10k puppetfile install -v

จะเป็นการติดตั้ง puppet module
::

    INFO	 -> Updating module /usr/share/openstack-puppet/modules/aodh
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/ceilometer
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/cinder
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/glance
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/gnocchi
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/heat
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/horizon
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/ironic
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/keystone
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/manila
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/neutron
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/nova
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/openstack_extras
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/openstacklib
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/oslo
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/sahara
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/swift
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/tempest
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/trove
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/vswitch
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/apache
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/certmonger
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/concat
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/firewall
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/inifile
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/memcached
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/mongodb
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/mysql
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/nssdb
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/rabbitmq
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/redis
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/remote
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/rsync
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/ssh
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/stdlib
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/sysctl
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/vcsrepo
    INFO	 -> Updating module /usr/share/openstack-puppet/modules/xinetd

copy module::

    cp -r packstack/puppet/modules/packstack /usr/share/openstack-puppet/modules

Run
::

    cd /etc/pki/tls/certs/
    openssl req -x509 -sha256 -newkey rsa:2048 -keyout certificate.key -out selfcert.crt -days 1024 -nodes
    ## answer question
    Country Name (2 letter code) [XX]:TH
    State or Province Name (full name) []:Bangkok
    Locality Name (eg, city) [Default City]:Bangkok
    Organization Name (eg, company) [Default Company Ltd]:MyOpenstack
    Organizational Unit Name (eg, section) []:ITDepartment
    Common Name (eg, your name or your server's hostname) []:controller.example.com
    Email Address []:admin@example.com
    Country Name (2 letter code) [XX]:TH
    State or Province Name (full name) []:Bangkok
    Locality Name (eg, city) [Default City]:Bangkok
    Organization Name (eg, company) [Default Company Ltd]:MyOpenstack
    Organizational Unit Name (eg, section) []:ITDepartment
    Common Name (eg, your name or your server's hostname) []:controller.example.com
    Email Address []:admin@example.com

    mv certificate.key /etc/pki/tls/private/selfkey.key
    mkdir -p ~/packstackca/certs/
    cp ssl_vnc.crt  ~/packstackca/certs/10.0.0.10ssl_vnc.crt


    packstack --answer-file answer-Jul-21-16.txt


.. image:: _images/openstack-two-machine-two-nic.png


ผลการ Run

.. image:: _images/openstack001.png


Network Management
------------------
.. note::
  เนื่องจากเป็นการทดสอบบน vagrant จึงใช้ eth0 สำหรับการเชื่อมต่อ internet เท่านั้น และใช้ eth1
  เป็น management network และ external network รวมกัน

packstack จะทำหน้าที่สร้าง ระบบโครงสร้าง virtual network (ovs-system)ให้ ได้แก่ bridge ชื่อ
ิbr-ex, br-int, br-tun และ เราจะต้องเชื่อมต่อ bridge นี้กับ interface จริง (physical interface)
::

  # openvswitch command
  ovs-vsctl show

backup::

	cp /etc/sysconfig/network-scripts/ifcfg-eth1  /root
	cp /etc/sysconfig/network-scripts/ifcfg-eth1  /etc/sysconfig/network-scripts/ifcfg-br-ex
	cd /etc/sysconfig/network-scripts/

edit ค่าของ interface eth1::

  vi ifcfg-eth1

	ONBOOT=yes
	DEVICE=eth1
	HWADDR=52:54:00:95:c4:b4
	TYPE=OVSPort
	DEVICETYPE=ovs
	OVS_BRIDGE=br-ex

	vi ifcfg-br-ex

	DEVICE=br-ex
	BOOTPROTO=static
	ONBOOT=yes
	TYPE=OVSBridge
	DEVICETYPE=ovs
	USERCTL=yes
	PEERDNS=yes
	IPV6INIT=no
	IPADDR=10.0.0.10
	NETMASK=255.255.255.0
	GATEWAY=192.168.121.1
	DNS1=8.8.8.8

restart::

	systemctl restart network
	ovs-vsctl show
	(show result  br-ex <--> eth1
	4b34f849-95d8-4651-bbae-40e05d088012
		Bridge br-ex
		    Port "eth1"
		        Interface "eth1"
		    Port br-ex
		        Interface br-ex
		            type: internal


	ip a s eth1
	(eth1 no ip)
	3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master ovs-system state UP qlen 1000
		link/ether 52:54:00:95:c4:b4 brd ff:ff:ff:ff:ff:ff
		inet6 fe80::5054:ff:fe95:c4b4/64 scope link
		   valid_lft forever preferred_lft forever

	ip a s br-ex
	(br-ex have ip)
	12: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN
		link/ether ce:d5:be:2d:03:40 brd ff:ff:ff:ff:ff:ff
		inet 10.0.0.10/24 brd 10.0.0.255 scope global br-ex
		   valid_lft forever preferred_lft forever
		inet6 fe80::ccd5:beff:fe2d:340/64 scope link
		   valid_lft forever preferred_lft forever

sethostname::

	hostnamectl set-hostname controller.example.com



upload image
------------
(packstack จะสร้าง ไฟล์ keystonerc_admin ใช้สำหรับการ login ทาง commandline)
::

  source keystonerc_admin
  curl http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img | glance \
         image-create --name='cirros image' --visibility=public --container-format=bare --disk-format=qcow2

	+------------------+--------------------------------------+
	| Property         | Value                                |
	+------------------+--------------------------------------+
	| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
	| container_format | bare                                 |
	| created_at       | 2016-07-06T06:30:13Z                 |
	| disk_format      | qcow2                                |
	| id               | 52835f4d-90fc-4dfd-85bd-d56a4c886ed7 |
	| min_disk         | 0                                    |
	| min_ram          | 0                                    |
	| name             | cirros image                         |
	| owner            | 4b2f4b8359614a2d930802d428cef551     |
	| protected        | False                                |
	| size             | 13287936                             |
	| status           | active                               |
	| tags             | []                                   |
	| updated_at       | 2016-07-06T06:30:42Z                 |
	| virtual_size     | None                                 |
	| visibility       | public                               |
	+------------------+--------------------------------------+


centos7 image::

	curl http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1606.qcow2 | glance image-create --name='centos7 image' --visibility=public --container-format=bare --disk-format=qcow2
