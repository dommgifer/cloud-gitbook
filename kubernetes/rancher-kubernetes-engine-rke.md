# Rancher Kubernetes Engine\(RKE\)

* Rancher Kubernetes Engine（RKE）是一款輕量級Kubernetes安裝工具，支持在裸機和虛擬機上安裝。 RKE解決了Kubernettes安裝的複雜步驟。
* 參考資料: [http://staging.rancher.com/docs/rke/v0.1.x/en/installation/](http://staging.rancher.com/docs/rke/v0.1.x/en/installation/)

{% hint style="warning" %}
RKE版本持續更新，請參考官網，本文紀錄時還是很舊的版本
{% endhint %}

### 前置作業 <a id="&#x524D;&#x7F6E;&#x4F5C;&#x696D;"></a>

* 準備好被佈署的node
* node需安裝docker

  ```text
  curl https://releases.rancher.com/install-docker/17.03.sh | sh
  usermod -aG docker ubuntu
  ```

### 下載rke <a id="&#x4E0B;&#x8F09;rke"></a>

* 根據作業系統，[下載rke](https://github.com/rancher/rke/releases/tag/v0.1.8-rc5)
  * MacOS: rke_darwin-amd64 \* Linux: rke_linux-amd64
  * Windows: rke_windows-amd64.exe \* 將rke設為執行檔&lt;code&gt; \# chmod +x rke_linux-amd64

\# mv rke\_linux-amd64 /usr/local/bin/rke &lt;/code&gt;編輯此段

### 設定config <a id="&#x8A2D;&#x5B9A;config"></a>

* 新增cluster.yml

  ```text
  # If you intened to deploy Kubernetes in an air-gapped environment,
  # please consult the documentation on how to configure custom RKE images.
  nodes:
  - address: 10.50.2.23
    port: "22"
    internal_address: ""
    role:
    - controlplane
    - etcd
    hostname_override: ""
    user: ubuntu
    docker_socket: /var/run/docker.sock
    ssh_key: ""
    ssh_key_path: /home/localadmin/rke.pem
    labels: {}
  - address: 10.50.2.45
    port: "22"
    internal_address: ""
    role:
    - worker
    hostname_override: ""
    user: ubuntu
    docker_socket: /var/run/docker.sock
    ssh_key: ""
    ssh_key_path: /home/localadmin/rke.pem
    labels: {}
  - address: 10.50.2.47
    port: "22"
    internal_address: ""
    role:
    - worker
    hostname_override: ""
    user: ubuntu
    docker_socket: /var/run/docker.sock
    ssh_key: ""
    ssh_key_path: /home/localadmin/rke.pem
    labels: {}
  services:
    etcd:
      image: ""
      extra_args: {}
      extra_binds: []
      extra_env: []
      external_urls: []
      ca_cert: ""
      cert: ""
      key: ""
      path: ""
      snapshot: false
      retention: ""
      creation: ""
    kube-api:
      image: ""
      extra_args: {}
      extra_binds: []
      extra_env: []
      service_cluster_ip_range: 10.43.0.0/16
      service_node_port_range: ""
      pod_security_policy: false
    kube-controller:
      image: ""
      extra_args: {}
      extra_binds: []
      extra_env: []
      cluster_cidr: 10.42.0.0/16
      service_cluster_ip_range: 10.43.0.0/16
    scheduler:
      image: ""
      extra_args: {}
      extra_binds: []
      extra_env: []
    kubelet:
      image: ""
      extra_args: {}
      extra_binds: []
      extra_env: []
      cluster_domain: cluster.local
      infra_container_image: ""
      cluster_dns_server: 10.43.0.10
      fail_swap_on: false
    kubeproxy:
      image: ""
      extra_args: {}
      extra_binds: []
      extra_env: []
  network:
    plugin: flannel
    options: {}
  authentication:
    strategy: x509
    options: {}
    sans: []
  addons: ""
  addons_include: []
  system_images:
    etcd: rancher/coreos-etcd:v3.1.12
    alpine: rancher/rke-tools:v0.1.10
    nginx_proxy: rancher/rke-tools:v0.1.10
    cert_downloader: rancher/rke-tools:v0.1.10
    kubernetes_services_sidecar: rancher/rke-tools:v0.1.10
    kubedns: rancher/k8s-dns-kube-dns-amd64:1.14.8
    dnsmasq: rancher/k8s-dns-dnsmasq-nanny-amd64:1.14.8
    kubedns_sidecar: rancher/k8s-dns-sidecar-amd64:1.14.8
    kubedns_autoscaler: rancher/cluster-proportional-autoscaler-amd64:1.0.0
    kubernetes: rancher/hyperkube:v1.10.3-rancher2
    flannel: rancher/coreos-flannel:v0.9.1
    flannel_cni: rancher/coreos-flannel-cni:v0.2.0
    calico_node: rancher/calico-node:v3.1.1
    calico_cni: rancher/calico-cni:v3.1.1
    calico_controllers: ""
    calico_ctl: rancher/calico-ctl:v2.0.0
    canal_node: rancher/calico-node:v3.1.1
    canal_cni: rancher/calico-cni:v3.1.1
    canal_flannel: rancher/coreos-flannel:v0.9.1
    wave_node: weaveworks/weave-kube:2.1.2
    weave_cni: weaveworks/weave-npc:2.1.2
    pod_infra_container: rancher/pause-amd64:3.1
    ingress: rancher/nginx-ingress-controller:0.10.2-rancher3
    ingress_backend: rancher/nginx-ingress-controller-defaultbackend:1.4
  ssh_key_path: /home/localadmin/mykey.pem
  ssh_agent_auth: false
  authorization:
    mode: rbac
    options: {}
  ignore_docker_version: false
  kubernetes_version: ""
  private_registries: []
  ingress:
    provider: ""
    options: {}
    node_selector: {}
    extra_args: {}
  cluster_name: ""
  cloud_provider:
    name: ""
  prefix_path: ""
  addon_job_timeout: 0
  bastion_host:
    address: ""
    port: ""
    user: ""
    ssh_key: ""
    ssh_key_path: ""
  ```

### 部屬k8s <a id="&#x90E8;&#x5C6C;k8s"></a>

* deploy k8s

  ```text
  # rke up
  ```

### 驗證k8s <a id="&#x9A57;&#x8B49;k8s"></a>

* [安裝kubectl](https://kubernetes.io/docs/reference/kubectl/kubectl/)
* use kubectl

  ```text
  kubectl --kubeconfig kube_config_cluster.yml get nodes
  ```

