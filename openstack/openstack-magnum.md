---
description: 此篇也是古文，將以前學習的過程搬上來
---

# OpenStack magnum



{% hint style="danger" %}
紀錄時間為:2017/09/12

此篇已過時，請勿參考  
當時是利用 DevStack 安裝 magnum，也是第一次接觸 Kubernetes，所以記錄下來
{% endhint %}

{% hint style="info" %}
本測試devstack裝在實體機的環境下是可行的，但是devstack裝在虛擬機上，  
建立k8s的master會失敗。   
\(更新\)官網Ocata和Pike版已經更新magnum了，可以不用透過devstack安裝最新版
{% endhint %}

Reference:

* [https://kubernetes.io/docs/home/](https://kubernetes.io/docs/home/)
* [https://goo.gl/PYBznn](https://goo.gl/PYBznn)
* [https://tachingchen.com/tw/blog/Kubernetes-Service/](https://tachingchen.com/tw/blog/Kubernetes-Service/)
* [http://www.inwinstack.com/index.php/zh/2017/06/15/the-kubernetes-package-manager-helm/](http://www.inwinstack.com/index.php/zh/2017/06/15/the-kubernetes-package-manager-helm/)



## Install OpenStack from Devstack

* Clone devstack

  ```text
  git clone https://github.com/openstack-dev/devstack.git
  ```

* Edit local.confg

  ```text
  [[local|localrc]]

  ADMIN_PASSWORD=openstack
  DATABASE_PASSWORD=openstack
  RABBIT_PASSWORD=openstack
  SERVICE_PASSWORD=$ADMIN_PASSWORD

  HOST_IP=10.0.2.7

  # Logging
  LOGFILE=$DEST/logs/stack.sh.log

  # Old log files are automatically removed after 7 days to keep things neat.  Change
  # the number of days by setting ``LOGDAYS``.
  LOGDAYS=2

  # Nova logs will be colorized if ``SYSLOG`` is not set; turn this off by setting
  # ``LOG_COLOR`` false.
  #LOG_COLOR=False

  # Swift
  # -----

  # Swift is now used as the back-end for the S3-like object store. Setting the
  # hash value is required and you will be prompted for it if Swift is enabled
  # so just set it to something already:
  SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5

  # For development purposes the default of 3 replicas is usually not required.
  # Set this to 1 to save some resources:
  SWIFT_REPLICAS=1

  # The data for Swift is stored by default in (``$DEST/data/swift``),
  # or (``$DATA_DIR/swift``) if ``DATA_DIR`` has been set, and can be
  # moved by setting ``SWIFT_DATA_DIR``. The directory will be created
  # if it does not exist.
  SWIFT_DATA_DIR=$DEST/data

  PUBLIC_INTERFACE=eno1
  FLOATING_RANGE=10.0.2.0/22
  PUBLIC_NETWORK_GATEWAY=10.0.3.254
  Q_FLOATING_ALLOCATION_POOL=start=10.0.2.70,end=10.0.2.79
  Q_USE_PROVIDERNET_FOR_PUBLIC=False
  Q_ASSIGN_GATEWAY_TO_PUBLIC_BRIDGE=False
  # Magnum
  enable_plugin barbican https://git.openstack.org/openstack/barbican
  enable_plugin heat https://git.openstack.org/openstack/heat
  enable_plugin magnum https://git.openstack.org/openstack/magnum
  enable_plugin magnum-ui https://github.com/openstack/magnum-ui

  VOLUME_BACKING_FILE_SIZE=40G
  enable_plugin neutron-lbaas https://git.openstack.org/openstack/neutron-lbaas
  enable_plugin octavia https://git.openstack.org/openstack/octavia

  # Disable LBaaS(v1) service
  disable_service q-lbaas
  # Enable LBaaS(v2) services
  enable_service q-lbaasv2
  enable_service octavia
  enable_service o-cw
  enable_service o-hk
  enable_service o-hm
  enable_service o-api
  ```

* install devstack

  ```text
  ./stack
  ```



## Magnum cluster template create

* create magnum cluster template

  ```text
  magnum cluster-template-create k8s-cluster-template\
         --image fedora-atomic-latest \
         --keypair testkey \
         --external-network public \
         --dns-nameserver 8.8.8.8 \
         --flavor m1.small \
         --docker-volume-size 15 \
         --network-driver flannel \
         --coe kubernetes
  ```



## Create magnum cluster

* Create magnum cluster

  ```text
  magnum cluster-create k8s-cluster \
         --cluster-template k8s-cluster-template \
         --node-count 1
  ```



## Setting up the environment and artifacts for TLS

* Setting up the environment and artifacts for TLS

  ```text
  mkdir key
  magnum cluster-config k8s-cluster-demo --dir key
  mkdir ~/.kube
  cd key
  cp config ~/.kube/config
  ```

* Edit config and modify path of **certificate-authority,client-certificate,client-key**

  ```text
  vim ~/.kube/config
  ```

  ```text
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority: /home/localadmin/key/ca.pem
      server: https://10.0.2.58:6443
    name: k8s-cluster
  contexts:
  - context:
      cluster: k8s-cluster
      user: k8s-cluster
    name: default/k8s-cluster
  current-context: default/k8s-cluster
  kind: Config
  preferences: {}
  users:
  - name: k8s-cluster
    user:
      client-certificate: /home/localadmin/key/cert.pem
      client-key: /home/localadmin/key/key.pem
  ```



## linux connect to k8s cluster

* linux connect to k8s cluster and Verify connection

  ```text
  curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
  sudo cp kubectl /usr/bin/kubectl
  sudo chmod +x /usr/bin/kubectl
  kubectl cluster-info
  kubectl -n kube-system get pod
  ```



## windows connect to k8s cluster

* Installing Chocolatey
* First, ensure that you are using an administrative shell - you can also install as a non-admin, check out Non-Administrative Installation.
* Copy the text specific to your command shell - cmd.exe.

  ```text
  @"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
  ```

* Paste the copied text into your shell and press Enter.
* Wait a few seconds for the command to complete.
* if you don't see any errors, you are ready to use Chocolatey!
* Use choco command to install kubectl.

  ```text
  choco install kubernetes-cli
  ```

* **Copy config and key from k8s-cluster server and create .kube directory.**

  ```text
  cd C:\users\yourusername
  mkdir .kube
  cd .kube
  ```

* Edit config and modify path of **certificate-authority,client-certificate,client-key**

  ```text
  apiVersion: v1
  clusters:
  - cluster:
      certificate-authority: C:\Users\170516/tls/ca.pem
      server: https://10.0.2.74:6443
    name: k8s-cluster
  contexts:
  - context:
      cluster: k8s-cluster
      user: user
    name: main
  current-context: main
  kind: Config
  preferences: {}
  users:
  - name: user
    user:
      client-certificate: C:\Users\170516/tls/user.pem
      client-key: C:\Users\170516/tls/user-key.pem
  ```

* Verify your connection

  ```text
  kubectl cluster-info
  kubectl -n kube-system get pod
  ```



## Run proxy and access dashboard from localhost

* Run your kubectl proxy

  ```text
  kubectl proxy
  ```

* Open browesr and k8s dashboard url is [**http://localhost:8001/ui**](http://localhost:8001/ui)

#### Access dashboard from NodePort <a id="access_dashboard_from_nodeport"></a>

* Change dashboard service type clusterIP to NodePort

  ```text
  kubectl edit svc kubernetes-dashboard --namespace=kube-system
  ```

  ```text
  apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: 2017-07-05T02:25:05Z
    labels:
      app: kubernetes-dashboard
    name: kubernetes-dashboard
    namespace: kube-system
    resourceVersion: "40"
    selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard
    uid: 2830e5dc-6129-11e7-811e-fa163e5fed51
  spec:
    clusterIP: 10.254.16.128
    ports:
    - nodePort: 30809
      port: 80
      protocol: TCP
      targetPort: 9090
    selector:
      app: kubernetes-dashboard
    sessionAffinity: None
    type: NodePort
  status:
    loadBalancer: {}
  ```

* Get dasboard port

  ```text
  $ kubectl get services
  NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
  kube-dns               10.254.0.10     <none>        53/UDP,53/TCP   6d
  kubernetes-dashboard   10.254.16.128   <nodes>       80:30809/TCP    6d
  ```

Open browesr and k8s dashboard url is\(master IP:port\)  
 [**http://10.0.2.74:30809**](http://10.0.2.74:30809/)



## Example : Create Guestbook app from yaml

* Clone example from k8s gitub

  ```text
  git clone https://github.com/kubernetes/kubernetes.git
  ```

* Edit example yaml and modify **type:LoadBalancer to type:NodePort** from frontend service section.

  ```text
  vim kubernetes/examples/guestbook/all-in-one/guestbook-all-in-one.yaml
  ```

  ```text
  apiVersion: v1
  kind: Service
  metadata:
    name: frontend
    labels:
      app: guestbook
      tier: frontend
  spec:
    # if your cluster supports it, uncomment the following to automatically create
    # an external load-balanced IP for the frontend service.
    type: NodePort
    ports:
    - port: 80
    selector:
      app: guestbook
      tier: frontend
  ```

* Create Guestbook app

  ```text
  cd kubernetes 
  kubectl create -f examples/guestbook/all-in-one/guestbook-all-in-one.yaml
  ```

* Check your guestbook service

  ```text
  $ kubectl get services

  NAME              CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
  frontend          10.254.116.165   <nodes>       80:32020/TCP   2d
  kubernetes        10.254.0.1       <none>        443/TCP        5d
  redis-master      10.254.255.211   <none>        6379/TCP       2d
  redis-slave       10.254.23.157    <none>        6379/TCP       2d
  ```

* Access your guestbook from k8s master or minion floating ip and service port  \(you can get port form command **kubectl get services**\) ,  For example: k8s master floating ip is 10.0.2.74, access guestgook url is 10.0.2.74:32020

## Use helm manage k8s package

### Install helm tool

* wget helm tool and install

  ```text
  wget https://kubernetes-helm.storage.googleapis.com/helm-v2.4.1-linux-amd64.tar.gz 
  tar zxvf helm-v2.4.1-linux-amd64.tar.gz
  sudo mv linux-amd64/helm /usr/local/bin/
  ```

* helm initialization

  ```text
  helm init
  ```

* Check tiller service has been created

  ```text
  $ kubectl get po,svc -n kube-system -l app=helm

  NAME                                READY     STATUS    RESTARTS   AGE
  po/tiller-deploy-3210876050-p6l61   1/1       Running   0          5d

  NAME                CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
  svc/tiller-deploy   10.254.91.237   <none>        44134/TCP   5d
  ```

### using helm to install app - jenkins

* helm repository update

  ```text
  helm repo update
  ```

* Search jenkins

  ```text
  $ helm search jenkins
  NAME          	VERSION	DESCRIPTION
  stable/jenkins	0.6.3  	Open source continuous integration server. It s...
  ```

* Create jenkins app

  ```text
  helm install --name jenkins stable/jenkins
  ```

{% hint style="warning" %}
 **WARNING**: This example didn't create Persistent Volume ,You will lose your data when the Jenkins pod is terminated.
{% endhint %}

* Check jenkins deploy complete

  ```text
  $ kubectl get po,svc
  NAME                                  READY     STATUS    RESTARTS   AGE
  po/jenkins-jenkins-2977593992-22btp   1/1       Running   0          6m

  NAME                        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
  svc/jenkins-jenkins         10.254.146.96    172.0.0.11    8080:31635/TCP   6m
  svc/jenkins-jenkins-agent   10.254.35.222    <none>        50000/TCP        6m
  svc/kubernetes              10.254.0.1       <none>        443/TCP          5d
  ```

* You can accsee jenkins by EXTERNAL-IP, but this k8s cluster in openstack vm , so you must access by master or minion floating ip and port, edit jenkins service .

  ```text
  kubectl edit svc jenkins-jenkins 
  ```

  
  \* Remove about Loadbalance and change type to NodePort

  ```text
  apiVersion: v1
  kind: Service
  metadata:
    creationTimestamp: 2017-07-10T09:46:35Z
    labels:
      app: jenkins-jenkins
      chart: jenkins-0.8.1
      component: jenkins-jenkins-master
      heritage: Tiller
      release: jenkins
    name: jenkins-jenkins
    namespace: default
    resourceVersion: "544166"
    selfLink: /api/v1/namespaces/default/services/jenkins-jenkins
    uid: a97aa98d-6554-11e7-811e-fa163e5fed51
  spec:
    clusterIP: 10.254.146.96
      ports:
    - name: http
      nodePort: 31635
      port: 8080
      protocol: TCP
      targetPort: 8080
    selector:
      component: jenkins-jenkins-master
    sessionAffinity: None
    type: NodePort
  ```

* Get jenkins service port

  ```text
  $ kubectl get svc
  NAME                        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
  svc/jenkins-jenkins         10.254.146.96    <nodes>       8080:31635/TCP   18m
  svc/jenkins-jenkins-agent   10.254.35.222    <none>        50000/TCP        18m
  svc/kubernetes              10.254.0.1       <none>        443/TCP          5d
  ```

* Get jenkins password

  ```text
  $ printf $(kubectl get secret --namespace default jenkins-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo 

  iyfUkr4uYK
  ```

 Login jenkins

```text
Dashboard IP : master or minion floating ip:port(example: 10.0.2.74:31635)
Account : admin
Password: iyfUkr4uYK
```

