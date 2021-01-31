# Kubernetes OpenStack Integration

* Kubernetes 所建立 pod 的資料可以儲存在 cinder 的 volume，建立的 service 也可以使用 neutron 的 loadbalancer 將對外服務開放，為了達成上述的目的，需要填入 openstack 相關資訊。
* [更多詳細說明請看官方文件](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#openstack)
* 一般而言在 kubernetes master 的節點 /etc/kubernetes/cloud.conf 內填入 openstack 資訊，內容如下:

  ```text
  [Global]
  auth-url    = http://10.50.0.101:5001/v3
  user-id     = bc45deeb36bf4cc79d275ae29ce84bb5
  password    = openstack
  tenant-id   = 8d3895eb8fef4fca8e38d29b8aa11259

  [LoadBalancer]
  subnet-id           = 796e12fd-a050-46a4-9bd0-0218a0dae931
  floating-network-id = f2d1eaa6-39a8-4560-94ff-49164542424f

  [BlockStorage]
  bs-version       = v2
  ignore-volume-az = true
  ```

* 填完後，把 k8s 上的 service 型態改為 LoadBalancer，OpenStack 會根據上面的使用者跟 project，建立loadbalancer到該專案底下

## Rancher 流程

* 透過 Rancher 所建立的 k8s ，可以在建立 k8s 前填完上面的參數，之後 Rancher 會自動建立此 config
* Rancher 使用建立 cluster API 時，需要加入[cloudProvider參數](http://10.50.0.12/wiki/doku.php?id=project:openstack:container:rancher_api#%E5%BB%BA%E7%AB%8B_cluster)，範例如下

  ```text
  {
    "type": "cluster",
    "rancherKubernetesEngineConfig": {
     .........
      "cloudProvider": {
        "type": "/v3/schemas/cloudProvider",
        "name": "openstack",
        "openstackCloudProvider": {
          "blockStorage": {
            "bs-version": "v2",
            "ignore-volume-az": false,
            "trust-device-path": false,
            "type": "/v3/schemas/blockStorageOpenstackOpts"
          },
          "global": {
            "auth-url": "http://10.50.2.10:5001/v3",
            "ca-file": "",
            "domain-id": "",
            "domain-name": "",
            "password": "openstack",
            "region": "",
            "tenant-id": "885c63821c3a43a78da157958eca3a99",
            "tenant-name": "",
            "trust-id": "",
            "type": "/v3/schemas/globalOpenstackOpts",
            "user-id": "bc45deeb36bf4cc79d275ae29ce84bb5",
            "username": ""
          },
          "loadBalancer": {
            "create-monitor": false,
            "floating-network-id": "d0456ee8-c9a9-4e0c-8be1-0974f1d13ac0",
            "lb-method": "",
            "lb-provider": "",
            "lb-version": "",
            "manage-security-groups": false,
            "monitor-delay": 0,
            "monitor-max-retries": 0,
            "monitor-timeout": 0,
            "subnet-id": "12c1529a-2f54-43be-8f04-f766369c2b1d",
            "type": "/v3/schemas/loadBalancerOpenstackOpts",
            "use-octavia": false
          },
          "metadata": {
            "request-timeout": 0,
            "search-order": "",
            "type": "/v3/schemas/metadataOpenstackOpts"
          },
          "route": {
            "router-id": "69ff6c93-c752-4cc3-9fa3-e3ce5adf2aee",
            "type": "/v3/schemas/routeOpenstackOpts"
          },
          "type": "/v3/schemas/openstackCloudProvider"
        }
      }
    },
    "name": "yournewcluster"
  }
  ```

### 問題 <a id="&#x554F;&#x984C;"></a>

* 由於需要知道 OpenStack 的使用者名稱和密碼，因此衍生出下列問題
  1. 當使用者更改密碼時，此 config 的密碼也只必須修改。

### 解決方法 <a id="&#x89E3;&#x6C7A;&#x65B9;&#x6CD5;"></a>

* OpenStack Keystone API v3 extensions 有個功能叫做 trust，可以建立委託者\(trust\)和受託者\(trustee\)的關係，委託者可以授權給受託者，讓受託者可以針對被授權的project進行資源管理。
* 例如: 使用者admin 委託 使用者trustee 有權限對於 client 這個 project 進行資源管理。
* 建立 trust 的流程如下:
  1. 建立使用者，角色為Member，名稱自訂\(此範例為trustee\)，密碼建議亂數
  2. 使用 admin token 建立 trust，trustor 為 admin使用者，trustee 為 trustee使用者，project 為 client，角色為 admin，使用指令如下

     ```text
     openstack trust create --role Admin --project client --impersonate admin trustee
     ```

  * 詳細API參數說明，[請參考官方文件](https://developer.openstack.org/api-ref/identity/v3-ext/?expanded=consuming-a-trust-detail,create-trust-detail#create-trust)，以下為API範例

    ```text
    URL:  /v3/OS-TRUST/trusts
    {
        "trust": {
            "impersonation": true,
            "project_id": "8d3895eb8fef4fca8e38d29b8aa11259",
            "roles": [
                {
                    "name": "Member"
                }
            ],
            "trustee_user_id": "bc45deeb36bf4cc79d275ae29ce84bb5",
            "trustor_user_id": "bfa1f7dd0a1646a5a27791163e5ae200"
        }
    }
    ```

  * trustor\_user\_id: 委託人ID，為 admin
  * trustee\_user\_id: 受託人ID，為 trustee
  * project\_id: 授權 project ID，為 client project ID
  * roles: 權限，只能授權 trust 所在 project 底下的權限

{% hint style="info" %}
檢查使用者 project 底下的 admin 成員，權限是否為 admin, Member 

如果不是，使用上面 API 時填 Member 會錯誤，因為 admin 無法授權 Member 權限給 trustee，只能授權 admin，但是授權 admin 極度不安全，因此要可以授權 Member 的角色，需要先去修改使用者 project 底下的 admin 成員，將權限改為 admin,Member ， [API 部分請參考官網](https://goo.gl/7p8qXF)
{% endhint %}

* 建立完成會得到此 trust 的 id，之後 Rancher 建立 cluster 的 API 中填入 trustee 使用者的ID跟密碼以及 trust 的 ID

  ```text
  "global": {
     "auth-url": "http://10.50.2.10:5001/v3",
     "password": "openstack",
     "tenant-id": "885c63821c3a43a78da157958eca3a99",
     "trust-id": "719e810d35a04c49a86a24a855c31ffc",
     "user-id": "bc45deeb36bf4cc79d275ae29ce84bb5"
    },
  ```

* 這樣 k8s 就會使用 trustee 使用者對指定的 project 做資源管理
* 利用這樣的作法解決了使用者變更密碼的問題，透過 admin 建立 trustee 的使用者，密碼也是 admin 決定，因此可以得知密碼，密碼也不會更換。
* 這裡衍伸出了一個問題
  1. 當使用者建立 第一個 k8s，系統產生了一個 trustee 給這個 k8s 使用，
  2. 使用者再建立一個 k8s ，這時候如過要填入剛剛產生的 trustee 密碼，這時候資料庫就必須記錄 trustee 的密碼，就需要對密碼做加密和解密，所以只能選擇對稱式加密的演算法，因為密碼需要名碼。
  3. 如果要避開密碼記錄在資料庫，只能每個 k8s 都建立一個 trustee，只是會產生很多使用者。

## 結論

* Kubernetes 和 OpenStack 結合需要以下的資料:
  1. Keystone URL
  2. user-id \(trustee 的 id\)
  3. password
  4. tenant-id \(使用者的 project id\)
  5. trust-id
  6. subnet-id \(k8s VM 所在的 subnet id\)
  7. floating-network-id \(外網的 network id\)
* 使用者得知 trustee 的 id 和密碼，是否可以使用 OpenStack API 操作資源?
  * 可以，但是無法使用一般登入API的方法\(/v3/auth/tokens\)，需要經過二次認證才能取得有效的 token
  * 如果只使用 trustee id 和密碼登入取得的 token 沒有什麼用處，因為 trustee 不屬於任何 project 底下的成員，因此無法操作任何 project 底下的資源。
  * 所謂的二次認證是指先使用一般的認證\(帳密認證\)取得 token，[再使用這個 token 和 trust id 呼叫認證](https://developer.openstack.org/api-ref/identity/v3-ext/?expanded=consuming-a-trust-detail,create-trust-detail,list-consumers-detail#consuming-a-trust)，回傳的 token 才能對授權的 project 進行資源操作，因此使用者必須了解其相關性，才能使用 API 操作資源，以下是第二次認證要填的資料

    ```text
    {
        "auth": {
            "identity": {
                "methods": [
                    "token"
                ],
                "token": {
                    "id": "一般登入的token"
                }
            },
            "scope": {
                "OS-TRUST:trust": {
                    "id": "trust 的 id"
                }
            }
        }
    }
    ```

