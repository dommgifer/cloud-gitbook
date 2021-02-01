---
description: 這篇超級古文，沒什麼參考價值，現在都使用 kolla 佈署跟升級 OpenStack，只是發現以前有記錄過程，所以搬上來
---

# OpenStack Upgrade 測試

{% hint style="danger" %}
本文紀錄時間為2018/01/22，看看就好，不要太認真，之前被公司要求如何升版，所以才做這測試
{% endhint %}

測試從Mitaka手動升級到Newton

## 環境

OS: Ubuntu 14.04

OpenStack version: Mitaka

Node:

* 1台 controller 
* 1台 compute
* 1台 storage

## All node

1. 所有節點更新OS至ubuntu 16.04，安裝都選第一個選項跟Y
2. 備份所有節點的config檔 
3. 備份資料庫\(controller\)

```text
# mysqldump -u root -p –opt –add-drop-database –all-databases > mitaka-db-backup.sql
```

* Enable the OpenStack repository

  ```text
  # add-apt-repository cloud-archive:newton
  ```

* Upgrade the packages on your host

  ```text
  # apt update
  ```

* Upgrade the OpenStack client

  ```text
  # apt install --only-upgrade python-openstackclient
  ```



## Controller node

### keystone

* Stop Keystone service

  ```text
  # service keystone stop
  ```

* Upgrade Keystone service

  ```text
  # apt install --only-upgrade keystone -y
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/keystone-install.html#install-and-configure-components)
* Restart the Apache service

  ```text
  # systemctl daemon-reload 
  # service keystone start
  ```

* Create keystone database

  ```text
  $ mysql -u root -p

  # DROP databases keystone;
  # CREATE databases keystone;
  ```

* Populate the Identity service database:

  ```text
  # su -s /bin/sh -c "keystone-manage db_sync" keystone 
  ```

* Import keystone db

  ```text
  # mysql -u root -p keystone < mitaka-db-backup.sql
  ```

* Create the service project:

  ```text
  # openstack project create --domain default --description "Service Project" service
  ```

* Initialize Fernet key repositories

  ```text
  # keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
  ```

* Mysql restart

  ```text
  # service mysql restart
  ```

* Backup wsgi-keystone.conf

  ```text
  # mv /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-available/wsgi-keystone.bak
  ```

* Remove site-enable

  ```text
  # rm /etc/apache2/sites-enabled
  ```

* Apache2 restart

  ```text
  # service apache2 reload 
  ```



### glance

* Stop all glance service

  ```text
  # for i in /etc/init.d/glance-*;do $i stop; done
  ```

* Upgrade glance service

  ```text
  # apt install --only-upgrade glance -y
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/glance-install.html#install-and-configure-components)
* Create glance database

  ```text
  # mysql -u root -p

  # DROP databases glance ;
  # CREATE databases glance ;
  ```

* Populate the Image service database:

  ```text
  # su -s /bin/sh -c "glance-manage db_sync" glance 
  ```

* Import glance database

  ```text
  # mysql -u root -p glance  < mitaka-db-backup.sql
  ```

* Start glance all service

  ```text
  # for i in /etc/init.d/glance-*;do $i start; done
  ```

* Confirm upload of the image and validate attributes:

  ```text
  # glance image-list 
  ```

{% hint style="warning" %}
有可能出現錯誤:\(錯誤:500 Internal Server Error: The server has either erred or is incapable of performing the requested operation.\) CantStartEngineError: No sql\_connection parameter is established 或\(An unexpected error prevented the server from fulfilling your request. \(HTTP 500\)
{% endhint %}

### nova

* Stop nova all service

  ```text
  # for i in /etc/init.d/nova-*;do $i stop; done
  ```

* Upgrade nova all service

  ```text
  # apt install --only-upgrade nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler -y
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/nova-controller-install.html#install-and-configure-components)
* Create nova database

  ```text
  # mysql -u root -p
  # DROP DATABASE nova_api;
  # DROP DATABASE nova;
  # CREATE DATABASE nova_api;
  # CREATE DATABASE nova;
  ```

* Populate the Compute databases:

  ```text
  # su -s /bin/sh -c "nova-manage api_db sync" nova
  # su -s /bin/sh -c "nova-manage db sync" nova
  ```

* Import nova database

  ```text
  # mysql -u root -p nova  < mitaka-db-backup.sql
  # mysql -u root -p nova_api  < mitaka-db-backup.sql
  ```

* Start nova all service

  ```text
  # for i in /etc/init.d/nova-*;do $i start; done
  ```



### neutron

* Stop nova all service

  ```text
  # for i in /etc/init.d/neutron-* ;do $i stop; done
  ```

* Upgrade neutron all service

  ```text
  # apt install --only-upgrade neutron-server neutron-plugin-ml2 \
    neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
    neutron-metadata-agent -y 
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/neutron-controller-install.html#configure-networking-options)
* Create neutron database

  ```text
  # mysql -u root -p
  # DROP DATABASE neutron;
  # CREATE DATABASE neutron;
  ```

* Populate the Neutron databases

  ```text
  su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
    --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
  ```

* Import neutron database

  ```text
  # mysql -u root -p neutron  < mitaka-db-backup.sql
  ```

* Start neutron all service

  ```text
  # for i in /etc/init.d/neutron-*; do $i start; done
  # service nova-api restart
  ```



### cinder

* Stop cinder all service

  ```text
  for i in /etc/init.d/cinder-* ;do $i stop; done
  ```

* Upgrade cinder all service

  ```text
  apt install --only-upgrade cinder-api cinder-scheduler
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/cinder-controller-install.html#install-and-configure-components)
* Create cinder database

  ```text
  # mysql -u root -p
  # DROP DATABASE cinder;
  # CREATE DATABASE cinder;
  ```

* Populate the cinder databases

  ```text
  su -s /bin/sh -c "cinder-manage db sync" cinder
  ```

* Import cinder database

  ```text
  mysql -u root -p cinder  < mitaka-db-backup.sql
  ```

* Start cinder all service

  ```text
  # for i in /etc/init.d/cinder-*; do $i start; done
  # service nova-api restart
  ```



### horizon

* Upgrade horizon service

  ```text
  # apt install --only-upgrade openstack-dashboard -y
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/horizon-install.html#install-and-configure-components)
* Restart apache2 service

  ```text
  service apache2 reload
  ```



## Compute node

### nova

* Stop nova-compute stop

  ```text
  # service nova-compute stop
  ```

* Upgrade nova-compute

  ```text
  # apt install --only-upgrade nova-compute -y
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/nova-compute-install.html#install-and-configure-components)
* Start nova-compute service

  ```text
  # service nova-compute start
  ```

### neutron

* Stop neutron-linuxbridge-agent stop

  ```text
  # service neutron-linuxbridge-agent stop
  ```

* Upgrade neutron-linuxbridge-agent

  ```text
  # apt install --only-upgrade neutron-linuxbridge-agent -y
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/neutron-compute-install.html#configure-the-common-component)
* Start nova-compute and neutron-linuxbridge-agent service

  ```text
  # service nova-compute start
  # service neutron-linuxbridge-agent start
  ```



## Volume node

* Stop cinder-volume stop

  ```text
  # service cinder-volume stop
  ```

* Upgrade cinder-volume

  ```text
  # apt install --only-upgrade cinder-volume -y
  ```

* [按照官網改config](https://docs.openstack.org/newton/install-guide-ubuntu/cinder-storage-install.html#install-and-configure-components)
* Start cinder-volume tgt service

  ```text
  # service tgt restart
  # service cinder-volume restart
  ```

## 備註

以下列出遇到的問題

1. nova list噴錯:

   ```text
   ClientException: Unexpected API Error. Please report this at http://bugs.launchpad.net/nova/ and attach the Nova API log if possible.
   [Wed May 24 13:59:15.475574 2017] [wsgi:error] [pid 8013:tid 140130413639424] <class 'oslo_db.exception.DBError'> (HTTP 500) (Request-ID: req-d786ae6a-f413-4654-b8d8-0297a0c5f52b)
   ```

2. LOG如果出現:DBError: \(pymysql.err.InternalError\) \(1054, u“Unknown column 'XXXXXXXXXXX 'field list'”\) 請重新sync db

   ```text
   # su -s /bin/sh -c "xxxxx-manage db_sync" xxxxx
   ```

3. 建立VM噴錯 因為compute上的nova-compute有問題，只要建立VM，nova-compute就會死掉:

   ```text
   Failed to perform requested operation on instance "test", the instance has an error status: Please try again later [Error: No valid host was found. There are not enough hosts available.].
   ```

4. 無法進入dashboard，查看apache2 error.log

ImportError: cannot import name security_group_rules 然後在controller輸入 openstack flavor list，出現下錯誤

```text
Unable to establish connection to http://controller:8774/v2.1/3fa5fc21995f4bf2b2da8d3073d9b819/flavors/detail: HTTPConnectionPool(host='controller', port=8774): Max retries exceeded with url: /v2.1/3fa5fc21995f4bf2b2da8d3073d9b819/flavors/detail (Caused by NewConnectionError('<requests.packages.urllib3.connection.HTTPConnection object at 0x7f613c181a90>: Failed to establish a new connection: [Errno 111] Connection refused',))
```

之後查看nova所有服務，發現狀態為dead，重啟後為active，但是再下一次openstack flavorlist或重整網頁，nova所有服務又死了  


