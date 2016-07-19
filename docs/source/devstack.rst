=======================
Openstack With DevStack
=======================

Devstack with Vagrant
=====================

.. literalinclude::  _source/Vagrantfile2

หลังจาก ที่เราได้ทำการ vagrant up server1 เรียบร้อยแล้ว vm นี้มีขนาด RAM  8 Gb เพื่อใช้การทดสอบ
openstack ทดสอบโดยการสร้าง Directory สำหรับการทดสอบ ``Devstack`` และ ให้ Download หรือ สร้าง
file Vagrant จากตัวอย่างด้านบน
::

  mkdir ~/Devstack
  cd ~/Devstack
  wget
  mv Vagrantfile2 Vagrantfile
  vagrant ssh server1
  sudo useradd -d /opt/stack stack
  sudo visudo
