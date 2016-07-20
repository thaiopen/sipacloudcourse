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
    ping compute
    getenforce
    systemctl is-active NetworkManager
    systemctl is-active firewalld

Disk prepare for cinder
-----------------------
::

    fdisk -l
    pvcreate /dev/vdb
    vgcreate cinder-volumes /dev/vdb
    packstack --gen-answer-file  answerfile001.txt

    ## การแก้ไขด้วยการใช้คำสั่ง ``sed``
    sed -i.orig s/192.168.121.158/10.0.0.10/g  answerfile001.txt
    sed -i s/CONFIG_HEAT_INSTALL=n/CONFIG_HEAT_INSTALL=y/g answerfile001.txt

	  ## ตัวอย่าง
    grep -n ADMIN_PW  answerfile001.txt
    vim  answerfile001.txt +(line no)

    CONFIG_KEYSTONE_ADMIN_PW=passwd
    CONFIG_LBAAS_INSTALL=y
    CONFIG_NEUTRON_METERING_AGENT_INSTALL=y
    CONFIG_NEUTRON_FWAAS=y

    CONFIG_NEUTRON_ML2_TYPE_DRIVERS=vlan
    CONFIG_NEUTRON_ML2_TENANT_NETWORK_TYPES=vlan
    CONFIG_NEUTRON_ML2_VLAN_RANGES=physnet1:1:1000

    CONFIG_NEUTRON_OVS_BRIDGE_MAPPINGS=physnet1:br-eth2
    CONFIG_NEUTRON_OVS_BRIDGE_IFACES=br-eth2:eth2

    CONFIG_HEAT_CFN_INSTALL=y
    CONFIG_HORIZON_SSL=y
    CONFIG_PROVISION_DEMO=n

Download complete file :download:`answerfile001.txt <./_source/answerfile001.txt>`.

Run
---
::

    packstack --answer-file answerfile001.txt


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
