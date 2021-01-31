---
description: 此篇說明如何安裝 ceilometer  和 gnocchi
---

# Ceilometer and Gnocchi Install

## 注意事項

在安裝過程發現 Queens 版官方的套件有問題，導致在安裝 gnocchi-api 時會有python2 和 python3 套件衝突，導致刪除 cinder-api、 keystone、libapache2-mod-wsgi、 nova-placement-api、 openstack-dashboard，因此需要加入新的 repository

Ubuntu 16.04版本要安裝 gnocchi-api 需要加入

```text
add-apt-repository cloud-archive:queens-proposed
```

{% hint style="info" %}
官方 BUG: : [https://bugs.launchpad.net/ubuntu/+source/gnocchi/+bug/1746992](https://bugs.launchpad.net/ubuntu/+source/gnocchi/+bug/1746992)
{% endhint %}

## 建立資料庫

請注意你的使用者和密碼

```text
# mysql -u root -p
MariaDB [(none)]> CREATE DATABASE gnocchi;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'localhost' \
  IDENTIFIED BY 'GNOCCHI_DBPASS';
  
MariaDB [(none)]> GRANT ALL PRIVILEGES ON gnocchi.* TO 'gnocchi'@'%' \
  IDENTIFIED BY 'GNOCCHI_DBPASS';
```

## 建立服務

建立 OpenStack gnocchi 和 ceilometer 使用者及服務

{% hint style="info" %}
請將  $ALL\_HOST 改為 controller IP
{% endhint %}

```text
# openstack user create --domain default --password-prompt ceilometer
# openstack user create --domain default --password-prompt gnocchi

# openstack role add --project service --user ceilometer admin
# openstack role add --project service --user gnocchi admin

# openstack service create --name gnocchi --description "Metric Service" metric

# openstack endpoint create --region RegionOne metric public http://$ALL_HOST:8041
# openstack endpoint create --region RegionOne metric internal http://$ALL_HOST:8041
# openstack endpoint create --region RegionOne metric admin http://$ALL_HOST:8041
```

## 修改 gnocchi.conf

修改 /etc/gnocchi/gnocchi.conf

{% hint style="info" %}
根據你的環境，修改下面參數
{% endhint %}

```text
[DEFAULT]
transport_url = rabbit://openstack:openstack@192.168.1.10

[api]
auth_mode = keystone
middlewares = oslo_middleware.cors.CORS
middlewares = keystonemiddleware.auth_token.AuthProtocol
auth_mode = keystone

[indexer]
url = mysql+pymysql://openstack:openstack@192.168.1.10/gnocchi

[keystone_authtoken]
auth_host = http://192.168.1.10:5000/v3/
auth_protocol = http
admin_user = admin
admin_password = openstack
admin_tenant_name = admin
auth_url = http://192.168.1.10:5000/v3
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = gnocchi
password = openstack
interface = internalURL
region_name = RegionOne

[storage]
coordination_url = file:///var/lib/gnocchi/locks
file_basepath = /var/lib/gnocchi
driver = file

```

## 新增 gnocchi-api.conf

新增 /etc/apache2/sites-available/gnocchi-api.conf

```text
Listen 8041

<VirtualHost *:8041>
    WSGIDaemonProcess gnocchi processes=2 threads=10 user=gnocchi display-name=%{GROUP}
    WSGIProcessGroup gnocchi
    WSGIScriptAlias / /usr/bin/gnocchi-api
    WSGIApplicationGroup %{GLOBAL}
    <IfVersion >= 2.4>
        ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/gnocchi_error.log
    CustomLog /var/log/apache2/gnocchi_access.log combined
    <Directory />
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

```

## 建立 gnocchi 資料表

使用 gnocchi-upgrade 建立資料表

```text
# gnocchi-upgrade
```

## 重啟 gnocchi 服務

```text
# service apache2 restart
# service gnocchi-metricd restart
```

## 修改 /var/lib/gnocchi 權限

請將 /var/lib/gnocchi 權限設為 gnocchi

```text
# chown -R gnocchi:gnocchi /var/lib/gnocchi
```

## 修改 admin-openrc

如果要使用 gnocchi 的指令，需要將 admin-openrc 加入以下參數，才能使用指令

```text
export OS_AUTH_TYPE=password
```

## 安裝 ceilometer

### Controller 節點

```text
# apt install -y ceilometer-agent-notification ceilometer-agent-central
```

#### 修改 ceilometer.conf

{% hint style="info" %}
跟據環境，修改以下參數
{% endhint %}

修改 /etc/ceilometer/ceilometer.conf

```text
[DEFAULT]
transport_url = rabbit://openstack:openstack@192.168.1.10

[dispatcher_gnocchi]
filter_service_activity = False
archive_policy = high

[service_credentials]
auth_url = http://10.50.2.10:5000/v3
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = ceilometer
password = openstack
interface = internalURL
region_name = RegionOne
```

#### 建立 ceilometer resource

在 gnocchi 裡建立 ceilometer 的 resource

```text
# ceilometer-upgrade
```

#### 重啟 ceilometer 服務

```text
# service ceilometer-agent-central restart
# service ceilometer-agent-notification restart
```

#### 修改 glance-api.conf 和 glance-registry.conf

修改 /etc/glance/glance-api.conf 和 /etc/glance/glance-registry.conf

```text
[DEFAULT]
transport_url = rabbit://openstack:openstack@192.168.1.10

[oslo_messaging_notifications]
driver = messagingv2

```

#### 修改 neutron.conf

修改 /etc/neutron/neutron.conf

```text
[oslo_messaging_notifications]
driver = messagingv2
```

#### 修改 cinder.conf

修改 /etc/cinder/cinder.conf \(volume節點上的也需要修改\)

```text
[oslo_messaging_notifications]
driver = messagingv2
```

#### 修改 heat.conf

修改 /etc/heat/heat.conf

```text
[oslo_messaging_notifications]
driver = messagingv2
```

#### 重啟上述服務

```text
# service glance-registry restart
# service glance-api restart

# service neutron-server restart

# service apache2 restart
# service cinder-api restart
# service cinder-scheduler restart
# service cinder-volume restart

# service heat-api restart
# service heat-api-cfn restart
# service heat-engine restart
```

### Compute 節點

安裝 ceilometer-agent-compute

```text
# apt-get install ceilometer-agent-compute
```

#### 修改 ceilometer.conf

{% hint style="info" %}
根據環境，修改以下參數
{% endhint %}

修改 /etc/ceilometer/ceilometer.conf

```text
[DEFAULT]
transport_url = rabbit://openstack:openstack@192.168.1.10
[service_credentials]
auth_url = http://10.50.2.10:5000/v3
auth_type = password
project_domain_id = default
user_domain_id = default
project_name = service
username = ceilometer
password = openstack
interface = internalURL
region_name = RegionOne

```

#### 修改 nova.conf

修改 /etc/nova/nova.conf

```text
[DEFAULT]
instance_usage_audit = True
instance_usage_audit_period = hour
notify_on_state_change = vm_and_task_state

[oslo_messaging_notifications]
driver = messagingv2
```

#### 重啟服務

```text
# service ceilometer-agent-compute restart
# service nova-compute restart
```

## 修改 polling.yaml

為了監測 VM 的 disk，增加下列至 /etc/ceilometer/polling.yaml 的 some\_pollsters

```text
        - disk.usage
        - disk.device.usage
        - disk.device.read.bytes
        - disk.device.read.requests
        - disk.device.write.bytes
        - disk.device.write.requests
```

## 其他事項

監測的間隔預設為5分鐘，政策為 low，可以使用 gnocchi 查看其他監測的政策，預設有三種: low, medium , high ，granularity 為間隔時間，也可以建立自己的政策，目前尚未嘗試。

```text
# gnocchi archive-policy list
+--------+-------------+-----------------------------------------------------------------------+---------------------------------+
| name   | back_window | definition                                                            | aggregation_methods             |
+--------+-------------+-----------------------------------------------------------------------+---------------------------------+
| bool   |        3600 | - points: 31536000, granularity: 0:00:01, timespan: 365 days, 0:00:00 | last                            |
| high   |           0 | - points: 3600, granularity: 0:00:01, timespan: 1:00:00               | std, count, min, max, sum, mean |
|        |             | - points: 10080, granularity: 0:01:00, timespan: 7 days, 0:00:00      |                                 |
|        |             | - points: 8760, granularity: 1:00:00, timespan: 365 days, 0:00:00     |                                 |
| low    |           0 | - points: 8640, granularity: 0:05:00, timespan: 30 days, 0:00:00      | std, count, min, max, sum, mean |
| medium |           0 | - points: 10080, granularity: 0:01:00, timespan: 7 days, 0:00:00      | std, count, min, max, sum, mean |
|        |             | - points: 8760, granularity: 1:00:00, timespan: 365 days, 0:00:00     |                                 |
+--------+-------------+-----------------------------------------------------------------------+---------------------------------+

```

若要修改間隔，請修改 ceilometer.conf 的 \[dispatcher\_gnocchi\]，以及/etc/ceilometer/polling.yaml

{% code title="ceilometer.conf" %}
```text
[dispatcher_gnocchi]
archive_policy = high
```
{% endcode %}

polling.yaml \(interval原為300，compute上的也要修改\)

{% code title="polling.yaml" %}
```text
sources:
    - name: some_pollsters
      interval: 60
```
{% endcode %}

conteroller 重啟服務

```text
# service ceilometer-agent-central restart
# service ceilometer-agent-notification restart
```

compute 重啟服務

```text
# service ceilometer-agent-compute restart
```

透過 gnocchi 的指令，可以取得 VM 的監控值，用 resource show --type instance VM\_UUID， 可以得到此 VM 的各資源監測的 ID

```text
# gnocchi resource show --type instance 7e157c6a-7b2a-4893-8819-bfc033e37f39
+-----------------------+---------------------------------------------------------------------+
| Field                 | Value                                                               |
+-----------------------+---------------------------------------------------------------------+
| created_by_project_id | 6b53bd8ecb01408d8d76a48ae3878723                                    |
| created_by_user_id    | 3942272c165f4c258663ed130e8171e2                                    |
| creator               | 3942272c165f4c258663ed130e8171e2:6b53bd8ecb01408d8d76a48ae3878723   |
| display_name          | test                                                                |
| ended_at              | None                                                                |
| flavor_id             | 2                                                                   |
| flavor_name           | m1.small                                                            |
| host                  | compute1                                                            |
| id                    | 7e157c6a-7b2a-4893-8819-bfc033e37f39                                |
| image_ref             | None                                                                |
| metrics               | compute.instance.booting.time: 53c30c1e-1c84-4946-8171-b15afe0ddfbd |
|                       | cpu.delta: 732c0318-195c-4735-a0db-0866cfb88189                     |
|                       | cpu: 2a273f20-917c-46f4-95b7-934c57ebd148                           |
|                       | cpu_l3_cache: f032f4e9-8947-4096-9fcd-222e93b255ef                  |
|                       | cpu_util: 4a0d8c08-56b1-47f7-8dc9-3fa8d56c53df                      |
|                       | disk.allocation: 063b2800-e26e-45ae-8840-8d53976743b1               |
|                       | disk.capacity: df6d2a85-19ec-4a69-8409-b2d56222635e                 |
|                       | disk.ephemeral.size: c04d6e90-dca6-454c-8be8-29d2ca765b89           |
|                       | disk.iops: cc8605a9-6687-42fd-af92-1372ae9692dc                     |
|                       | disk.latency: 6d1205a6-8e6b-45ce-a4b5-a48f62b26880                  |
|                       | disk.read.bytes.rate: d9f5dbc8-24c5-40e5-88f4-0b9c36bfd197          |
|                       | disk.read.bytes: 8ff22fe3-c91a-41de-b7f6-06ce77098b79               |
|                       | disk.read.requests.rate: 7b820453-7455-40f2-8808-7437651fc351       |
|                       | disk.read.requests: 60e98d20-36b0-4fe8-adf3-86e827be15d4            |
|                       | disk.root.size: 8da968ca-9703-4625-b7c3-8f13460dae77                |
|                       | disk.usage: 8046e635-bfa6-45e5-bc28-a8ce8aefeefa                    |
|                       | disk.write.bytes.rate: b186f91d-f61c-40a3-8577-a0e443b92f7a         |
|                       | disk.write.bytes: 9c414ba1-597e-4bda-a154-c0131af03505              |
|                       | disk.write.requests.rate: 235445a2-d278-4a55-955b-ab1e6a5c567a      |
|                       | disk.write.requests: 8cf3b36b-c4f0-4b30-bcce-9463b89e2e3a           |
|                       | memory.bandwidth.local: 054870ba-06e6-473f-a37b-e7efcf9188e9        |
|                       | memory.bandwidth.total: ee244a7a-1532-40db-8fb1-96bfcfdee2d9        |
|                       | memory.resident: a045ff95-c7e5-4162-9345-ac68a2993981               |
|                       | memory.swap.in: 6162d2f2-6f6c-4fab-99e5-b4fbb9a8844e                |
|                       | memory.swap.out: a6d9846c-ffd9-471f-8175-4c900af55c99               |
|                       | memory.usage: 8b8706a2-d108-445c-8a22-c8dcab2d4950                  |
|                       | memory: 3d2e70c8-7fd2-4557-b30c-3090749c5014                        |
|                       | perf.cache.misses: dae14570-1c80-472b-9bb2-9818921d03cd             |
|                       | perf.cache.references: 0a294dea-decf-4707-8b4d-9ef92ffc2e87         |
|                       | perf.cpu.cycles: be19335c-9873-4c9d-9011-0e4ef9931ab4               |
|                       | perf.instructions: d5d99bcc-044f-4543-a27f-7526a35826b9             |
|                       | vcpus: 6e2b7fd0-3fbd-4481-843f-cd705408cb05                         |
| original_resource_id  | 7e157c6a-7b2a-4893-8819-bfc033e37f39                                |
| project_id            | 885c63821c3a43a78da157958eca3a99                                    |
| revision_end          | None                                                                |
| revision_start        | 2018-05-03T06:51:06.014722+00:00                                    |
| server_group          | None                                                                |
| started_at            | 2018-05-03T06:51:06.014643+00:00                                    |
| type                  | instance                                                            |
| user_id               | be23e663a2ef4873bbbfee411b421049                                    |
+-----------------------+---------------------------------------------------------------------+

```

再透過 gnocchi measures show METRICS\_ID，查詢上面的 metrics 的監測值，例如要查詢上述 VM 的 cpu\_util \(CPU 使用率\)

```text
# gnocchi measures show 4a0d8c08-56b1-47f7-8dc9-3fa8d56c53df
+---------------------------+-------------+-----------------+
| timestamp                 | granularity |           value |
+---------------------------+-------------+-----------------+
| 2018-05-03T13:24:00+00:00 |        60.0 |  0.149982089639 |
| 2018-05-03T13:25:00+00:00 |        60.0 |  0.166708385488 |
| 2018-05-03T13:26:00+00:00 |        60.0 |   0.14998932576 |
| 2018-05-03T13:27:00+00:00 |        60.0 |  0.149992996335 |
| 2018-05-03T13:28:00+00:00 |        60.0 |  0.133337124552 |
| 2018-05-03T13:29:00+00:00 |        60.0 |  0.166712370593 |
| 2018-05-03T13:30:00+00:00 |        60.0 |  0.150029974686 |
+---------------------------+-------------+-----------------+

```

若要取得 VM 網路的監測值，使用 gnocchi resource list --type instance\_network\_interface， 取得 ID

```text
# gnocchi resource list --type instance_network_interface | grep 7e157c6a-7b2a-4893-8819-bfc033e37f39
+--------------------------------------+----------------------------+----------------------------------+----------------------------------+-----------------------------------------------------------------------+----------------------------------+----------------------------------+----------------------------------+--------------+--------------------------------------+-------------------------------------------------------------------+----------------+
| id                                   | type                       | project_id                       | user_id                          | original_resource_id                                                  | started_at                       | ended_at                         | revision_start                   | revision_end | instance_id                          | creator                                                           | name           |
+--------------------------------------+----------------------------+----------------------------------+----------------------------------+-----------------------------------------------------------------------+----------------------------------+----------------------------------+----------------------------------+--------------+--------------------------------------+-------------------------------------------------------------------+----------------+

| 06d1109b-fc23-5369-947d-564aff231fc4 | instance_network_interface | 885c63821c3a43a78da157958eca3a99 | be23e663a2ef4873bbbfee411b421049 | instance-00000011-7e157c6a-7b2a-4893-8819-bfc033e37f39-tap594786b7-d9 | 2018-05-03T06:51:55.592433+00:00 | None                             | 2018-05-03T06:51:55.592459+00:00 | None         | 7e157c6a-7b2a-4893-8819-bfc033e37f39 | 3942272c165f4c258663ed130e8171e2:6b53bd8ecb01408d8d76a48ae3878723 | tap594786b7-d9 |

```

再使用 gnocchi resource show 取得監測資源 ID

```text
# gnocchi resource show --type instance_network_interface 06d1109b-fc23-5369-947d-564aff231fc4
+-----------------------+-----------------------------------------------------------------------+
| Field                 | Value                                                                 |
+-----------------------+-----------------------------------------------------------------------+
| created_by_project_id | 6b53bd8ecb01408d8d76a48ae3878723                                      |
| created_by_user_id    | 3942272c165f4c258663ed130e8171e2                                      |
| creator               | 3942272c165f4c258663ed130e8171e2:6b53bd8ecb01408d8d76a48ae3878723     |
| ended_at              | None                                                                  |
| id                    | 06d1109b-fc23-5369-947d-564aff231fc4                                  |
| instance_id           | 7e157c6a-7b2a-4893-8819-bfc033e37f39                                  |
| metrics               | network.incoming.bytes.rate: e8c3d083-0ede-4719-a88a-eca92b653fed     |
|                       | network.incoming.bytes: 92ccff55-2ba8-452a-9ca9-b11daf7a3e8c          |
|                       | network.incoming.packets.drop: 889d6986-dfc6-40c7-a28c-d2543616d059   |
|                       | network.incoming.packets.error: d3dc4693-b6cd-4c15-89e5-9bf92a2bcae0  |
|                       | network.incoming.packets.rate: c6422bd1-cf03-4cfc-8e79-edd37851ae00   |
|                       | network.incoming.packets: fd860305-7c1c-45fe-a3ef-a5e843f24e4b        |
|                       | network.outgoing.bytes.rate: 1431543c-91ae-4aed-b506-330f18a18805     |
|                       | network.outgoing.bytes: 4db6ea32-9ea3-4e7c-a881-40f0ef82203d          |
|                       | network.outgoing.packets.drop: c49c0019-5a1a-4582-8abd-d82608a111a8   |
|                       | network.outgoing.packets.error: 7e725029-d676-4c98-b1ea-d568d2bc80b0  |
|                       | network.outgoing.packets.rate: bc28b58f-b848-48d3-bfd6-c3995b09a50e   |
|                       | network.outgoing.packets: f7057ae3-0080-4f6f-8fdb-04b5129b7fd0        |
| name                  | tap594786b7-d9                                                        |
| original_resource_id  | instance-00000011-7e157c6a-7b2a-4893-8819-bfc033e37f39-tap594786b7-d9 |
| project_id            | 885c63821c3a43a78da157958eca3a99                                      |
| revision_end          | None                                                                  |
| revision_start        | 2018-05-03T06:51:55.592459+00:00                                      |
| started_at            | 2018-05-03T06:51:55.592433+00:00                                      |
| type                  | instance_network_interface                                            |
| user_id               | be23e663a2ef4873bbbfee411b421049                                      |
+-----------------------+-----------------------------------------------------------------------+
```

接下來用gnocchi measures show METRICS\_ID，根據上述的 ID 取得監測值

```text
# gnocchi measures show 06d1109b-fc23-5369-947d-564aff231fc4
+---------------------------+-------------+------------+
| timestamp                 | granularity |    value   |
+---------------------------+-------------+------------+
| 2018-05-04T07:00:00+00:00 |        60.0 |   208353.0 |
| 2018-05-04T07:01:00+00:00 |        60.0 |   209253.0 |
| 2018-05-04T07:02:00+00:00 |        60.0 |   210153.0 |
| 2018-05-04T07:03:00+00:00 |        60.0 |   211053.0 |
| 2018-05-04T07:04:00+00:00 |        60.0 |   211953.0 |
| 2018-05-04T07:05:00+00:00 |        60.0 |   212853.0 |
| 2018-05-04T07:06:00+00:00 |        60.0 |   213753.0 |
+---------------------------+-------------+------------+
```

更快的查詢方式，根據VM ID查詢，查詢 instance 相關監測值: \(resource-type 為 instance，最後面的參數為要查詢的 metrics，範例為 cpu\_util\)

```text
# gnocchi measures aggregation --resource-type instance --query 'instance_id="7e157c6a-7b2a-4893-8819-bfc033e37f39"' --granularity 60 --aggregation mean -m cpu_util
```

查詢 instance network interface 相關監測值:\(resource-type為 instance\_network\_interface，最後面的參數為要查詢的metrics，  範例為network.incoming.bytes

```text
# gnocchi measures aggregation --resource-type instance_network_interface  --query 'instance_id="7e157c6a-7b2a-4893-8819-bfc033e37f39"' --granularity 60 --aggregation mean -m network.incoming.bytes
```

