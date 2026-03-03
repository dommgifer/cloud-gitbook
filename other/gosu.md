# gosu 管理 Docker 非 root 執行的方式

{% hint style="info" %}
本文轉寫時間為 2025年04月11日，內容可能會有變動，僅記錄
{% endhint %}

##  什麼是 `gosu`？

- `gosu` 是一個輕量級工具，用來 **安全切換使用者身分執行應用程式**。
- 與 `su` / `sudo` 不同，它非常適合在 Docker 容器中使用。
- **目的：** 以非 root 執行程式、避免訊號與 TTY 問題、保持容器結構正確。

---

##  為什麼不要用 `su` 或 `sudo`？

| 問題 | 說明 |
|------|------|
| 訊號無法轉發 | `SIGINT`（Ctrl+C）、`SIGTERM`（docker stop） 會被 `su` 攔截，無法傳給子程式 |
| TTY 依賴性高 | `sudo` 在非互動終端（如 CI/CD）下可能無法執行 |
| PID 結構錯誤 | 真正執行的應用不是 PID 1，訊號與容器生命週期難以管理 |

---

##  `gosu` 的優點

- 切換 user 後直接 `exec` 應用程式 → 應用成為 PID 1
- 所有訊號（如 Ctrl+C）都會正確傳遞給應用程式
- 無需額外設定互動終端機
- 完美配合 Docker 容器的訊號機制與資安需求

---

##  訊號與 TTY 快速理解

| 概念     | 說明 |
|----------|------|
| TTY      | 使用者輸入/輸出的終端介面（例：Ctrl+C） |
| Signal   | 作業系統發給程式的控制訊號（SIGINT, SIGTERM） |
| 前景進程 | 控制 TTY 的進程，能接收使用者指令 |
| PID 1    | 容器的主進程，Docker 傳送訊號的對象 |

> 如果應用不是 PID 1、不是前景進程，就不會收到訊號！

---

##  測試範例

###  目錄結構

```
demo-gosu/
├── Dockerfile
├── entrypoint.sh
└── print_signal.py
```

---

###  Dockerfile

```Dockerfile
FROM debian:bullseye

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    python3 curl gosu procps && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

RUN useradd -m agent

COPY entrypoint.sh /entrypoint.sh
COPY print_signal.py /print_signal.py

RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

---

###  print_signal.py

```python
import signal, time, os

def handle(sig, frame):
    print(f"\n[PID {os.getpid()}]  Received signal: {sig}, exiting.")
    exit(0)

signal.signal(signal.SIGINT, handle)
signal.signal(signal.SIGTERM, handle)

print(f"[PID {os.getpid()}]  Running. Waiting for Ctrl+C or SIGTERM...")

counter = 0
while True:
    time.sleep(1)
    counter += 1
    print(f"[PID {os.getpid()}]  Alive for {counter} seconds...")
```

---

###  entrypoint.sh

```bash
#!/bin/bash
set -e

echo "Using test mode: $TEST_MODE"
echo "Container PID 1 is: $$"

whoami
ps -o pid,ppid,pgid,sid,comm -e | grep -E "(PID|agent|entrypoint|bash|python)"

case "$TEST_MODE" in
  "su")
    echo "[su mode] Launching as: su agent -c 'python3 /print_signal.py'"
    su agent -c "python3 /print_signal.py"
    ;;
  "gosu")
    echo "[gosu mode] Launching as: gosu agent python3 /print_signal.py"
    exec gosu agent python3 /print_signal.py
    ;;
  *)
    echo "Unknown TEST_MODE. Use -e TEST_MODE=su or gosu"
    ;;
esac
```

---

##  測試指令

### 建置映像：

```bash
docker build -t demo-gosu ./demo-gosu
```

---

### 測試 su 模式（訊號會失敗）

```bash
docker run -it --rm -e TEST_MODE=su demo-gosu
```

輸出：
```
[PID 12]  Alive for 5 seconds...
^C
Session terminated, killing shell... ...killed.
```

---

### 測試 gosu 模式（訊號會成功）

```bash
docker run -it --rm -e TEST_MODE=gosu demo-gosu
```

輸出：
```
[PID 1]  Alive for 3 seconds...
^C
[PID 1]  Received signal: 2, exiting.
```

---

##  總結對比表

| 特性                  | `su` / `sudo` | `gosu` |
|-----------------------|---------------|--------|
| 切換使用者            | v            | v     |
| 訊號正常轉發          | x            | v     |
| 應用為容器 PID 1      | x            | v     |
| 保留進程殘留（如 su） | v            | x    |
| 容器啟停一致性高      | x            | vvv |

---

## 結論
在容器中使用非 root 使用者，建議一律使用 `gosu` 搭配 exec，以確保應用正確處理訊號、保持最小權限原則，並避開 su/sudo 的 TTY 陷阱。
