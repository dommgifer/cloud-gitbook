# Measure Your Maven Build

{% hint style="info" %}
本文轉寫時間為 2024年03月22日，內容可能會有變動，僅記錄
{% endhint %}

參考文章: https://maarten.mulders.it/2024/03/measure-your-maven-build/

此文章主要在說明

* 緩慢的build問題：緩慢的Maven build 是令人煩躁，並且會降低生產力和工作滿意度。
* 效率投資回報：引用Hans Dockter的報告，強調投資於更高效的建構系統將帶來顯著的投資回報，並且是最成功軟體團隊的特點。
* 工具比較：文章評估了三種工具: Maven Profiler、Maven BuildTime Profiler和Maven OpenTelemetry擴展，並根據易用性、視覺報告和報告格式多樣性進行了比較。



以下三套工具來找出Maven build 的執行時間

* 前置作業為 maven 3.9 以上版本
* 所有build 所需的 plugins 是最新的

## Maven Profiler
https://github.com/jcgay/maven-profiler

Maven 的時間執行記錄器，記錄每個Mojo在您的構建生命周期中所花費的時間。

1. 在 `${maven.multiModuleProjectDirectory}/.mvn/` 建立 extensions.xml，以便 maven 執行擴充功能，以下是 extensions.xml的內容
    ```
    <?xml version="1.0" encoding="UTF-8" ?>
    <extensions xmlns="http://maven.apache.org/EXTENSIONS/1.0.0"
                xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                xsi:schemaLocation="http://maven.apache.org/EXTENSIONS/1.0.0 https://maven.apache.org/xsd/core-extensions-1.1.0.xsd">
        <extension>
            <groupId>fr.jcgay.maven</groupId>
            <artifactId>maven-profiler</artifactId>
            <version>3.2</version>
        </extension>
    </extensions>
    ```

2. 執行 mvn install ，並帶入 -Dprofile 參數
    ```
    $ mvn install  -Dprofile
    ............
    [INFO] Profiling mvn execution..
    [INFO]  T E S T S
    [INFO] -------------------------------------------------------
    [INFO] Running com.mycompany.app.AppTest
    [INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.125 s -- in com.mycompany.app.AppTest
    [INFO] 
    [INFO] Results:
    [INFO] 
    [INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
    [INFO] 
    [INFO] 
    [INFO] --- jar:3.3.0:jar (default-jar) @ my-app ---
    [INFO] 
    [INFO] --- install:3.1.1:install (default-install) @ my-app ---
    [INFO] Installing /app/simple-java-maven-app/pom.xml to /root/.m2/repository/com/mycompany/app/my-app/1.0-SNAPSHOT/my-app-1.0-SNAPSHOT.pom
    [INFO] Installing /app/simple-java-maven-app/target/my-app-1.0-SNAPSHOT.jar to /root/.m2/repository/com/mycompany/app/my-app/1.0-SNAPSHOT/my-app-1.0-SNAPSHOT.jar
    [INFO] ------------------------------------------------------------------------
    [INFO] BUILD SUCCESS
    [INFO] ------------------------------------------------------------------------
    [INFO] Total time:  3.271 s
    [INFO] Finished at: 2024-03-22T06:43:47Z
    [INFO] ------------------------------------------------------------------------
    [INFO] HTML profiling report has been saved in: /app/simple-java-maven-app/.profiler/profiler-report-2024-03-22-06-43-47.html
    ```
    <figure><img src="../.gitbook/assets/measure-maven-build/maven-profiler-console-output.png" alt=""><figcaption></figcaption></figure>

    完成後會看到一個輸出
    `[INFO] HTML profiling report has been saved in: /app/simple-java-maven-app/.profiler/profiler-report-2024-03-22-06-43-47.html` 
    
    這就是執行 mvn install 執行所花的時間報告位置
    
    <figure><img src="../.gitbook/assets/measure-maven-build/maven-profiler-html-report.png" alt=""><figcaption></figcaption></figure>

   
 
 ## Maven BuildTime Profiler
 https://github.com/khmarbaise/maven-buildtime-profiler
 
 這是 EventSpy 實做，它收集所有階段和 mojo 執行的所有訊息，在構建結束時進行總結輸出。
 
 1. 依樣可以透過 mvn extenstions使用， extensions.xml的內容如下
 
     ```
     <?xml version="1.0" encoding="UTF-8" ?>
    <extensions xmlns="http://maven.apache.org/EXTENSIONS/1.0.0"
                xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                xsi:schemaLocation="http://maven.apache.org/EXTENSIONS/1.0.0 https://maven.apache.org/xsd/core-extensions-1.1.0.xsd">
        <extension>
            <groupId>com.soebes.maven.extensions</groupId>
            <artifactId>maven-buildtime-profiler</artifactId>
            <version>0.2.0</version>
        </extension>
    </extensions>
     ```
2. 執行 mvn install ，並帶入 -Dprofile 參數，完成後，可以在 console 看到結果，沒有額外的輸出報告

    <figure><img src="../.gitbook/assets/measure-maven-build/buildtime-profiler-result-1.png" alt=""><figcaption></figcaption></figure>

    <figure><img src="../.gitbook/assets/measure-maven-build/buildtime-profiler-result-2.png" alt=""><figcaption></figcaption></figure>


## Maven OpenTelemetry extension

https://github.com/open-telemetry/opentelemetry-java-contrib/tree/main/maven-extension

將 Maven build 過程當作是分散式的方式觀察和追蹤

1. 由於使用到 OpenTelemetry，需要先建立一個服務蒐集 trace 結果，這裡使用 jaeger，透過docker啟動一個all in one 的架構

    ```
    $ docker run --rm --name jaeger \
      -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 \
      -p 6831:6831/udp \
      -p 6832:6832/udp \
      -p 5778:5778 \
      -p 16686:16686 \
      -p 4317:4317 \
      -p 4318:4318 \
      -p 14250:14250 \
      -p 14268:14268 \
      -p 14269:14269 \
      -p 9411:9411 \
      jaegertracing/all-in-one:1.54
    ```

2. 一樣在 extensions.xml 寫入以下內容
    ```
    <?xml version="1.0" encoding="UTF-8" ?>
    <extensions xmlns="http://maven.apache.org/EXTENSIONS/1.0.0"
                xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                xsi:schemaLocation="http://maven.apache.org/EXTENSIONS/1.0.0 https://maven.apache.org/xsd/core-extensions-1.1.0.xsd">
        <extension>
            <groupId>io.opentelemetry.contrib</groupId>
            <artifactId>opentelemetry-maven-extension</artifactId>
            <version>1.33.0-alpha</version>
        </extension>
    </extensions>
    ```

3. 執行 mvn install 並帶入 opentelemetry 相關參數
    * otel.traces.exporter: 輸出的協定， 目前只有none 跟 otlp
    * otel.exporter.otlp.endpoint: otlp 接收的位置，jaeger預設的otlp接受port號是4317
```
$ mvn install -Dotel.traces.exporter=otlp -Dotel.exporter.otlp.endpoint=http://172.17.0.3:4317
```

4. 完成build 後，到 jaeger 的頁面查看(16686 port)，可以完整看到所有build的時間跟相依性，還有每一項的span的屬性

<figure><img src="../.gitbook/assets/measure-maven-build/opentelemetry-jaeger-traces.png" alt=""><figcaption></figcaption></figure>


## 工具比較

| Tool                          | 方便度     | 報告豐富度 | 報告格式                    |
| ----------------------------- | ---------- | ---------- | --------------------------- |
| Maven Profiler                | ⭐⭐⭐⭐⭐ | ⭐         | ⭐⭐⭐⭐ (JSON)             |
| Maven BuildTime Profiler      | ⭐⭐⭐⭐⭐ | ⭐         | ⭐ (none)                   |
| Maven OpenTelemetry extension | ⭐⭐       | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ (JSON, support otlp) |


## 結論
**如果只是尋求快速了解 Maven Build 執行時間，Maven Profiler 是最簡單的選擇**，簡單的設定和輸出html 報告。**如果想要更進一步的了解或是在 build 的時候使用 multi thread ，可以選擇 Maven OpenTelemetry extension** 提供了詳細的視覺化的 build 過程，有助於發現瓶頸。