# 地端模型使用 MCP

{% hint style="info" %}
本文轉寫時間為 2025年07月09日，內容可能會有變動，僅記錄
{% endhint %}

## 測試工具

MCP Server:

- Kubernetes-mcp-server
- Grafana-mcp-server

Ollama 使用 Model:

- qwen3:8b
- llama3.2:3b

MCP Client:

- Github Copilot
- Open-WebUI

## 啟動 MCP Server
>
> [!Note] MCP Server 的兩種主要傳輸模式如下：
>
>### 1. **STDIO 傳輸模式**
>
>- 透過標準輸入（stdin）與標準輸出（stdout）進行 JSON-RPC 2.0 訊息交換。
>- **適合場合**：
> - 本機整合／CLI 工具
> - Shell 腳本或 Docker 容器內部使用
> - 無需網路，單一用戶執行環境
>
>### 2. **HTTP-Based 傳輸模式**（遠端模式）
>
>#### HTTP + SSE（Server-Sent Events）
>
> - 客戶端用 POST 發送請求；使用 GET + `Accept: text/event‑stream` 開一條 SSE 單向通道接收伺服器推送。
>
>#### Streamable HTTP（單端點雙向）
>
> - 透過單一 `/mcp` HTTP 端點發送請求與串流回應。
> - 2025 年引入，預期成為遠端 MCP 標準傳輸方式

**本次測試使用 HTTP + SSE 傳輸模式**

### Kubernetes-MCP-Server + Grafana-MCP-Server

透過 docker-compose 啟動

```yaml
services:
  k8s-mcp-server:
    image: quay.io/manusa/kubernetes_mcp_server:v0.0.43
    container_name: k8s-mcp-server
    ports:
      - "8080:8080"
    volumes:
      - ~/.kube/config:/root/.kube/config:ro
    restart: unless-stopped

  grafana-mcp-server:
    image: mcp/grafana:latest
    container_name: grafana-mcp-server
    ports:
      - "8081:8000"
    environment:
      GRAFANA_URL: http://grafana.192.168.1.21.nip.io
      GRAFANA_API_KEY: "xxxxxxxxxxxxxxxxx"

```

- k8s-mcp-server:
  - 這裡使用 manusa 的 [k8s-mcp-server](https://github.com/manusa/kubernetes-mcp-server)，有兩種認證方式
        1. Token 認證（k8s內）
            當您的應用程式在 Kubernetes 集群內運行時，無需額外配置，系統會自動使用 Service Account token。
        2. Kubeconfig 認證
            可以透過 --kubeconfig 參數指定配置檔案路徑，或是使用預設路徑

    這裡使用 Kubeconfig 認證，但是僅測試用，請不要用 admin 的 kubeconfig，有很大的機率用嘴巴破壞 k8s ，建議先從只給 View 權限開始測

- grafana-mcp-server:
  - 使用官方的 [mcp-server](https://github.com/grafana/mcp-grafana)
  - 須提供以下環境變數
    - GRAFANA_URL
    - GRAFANA_API_KEY: 可以從 Grafana 介面產生 service account 並建立 key

啟動 container&#x20;

<figure><img src="../.gitbook/assets/local-llm-mcp/containers-started.png" alt=""><figcaption></figcaption></figure>

## 啟動 mcpo (MCP-to-OpenAPI Proxy)

- MCP 是一個讓 LLM 客戶端和外部工具/資料源串接的通用協定。
- [mcpo](https://github.com/open-webui/mcpo) 則是讓這些 MCP 工具能 像普通 REST API 一樣被使用的橋樑，提供認證、安全、可視化文件等功能，使開發與整合都更加順暢。
- 因為 Client 端預計使用 Open‑WebUI 而 Open‑WebUI 的後端模組（backend）預期透過 HTTP/OpenAPI 協定與工具互動，而不是透過 stdin/stdout 這種 MCP 原生方式 ，而 mcpo 正好充當橋樑，把 MCP 工具 wrap 成一個標準 HTTP+OpenAPI 伺服器，因此可以無縫整合到 Open‑WebUI 中 。

設定 mcpo config.json 串接 mcp server

```json
{
  "mcpServers": {
    "kubernetes-mcp-server": {
      "type": "sse",
      "url": "http://k8s-mcp-server:8080/sse"
    },
    "grafana-mcp-server": {
      "type": "sse",
      "url": "http://grafana-mcp-server:8000/sse"
    }
  }
}
```

透過 contaienr 啟動 mcpo，加入 mcpo 到剛剛的 dokcer-compose.yaml

```yaml
services:
  k8s-mcp-server:
    image: quay.io/manusa/kubernetes_mcp_server:v0.0.43
    container_name: k8s-mcp-server
    ports:
      - "8080:8080"
    volumes:
      - ~/.kube/config:/root/.kube/config:ro
    restart: unless-stopped

  grafana-mcp-server:
    image: mcp/grafana:latest
    container_name: grafana-mcp-server
    ports:
      - "8081:8000"
    environment:
      GRAFANA_URL: http://grafana.192.168.1.21.nip.io
      GRAFANA_API_KEY: "xxxxxxxx"
  
  mcpo:
    image: ghcr.io/open-webui/mcpo:main
    container_name: mcpo
    depends_on:
      - k8s-mcp-server
      - grafana-mcp-server
    volumes:
      - ./config.json:/config.json:ro
    ports:
      - "8000:8000"
    command: --config /config.json
    restart: unless-stopped
```

重新啟動 compose&#x20;

<figure><img src="../.gitbook/assets/local-llm-mcp/mcpo-compose-started.png" alt=""><figcaption></figcaption></figure>

### 查看MCP Tool API

輸入 mcpo 的網址加上 MCP Tool 的 path，可以看到 Swagger 頁面
例如: <http://192.168.1.19:8000/kubernetes-mcp-server/docs>

<figure><img src="../.gitbook/assets/local-llm-mcp/mcpo-swagger-k8s.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/local-llm-mcp/mcpo-swagger-grafana.png" alt=""><figcaption></figcaption></figure>

## 建立 Ollama

直接至ollama 官網下載安裝包安裝

## Github Copliot 使用 MCP

透過 vscode 安裝 Github Copliot 套件，此方案是直接連線到 MCP Server 的 SSE Endopint

- 設定 mcp

<figure><img src="../.gitbook/assets/local-llm-mcp/copilot-mcp-settings.png" alt=""><figcaption></figcaption></figure>
- 設定 ollama 位置
<figure><img src="../.gitbook/assets/local-llm-mcp/copilot-ollama-settings.png" alt=""><figcaption></figcaption></figure>

- 到聊天界面設定模型來源為 Ollama，並選取要使用 Ollama 上哪個模型

<figure><img src="../.gitbook/assets/local-llm-mcp/copilot-select-model.png" alt=""><figcaption></figcaption></figure>

- 點選右下角的 Tool 查看可用的 MCP Server，聊天模式選 Agent

<figure><img src="../.gitbook/assets/local-llm-mcp/copilot-tool-agent-mode.png" alt=""><figcaption></figcaption></figure>

### 測試MCP

地端模型:

- llama3.2:3b 無法理解語意並使用MCP Tool
    <figure><img src="../.gitbook/assets/local-llm-mcp/llama3-mcp-fail.png" alt=""><figcaption></figcaption></figure>
- qwen2.5:7b 可以了解語意並使用MCP Tool，但一直處於呼叫工具的迴圈無法結束
    <figure><img src="../.gitbook/assets/local-llm-mcp/qwen25-mcp-loop-1.png" alt=""><figcaption></figcaption></figure>
    <figure><img src="../.gitbook/assets/local-llm-mcp/qwen25-mcp-loop-2.png" alt=""><figcaption></figcaption></figure>

GPT-4.1:
    <figure><img src="../.gitbook/assets/local-llm-mcp/gpt41-mcp-result.png" alt=""><figcaption></figcaption></figure>

## Open-WebUI 使用 MCP

可直接透過container 啟動，這裡不贅述

- 到管理員控制設定串接 ollama

<figure><img src="../.gitbook/assets/local-llm-mcp/openwebui-ollama-settings.png" alt=""><figcaption></figcaption></figure>
- 設定 mcpo
<figure><img src="../.gitbook/assets/local-llm-mcp/openwebui-mcpo-settings.png" alt=""><figcaption></figcaption></figure>

### 測試 MCP

Open-WebUI 可以針對同一個user prompt 選定不同的模型回答，方便比較
以下測試都會跟 GPT-4o 做比較

Kubernetes-mcp-server:

1. 請LLM 協助整理 k8s sock-shop ns 內 pod 的資源使用量 並用表格呈現
    - llama3.2 vs GPT-4o:

      - llama3.2: 理解成整理名稱為k8s-sock-shop的 namespace(理解錯誤)，工具使用正確
      - GPT-4o: 理解正確，工具使用正確
    <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test1-llama-gpt.png" alt=""><figcaption></figcaption></figure>

2. 重新敘述更完整需求，整理 k8s 內 sock-shop ns 內 pod 的資源使用量 並用表格呈現
    - llama3.2 vs GPT-4o:

      - llama3.2: 理解正確，工具使用正確
      - GPT-4o: 理解正確，工具使用正確
      <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test2-llama-gpt.png" alt=""><figcaption></figcaption></figure>

3. 整理 sock-shop namespace 内的 pod image tag 並整理成表格
    - llama3.2 vs GPT-4o:

      - llama3.2: 理解正確，但沒有使用工具
      - GPT-4o: 理解正確，工具使用正確
      <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test3-image-tag-1.png" alt=""><figcaption></figcaption></figure>
      <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test3-image-tag-2.png" alt=""><figcaption></figcaption></figure>

4. 整理 sock-shop namespace 內的 deploy的image tag 並整理成表格
   這裡多測試一項內容，如果只提供一個MCP Tool，地端模型可以正常使用 MCP Tool
    - qwen3:8b vs GPT-4o: 只提供 Kubetnetes-mcp-server
      - qwen3:8b: 理解正確，工具使用正確
      - GPT-4o: 理解正確，工具使用正確
      <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test4-single-tool-qwen-1.png" alt=""><figcaption></figcaption></figure>
      <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test4-single-tool-qwen-2.png" alt=""><figcaption></figcaption></figure>
    - qwen3:8b vs GPT-4o: 提供 Kubetnetes-mcp-server和 Grafana-mcp-server
      - qwen3:8b: 理解正確，沒有使用工具
      - GPT-4o: 理解正確，工具使用正確
      <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test4-multi-tool-qwen-1.png" alt=""><figcaption></figcaption></figure>
      <figure><img src="../.gitbook/assets/local-llm-mcp/k8s-test4-multi-tool-qwen-2.png" alt=""><figcaption></figcaption></figure>

Grafana-mcp-server:

1. 請LLM 幫我建立關於 Java spring boot 監控 Latency 折線圖的 Grafana dashboard
    這個 container 的metrics 是由以下套件生成
    • spring-boot-starter-actuator
    • micrometer-core
    • micrometer-registry-prometheus
    dashboard name: Spring Boot Latency Monitoring
    Metrics 類型：99 百分位延遲值

    - llama3.2: 理解正確，但沒有使用工具
    - GPT-4o: 理解正確，工具使用正確，但沒有回覆結果，實際上有完成任務，
    <figure><img src="../.gitbook/assets/local-llm-mcp/grafana-test1-create-dashboard-1.png" alt=""><figcaption></figcaption></figure>
    <figure><img src="../.gitbook/assets/local-llm-mcp/grafana-test1-create-dashboard-2.png" alt=""><figcaption></figcaption></figure>
    GPT-4o 建立的 Dashboard
    <figure><img src="../.gitbook/assets/local-llm-mcp/grafana-test1-dashboard-result-1.png" alt=""><figcaption></figcaption></figure>
    <figure><img src="../.gitbook/assets/local-llm-mcp/grafana-test1-dashboard-result-2.png" alt=""><figcaption></figcaption></figure>

## 總結

整體來說，地端模型在使用 MCP Tool 時還不穩定，有時候問同樣的問題會用工具，有時候又不會。如果只提供一個 MCP Tool，地端模型大多能正確使用；但如果同時給多個 MCP Tool，它就會搞不清楚該用哪個。這種行為滿不一致的，不知道是不是模型的參數太小，像 llama3.2:3b 就常常理解錯或無法正確使用工具，也要注意地端模型是否支援 function calling。相對來說 GPT-4o 就穩定很多，可以完整搭配 MCP Tool 使用
