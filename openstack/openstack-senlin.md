---
description: >-
  此專案為集群及服務，可將VM和Container作為集群，對於集群內的資源可以管理，規模可彈性伸縮、成員間負載均衡、被管對象和策略可定制等等。Auto
  scaling的部分也由senlin管理，透過簡單的yaml定義，不需寫得像heat模板般複雜，目前Auto scaling 可以使用 senlin 或
  Heat 來實現。
---

# OpenStack Senlin

{% hint style="danger" %}
由於此篇是由很久之前的個人紀錄搬至gitbook

內容已過時，僅供參考
{% endhint %}

{% hint style="warning" %}
使用 Auto scaling 前，請先安裝 ceilometer、gnocchi、aodh 專案，以下範例為 Queens 版。
{% endhint %}

## 安裝步驟 <a id="&#x5B89;&#x88DD;&#x6B65;&#x9A5F;"></a>

### 安裝服務

以下步驟在 Controller 執行。 確認 openstack sdk 版本是否為 0.17.2，若不是可以用 pip 安裝

```
# pip show openstacksdk
---
Metadata-Version: 2.1
Name: openstacksdk
Version: 0.17.2

# pip install openstacksdk==0.17.2
```

安裝 senlin 請至 pypi 查看版號

```text
# pip install senlin==6.0.0
```

建立資料夾及 user

```text
# groupadd -r senlin
# adduser --system selin senlin
# mkdir /etc/senlin /var/log/senlin
```

建立資料庫 \(注意 database 的 user\)

```text
#mysql -u root -p

CREATE DATABASE senlin;

GRANT ALL PRIVILEGES ON senlin.* TO 'admin'@'localhost' \ 
IDENTIFIED BY 'openstack';

GRANT ALL PRIVILEGES ON senlin.* TO 'admin'@'%' \
 IDENTIFIED BY 'openstack';
```

建立資料表

```text
# senlin-manage db_sync
```

設定 /etc/senlin/senlin.conf，根據環境替換 IP 和 hostname

```text
[DEFAULT]
log_dir = /var/log/senlin
transport_url = rabbit://openstack:openstack@controller1

[senlin_api]
bind_host = 10.50.2.4
bind_port = 8780

[receiver]
host = 10.50.2.4
port = 8780

[database]
connection = mysql+pymysql://senlin:openstack@10.50.2.3/senlin

[keystone_authtoken]
www_authenticate_uri = http://10.50.2.3:5000/v3
auth_url = http://10.50.2.3:5000
auth_type = password
project_name = service
username = senlin
password = openstack
project_domain_name = Default
user_domain_name = Default
memcached_servers = controller1:11211


[authentication]
auth_url = http://10.50.2.3:5000/v3
service_username = senlin
service_password = openstack
service_project_name = service
service_user_domain = Default
service_project_domain = Default

[oslo_messaging_notifications]
driver = messagingv2
transport_url = rabbit://openstack:openstack@10.50.2.14

[oslo_policy]
policy_file = /etc/senlin/policy.yaml
```

建立 user 和 Endpoint，根據環境替換 IP 和 hostname

```text
# openstack user create --domain default --password-prompt senlin
# openstack role add --project service --user senlin admin
# openstack service create --name senlin--description "Senlin Clustering Service V1"
# openstack endpoint create --region RegionOne clustering public http://ip:8780
# openstack endpoint create --region RegionOne clustering internal http://ip:8780
# openstack endpoint create --region RegionOne clustering admin http://ip:8780

```

建立 senlin-api service

```text
# vim /lib/systemd/system/senlin-api.service

[Unit]
Description = OpenStack Clustering Service API

[Service]
ExecStart = /usr/local/bin/senlin-api --config-file /etc/senlin/senlin.conf --log-file /var/log/senlin/senlin-api.log
User = senlin
TimeoutStartSec=120
TimeoutStopSec=300
Restart=on-failure
RestartSec=2

[Install]
WantedBy = multi-user.target
```

建立 senlin-engine 服務

```text
# vim /lib/systemd/system/senlin-engine.service

[Unit]
Description = OpenStack Clustering engine Service API

[Service]
ExecStart = /usr/local/bin/senlin-engine --config-file /etc/senlin/senlin.conf --log-file /var/log/senlin/senlin-engine.log
User = senlin
TimeoutStartSec=120
TimeoutStopSec=300
Restart=on-failure
RestartSec=2

[Install]
WantedBy = multi-user.target
```

建立 /etc/senlin/policy.yaml

```text
# vim /etc/senlin/policy.yaml

#
#"context_is_admin": "role:admin"

#
#"deny_everybody": "!"

# Show build information
# GET  /v1/build-info
#"build_info:build_info": ""

# List profile types
# GET  /v1/profile-types
#"profile_types:index": ""

# Show profile type details
# GET  /v1/profile-types/{profile_type}
#"profile_types:get": ""

# List profile type operations
# GET  /v1/profile-types/{profile_type}/ops
#"profile_types:ops": ""

# List policy types
# GET  /v1/policy-types
#"policy_types:index": ""

# Show policy type details
# GET  /v1/policy-types/{policy_type}
#"policy_types:get": ""

# List clusters
# GET  /v1/clusters
#"clusters:index": ""

# Create cluster
# POST  /v1/clusters
#"clusters:create": ""

# Delete cluster
# DELETE  /v1/clusters/{cluster_id}
#"clusters:delete": ""

# Show cluster details
# GET  /v1/clusters/{cluster_id}
#"clusters:get": ""

# Perform specified action on a cluster.
# POST  /v1/clusters/{cluster_id}/actions
#"clusters:action": ""

# Update cluster
# PATCH  /v1/clusters/{cluster_id}
#"clusters:update": ""

# Collect Attributes Across a Cluster
# GET  v1/clusters/{cluster_id}/attrs/{path}
#"clusters:collect": ""

# Perform an Operation on a Cluster
# POST  /v1/clusters/{cluster_id}/ops
#"clusters:operation": ""

# List profiles
# GET  /v1/profiles
#"profiles:index": ""

# Create profile
# POST  /v1/profiles
#"profiles:create": ""

# Show profile details
# GET  /v1/profiles/{profile_id}
#"profiles:get": ""

# Delete profile
# DELETE  /v1/profiles/{profile_id}
#"profiles:delete": ""

# Update profile
# PATCH  /v1/profiles/{profile_id}
#"profiles:update": ""

# Validate profile
# POST  /v1/profiles/validate
#"profiles:validate": ""

# List nodes
# GET  /v1/nodes
#"nodes:index": ""

# Create node
# GET  /v1/nodes
#"nodes:create": ""

# Adopt node
# POST  /v1/nodes/adopt
#"nodes:adopt": ""

# Adopt node (preview)
# POST  /v1/nodes/adopt-preview
#"nodes:adopt_preview": ""

# Show node details
# GET  /v1/nodes/{node_id}
#"nodes:get": ""

# Perform specified action on a Node.
# POST  /v1/nodes/{node_id}/actions
#"nodes:action": ""

# Update node
# PATCH  /v1/nodes/{node_id}
#"nodes:update": ""

# Delete node
# DELETE  /v1/nodes/{node_id}
#"nodes:delete": ""

# Perform an Operation on a Node
# POST  /v1/nodes/{node_id}/ops
#"nodes:operation": ""

# List policies
# GET  /v1/policies
#"policies:index": ""

# Create policy
# POST  /v1/policies
#"policies:create": ""

# Show policy details
# GET  /v1/policies/{policy_id}
#"policies:get": ""

# Update policy
# PATCH  /v1/policies/{policy_id}
#"policies:update": ""

# Delete policy
# DELETE  /v1/policies/{policy_id}
#"policies:delete": ""

# Validate policy.
# POST  /v1/policies/validate
#"policies:validate": ""

# List cluster policies
# GET  /v1/clusters/{cluster_id}/policies
#"cluster_policies:index": ""

# Attach a Policy to a Cluster
# POST  /v1/clusters/{cluster_id}/actions
#"cluster_policies:attach": ""

# Detach a Policy from a Cluster
# POST  /v1/clusters/{cluster_id}/actions
#"cluster_policies:detach": ""

# Update a Policy on a Cluster
# POST  /v1/clusters/{cluster_id}/actions
#"cluster_policies:update": ""

# Show cluster_policy details
# GET  /v1/clusters/{cluster_id}/policies/{policy_id}
#"cluster_policies:get": ""

# List receivers
# GET  /v1/receivers
#"receivers:index": ""

# Create receiver
# POST  /v1/receivers
#"receivers:create": ""

# Show receiver details
# GET  /v1/receivers/{receiver_id}
#"receivers:get": ""

# Update receiver
# PATCH  /v1/receivers/{receiver_id}
#"receivers:update": ""

# Delete receiver
# DELETE  /v1/receivers/{receiver_id}
#"receivers:delete": ""

# Notify receiver
# POST  /v1/receivers/{receiver_id}/notify
#"receivers:notify": ""

# List actions
# GET  /v1/actions
#"actions:index": ""

# Show action details
# GET  /v1/actions/{action_id}
#"actions:get": ""

# Update action
# PATCH  /v1/actions/{action_id}
#"actions:update": ""

# List events
# GET  /v1/events
#"events:index": ""

# Show event details
# GET  /v1/events/{event_id}
#"events:get": ""

# Trigger webhook action
# POST  /v1/webhooks/{webhook_id}/trigger
#"webhooks:trigger": ""

# List services
# GET  /v1/services
#"services:index": "role:admin"
```

修改資料夾權限

```text
# chown -R senlin:senlin /etc/senlin /var/log/senlin
```

啟用服務

```text
# systemctl enable senlin-api 
# systemctl enable senlin-engine

```

### 安裝 senlin-dashboard

```text
# git clone https://github.com/openstack/senlin-dashboard.git -b stable/queens
# cd senlin-dashboard
# pip install .
```

啟用 dashboard

```text
# cp /opt/senlin-dashboard/senlin_dashboard/enabled/_50_senlin.py /usr/share/openstack-dashboard/openstack_dashboard/local/enabled/_50_senlin.py
```

## Auto Scaling 前置設定 <a id="Auto-Scaling- &#x524D;&#x7F6E;&#x8A2D;&#x5B9A;"></a>

* 請先安裝 ceilometer，gnocchi，aodh
* senlin 和 gnocchi、整合有些 bug，因此需要修改一下程式碼
* Heat 的 autoscaling 是透過 vm 中 metadata 的 server\_group=stack\_id 來辨識是否為同一個 heat stack，gnocchi 所儲存的資源有包含 server\_group。
* senlin 所產生的 cluster 會有 cluster\_id，並且在 VM 的 metadata 加上 clusetr\_id 加以辨認，但是 gnocchi 的資源並沒有 clusetr\_id，因此需要手動增加。
* 修改 /usr/lib/python2.7/dist-packages/ceilometer/gnocchi\_client.py，在 server\_group 的下一行增加 cluster\_id resource type

```text
resources_initial = {
        "instance": {
            "server_group": {"type": "string", "min_length": 0, "max_length": 255,
                         "required": False},
            "cluster_id": {"type": "string", "min_length": 0, "max_length": 255,
                         "required": False},
         },
         

resources_update_operations = [
    {"desc": "add cluster_id to instance",
     "type": "update_attribute_type",
     "resource_type": "instance",
     "data": [{
         "op": "add",
         "path": "/attributes/cluster_id",
         "value": {"type": "string", "min_length": 0, "max_length": 255,
                   "required": False}
     }]},
```

修改 /usr/lib/python2.7/dist-packages/ceilometer/publisher/data/gnocchi\_resources.yaml， 在 server\_group: 下一行加入 cluster\_id

```text
- resource_type: instance
  attributes:
     server_group: resource_metadata.user_metadata.server_group
     cluster_id: resource_metadata.user_metadata.cluster_id   
```

刪除已編譯的 gnocchi\_client.pyc

```text
# rm /usr/lib/python2.7/dist-packages/ceilometer/gnocchi_client.pyc
```

更新 ceilometer 和 gnocchi 資料庫

```text
# gnocchi-upgrade
# ceilometer-upgrade
```

{% hint style="info" %}
如果不想修改程式碼 可以使用 gnocchi api 更新 resource
{% endhint %}

```text
PATCH /v1/resource_type/my_custom_type HTTP/1.1
Content-Type: application/json-patch+json

[
	{
    "op": "add",
    "path": "/attributes/cluster_id",
    "value": {
    	"type": "string",
    	"min_length": 0, 
    	"max_length": 255, 
    	"required": false
		}
	}
]
```

在 /etc/ceilometer/ceilometer.conf 增加

```text
[DEFAULT]
reserved_metadata_keys = cluster_id
```

重啟 ceilometer 服務 \(controller\)

```text
# service ceilometer-agent-notification restart
# service ceilometer-agent-central restart
```

重啟 ceilometer-agent-compute \(compute\)

```text
# service ceilometer-agent-compute restart

```

{% hint style="warning" %}
此段官方好像修復了，應該不用改了

[https://github.com/openstack/aodh/blob/stable/queens/aodh/notifier/rest.py\#L80](https://github.com/openstack/aodh/blob/stable/queens/aodh/notifier/rest.py#L80)
{% endhint %}

~~再來修改 aodh，senlin 的 VM scaling 是透過呼叫 webhook 的方式增加減少 VM，但是呼叫 webhook 時，不能送出有值的 body，而 aodh alarm 所觸發的 webhook 一定會送出含有資料的 body，之後 senlin 增加了功能，如果呼叫 scaling 的 webhook，要送出有值的 body ，必須要加入 header openstack-api-version = clustering 1.10，但是 aodh 的 alarm 所送出的 header 只有 content-type = application/json，因此需要修改程式碼。~~

~~修改 /usr/lib/python2.7/dist-packages/aodh/notifier/rest.py，在 headers\[‘content-type’\]下一行增加 header ‘openstack-api-version’~~

```text
headers['content-type'] = 'application/json'
headers['openstack-api-version'] = 'clustering 1.10'
```

~~刪除已編譯的 rest.pyc~~

```text
# rm /usr/lib/python2.7/dist-packages/aodh/notifier/rest.pyc
```

重啟 aodh 服務

```text
# service apache2 restart
# service aodh-evaluator restart
# service aodh-notifier restart
# service aodh-listener restart
```

檢查是否成功新增 gnocchi instance 的 resource type 裡的 cluster\_id 屬性

```text
openstack metric resource-type show instance
+-------------------------+-----------------------------------------------------------+
| Field                   | Value                                                     |
+-------------------------+-----------------------------------------------------------+
| attributes/cluster_id   | max_length=255, min_length=0, required=False, type=string |
| attributes/created_at   | required=False, type=datetime                             |
| attributes/deleted_at   | required=False, type=datetime                             |
| attributes/display_name | max_length=255, min_length=0, required=True, type=string  |
| attributes/flavor_id    | max_length=255, min_length=0, required=True, type=string  |
| attributes/flavor_name  | max_length=255, min_length=0, required=True, type=string  |
| attributes/host         | max_length=255, min_length=0, required=True, type=string  |
| attributes/image_ref    | max_length=255, min_length=0, required=False, type=string |
| attributes/launched_at  | required=False, type=datetime                             |
| attributes/server_group | max_length=255, min_length=0, required=False, type=string |
| name                    | instance                                                  |
| state                   | active                                                    |
+-------------------------+-----------------------------------------------------------+
```

## 使用 Auto Scaling <a id="&#x4F7F;&#x7528; -Auto-Scaling"></a>

建立 VM 模板，此模板為 scaling 的 VM，建立 server.yaml

```text
type: os.nova.server
version: 1.0
properties:
  name: cirros_server
  flavor: m1.tiny
  image: cirros-0.3.5-x86_64-disk
  key_name: oskey
  networks:
    - network: private
```

建立 profile ，也就是樣板

```text
# openstack cluster profile create --spec-file server.yaml pserver

+------------+----------------------------------------------+
| Field      | Value                                        |
+------------+----------------------------------------------+
| created_at | 2019-02-19T07:08:10Z                         |
| domain_id  | None                                         |
| id         | 3a3ad404-5676-4f9d-b424-a74b25abce88         |
| location   | None                                         |
| metadata   | {}                                           |
| name       | pserver                                      |
| project_id | e214ad5b451a4616a4689afb58e7dbc8             |
| spec       | +------------+-----------------------------+ |
|            | | property   | value                       | |
|            | +------------+-----------------------------+ |
|            | | properties | {                           | |
|            | |            |   "key_name": "mykey",      | |
|            | |            |   "flavor": "m1.tiny",      | |
|            | |            |   "networks": [| |
|            | |            |     {                       | |
|            | |            |       "network": "demo-net" | |
|            | |            |     }                       | |
|            | |            |   ],                        | |
|            | |            |   "image": "cirros",        | |
|            | |            |   "name": "cirros_server"   | |
|            | |            | }                           | |
|            | | type       | os.nova.server              | |
|            | | version    | 1.0                         | |
|            | +------------+-----------------------------+ |
| type       | os.nova.server-1.0                           |
| updated_at | None                                         |
| user_id    | e26bd520e7044d78b8966c7f3ce9b364             |
+------------+----------------------------------------------+


```

建立 cluster，–desired-capacity 指定 cluster node 起始數量

```text
# openstack cluster create --profile pserver --desired-capacity 1 --min-size 1 mycluster

+------------------+--------------------------------------+
| Field            | Value                                |
+------------------+--------------------------------------+
| config           | {}                                   |
| created_at       | None                                 |
| data             | {}                                   |
| dependents       | {}                                   |
| desired_capacity | 1                                    |
| domain_id        | None                                 |
| id               | 2682c9e0-0231-4506-8a69-a95bb764bcea |
| init_at          | 2019-02-19T07:30:29Z                 |
| location         | None                                 |
| max_size         | -1                                   |
| metadata         | {}                                   |
| min_size         | 0                                    |
| name             | mycluster                            |
| node_ids         |                                      |
| profile_id       | 3a3ad404-5676-4f9d-b424-a74b25abce88 |
| profile_name     | pserver                              |
| project_id       | e214ad5b451a4616a4689afb58e7dbc8     |
| status           | INIT                                 |
| status_reason    | Initializing                         |
| timeout          | 3600                                 |
| updated_at       | None                                 |
| user_id          | e26bd520e7044d78b8966c7f3ce9b364     |
+------------------+--------------------------------------+
```

查看 cluster 資訊

```text
# openstack cluster show mycluster

+------------------+--------------------------------------------------------------------------------+
| Field            | Value                                                                          |
+------------------+--------------------------------------------------------------------------------+
| config           | {}                                                                             |
| created_at       | 2019-02-19T07:31:00Z                                                           |
| data             | {}                                                                             |
| dependents       | {}                                                                             |
| desired_capacity | 1                                                                              |
| domain_id        | None                                                                           |
| id               | 2682c9e0-0231-4506-8a69-a95bb764bcea                                           |
| init_at          | 2019-02-19T07:30:29Z                                                           |
| location         | None                                                                           |
| max_size         | -1                                                                             |
| metadata         | {}                                                                             |
| min_size         | 0                                                                              |
| name             | mycluster                                                                      |
| node_ids         | 27aab314-9d4a-4a3f-9545-788ec12bd5f9                                           |
| profile_id       | 3a3ad404-5676-4f9d-b424-a74b25abce88                                           |
| profile_name     | pserver                                                                        |
| project_id       | e214ad5b451a4616a4689afb58e7dbc8                                               |
| status           | ACTIVE                                                                         |
| status_reason    | CLUSTER_CREATE: number of active nodes is equal or above desired_capacity (1). |
| timeout          | 3600                                                                           |
| updated_at       | 2019-02-19T07:31:00Z                                                           |
| user_id          | e26bd520e7044d78b8966c7f3ce9b364                                               |
+------------------+--------------------------------------------------------------------------------+

```

將上述所得到的 cluster id 匯入環境變數

```text
# export MYCLUSTER_ID=2682c9e0-0231-4506-8a69-a95bb764bcea
```



建立 Receiver webhook，透過觸發 webhook，可以執行所設定的 action，Receiver 可以設定的 action 如下:

* CLUSTER\_SCALE\_OUT
* CLUSTER\_SCALE\_IN
* CLUSTER\_RESIZE
* CLUSTER\_CHECK
* CLUSTER\_UPDATE
* CLUSTER\_DELETE
* CLUSTER\_ADD\_NODES
* CLUSTER\_DEL\_NODES
* NODE\_CREATE
* NODE\_DELETE
* NODE\_UPDATE
* NODE\_CHECK
* NODE\_RECOVER

Auto scaling 需要 scale\_out 和 sacle\_in ，因此建立兩個 Receiver，count 是指觸發一次後要增加減少的 node 數量 建立 Receiver CLUSTER\_SCALE\_OUT

```text
# openstack cluster receiver create --action CLUSTER_SCALE_OUT --params count=1 --cluster mycluster scale_out
+------------+-------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                       |
+------------+-------------------------------------------------------------------------------------------------------------+
| action     | CLUSTER_SCALE_OUT                                                                                           |
| actor      | {                                                                                                           |
|            |   "trust_id": "1795bf3ee52a44f6bcf681144f15c4c3"                                                            |
|            | }                                                                                                           |
| channel    | {                                                                                                           |
|            |   "alarm_url": "http://10.50.2.4:8780/v1/webhooks/706eab3c-6269-409f-9675-abf14523de31/trigger?V=1&count=1" |
|            | }                                                                                                           |
| cluster_id | 2682c9e0-0231-4506-8a69-a95bb764bcea                                                                        |
| created_at | 2019-02-19T07:40:22Z                                                                                        |
| domain_id  | None                                                                                                        |
| id         | 706eab3c-6269-409f-9675-abf14523de31                                                                        |
| location   | None                                                                                                        |
| name       | scale_out                                                                                                   |
| params     | {                                                                                                           |
|            |   "count": "1"                                                                                              |
|            | }                                                                                                           |
| project_id | e214ad5b451a4616a4689afb58e7dbc8                                                                            |
| type       | webhook                                                                                                     |
| updated_at | None                                                                                                        |
| user_id    | e26bd520e7044d78b8966c7f3ce9b364                                                                            |
+------------+-------------------------------------------------------------------------------------------------------------+
```

建立 Receiver CLUSTER\_SCALE\_IN

```text
# openstack cluster receiver create --action CLUSTER_SCALE_IN --params count=1 --cluster mycluster scale_in
+------------+-------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                       |
+------------+-------------------------------------------------------------------------------------------------------------+
| action     | CLUSTER_SCALE_IN                                                                                            |
| actor      | {                                                                                                           |
|            |   "trust_id": "1795bf3ee52a44f6bcf681144f15c4c3"                                                            |
|            | }                                                                                                           |
| channel    | {                                                                                                           |
|            |   "alarm_url": "http://10.50.2.5:8780/v1/webhooks/e8079cfd-2e21-4630-93f0-5dfed1c55299/trigger?V=1&count=1" |
|            | }                                                                                                           |
| cluster_id | 2682c9e0-0231-4506-8a69-a95bb764bcea                                                                        |
| created_at | 2019-02-19T07:40:56Z                                                                                        |
| domain_id  | None                                                                                                        |
| id         | e8079cfd-2e21-4630-93f0-5dfed1c55299                                                                        |
| location   | None                                                                                                        |
| name       | scale_in                                                                                                    |
| params     | {                                                                                                           |
|            |   "count": "1"                                                                                              |
|            | }                                                                                                           |
| project_id | e214ad5b451a4616a4689afb58e7dbc8                                                                            |
| type       | webhook                                                                                                     |
| updated_at | None                                                                                                        |
| user_id    | e26bd520e7044d78b8966c7f3ce9b364                                                                            |
+------------+-------------------------------------------------------------------------------------------------------------+


```

匯入環境變數 alarm\_url，此 URL 為 webhook

```text
# export ALRM_URLOUT="http://10.50.2.4:8780/v1/webhooks/706eab3c-6269-409f-9675-abf14523de31/trigger?V=1&count=1"
# export ALRM_URLIN="http://10.50.2.5:8780/v1/webhooks/e8079cfd-2e21-4630-93f0-5dfed1c55299/trigger?V=1&count=1"
```

建立 aodh alarm，以下為參數設定

* metric: 根據什麼發出 alarm，這裡選擇 cpu 使用率，cpu\_util 
* alarm-action: alarm 時要觸發的 webhook，為上述的 alarm\_url 
* query: query 出 VM 的 metadata 中相同 cluster\_id 的 VM，並聚合所有的 metric，請填入 cluster\_id 
* threshold: 閥值，當監測的量此值，會觸發 1 次 alarm，這裡設定 cpu 70% 
* comparison-operator: 跟 threshold 結合，gt 就是大於，lt 小於，範例 gt 就是大於 70 
* evaluation-periods: 週期，到達幾次定的 threshold，就會發出 alarm，此範例為到達 1 次 cpu 大於 70%，就發出 alarm 
* aggregation-method: 如何計算 query 出的 metric 值，這裡填入平均 mean， repeat-actions: 重複執行 alarm-action，當 alarm 持續發生時，是否重複觸發 alarm-action

建立 cpu-high 警示



```text
# aodh alarm create \
--type gnocchi_aggregation_by_resources_threshold \
--name cpu-high \
--metric cpu_util \
--threshold 70 \
--comparison-operator gt \
--description 'instance running hot' \
--evaluation-periods 1 \
--aggregation-method mean \
--alarm-action $ALRM_URLOUT \
--granularity 60 \
--repeat-actions True \
--query '{"=": {"cluster_id": "2682c9e0-0231-4506-8a69-a95bb764bcea"}}' \
--resource-type instance

+---------------------------+-------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                           |
+---------------------------+-------------------------------------------------------------------------------------------------+
| aggregation_method        | mean                                                                                            |
| alarm_actions             | [u'http://10.50.2.4:8780/v1/webhooks/706eab3c-6269-409f-9675-abf14523de31/trigger?V=1&count=1'] |
| alarm_id                  | c541927c-2611-48a3-8e45-3bdcd4ccb533                                                            |
| comparison_operator       | gt                                                                                              |
| description               | instance running hot                                                                            |
| enabled                   | True                                                                                            |
| evaluation_periods        | 1                                                                                               |
| granularity               | 60                                                                                              |
| insufficient_data_actions | []                                                                                              |
| metric                    | cpu_util                                                                                        |
| name                      | cpu-high                                                                                        |
| ok_actions                | []                                                                                              |
| project_id                | e214ad5b451a4616a4689afb58e7dbc8                                                                |
| query                     | {"=": {"cluster_id": "2682c9e0-0231-4506-8a69-a95bb764bcea"}}                                   |
| repeat_actions            | True                                                                                            |
| resource_type             | instance                                                                                        |
| severity                  | low                                                                                             |
| state                     | insufficient data                                                                               |
| state_reason              | Not evaluated yet                                                                               |
| state_timestamp           | 2019-02-19T10:02:16.086730                                                                      |
| threshold                 | 70.0                                                                                            |
| time_constraints          | []                                                                                              |
| timestamp                 | 2019-02-19T10:02:16.086730                                                                      |
| type                      | gnocchi_aggregation_by_resources_threshold                                                      |
| user_id                   | e26bd520e7044d78b8966c7f3ce9b364                                                                |
+---------------------------+-------------------------------------------------------------------------------------------------+
```

建立 cpu-low 警示



```text
# aodh alarm create \
--type gnocchi_aggregation_by_resources_threshold \
--name cpu-low \
--metric cpu_util \
--threshold 30 \
--comparison-operator lt \
--description 'instance running cold' \
--evaluation-periods 1 \
--aggregation-method mean \
--alarm-action $ALRM_URLIN \
--granularity 60 \
--repeat-actions True \
--query '{"=": {"cluster_id": "2682c9e0-0231-4506-8a69-a95bb764bcea"}}' \
--resource-type instance
+---------------------------+-------------------------------------------------------------------------------------------------+
| Field                     | Value                                                                                           |
+---------------------------+-------------------------------------------------------------------------------------------------+
| aggregation_method        | mean                                                                                            |
| alarm_actions             | [u'http://10.50.2.5:8780/v1/webhooks/e8079cfd-2e21-4630-93f0-5dfed1c55299/trigger?V=1&count=1'] |
| alarm_id                  | 7b0e8f10-251c-42db-be40-f46a1e276faa                                                            |
| comparison_operator       | lt                                                                                              |
| description               | instance running cold                                                                           |
| enabled                   | True                                                                                            |
| evaluation_periods        | 1                                                                                               |
| granularity               | 60                                                                                              |
| insufficient_data_actions | []                                                                                              |
| metric                    | cpu_util                                                                                        |
| name                      | cpu-low                                                                                         |
| ok_actions                | []                                                                                              |
| project_id                | e214ad5b451a4616a4689afb58e7dbc8                                                                |
| query                     | {"=": {"cluster_id": "2682c9e0-0231-4506-8a69-a95bb764bcea"}}                                   |
| repeat_actions            | True                                                                                            |
| resource_type             | instance                                                                                        |
| severity                  | low                                                                                             |
| state                     | insufficient data                                                                               |
| state_reason              | Not evaluated yet                                                                               |
| state_timestamp           | 2019-02-20T02:50:17.389596                                                                      |
| threshold                 | 30.0                                                                                            |
| time_constraints          | []                                                                                              |
| timestamp                 | 2019-02-20T02:50:17.389596                                                                      |
| type                      | gnocchi_aggregation_by_resources_threshold                                                      |
| user_id                   | e26bd520e7044d78b8966c7f3ce9b364                                                                |
+---------------------------+-------------------------------------------------------------------------------------------------+
```

進入 VM，對 VM 做 CPU 壓力測試，此為 cirros 範例

```text
$ cat /dev/zero > /dev/bull
```

使用 openstack alarm 查看警示狀態

```text
# watch openstack alarm list
+--------------------------------------+--------------------------------------------+----------+-------+----------+---------+
| alarm_id                             | type                                       | name     | state | severity | enabled |
+--------------------------------------+--------------------------------------------+----------+-------+----------+---------+
| 7b0e8f10-251c-42db-be40-f46a1e276faa | gnocchi_aggregation_by_resources_threshold | cpu-low  | ok    | low      | True    |
| c541927c-2611-48a3-8e45-3bdcd4ccb533 | gnocchi_aggregation_by_resources_threshold | cpu-high | alarm | low      | True    |
+--------------------------------------+--------------------------------------------+----------+-------+----------+---------+
```

使用 openstack cluster 查看集群是否有擴展

```text
# watch openstack cluster show 
+------------------+------------------------------------------+
| Field            | Value                                    |
+------------------+------------------------------------------+
| config           | {}                                       |
| created_at       | 2019-02-19T07:31:00Z                     |
| data             | {}                                       |
| dependents       | {}                                       |
| desired_capacity | 4                                        |
| domain_id        | None                                     |
| id               | 2682c9e0-0231-4506-8a69-a95bb764bcea     |
| init_at          | 2019-02-19T07:30:29Z                     |
| location         | None                                     |
| max_size         | -1                                       |
| metadata         | {}                                       |
| min_size         | 0                                        |
| name             | mycluster                                |
| node_ids         | 2e23692b-388c-4fd6-b159-8398bf1a5688     |
|                  | 3fc73090-80d5-4e31-b3fc-15901c1d1cd1     |
|                  | c2ef243d-697c-476b-b74a-cdbb61f5c4b2     |
|                  | eb959317-5dc8-466f-a054-aaf75134875f     |
| profile_id       | 3a3ad404-5676-4f9d-b424-a74b25abce88     |
| profile_name     | pserver                                  |
| project_id       | e214ad5b451a4616a4689afb58e7dbc8         |
| status           | RESIZING                                 |
| status_reason    | Node node-Zb9FrKHm: Creation in progress |
| timeout          | 3600                                     |
| updated_at       | 2019-02-20T03:07:01Z                     |
| user_id          | e26bd520e7044d78b8966c7f3ce9b364         |
+------------------+------------------------------------------+
```

