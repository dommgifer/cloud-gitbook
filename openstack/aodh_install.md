---
description: 此篇說明如何安裝 aodh 服務
---

# Aodh Install

{% hint style="info" %}
以下範例為 ubuntu queens 版本
{% endhint %}

## 建立資料庫

進入資料庫後，建立資料庫和使用者 

```
# mysql -u root -p
MariaDB [(none)]> CREATE DATABASE aodh;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'localhost' \ 
IDENTIFIED BY 'openstack';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'%' \
 IDENTIFIED BY 'openstack';

```

## 建立 Aodh 服務

建立 OpenStack 的 aodh service

{% hint style="info" %}
請將 $ALL\_HOST 換成自己環境的 controller IP
{% endhint %}

```
# openstack user create --domain default --password-prompt aodh 
# openstack role add --project service --user aodh admin
# openstack service create --name aodh --description "Telemetry" alarming
# openstack endpoint create --region RegionOne alarming  public http://$ALL_HOST:8042
# openstack endpoint create --region RegionOne alarming  internal http://$ALL_HOST:8042
# openstack endpoint create --region RegionOne alarming  admin http://$ALL_HOST:8042
```

## 安裝 aodh

使用 apt install 安裝 aodh

```text
apt install -y aodh-api aodh-evaluator aodh-notifier \
  aodh-listener aodh-expirer python-aodhclient

```

## 修改 aodh.conf

修改 /etc/aodh/aodh.conf

{% hint style="info" %}
根據你的環境，修改下面參數
{% endhint %}

```text
[DEFAULT]
transport_url = rabbit://openstack:openstack@10.50.2.10
auth_strategy = keystone

[database]
connection = mysql+pymysql://openstack:openstack@10.50.2.10/aodh

[keystone_authtoken]
auth_uri = http://10.50.2.10:5000
auth_url = http://10.50.2.10:5001
memcached_servers = 10.50.2.10:11211
auth_type = password
project_domain_id = 33464b562a3941bd826231660e1f7d0b
user_domain_id = 33464b562a3941bd826231660e1f7d0b
project_name = service
username = aodh
password = openstack

[service_credentials]
auth_type = password
auth_url = http://10.50.2.10:5000/v3
project_domain_id = 33464b562a3941bd826231660e1f7d0b
user_domain_id = 33464b562a3941bd826231660e1f7d0b
project_name = service
username = aodh
password = openstack
interface = internalURL
region_name = RegionOne

```

## 建立資料表

使用 aodh-dbsync 建立 aodh 的資料表

```text
aodh-dbsync
```

## 重啟服務

{% hint style="info" %}
Aodh-api 已改用 WSGI，所以需要重啟 apache2
{% endhint %}

```text
# service apache2 restart
# service aodh-evaluator restart
# service aodh-notifier restart
# service aodh-listener restart
```



