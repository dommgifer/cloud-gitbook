# DRBD9 使用說明

* DRBD9 支援多節點同步，相較之前的版本，有提供更容易配置 DRBD 的工具，可以使用 drbdmanage 或是 linstor 指令管理，由於 drbdmanage 指令到2018年底變為 EOL，後續指令改用 linstor 來維護，因此本篇使用 linstor 作說明。
* LINSTOR 是一個用於在Linux系統上存儲的配置管理系統。 管理節點集群上的LVM 或 ZFS ZVOL， 利用DRBD在不同節點之間進行複制。
* LINSTOR 的架構為 一個 controller 和多個 satellites\(需要備份的節點\) 。 linstor-controller 儲存了整個集群的所有設定，並且維護整個集群，因此 linstor-controller 通常使用Pacemaker和DRBD作為HA服務，因為 controller 系統的關鍵部分。
* 更多詳細資訊及操作請至[ LINBIT官網](https://docs.linbit.com/docs/users-guide-9.0)

## 安裝 DRBD9

* 安裝相關套件

  ```text
  # apt install -y linstor-controller linstor-satellite linstor-client drbd-dkms lvm2
  ```

* 啟用和啟動 linstor-controller 和 linstor-client 服務

  ```text
  # systemctl enable linstor-controller.service
  # systemctl start linstor-controller.service
  # systemctl enable linstor-satellite.service
  # systemctl start linstor-satellite.service
  ```

* 重啟server

  ```text
  # reboot
  ```

### 建立節點 <a id="&#x5EFA;&#x7ACB;&#x7BC0;&#x9EDE;"></a>

* 此範例為三台server，一個controller和三個satellite，其中一個節點為controller和satellite
* 以下所有指令都在 nodeA 執行。

| 節點名稱 | IP | 角色 |
| :--- | :--- | :--- |
| nodeA | 10.50.2.4 | Controller, satellite |
| nodeB | 10.50.2.5 | satellite |
| nodeC | 10.50.2.6 | satellite |



 建立節點，由於 nodeA 的角色為 controller 和 satellite，因此 type 為 Cmbined

```text
# linstor node create nodeA 10.50.2.4 --node-type Combined
# linstor node create nodeB 10.50.2.5 --node-type satellite
# linstor node create nodeC 10.50.2.6 --node-type satellite
```

使用 linstor node list 確認節點

```text
# linstor node list
╭───────────────────────────────────────────────────╮
┊ Node  ┊ NodeType ┊ Addresses              ┊ State  ┊
╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
┊ nodeA ┊ COMBINED ┊ 10.50.2.4:3366 (PLAIN) ┊ Online ┊
┊ nodeB ┊ SATELLITE┊ 10.50.2.5:3366 (PLAIN) ┊ Online ┊
┊ nodeC ┊ SATELLITE┊ 10.50.2.6:3366 (PLAIN) ┊ Online ┊
╰────────────────────────────────────────────────────╯
```

建立 LVM 的 drbdpool群組，此範例的drbd的硬碟為vdb

```text
# pvcreate /dev/vdb
# vgcreate drbdpool /dev/vdb
```

### 定義 storage pool <a id="&#x5B9A;&#x7FA9;_storage_pool"></a>

定義一個 storage pool 名為 drbdpool

```text
# linstor storage-pool-definition create drbdpool
```

### 建立 storage pool <a id="&#x5EFA;&#x7ACB;_storage_pool"></a>

各節點建立 lvm 並指定 drbdpool

```text
# linstor storage-pool create lvm nodeA drbdpool drbdpool
# linstor storage-pool create lvm nodeB drbdpool drbdpool
# linstor storage-pool create lvm nodeC drbdpool drbdpool
```

### 定義 Resource 和 volume <a id="&#x5B9A;&#x7FA9;_resource_&#x548C;_volume"></a>

* 定義一個 demo 的 Resource，並指定volume的size，此範例的 /dev/vdb 有 20G，這裡分 15G 給 demo 的resource

  ```text
  # linstor resource-definition create demo
  # linstor volume-definition create demo 15G
  ```

* 確認已定義的volume

  ```text
  # linstor volume-definition list
  ╭──────────────────────────────────────────────────────────────╮
  ┊ ResourceName ┊ VolumeNr ┊ VolumeMinor ┊ Size    ┊ State ┊
  ╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
  ┊ demo          ┊ 0        ┊ 1000        ┊ 15 GiB  ┊ ok   ┊
  ╰──────────────────────────────────────────────────────────────╯
  ```

### 建立 resource <a id="&#x5EFA;&#x7ACB;_resource"></a>

* 在三個節點建立 resource

  ```text
  # linstor resource create nodeA demo --storage-pool drbdpool
  # linstor resource create nodeB demo --storage-pool drbdpool
  # linstor resource create nodeC demo --storage-pool drbdpool
  ```

* 查看磁碟並格式化 drbd 磁區

  ```text
  # lsblk
  vdb                        253:16    0    20G  0 disk 
  ├─drbdpool-demo_00000      252:2     0    15G  0 lvm  
  │ └─drbd1000               147:1000  0    15G  1 disk 

  # mkfs.xfs /dev/drbd1000
  ```

* 查看 resource

  ```text
  # linstor resource list
  ╭─────────────────────────────────────────────────────╮
  ┊ ResourceName  ┊ Node  ┊ Port ┊ Usage  ┊        State ┊
  ╞┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄┄╡
  ┊ demo          ┊ nodeA ┊ 7000 ┊ Unused ┊     UpToDate ┊
  ┊ demo          ┊ nodeB ┊ 7000 ┊ Unused ┊ Inconsistent ┊
  ┊ demo          ┊ nodeC ┊ 7000 ┊ Unused ┊ Inconsistent ┊
  ╰─────────────────────────────────────────────────────╯
  ```

* State 為 UpToDate 代表資料完全同步， Inconsistent 代表尚未完全同步， 可以用 drbdadm 查看進度

  ```text
  # rbdadm status
  demo role:Secondary
    disk:UpToDate
    nodeB role:Primary
      replication:SyncSource peer-disk:Inconsistent done:42.18
    nodeC role:Secondary
      replication:SyncSource peer-disk:Inconsistent done:39.49
  ```

## 驗證

* 完成後可以 mount /dev/drbd1000 至目錄，測試是否有完成資料同步
* nodeA 節點

  ```text
  # mount /dev/drbd1000 /tmp
  # touch /tmp/test.txt 
  # umonut /tmp
  ```

* nodeB 節點

  ```text
  # mount /dev/drbd1000 /tmp
  # ls /tmp
  test.txt
  ```

