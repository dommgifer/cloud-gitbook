---
description: >-
  Octavia 是 OpenStack 新的 load balancer 專案，也是官方推薦的 lbaas，夠對外提供獨立而穩定的API而 neutron
  lbaasv2則在 Queens 版本被標為棄用。
---

# OpenStack Octavia

{% hint style="danger" %}
由於此篇是由很久之前的個人紀錄搬至gitbook

內容已過時，僅供參考
{% endhint %}

* 因為無法處理大量的流量，也無法擁有 HA 的機制，因此將此專案獨立出來。
* Octavia 透過建立 HAProxy 的虛擬機，將流量導入所指定的 member，而 HAProxy 的虛擬機的 image 是官方製作的，裡面包含了 haproxy 以及 keepalive，還有 python Flask 的 API，開啟 9443 port，讓 Octavia-worker 服務可以根據使用者所選擇的 load balancer 的 pool 和 member 等等設定，透過 API 方式，修改 HAProxy 虛擬機的設定。
* Octavia 支援 Active 和 Standby，也就是建立兩台 HAProxy 虛擬機，一台為 Master，另一台為 Slave，兩個是透過 keepalive 做溝通，而 keepalive 的設定也是透過 API 來設定。

## 安裝步驟 <a id="&#x5B89;&#x88DD;&#x6B65;&#x9A5F;"></a>

以下步驟安裝至 controller 節點，版本為 Queens

### 建立所需資源 <a id="&#x5EFA;&#x7ACB;&#x6240;&#x9700;&#x8CC7;&#x6E90;"></a>

建立 Octavia 使用者和群組

```text
# addgroup --system octavia 
# adduser --system octavia octavia --home /var/lib/octavia 
```

建立 octavia 資料夾

```text
# mkdir /var/lib/octavia /etc/octavia /var/cache/octavia /var/log/octavia
# chmod 755 /var/lib/octavia /etc/octavia /var/cache/octavia /var/log/octavia
# chown -R octavia:octavia /var/lib/octavia /etc/octavia /var/cache/octavia /var/log/octavia
```

安裝 octavia，如果要安裝 Queens 版的，版本為 2.0.3

```text
# pip install octavia==2.0.3
```

新增 octavia 使用者

```text
# openstack user create --domain default --password-prompt octavia
```

新增 octavia role 權限

```text
# openstack role add --project service --user octavia admin
```

新增 octavia 服務

```text
# openstack service create --name octavia --description "Octavia Load Balancing Service" load-balancer
```

新增 octavia API endpoints

```text
# openstack endpoint create --region RegionOne load-balancer public http://controller:9876
# openstack endpoint create --region RegionOne load-balancer internal http://controller:9873
# openstack endpoint create --region RegionOne load-balancer admin http://controller:9873
```

新增 octavia role

```text
# openstack role create load-balancer_observer
# openstack role create load-balancer_global_observer
# openstack role create load-balancer_member
# openstack role create load-balancer_admin
# openstack role create load-balancer_quota_admin
```

建立 CA 證書 資料夾

```text
# mkdir /etc/octavia/certs /etc/octavia/certs/private
# chown -R octavia:octavia /etc/octavia/certs /etc/octavia/certs/private
```

Clone github Octaiva 專案，裡面已經有建立 CA 證書的腳本

```text
# git clone https://github.com/openstack/octavia.git
```

設定 CA 密碼，此範例為 openstack，並取代腳本內預設密碼後，執行腳本

```text
# cd octavia
# sed -i 's/foobar/openstack/g' bin/create_certificates.sh
# ./bin/create_certificates.sh cert $(pwd)/etc/certificates/openssl.cnf
```

複製到 /etc/octavia/certs

```text
# cp certs/ca_01.pem certs/client.pem  /etc/ocatava/certs/
# cp certs/private/cakey.pem /etc/ocatava/certs/private
# chown -R octavia:octavia  /etc/ocatava/certs/
```

下載 amphora-haproxy image，並上傳

```text
# wget http://tarballs.openstack.org/octavia/test-images/test-only-amphora-x64-haproxy-ubuntu-xenial.qcow2
# openstack image create --container-format bare --disk-format qcow2 \ 
--private --file test-only-amphora-x64-haproxy-ubuntu-xenial.qcow2 \ 
--tag octavia-amphora-image amphora-x64-haproxy
```

建立 flavor

```text
# openstack flavor create --ram 1024 --disk 20 \ 
--vcpu 1 --private --project service m1.amphora
```

{% hint style="info" %}
建立 loadbalancer 的管理網路，這個網段需要 controller 可以抵達，因為需要呼叫 haproxy VM 裡面的 API，去新增設定，因此可以有以下幾種方式:

* 不建立管理網段，直接將 loadbalancer 建在外網上，但是這樣每建立一個 loadbalancer ，就會占一個 外網 IP，不建議
* 建立跟 OpenStack 管理網段一樣的網段，就是建一個外網，但是這個外網是通到 OpenStack 的管理網段。
* 建立一條 neutron 內網和 router，並接上外網，然後再由 controller 節點新增路由，通往 neutron 內網的網段全部通過 neutron 建的 router
{% endhint %}

此範例為第三種方式，先建立 neutron 內網

```text
# openstack network create --provider-network-type vxlan lbaas-mgmt
# openstack subnet create --subnet-range 172.16.0.0/24 --network lbaas-mgmt \
    --gateway 172.16.0.1 --dns-nameserver 8.8.8.8 lbaas-mgmt-subnet
```

建立 router，並接上內外網

```text
# openstack router create lb-router
# openstack router add subnet lb-router lbaas-mgmt-subnet
# openstack router set --external-gateway public lb-router
```

在 controller 新增路由，請先查看 lb-router 所拿到的 IP，然後設定 172.16.0.0/24 都經由此 router IP，dev 則指定外網的網卡

```text
#  route add -net 172.16.0.0/24 gw 10.40.0.28 dev eno1
```

設定 service project quota 數為 -1

```text
# openstack quota set --cores -1 \
  --instances -1 \
  --ram -1 \
  --server-groups -1 \
  --server-group-members -1 \
  --secgroups -1 \
  --ports -1 \
  --secgroup-rules -1 \
  service
```

建立 Octavia VM 的 security group，9443 為 API 的 Port

```text
# openstack security group create octavia_sec_grp
# openstack security group rule create --ingress --ethertype IPv4 \
    --protocol tcp --dst-port 22 octavia_sec_grp
# openstack security group rule create --ingress --ethertype IPv4 \
    --protocol tcp --dst-port 9443 octavia_sec_grp
```

建立 keypair ，用來 ssh 進 haproxy VM 除錯用的

```text
# openstack keypair create --public-key ~/.ssh/id_rsa.pub octavia_key
```

### Config 設定 <a id="Config- &#x8A2D;&#x5B9A;"></a>

編輯 /etc/octavia/octavia.conf， IP、帳號、密碼請自行更改，下方有更進一步說明

```text
[DEFAULT]
transport_url = rabbit://openstack:openstack@openstack

[api_settings]
auth_strategy = keystone

[certificates]
ca_private_key = /etc/octavia/certs/private/cakey.pem
ca_certificate = /etc/octavia/certs/ca_01.pem
ca_private_key_passphrase = openstack

[haproxy_amphora]
server_ca = /etc/octavia/certs/ca_01.pem
client_cert = /etc/octavia/certs/client.pem

[database]
connection = mysql+pymysql://octavia:openstack@10.40.0.7/octavia
max_retries = -1

[service_auth]
auth_url = http://10.40.0.7:5000/v3
auth_type = password
username = octavia
password = openstack
user_domain_name = Default
project_name = service
project_domain_name = Default
memcached_servers = openstack:11211

[keystone_authtoken]
www_authenticate_uri = http://10.40.0.7:5000/v3
auth_url = http://10.40.0.7:5000/v3
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = octavia
password = openstack
memcached_servers = openstack:11211

[health_manager]
bind_port = 5555
bind_ip = 0.0.0.0
controller_ip_port_list = 10.40.0.7:5555
heartbeat_key = insecure

[controller_worker]
amp_image_tag = octavia-amphora-image 
amp_flavor_id = 08dfaafc-af18-42c2-9e4d-1e0ea328ded5
amp_boot_network_list = 2f7c376b-3620-4bc6-92ff-bca73dcb3007
amp_ssh_key_name = octavia_key
amp_secgroup_list = octavia_sec_grp
amphora_driver = amphora_haproxy_rest_driver
compute_driver = compute_nova_driver
network_driver = allowed_address_pairs_driver
loadbalancer_topology = SINGLE

[oslo_messaging]
topic = octavia_prov
thread_pool_size = 2

[oslo_messaging_notifications]
transport_url = rabbit://openstack:openstack@10.40.0.7
```

參數說明:

* \[certificates\] 內的 ca\_private\_key\_passphrase 就是建 CA 時所輸入的密碼
* \[controller\_worker\]
  * amp\_flavor\_id: 請填入剛剛建立的 flavor ID，haproxy 的 VM 為此規格
  * amp\_boot\_network\_list: haproxy 的 VM 會建在此網段下，填入 network ID
  * loadbalancer\_topology: 可以設定 haproxy 的 VM 是否為 HA 模式，選項為 ACTIVE\_STANDBY 或 SINGLE，選擇 ACTIVE\_STANDBY 會建立 2 個 VM，並透過 keepalived 來實作 HA

修改 /etc/neutron/neutron.conf，增加以下

```text
[octavia]
base_url = http://10.40.0.7:9876
```

新增 /etc/neutron/neutron\_lbaas.conf

```text
[service_providers]
service_provider = LOADBALANCERV2:Octavia:neutron_lbaas.drivers.octavia.driver.OctaviaDriver:default
```

### 建立資料庫 <a id="&#x5EFA;&#x7ACB;&#x8CC7;&#x6599;&#x5EAB;"></a>

建立 Octavia 資料庫，密碼請自行更換

```text
# CREATE DATABASE octavia;
# GRANT ALL PRIVILEGES ON octavia.* TO octavia@'localhost' \
  IDENTIFIED BY 'openstack';
# GRANT ALL PRIVILEGES ON octavia.* TO octavia@'%' \
  IDENTIFIED BY 'openstack';
```

### 建立 Octavia 服務設定 <a id="&#x5EFA;&#x7ACB; -Octavia- &#x670D;&#x52D9;&#x8A2D;&#x5B9A;"></a>

建立服務，以利透過 service 指令啟動 Octavia  
建立 Octavia wsgi API 服務，新增 /etc/apache2/sites-available/octavia-api.conf

```text
Listen 9876

<VirtualHost *:9876>
    WSGIDaemonProcess octavia-wsgi processes=2 threads=1 user=octavia display-name=%{GROUP}
    WSGIProcessGroup octavia-wsgi
    WSGIScriptAlias / /usr/local/bin/octavia-wsgi
    WSGIApplicationGroup %{GLOBAL}

    ErrorLog /var/log/apache2/octavia_error.log
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    CustomLog /var/log/apache2/octavia_access.log combined


     <Directory /usr/local/bin/>
        WSGIProcessGroup octavia-wsgi
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

建立 octavia-health-manager 服務，新增 /lib/systemd/system/octavia-health-manager.service

```text
[Unit]
Description=OpenStack Octavia Health-Manager Service
After=syslog.target network.target

[Service]
Type=simple
User=octavia
Group=octavia
ExecStart=/usr/local/bin/octavia-health-manager --config-file /etc/octavia/octavia.conf --log-file /var/log/octavia/octavia-health-manager.log

TimeoutStartSec=120
TimeoutStopSec=300
Restart=on-failure
RestartSec=2

Slice=nova.slice
CPUAccounting=true
BlockIOAccounting=true
MemoryAccounting=true
TasksAccounting=true

[Install]
WantedBy=multi-user.target
```

建立 octavia-housekeeping 服務，新增 /lib/systemd/system/octavia-housekeeping.service

```text
[Unit]
Description=OpenStack Octavia Housekeeping Service
After=syslog.target network.target

[Service]
Type=simple
User=octavia
Group=octavia
ExecStart=/usr/local/bin/octavia-housekeeping --config-file /etc/octavia/octavia.conf --log-file /var/log/octavia/octavia-housekeeping.log

TimeoutStartSec=120
TimeoutStopSec=300
Restart=on-failure
RestartSec=2

Slice=nova.slice
CPUAccounting=true
BlockIOAccounting=true
MemoryAccounting=true
TasksAccounting=true

[Install]
WantedBy=multi-user.target
```

建立 octavia-worker 服務，新增 /lib/systemd/system/octavia-worker.service

```text
[Unit]
Description=OpenStack Octavia Worker Service
After=syslog.target network.target

[Service]
Type=simple
User=octavia
Group=octavia
ExecStart=/usr/local/bin/octavia-worker --config-file /etc/octavia/octavia.conf --log-file /var/log/octavia/octavia-worker.log

TimeoutStartSec=120
TimeoutStopSec=300
Restart=on-failure
RestartSec=2

Slice=nova.slice
CPUAccounting=true
BlockIOAccounting=true
MemoryAccounting=true
TasksAccounting=true

[Install]
WantedBy=multi-user.target
```

### 啟動服務 <a id="&#x555F;&#x52D5;&#x670D;&#x52D9;"></a>

Apache2 啟用 Octavia api

```text
# a2ensite octavia-api
```

啟用 Octavia 相關服務

```text
# systemctl enable octavia-housekeeping.service
# systemctl enable octavia-worker.service
# systemctl enable octavia-health-manager.service
```

重啟服務

```text
# service apache2 restart
# service octavia-housekeeping restart
# service octavia-worker restart
# service octavia-health-manager restart
# service neutron-server restart
```

### 安裝 Octavia dashboard <a id="&#x5B89;&#x88DD; -Octavia-dashboard"></a>

安裝 dashboard，Queens 版為 1.X.X，Rocky 版為 2.X.X

```text
# pip install octavia-dashboard==1.0.1
```

複製所需檔案

```text
# cp /usr/local/lib/python2.7/dist-packages/octavia_dashboard/enabled/_1482_*.py /usr/share/openstack-dashboard/openstack_dashboard/local/enabled/
```

Django 更新

```text
# cd /usr/share/openstack-dashboard/
# python manage.py compilemessages
# DJANGO_SETTINGS_MODULE=openstack_dashboard.settings python manage.py collectstatic --noinput
# DJANGO_SETTINGS_MODULE=openstack_dashboard.settings python manage.py compress --force
```

重啟 Apache2

```text
# service apache2 restart
```

## 解說 <a id="&#x89E3;&#x8AAA;"></a>

* 建立了兩台 web server，並建立了 active standby 的 loadbalancer， 然後 loadbalancer 的 VIP 建在跟 web server 一樣地網段下，連接之後，在 user 端只會看到 loadbalancer 的資訊，不會看見 2 台 haproxy VM![](https://dommgifer0604.herokuapp.com/2019/06/04/octavia/user_look.JPG)
* 登入 octavia 使用者，可以看到 2 台 haproxy VM 建在 service project，其中一台為 Master，另一台為 Backup![](https://dommgifer0604.herokuapp.com/2019/06/04/octavia/octavia_look.JPG)
* VIP 為 private 網段\(與 web server 同段\)，將兩者網路拓樸組合，就變成以下架構![](https://dommgifer0604.herokuapp.com/2019/06/04/octavia/all_topo.JPG)
* VIP 為新建的 vip 網段\(與 web server 不同段\)，將兩者網路拓樸組合，就變成以下架構![](https://dommgifer0604.herokuapp.com/2019/06/04/octavia/vip_all_topo.jpg)

