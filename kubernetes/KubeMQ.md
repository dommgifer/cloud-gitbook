# KubeMQ 介紹

{% hint style="info" %}
本文轉寫時間為 2025年10月07日，內容可能會有變動，僅記錄
並同時發佈在 [OSI Tech Blog](https://medium.com/osi-tech-blog/kubemq-%E4%BB%8B%E7%B4%B9-22a709677f24)
{% endhint %}

## 介紹

KubeMQ 是一套雲原生訊息佇列（Message Queue）與事件串流中介軟體，支援多種訊息模型（Queue、Pub/Sub、Command/Query），並提供擴充模組如 Targets、Sources、Bridges，方便與外部系統整合。

- 適用於 Kubernetes 與多雲環境
- 支援多種通訊模式（async/sync）
- 提供 UI、CLI、SDK 管理與測試介面
- 架構模組化，可插拔式擴充

## 安裝 KubeMQ

```bash
helm repo add kubemq-charts https://kubemq-io.github.io/charts
````

安裝 Controller：

```bash
helm install --wait -n kubemq kubemq-controller kubemq-charts/kubemq-controller
```

安裝 Cluster，key 的部分需要官網註冊，提供30天試用：

```bash
helm install --wait -n kubemq kubemq-cluster --set key={your-license-key} kubemq-charts/kubemq-cluster
```
- 檢查 KubeMQ Cluster 狀態&#x20;

    <figure><img src="../.gitbook/assets/kubemq/cluster-status-check.png" alt=""><figcaption></figcaption></figure>
- KubeMQ 提供 Dashboard 檢視KubeMQ Cluster 狀態&#x20;

    <figure><img src="../.gitbook/assets/kubemq/dashboard-cluster-overview.png" alt=""><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/kubemq/dashboard-cluster-detail.png" alt=""><figcaption></figcaption></figure>


# Queue
KubeMQ 的 Queue 模式是一種「可靠且可保序」的非同步通訊方式，適用於：
- 生產者與消費者不同步的情境（例如：前台快速送出任務、後台慢慢處理）
- 保證訊息至少被處理一次（At-Least-Once Delivery）
- 任務佇列、批次處理、非同步工作分派

特性：
- 支援 Ack（確認已處理）
- 支援 Dead-letter（處理失敗時轉移）
- 支援 FIFO 順序性

以下範例透過[KubeMQ 官方的範例程式](https://github.com/kubemq-io/kubemq-Python/tree/master/examples)收發訊息

範例：
- 建立 queue channel&#x20;

    <figure><img src="../.gitbook/assets/kubemq/queue-channel-create.png" alt=""><figcaption></figcaption></figure>
- 送出訊息至 auto_ack channel&#x20;

    <figure><img src="../.gitbook/assets/kubemq/queue-send-message.png" alt=""><figcaption></figcaption></figure>

# Pub/Sub

Pub/Sub（發佈／訂閱） 模式適合用於：
- 多個接收者「同時」收到相同訊息（像廣播）
- 微服務間的鬆耦合事件流設計
- 實時通知、活動推播、監控事件

特性：
- 非同步、即時事件推播
- 一對多傳遞模型
- 不保證順序、不保證持久（除非搭配 Events Store）

範例：
- 透過 Publisher  Subscriber 的程式 至 demo-channel 收發訊息
 - Publisher&#x20;

    <figure><img src="../.gitbook/assets/kubemq/pubsub-publisher.png" alt=""><figcaption></figcaption></figure>
 - Subscriber&#x20;

    <figure><img src="../.gitbook/assets/kubemq/pubsub-subscriber.png" alt=""><figcaption></figcaption></figure>

Dashboard 查看 channel 狀況:

<figure><img src="../.gitbook/assets/kubemq/pubsub-dashboard-channel.png" alt=""><figcaption></figcaption></figure>

- 可以用拓樸圖呈現 滑鼠移到client 還可顯示關於client 的相關資訊

<figure><img src="../.gitbook/assets/kubemq/pubsub-topology-graph.png" alt=""><figcaption></figcaption></figure>

- 在client 的頁籤，還可以看到每一個client處理訊息的資訊

<figure><img src="../.gitbook/assets/kubemq/pubsub-client-info.png" alt=""><figcaption></figcaption></figure>




# Commands & Queries 簡介

KubeMQ 的 **Commands & Queries** 模式，提供一種同步請求回應的通訊方式，可用於需要「立即得到結果」或「返回資料」的場景。


##  Command 模式

- **用途**：變更系統狀態、觸發動作
- **通訊特性**：同步請求 → 等待回應
- **回應內容**：成功／失敗狀態（`is_executed`）
- **範例**：
  - 下訂單並返回是否成功
  - 執行異動並確認已套用

---

### 範例
- 先測試只有 command client 也就是送出指令的一端，可以看到 timeout，因為沒有收到回應，所以不像 Qeueu 一樣，送出訊息就完成&#x20;

    <figure><img src="../.gitbook/assets/kubemq/command-client-timeout.png" alt=""><figcaption></figcaption></figure>

1. 這次多執行 command server 也就是執行命令端

<figure><img src="../.gitbook/assets/kubemq/command-server-start.png" alt=""><figcaption></figcaption></figure>

2. command client 送出指令，得到了回應

<figure><img src="../.gitbook/assets/kubemq/command-client-response.png" alt=""><figcaption></figcaption></figure>

3. 查看 command server，確實收到請求，這範例程式只要收到訊息，就回應說我完成了

<figure><img src="../.gitbook/assets/kubemq/command-server-received.png" alt=""><figcaption></figcaption></figure>

- Dashboard 查看 channel 狀況:&#x20;

    <figure><img src="../.gitbook/assets/kubemq/command-dashboard-channel.png" alt=""><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/kubemq/command-dashboard-detail.png" alt=""><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/kubemq/command-dashboard-client.png" alt=""><figcaption></figcaption></figure>
- 透過Dashboard也可以看到訊息的收發&#x20;

    <figure><img src="../.gitbook/assets/kubemq/command-dashboard-messages.png" alt=""><figcaption></figcaption></figure>

---

## Query 模式

- **用途**：讀取系統資料、查詢狀態
- **通訊特性**：同步請求 → 等待回應
- **回應內容**：實際資料（回傳 `body`）
- **範例**：
  - 查詢用戶餘額並返回金額
  - 取得訂單詳情清單

### 範例
- 啟動 query server 也就是執行命令端 (這裡就不示範timeout了，因為跟command 模式一樣)

<figure><img src="../.gitbook/assets/kubemq/query-server-start.png" alt=""><figcaption></figcaption></figure>

- 啟動 query client 也就是送命令端

<figure><img src="../.gitbook/assets/kubemq/query-client-start.png" alt=""><figcaption></figcaption></figure>

- 透過Dashboard 查看channel 資訊(其餘頁面都差不多)&#x20;

    <figure><img src="../.gitbook/assets/kubemq/query-dashboard-channel.png" alt=""><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/kubemq/query-dashboard-detail.png" alt=""><figcaption></figcaption></figure>


---

## Command 和 Query 比較

概念跟RESTful API 裡的「語意約定」很像：
- Query → GET
GET 請求只讀不改變伺服器狀態。
- Command → POST / PUT / DELETE / PATCH
這些動詞承載「副作用」，用來建立、修改或刪除資源，伺服器會依請求做出變動。

| 面向      | Command                    | Query                     |
|-----------|----------------------------|---------------------------|
| 目的      | 寫入／觸發（改變狀態）     | 讀取資料（不改變狀態）    |
| 回應內容  | 執行結果（成功／失敗）     | 查詢結果（資料本身）      |
| 幂等性    | 視業務而定，可能非幂等     | 應為幂等，多次請求相同結果 |
| 使用時機  | 需要確認操作有被執行       | 只需取回資料，不產生副作用 |

---

## 使用建議

- **Command**：適用於需修改資料或執行動作且必須確認結果的場景。
- **Query**：適用於只需讀取／查詢資料，不影響系統狀態的場景。


# Connector

KubeMQ 提供的 **Connector 系列** 是一組支援外部系統整合的元件，主要包含：

| 類型          | 功能說明                   | 範例                               |
| ----------- | ---------------------- | -------------------------------- |
| **Targets** | 把 KubeMQ 中的訊息「寫出」到外部系統 | 寫入資料庫、推送到 Kafka、觸發 REST API      |
| **Sources** | 從外部系統「接收」資料進 KubeMQ    | 拉 Kafka 訊息、接 webhook、讀 Redis key |
| **Bridges** | 在多個 KubeMQ 叢集之間「橋接」訊息  | 跨地區多叢集同步、分散式服務整合                 |

這些 Connector 可以大幅簡化與外部服務的整合，讓開發者只需與 KubeMQ 溝通，**不需要直接管理 Redis、PostgreSQL、Kafka 等第三方 client library**，提高系統一致性與維運效率。

---


## KubeMQ Targets
KubeMQ Target 是 Connector 家族的一員，負責將 KubeMQ Channel 內的訊息「寫出」到外部系統，例如：
- 資料庫：PostgreSQL、MySQL、MongoDB
- 快取：Redis、Memcached
- 訊息平台：Kafka、RabbitMQ、ActiveMQ
- 雲服務：AWS S3 / SQS、GCP Pub/Sub、Azure Service Bus
- 任何 HTTP / REST API

服務只要把資料送進 KubeMQ，Target 會依 YAML 設定自動執行 INSERT、SET、HTTP POST…​，省去程式端直接連外部系統的需求。

### 安裝kubemq-targets
- 準備 helm charts
    ```bash
    git clone https://github.com/kubemq-io/charts.git
    cd charts/kubemq-targets
    ```

- 更新 values.yaml 內config 內容對接 redis 和 postgresql
設定的部分請參考[kubenq-targets的github](https://github.com/kubemq-io/kubemq-targets/tree/master?tab=readme-ov-file#target)
    ```yaml
    bindings:
      - name: redis-set
        source:
          kind: kubemq.query
          properties:
            address: "kubemq-cluster:50000"
            channel: "query.redis"
            client_id: "src-redis"
        target:
          kind: cache.redis
          properties:
            url: "redis://redis:6379"

      - name: postgres-exec
        source:
          kind: kubemq.query
          properties:
            address: "kubemq-cluster:50000"
            channel: "query.postgres"
            client_id: "src-pg"
        target:
          kind: stores.postgres
          properties:
            connection: "postgres://app:secret@postgres:5432/sales?sslmode=disable"
    ```
- 安裝 targets
    ```
    helm install -n kubemq kubemq-targets . -f values.yaml
    ```

### 範例
付款成功後，同步寫入 Redis 與 PostgreSQL，配合 KubeMQ架構如下

<figure><img src="../.gitbook/assets/kubemq/targets-architecture.png" alt=""><figcaption></figcaption></figure>


測試流程：
1. payment-service 發布訊息至 **query.postgresql** 和 **query.redis** channel
2. kubemq-targets 根據 bindings.yaml：
    - 同步SET 到 Redis 快取
    - 同步INSERT 到 PostgreSQL

3. 撰寫 payment-service 程式把資料庫指令丟到 targets
    ```python
    import json, uuid, base64, time
    from kubemq.cq import Client, QueryMessage

    client = Client(address="192.168.1.19:50000", client_id="targer‑client")

    pid = f"pmt_{uuid.uuid4().hex[:6]}"
    value_bytes = json.dumps({"id": pid, "amount": 888}).encode()

    # ---------- Redis SET ----------
    redis_request = {
        "metadata": {
            "method": "set",
            "key": f"payment:{pid}"
        },
        "data": base64.b64encode(value_bytes).decode()
    }
    redis_msg = QueryMessage(
        channel="query.redis",
        metadata="",
        body=json.dumps(redis_request).encode(),
        timeout_in_seconds=5
    )
    print("Redis →", client.send_query_request(redis_msg).error or "OK")

    # ---------- Postgres EXEC ----------
    sql = f"INSERT INTO payments(id,amount) VALUES ('{pid}',888);"
    pg_request = {
        "metadata": {
            "method": "exec"
        },
        "data": base64.b64encode(sql.encode()).decode()
    }
    pg_msg = QueryMessage(
        channel="query.postgres",
        metadata="",
        body=json.dumps(pg_request).encode(),
        timeout_in_seconds=5
    )
    print("Postgres →", client.send_query_request(pg_msg).error or "OK")

    time.sleep(1)

    ```
> 要傳什麼格式到queue內，請參考github Readme
Postgresql: https://github.com/kubemq-io/kubemq-targets/tree/master/targets/stores/postgres
Redis: https://github.com/kubemq-io/kubemq-targets/tree/master/targets/cache/redis

4. 執行程式
可以看到建立了 2個channel

<figure><img src="../.gitbook/assets/kubemq/targets-channels-created.png" alt=""><figcaption></figcaption></figure>

- query.postgresql 的channel 的 traffic 看到流量

<figure><img src="../.gitbook/assets/kubemq/targets-postgresql-traffic.png" alt=""><figcaption></figcaption></figure>

- query.redis 的channel 的 traffic 看到流量

<figure><img src="../.gitbook/assets/kubemq/targets-redis-traffic.png" alt=""><figcaption></figcaption></figure>

- 檢查 postgresql 的 資料庫內容

<figure><img src="../.gitbook/assets/kubemq/targets-postgresql-data.png" alt=""><figcaption></figcaption></figure>

- 檢查 redis 的資料內容

<figure><img src="../.gitbook/assets/kubemq/targets-redis-data.png" alt=""><figcaption></figcaption></figure>


所以 Service 無需安裝任何 Redis/PostgreSQL driver，只維護與 KubeMQ 的連線

> Target 只負責「寫」，若要把外部系統的更新事件反推回 KubeMQ，請搭配 Source。


##  KubeMQ Sources

**KubeMQ Sources** 是一組「來源連接器 (source connectors)」，讓你能從外部系統（如：Kafka、MySQL、Redis、MongoDB、PostgreSQL、AWS SQS、GCP Pub/Sub 等）**接收資料**，然後自動轉發到 **KubeMQ broker** 中的 channel。

簡單講，就是「把外部資料拉進來」變成 KubeMQ 的消息。

---

### KubeMQ Sources 架構圖

<img src="https://github.com/kubemq-io/kubemq-sources/raw/master/.github/assets/binding.jpg" width="600"/>

> 這裡的 Targets 指的是 KubeMQ 的模式，不是上一章節的 Targets，例如可以丟到 Queues 或是 Commands 或是 Events 內的 channel
---

###  支援的 source 種類

| 類型          | 範例                             |
| ----------- | ------------------------------ |
| Queue       | Kafka, RabbitMQ, AWS SQS       |
| Database    | MySQL, PostgreSQL, MongoDB     |
| Cache       | Redis                          |
| API/Webhook | REST, Webhook                  |
| Cloud       | GCP Pub/Sub, Azure Service Bus |
| Others      | Cron, File, etc.               |

更多種類請看 GitHub 中的子目錄列表：[kubemq-sources](https://github.com/kubemq-io/kubemq-sources/tree/master/sources)

---

###  簡易應用

使用 KubeMQ Sources 可以解決這些需求：

* 把 Kafka 裡的資料丟進 microservices 中處理。
* 定時從 MySQL 抓資料送給 downstream。
* 讓 Redis 的 key 更新能通知到下游服務。
*  webhook，要轉成 KubeMQ 事件流程。

---

###  設定範例（以 Kafka 為例）

```yaml
sources:
  - name: kafka-orders
    kind: messaging.kafka
    properties:
      brokers: "kafka:9092"
      topics: "orders"
      group: "consumer-group-1"
    target:
      kind: kubemq.queue
      properties:
        address: "kubemq-cluster:50000"
        channel: "orders.new"
        client_id: "source-kafka-orders"
```

這段會把 Kafka topic `orders` 的訊息送進 KubeMQ 的 queue 內 `orders.new` channel。

---

###  Source 與 Target 功能整理

| Connector | 功能說明                   |
| --------- | ---------------------- |
| Source    | 從外部系統「接收資料」→ 傳給 KubeMQ |
| Target    | 從 KubeMQ「接收資料」→ 寫入外部系統 |

---

## KubeMQ Bridges

**KubeMQ Bridges** 是一個**跨叢集訊息橋接器**，能夠將訊息在**多個 KubeMQ** 叢集之間「轉送（Bridge）」、「複製（Replicate）」、「彙整（Aggregate）」或「轉換（Transform）」，讓你在任意環境（K8s、VM、雲端或地端）建立一個全球性的訊息網路。

- 支援任意拓樸（1:1、1\:N、N:1、N\:N）
- 可部署於任意平台（Docker、Kubernetes、裸機）
- 設定檔簡單、可觀察性高（Log、Retry、Rate Limit）
- 各類 Source/Target 型別（queue、events、command、query 等）皆可互通

---

###  簡單的 YAML 範例

以下為一個 **Bridge (1:1)** 範例，把 `events.source` channel 的事件從來源叢集(kubemq-source)送到 目標叢集(kubemq-target) 的`events.target` channel：

```yaml
bindings:
  - name: simple-bridge
    sources:
      kind: source.events
      name: cluster-source
      connections:
        - address: "kubemq-source:50000"
          client_id: "bridge-source"
          channel: "events.source"
    targets:
      kind: target.events
      name: cluster-target
      connections:
        - address: "kubemq-target:50000"
          client_id: "bridge-target"
          channels: "events.target"
```
---

### 四種拓樸功能
###  Binding 核心

KubeMQ Bridges 的基本架構是透過「**Binding**」來建立 **Source → Target** 之間的訊息連線。

每一個 binding 就像一條橋，把訊息從一邊送到另一邊。你可以設定 middleware，例如 log、重試、速率限制等。

<figure>
  <img src="https://raw.githubusercontent.com/kubemq-io/kubemq-bridges/master/.github/assets/concept.jpeg" width="480"/>
  <figcaption>▲ Binding 的核心概念：source ↔ target</figcaption>
</figure>

---

### Bridge：一對一橋接（1:1）

用於同步型應用，例如「一個系統→另一個系統」，雙向或單向同步。

<figure>
  <img src="https://raw.githubusercontent.com/kubemq-io/kubemq-bridges/master/.github/assets/bridge.jpeg" width="400"/>
  <figcaption>▲ Bridge：一對一（1 source → 1 target）</figcaption>
</figure>

---

### Replicate：一對多複製（1\:N）

將來源訊息**同步複製**到多個目標，例如一個主叢集複製到多個地區以便高可用與災難備援。

<figure>
  <img src="https://raw.githubusercontent.com/kubemq-io/kubemq-bridges/master/.github/assets/replicate.jpeg" width="480"/>
  <figcaption>▲ Replicate：一對多（1 source → 多 targets）</figcaption>
</figure>

---

###  Aggregate：多對一聚合（N:1）

適合彙整來自多個地區、服務、分區的訊息進單一目標，例如日誌彙整、分析平台輸入。

<figure>
  <img src="https://raw.githubusercontent.com/kubemq-io/kubemq-bridges/master/.github/assets/aggregate.jpeg" width="420"/>
  <figcaption>▲ Aggregate：多對一（多 sources → 1 target）</figcaption>
</figure>

---

###  Transform：多對多轉送（N\:N）

支援多來源與多目標混搭轉送，適合建立複雜的全域訊息網路，例如各地服務即時同步。

<figure>
  <img src="https://raw.githubusercontent.com/kubemq-io/kubemq-bridges/master/.github/assets/transform.jpeg" width="420"/>
  <figcaption>▲ Transform：多對多（多 sources → 多 targets）</figcaption>
</figure>

---

### Bridge 功能整理

| 模式        | 拓樸   | 用途            |
| --------- | ---- | ------------- |
| Bridge    | 1:1  | 單一來源對單一目標     |
| Replicate | 1\:N | 主→多從，做資料備援或同步 |
| Aggregate | N:1  | 多來源彙整到分析系統    |
| Transform | N\:N | 任意來源對接任意目標    |

---


# 驗證 HA
透過簡易的Pub Sub實驗，驗證 KubeMQ HA 的功能，實驗步驟如下
1. 啟動 Subscriber，接收訊息，將訊息寫入檔案紀錄
2. 啟動 Publisher，每5秒發送訊息至 ha-channel

<figure><img src="../.gitbook/assets/kubemq/ha-subscriber-start.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/kubemq/ha-publisher-start.png" alt=""><figcaption></figcaption></figure>

3. 使用 Dashboard 觀察 ha-channel channel 的流量是否穩定

<figure><img src="../.gitbook/assets/kubemq/ha-dashboard-traffic.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/kubemq/ha-dashboard-traffic-detail.png" alt=""><figcaption></figcaption></figure>

4. 模擬 Broker 當機，確認 Leader

<figure><img src="../.gitbook/assets/kubemq/ha-leader-confirm.png" alt=""><figcaption></figcaption></figure>

5. 強制刪除Leader broker Pod
    ```bash
    kubectl delete pod -n kubemq kubemq-cluster-1
    ```

6. 再次查看Cluster，可以發現pod 馬上重建，Leader 切回 cluster-0 pod

<figure><img src="../.gitbook/assets/kubemq/ha-pod-rebuild-leader.png" alt=""><figcaption></figcaption></figure>

7. 查看 Publisher和Subscriber
- Publisher&#x20;

    <figure><img src="../.gitbook/assets/kubemq/ha-publisher-result.png" alt=""><figcaption></figcaption></figure>

- Subscriber&#x20;

    <figure><img src="../.gitbook/assets/kubemq/ha-subscriber-result.png" alt=""><figcaption></figcaption></figure>

可以看到 KubeMQ 具備高可用（HA）能力以及 Leader 切換流程 ，但預設情況下並**不保證訊息零遺失**，尤其是使用 **Pub/Sub 模式（`events`）** 時。若希望保障訊息「不會遺失」，請使用：

| 模式               | 功能描述                            |
| ---------------- | ------------------------------- |
| **Queue**        | 支援持久化、ack、重試，適用於要求可靠性的情境        |
| **Events Store** | PubSub + 儲存功能，支援 replay，可避免訊息遺失 |

## 結論
KubeMQ 是我認為在 Kubernetes 上部署Message Queue服務是合適的選擇。它的安裝和設定非常簡單，我們只需要透過 Helm 幾行指令，就能快速完成 controller 和 broker 的部署，整體比起 Kafka 或 ActiveMQ 輕巧很多。KubeMQ 採用輕量化設計，不需要像 Kafka 之前的版本那樣依賴 ZooKeeper，這對維運團隊來說是很大的優勢。

另外，它本身就有內建的 Dashboard，可以即時看到 channel 狀況、流量、Client 拓樸圖，甚至能直接在 UI 中觀察訊息的收發，這對於日常管理和問題排查非常方便。

不過生態系和社群規模相較於 Kafka 或 ActiveMQ 仍小很多，在查找資源或範例時，會遇到資料較少或支援較有限的情況。此外，KubeMQ 是商業授權模式，若要在 Kubernetes 上運作，必須使用企業版，社群版只支援單機部署

>以上僅根據文件及試用經驗，若有更多實戰經驗歡迎補充交流！