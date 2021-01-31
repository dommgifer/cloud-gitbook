---
description: 使用 rook 安裝 ceph
---

# Ceph On Kubernetes

{% hint style="info" %}
此篇記錄時間為 2020/01/31，內容可能過時，請斟酌參考
{% endhint %}

* [關於 rook](https://rook.io/docs/rook/v1.2/)
* [參考文件](https://rook.io/docs/rook/v1.2/ceph-quickstart.html)
* 請先準備好 Kubernetes
* Kubernetes 至少3個worker節點
* worker節點請準備第2顆硬碟當OSD

## 安裝 Rook-Ceph

* 請根據目前版本 Clone rook-ceph

  ```text
  # git clone --single-branch --branch release-1.2 https://github.com/rook/rook.git
  # cd roook/cluster/examples/kubernetes/ceph
  # kubectl create -f common.yaml
  # kubectl create -f operator.yaml
  # kubectl create -f cluster.yaml
  ```

* 確認 pod 是否正常

  ```text
  # kubectl -n rook-ceph get pod
  NAME                                                READY   STATUS      RESTARTS   AGE
  csi-cephfsplugin-48l98                              3/3     Running     0          110m
  csi-cephfsplugin-9lcz5                              3/3     Running     0          110m
  csi-cephfsplugin-nfx4d                              3/3     Running     0          110m
  csi-cephfsplugin-provisioner-799547ccdd-565gt       4/4     Running     0          110m
  csi-cephfsplugin-provisioner-799547ccdd-wqv2n       4/4     Running     0          110m
  csi-rbdplugin-445z7                                 3/3     Running     0          110m
  csi-rbdplugin-f8gtt                                 3/3     Running     0          110m
  csi-rbdplugin-nj4sv                                 3/3     Running     0          110m
  csi-rbdplugin-provisioner-78547ddbc9-fr4s6          5/5     Running     0          110m
  csi-rbdplugin-provisioner-78547ddbc9-l6ptp          5/5     Running     0          110m
  rook-ceph-crashcollector-worker1-785c889774-xzg5t   1/1     Running     0          106m
  rook-ceph-crashcollector-worker2-dc8cf4df5-rl79v    1/1     Running     0          107m
  rook-ceph-crashcollector-worker3-68fdf7d8d-ghznf    1/1     Running     0          87m
  rook-ceph-mgr-a-7cb964c4f9-7sxhw                    1/1     Running     0          106m
  rook-ceph-mon-a-57ddf967f-hxdm8                     1/1     Running     0          107m
  rook-ceph-mon-b-69d7576d7-nvt65                     1/1     Running     0          107m
  rook-ceph-mon-c-df5c97cf9-t7b5f                     1/1     Running     0          106m
  rook-ceph-operator-586d948744-vc8sz                 1/1     Running     0          115m
  rook-ceph-osd-0-8854465d-zmzrb                      1/1     Running     0          87m
  rook-ceph-osd-1-6c77d4b8fd-zw5tr                    1/1     Running     0          87m
  rook-ceph-osd-2-75c88d95b8-9ltc7                    1/1     Running     0          87m
  rook-ceph-osd-prepare-worker1-7rqgb                 0/1     Completed   0          25m
  rook-ceph-osd-prepare-worker2-7hvpw                 0/1     Completed   0          25m
  rook-ceph-osd-prepare-worker3-qdrlh                 0/1     Completed   0          25m
  rook-discover-2f82d                                 1/1     Running     0          114m
  rook-discover-qd79q                                 1/1     Running     0          114m
  rook-discover-qlhxf                                 1/1     Running     0          114m
  ```

如果發現沒有 rook-ceph-osd-0、1、2 請檢查是否有第2顆硬碟編輯此段

### 建立 ceph-dashboard

* 啟用 ceoh-dashboard

  ```text
  # kubectl create -f dashboard-external-https.yaml
  ```

* 取得 dashboard port

  ```text
  # kubectl get svc -n rook-ceph | grep rook-ceph-mgr-dashboard-external-https
  rook-ceph-mgr-dashboard-external-https   NodePort    10.43.156.215   <none>        8443:32607/TCP      110m
  ```

* 取得 dashboard 密碼

  ```text
  # kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

  5G14UM0iWP
  ```

進入 dashboard 請輸入 https://node\_ip:port  
此範例 port為 32607 密碼為 5G14UM0iWP  
帳號預設為 admin

### 建立 storage class

* 切換目錄至 csi/rbd

  ```text
  # cd csi/rbd
  ```

* 建立 rook-ceph-block storageclass

  ```text
  # kubectl apply -f storageclass.yaml
  ```

### 建立 pvc 測試

* 建立一個 wordpress 和 mysql 測試功能是否正常
* 回到 kubernetes 目錄並建立服務

  ```text
  # cd cluster/examples/kubernetes
  # kubectl create -f mysql.yaml
  # kubectl create -f wordpress.yaml
  ```

* 查看 pvc 是否正常

  ```text
  # kubectl get pvc
  NAME             STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
  mysql-pv-claim   Bound     pvc-95402dbc-efc0-11e6-bc9a-0cc47a3459ee   20Gi       RWO           1m
  wp-pv-claim      Bound     pvc-39e43169-efc1-11e6-bc9a-0cc47a3459ee   20Gi       RWO           1m
  ```

