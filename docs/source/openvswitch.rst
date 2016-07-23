=====================================
Understand Linux Bridge & Openvswitch
=====================================

***********
Openvswitch
***********
openstack จะมี bridge อยู่ 2 ชนิด คือ bridge ที่สร้าง linux bridge และ bridge ที่สร้างจาก openvswitch
โดย linux bridge ทำหน้าเป็น network l2 broadcast domain ให้ แก่ vm มาเกาะ และ linux bridge
ก็จะไปเชื่อมต่อกับ br-int ที่สร้างโดย openvswitch



Openvswitch Install
===================

1). prepare vagrant::

  mkdir openvswitch
  vagrant init centos/7
  vagrant up
  vagrant ssh
  sudu su -

2). Install requisite packages::

  yum -y install wget gcc make python-devel openssl-devel kernel-devel graphviz kernel-debug-devel autoconf automake rpm-build redhat-rpm-config libtool
  mkdir -p ~/rpmbuild/SOURCES
  wget http://openvswitch.org/releases/openvswitch-2.5.0.tar.gz
  tar xvf openvswitch-2.5.0.tar.gz
  cd openvswitch-2.5.0

3). Build RPM package::

  ls
  openvswitch-2.5.0  openvswitch-2.5.0.tar.gz

  rpmbuild -bb --nocheck openvswitch-2.5.0/rhel/openvswitch.spec
  ls ~/rpmbuild/RPMS/x86_64/

  openvswitch-2.5.0-1.x86_64.rpm  openvswitch-debuginfo-2.5.0-1.x86_64.rpm

  yum localinstall ~/rpmbuild/RPMS/x86_64/openvswitch-2.5.0-1.x86_64.rpm

  systemctl enable openvswitch
  systemctl start openvswitch
  lsmod | grep openvswitch

  openvswitch            84543  0
  libcrc32c              12644  1 openvswitch

4). Test OVS::

  ovs-vsctl -V
  ##
  ovs-vsctl (Open vSwitch) 2.5.0
  Compiled Jul 23 2016 06:50:04
  DB Schema 7.12.1

  ovs-vsctl show
  ##
  7369b72b-22a1-4d0a-b747-0ca20f3b55ec
      ovs_version: "2.5.0"
