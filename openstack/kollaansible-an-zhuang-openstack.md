# Kolla-ansible 安裝 OpenStack

* Kolla: 將 OpenStack 各個專案包成 docker image
* Kolla-ansible: 將 OpenStack 各個專案包成 docker image，部署至節點上，也就是 OpenStack所有服務都在 container 內執行。
* 此篇內容參考**郭靖**大大的文章: [https://igene.tw/kolla-ansible-deploy](https://igene.tw/kolla-ansible-deploy)

## 前置準備

* 準備節點，這範例準備了8台
  * 1台部屬節點，即 kolla-ansible執行的節點
  * 3台controller
  * 3台compute
  * 1台monitor

| hosname | 外部網段 | VM和管理網段 | 儲存網段 | Neutron 外網網段 |
| :--- | :--- | :--- | :--- | :--- |
|  | ens3 | ens9 | ens10 | ens11 |
| controller1 | 10.50.2.4 | 192.168.2.4 | 192.168.4.4 |  |
| controller2 | 10.50.2.5 | 192.168.2.5 | 192.168.4.5 |  |
| controller3 | 10.50.2.6 | 192.168.2.6 | 192.168.4.6 |  |
| compute1 \(ceph-osd1\) | 10.50.2.7 | 192.168.2.7 | 192.168.4.7 |  |
| compute2 \(ceph-osd2\) | 10.50.2.8 | 192.168.2.8 | 192.168.4.8 |  |
| compute3 \(ceph-osd3\) | 10.50.2.13 | 192.168.2.13 | 192.168.4.13 |  |
| monitor | 10.50.2.14 | 192.168.2.14 | 192.168.4.14 |  |

* 網卡數量，最少兩張，此範例為4張
  * 外部網段: ens3
  * Neutron 外網: ens11
  * VM、管理網段: ens9
  * 儲存網段: ens10
* 佈署節點需要無密碼登入各節點，可以使用ssh-copy-id 輸入公鑰至各節點
* Kolla-ansible 也可以佈署 ceph，如果要佈署 ceph，請到 ceph-osd 的節點，輸入以下指令:  
  \($DISK 為你的osd硬碟 例如:/dev/vdb\)

  ```text
  parted $DISK -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP_BS 1 -1
  ```

* Kolla-ansible 會幫你安裝 docker，但是版本是舊的，如果你想要裝新版 docker，可以自行到各節點安裝 docker

### 各節點準備

* 設定好各網卡 IP
* 設定好 hostname
* Neutron 外網的網卡請不要設定IP，此範例ens11為Neutron 外網，因此設定為

  ```text
  auto ens11
  iface ens11 inet manual
  ```

## Kolla-ansible 佈署節點準備

* 請在 /etc/hosts 自行填入其他節點的 hostname 和 IP，以便使用hostname。
* 安裝 pip, ansible

  ```text
  # apt update
  # apt install python-pip
  # pip install ansible
  ```

* 安裝相關套件

  ```text
  # apt-get install python-dev libffi-dev gcc libssl-dev python-selinux python-setuptools
  ```

* 新增/etc/ansible/資料夾

  ```text
  # mkdir -p /etc/ansible
  ```

* 新增/etc/ansible/ansible.cfg檔案，內容如下:

  ```text
  [defaults]
  host_key_checking=False
  pipelining=True
  forks=100
  ```

* 從github clone kolla-ansible，安裝所需套件

  ```text
  # git clone https://github.com/openstack/kolla-ansible -b stable/rocky
  # cd kolla-ansible
  # pip install -r requirements.txt
  ```

* 複製kolla設定檔至/etc/kolla/

  ```text
  # cp kolla-ansible/etc/kolla/ /etc/kolla
  ```

* 複製 inventory 檔案至家目錄

  ```text
  # cp ansible/inventory/* ~/
  ```

* 編輯 /etc/kolla/globals.yml，裡面可以設定環境

### globals.yml 參數說明

kolla Options

* kolla base distro: image的基底作業系統這裡使用ubuntu

  ```text
  kolla_base_distro: "ubuntu"
  ```

* kolla install type:  OpenStack 安裝的來源，有binary 跟 source 可選

  ```text
  kolla_install_type: "source"
  ```

* openstack release: OpenStack版本

  ```text
  openstack_release: "rocky"
  ```

* kolla internal vip address:  OpenStack 內部溝通的VIP，也是HA環境時keepalived所使用的IP
* kolla external vip address:  OpenStack 外部溝通的VIP，也就是 endpoint 的 public url的 IP
* Neutron - Networking Options

  * API interface: OpenStack 各 component 溝通的網路
  * External VIP Interface: OpenStack Public 的 endpoint
  * Storage Interface: OpenStack VM 和 Ceph 溝通的網路
  * Tunnel Interface: OpenStack VM之間的溝通網路
  * Neutron External Interface: VM 連接對外網路的 interface，為Floating IP的網路，不能與其他人共用也不能設IP



  預設 network\_interface 的值， 這個{{ network\_interface }}值， 就會等於你所填的network\_interface， 也就是說，ansible會把 變數解析為 ens9



根據範例的網段設定如下

```text
kolla_internal_vip_address: "192.168.2.3"

kolla_external_vip_address: "10.50.2.3"

network_interface: "ens9"

kolla_external_vip_interface: "ens3"

#api_interface: "{{ network_interface }}"

storage_interface: "ens10"

#cluster_interface: "{{ network_interface }}"

tunnel_interface: "ens9"

#dns_interface: "{{ network_interface }}"

neutron_external_interface: "ens11"
```

OpenStack options: 根據需求啟用OpenStack各種服務，被註解的部分就是預設值，此範例額外啟用ceph，然而有關儲存的服務，像是 glance 和 cinder 等等，都要啟用 backend 為 ceph

如果要啟用 ceilometer，請將 ceilometer、gnocchi、redis開啟

```text
###################
# OpenStack options
###################
....
enable_ceph: "yes"
enable_ceph_rgw: "yes"
enable_cinder: "yes"
....
########################
# Glance - Image Options
########################
# Configure image backend.
glance_backend_ceph: "yes"
glance_backend_file: "no"

################################
# Cinder - Block Storage Options
################################
# Enable / disable Cinder backends
cinder_backend_ceph: "yes"

########################
# Nova - Compute Options
########################
nova_backend_ceph: "yes"
```

## Ansible Inventory 設定說明

Inventory 檔案可以設定 OpenStack 的主機和角色，此範例為: 3台 controller、3台 compute、compute 包含 OSD，1台 monitor。

編輯 ~/multinode

```text
[control]
controller[1:3] ansible_ssh_user=localadmin ansible_become_pass=openstack

[network]
controller[1:3] ansible_ssh_user=localadmin ansible_become_pass=openstack

[external-compute]
compute[1:3] ansible_ssh_user=localadmin ansible_become_pass=openstack

[monitoring]
monitor ansible_ssh_user=localadmin ansible_become_pass=openstack

[storage]
compute[1:3] ansible_ssh_user=localadmin ansible_become_pass=openstack
```

* ansible ssh user: ansible要使用哪個 user ssh 登入
* ansible become pass: ansible ssh 進入後，要使用 sudo 指令時的密碼

## 佈署 OpenStack

* 產生OpenStack所需的密碼，產出的檔案在/etc/kolla/passwords.yml

  ```text
  # tools/generate_passwords.py
  ```

* 產出後，可以根據喜好更改密碼，若想要自訂 horizon 的 admin 登入密碼，請修改keystone admin password

  ```text
  # vim /etc/kolla/passwords.yml
  keystone_admin_password: openstack
  ```

* 執行 bootstrap-servers，kolla-ansible會至各節點安裝設定所需的套件，像是docker

  ```text
  # tools/kolla-ansible -i ~/multinode bootstrap-servers
  ```

* 執行 prechecks ，會檢查所需的 IP 和 port 是否可以使用等等

  ```text
  # tools/kolla-ansible -i ~/multinode prechecks
  ```

* 執行 deploy，佈署 OpenStack

  ```text
  # tools/kolla-ansible -i ~/multinode deploy
  ```

* 完成後，可至瀏覽器輸入 kolla internal vip\_address 或 kolla external vip address，登入OpenStack
* 請查看 /var/log/kolla/gnocchi/gnocchi-metricd.log 如果有錯誤是關於 not 'NoneType' 請隨意進入一個 gnocchi 的容器，輸入以下指令

  ```text
  # gnocchi-upgrade
  ```

