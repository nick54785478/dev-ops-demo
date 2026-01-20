# RBAC 雲原生 CI/CD 與全棧監控系統

本文件旨在詳述 **rbac-frontend** 與 **rbac-backend** 專案的 Jenkins 流水線（Pipeline）設計與執行流程。

內容涵蓋自動化部署的全生命週期，包括：

* **壹、 Jenkins 核心 Pipeline 流程。**

* **貳、 可觀測性架構：整合 Prometheus 與 Loki 進行數據採集，並透過 Grafana 實現指標監控與日誌聚合，建構全方位運維解決方案。**



---
# 壹、 Jenkins 核心 Pipeline 流程

### 流程: 

* 自原始碼拉取（Checkout）
* 前端構建、靜態代碼掃描
* Docker 映像檔封裝
* Kubernetes 叢集部署。



---

## 一、Pipeline 概述

<strong>
註：在執行 Jenkins CI/CD Pipeline 之前，必須先完成基礎服務 MySQL 、 Jenkins 與 SonarQube 和 Kubernetes 集群相關建置 ( 使用三台虛擬機，並搭配 Docker 進行建置 )。
</strong>
<br><br/>

本 CI/CD 文件同時涵蓋 前端（rbac-frontend） 與 後端（rbac-service） 兩條 Jenkins Pipeline，兩者共用 Kubernetes Jenkins Agent 架構，但在編譯、掃描與產出物上有所差異。

此 Pipeline 採用 Jenkins Declarative Pipeline，並透過 Kubernetes Plugin 動態建立 Agent Pod 來執行各階段任務。



---

### Pipeline 特色

* Kubernetes 動態 Agent（多 Container）
* Angular 前端專案建置
* SonarQube 程式碼品質掃描
* Docker Image 建置與 Harbor 推送
* Kubernetes 自動部署

---

## 二、Kubernetes Agent 架構

Pipeline 啟動時，Jenkins 會動態建立一個 Pod，其結構如下：

| Container | Image                        | 說明                       |
| --------- | ---------------------------- | ------------------------ |
| jnlp      | jenkins/inbound-agent:latest | Jenkins 與 Agent 通訊       |
| node      | node:18-alpine               | Angular 編譯與 SonarQube 掃描 |
| docker    | docker:latest                | Docker build / push      |
| kubectl   | bitnami/kubectl:1.28.4       | Kubernetes 部署            |

### 特殊設定

* `docker` container 掛載 `/var/run/docker.sock`
* `kubectl` container 以 root 身份執行
* Pod 生命週期由 Jenkins Pipeline 控制

---

## 三、Pipeline 參數與環境變數

### 1. 參數（Parameters）

| 名稱     | 說明                 |
| ------ | ------------------ |
| Branch | Git 分支選擇（預設：`dev`） |

> 使用 Git Parameter Plugin 提供分支選擇功能

### 2. 環境變數（Environment）

| 變數                 | 值    |
| ------------------ | ---- |
| DOCKER_API_VERSION | 1.43 |

---

## 四、Pipeline 執行流程

以下分為 **前端 Pipeline** 與 **後端 Pipeline** 兩個部分說明。

## 4.1、前端 Pipeline（rbac-frontend）

### Stage 1：拉取程式碼

**目的**：從 GitLab 拉取前端原始碼。

流程說明：

1. 顯示選擇的 Git 分支
2. 使用 Jenkins Credential 存取 GitLab Repository
3. Checkout `rbac-frontend` 專案程式碼

---

### Stage 2：程式碼編譯（Angular Build）

**執行 Container**：`node`

流程說明：

1. 安裝 Angular CLI
2. 安裝 `@angular/cdk@18`
3. 顯示 `environment.ts` 內容
4. 動態產生 `environment.server.ts`
5. 使用 Production 設定進行 Angular Build
6. 編譯產物輸出至 `dist/rbac-frontend`

編譯產物：

* Angular 靜態檔案（HTML / JS / CSS）

---

### Stage 3：程式碼掃描（SonarQube）

**執行 Container**：`node`

流程說明：

1. 安裝 OpenJDK 11
2. 安裝 Sonar Scanner
3. 執行 SonarQube 掃描
4. 排除以下目錄與檔案：

   * `node_modules`
   * `dist`
   * `*.spec.ts`

產出：

* SonarQube Code Quality 報告

---

### Stage 4：Docker Image 建置與推送

**執行 Container**：`docker`

流程說明：

1. 檢查 `dist` 目錄內容
2. 顯示 Dockerfile 內容
3. 建立 Docker Image：

   ```
   192.168.68.16:80/rbac-frontend/rbac-frontend:${BUILD_NUMBER}
   ```
4. 登入 Harbor Registry
5. 將 Image 推送至 Harbor

產出：

* 已版本化的前端 Docker Image

---

### Stage 5：Kubernetes 部署

**執行 Container**：`kubectl`

流程說明：

1. 載入 kubeconfig
2. 建立或更新 Harbor Image Pull Secret
3. 動態替換 Deployment YAML 中的 Image Tag
4. 執行 `kubectl apply`
5. 等待 Pod 啟動
6. 顯示 Pod 與 Service 狀態

部署資訊：

* Namespace：`rbac`

---

## 4.2、後端 Pipeline（rbac-service）

### Stage 1：拉取程式碼

**目的**：從 GitLab 拉取後端服務原始碼。

流程說明：

1. 透過 Git Parameter 選擇分支（預設 `dev`）
2. 使用 Jenkins Credential 存取 GitLab Repository
3. Checkout `rbac-service` 專案程式碼

---

### Stage 2：程式碼編譯（Maven Build）

**執行 Container**：`maven`

流程說明：

1. 使用 Maven 3.8.5 + OpenJDK 17
2. 執行 `mvn clean package`
3. 產出後端服務 JAR

編譯產物：

* `target/*.jar`

---

### Stage 3：程式碼掃描（SonarQube）

**執行 Container**：`maven`

流程說明：

1. 透過 Maven Plugin 執行 SonarQube 掃描
2. 掃描 Java 編譯後的 class 檔案

Sonar 主要參數：

* Project Key：`rbac-service`
* Java binaries：`target/classes`

產出：

* SonarQube Code Quality 報告（後端）

---

### Stage 4：Docker Image 建置與推送

**執行 Container**：`docker`

流程說明：

1. 建立 Docker Image：

   ```
   192.168.68.16:80/rbac-service/rbac-service:${BUILD_NUMBER}
   ```
2. 登入 Harbor Registry
3. 推送 Image 至 Harbor

產出：

* 已版本化的後端服務 Docker Image

---

### Stage 5：Kubernetes 部署

**執行 Container**：`kubectl`

流程說明：

1. 載入 kubeconfig
2. 建立 / 更新 Harbor Image Pull Secret
3. 動態替換 Deployment YAML 中的 Image Tag
4. 執行 `kubectl apply`
5. 顯示 Pod / Service / Ingress 狀態

部署資訊：

* Namespace：`rbac`

---

## 五、Pipeline 結果處理（Post Actions）

| 狀態      | 行為         |
| ------- | ---------- |
| always  | 顯示「構建流程結束」 |
| success | 顯示「構建成功」   |
| failure | 顯示「構建失敗」   |

---

## 六、整體流程示意

以下分別說明 前端（rbac-frontend） 與 後端（rbac-service） 的完整 CI/CD 執行流程。

前端 Pipeline 流程（rbac-frontend）
```
Jenkins 觸發
↓
Kubernetes Agent Pod 建立
↓
Git Checkout
↓
Angular Build
↓
SonarQube 掃描（Frontend）
↓
Docker Build & Push（Frontend Image）
↓
Kubernetes 部署（Frontend）
↓
Pipeline 結束
```

後端 Pipeline 流程（rbac-service）
```
Jenkins 觸發
↓
Kubernetes Agent Pod 建立
↓
Git Checkout
↓
Maven Build（Java / Spring Boot）
↓
SonarQube 掃描（Backend / Java）
↓
Docker Build & Push（Backend Image）
↓
Kubernetes 部署（Service / Ingress）
↓
Pipeline 結束
```
---

## 七、基礎環境與先置作業（Infrastructure & Pre-requisites）

本系統的 CI/CD 與執行環境由 Kubernetes 叢集 與 Infrastructure 虛擬機 組成，兩者職責明確區分。

本 CI/CD 架構在 Jenkins Pipeline 執行前，需先於 Kubernetes 叢集中完成以下基礎服務建置，否則 Pipeline 將無法正常運作。

### 7.1 Kubernetes 叢集說明

說明：SonarQube 與 MySQL 皆以 Pod 形式部署於 Kubernetes 叢集中。

**Kubernetes Node 規劃**

| IP | 角色	| 說明
| ------ | ------ | ------------------ |
|	192.168.68.39 | Master Node  |	Kubernetes Control Plane / Master
|	192.168.68.159 | Worker Node  |	Kubernetes Worker（承載應用 Pod）

### 7.2 Infrastructure 虛擬機

Infrastructure VM 提供 CI/CD 與共用基礎服務，不直接承載業務 Pod。

| IP | 角色 | 說明
| ------ | ------ | ------------------ |
| 192.168.68.16	| Infrastructure | Jenkins / Harbor / GitLab / NFS Server

**Infrastructure 職責**

 * Jenkins（CI/CD Pipeline）
 * Harbor（Docker Image Registry）
 * GitLab（原始碼管理）
 * NFS Server（提供 MySQL 及 SonarQube PersistentVolume）

### 7.3 MySQL 主從叢集（MySQL Master / Slave）

本系統使用 MySQL 8.0 主從架構，並透過 StatefulSet + NFS PersistentVolume 方式部署。

**架構說明**
>* 架構類型：Master / Slave Replication
>* Namespace：rbac
>* 儲存體：NFS（ReadWriteMany）
>* Workload：StatefulSet

**元件組成**

| 元件 | 說明 |
| ------ | ------ |
| PersistentVolume	| Master / Slave 各自一組 NFS PV |
| PersistentVolumeClaim	| 綁定對應 PV |
| ConfigMap |	MySQL 設定（server-id、binlog、relay-log）|
| Secret |	MySQL Root 密碼 |
| Service	| NodePort 對外服務 |
| StatefulSet |	確保 MySQL Pod 穩定識別 |
| Service | Port 對應 |
| 角色 | Service	NodePort |
| Master | mysql-master-svc	30306
| Slave |	mysql-slave-svc	30308


**關鍵設定說明**

* Master
>* server-id = 1
>* 啟用 log-bin
>* 指定 binlog_do_db

* Slave
>* server-id = 2
>* 啟用 relay-log

**注意：Slave 與 Master 的實際 replication（CHANGE MASTER TO）需於 MySQL 啟動後手動設定或另行自動化腳本完成。**

---

### 7.4 SonarQube 服務

Jenkins Pipeline 中的 程式碼掃描（SonarQube）階段，需事先完成 SonarQube 服務部署。

**SonarQube 角色**
>* Frontend Pipeline：掃描 Angular / TypeScript
>* Backend Pipeline：掃描 Java / Spring Boot

**Jenkins 依賴項目**
| 項目 | 說明 |
| ------ | ------ |
|SonarQube Server	| Jenkins 設定中的 sonarqube Server Name |
|Sonar Token |	由 Jenkins Credential 管理 |

網路存取	Jenkins Agent Pod 必須可存取 SonarQube Endpoint

**注意：若 SonarQube 未就緒，Pipeline 將於「程式碼掃描」階段失敗。**

---

## 貳、監控與可觀測性架構 (Observability Architecture)

本系統將監控視為生產級別的必要組件，透過以下配置實現自動化採集：

* 指標監控 (Prometheus & Grafana)
* 日誌聚合 (Loki)

---

## 一、Observability 概述

本系統建構於 Kubernetes 環境，整合 Prometheus、Grafana 與 Loki (PLG Stack)，旨在提供全方位的可觀測性解決方案。

透過自動化指標採集與日誌聚合，實現對微服務架構的即時監控、故障排查與數據持久化。

---

## 二、Prometheus & Grafana 架構

系統採用 Prometheus Operator 模式，透過自定義資源（CRD）管理監控對象，並結合分散式儲存確保數據安全。

* 指標層 (Metrics)：使用 Prometheus 採集系統、資料庫 (MySQL) 及 Java 應用指標。
* 日誌層 (Logging)：透過 Loki 實現容器日誌的統一收納與檢索。
* 展示層 (Visualization)：Grafana 整合多數據源，提供一站式監控面板。
* 儲存層 (Storage)：全組件透過 NFS Server 進行數據持久化，防止 Pod 重啟導致數據丟失。

---

## 三、參數與環境變數

**3.1 Prometheus 儲存參數 (values-prometheus-nfs.yaml)**


| 參數名稱 | 設定值 | 說明 |
| ------ | ------ | ------ |
|retention | 15d	| 監控數據留存天數 |
|retentionSize | 18GB	| 數據儲存上限 |
|storage | 20Gi	| 申請之 NFS 持久化容量 |
|storageClassName | nfs-static	| 指定之儲存類別 |


**3.2 MySQL Exporter 配置 (mysql-exporter.yaml)**

* DATA_SOURCE_NAME: exporter:exporter_password@tcp(mysql-master-svc.rbac.svc.cluster.local:3306)/
* 採集端口: 9104

**3.3 Java 應用監控環境變數**

* MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,prometheus"
* MANAGEMENT_ENDPOINTS_WEB_BASE_PATH: "/actuator"

---

## 四、執行流程

**4.1 儲存準備**

* 在 NFS Server (192.168.68.16) 建立實體目錄 /data/nfs/prometheus 與 /data/nfs/loki。
* 建立 PersistentVolume (PV) 與標籤 pv: prometheus-nfs，確保精準綁定。

**4.2 服務部署**

* 透過 Helm 安裝 kube-prometheus-stack 並載入自定義 values。
* 部署 mysqld-exporter 及其對應的 ConfigMap 與 Secret。

**4.3 自動發現 (Service Discovery)**

* 部署 ServiceMonitor 資源。
* Prometheus Operator 根據 release: kube-prometheus 標籤識別 ServiceMonitor，自動更新採集清單。

---

## 五、結果處理

**5.1 數據驗證**

* Prometheus Targets：確認 rbac-service 與 mysql-exporter 狀態為 UP。
* NFS 寫入確認：檢查 /data/nfs/prometheus/prometheus-db 內是否產生 wal 與數據塊。

**5.2 Grafana 面板展示**

* MySQL Overview：呈現 QPS、連接數及 Buffer Pool 命中率。
* Loki Logs：在 Grafana 中透過 LogQL 檢索應用即時日誌。

---

## 六、整體示意

系統監控流向如下：

<pre>
[應用服務/資料庫] ➔ [Exporters / Actuator] ➔ [K8s Service] 
                                            ↓
[Grafana 面板] ↞ [Prometheus/Loki 存儲] ↞ [ServiceMonitor 自動發現]
      ↑                  ↓
[NFS 數據持久化] ←───────┘
</pre>

---
## 七、 基礎環境與先置作業（Infrastructure & Pre-requisites）

**7.1 儲存節點 (NFS Server)**

* IP: 192.168.68.16
* 職責: 提供 Prometheus、Loki、MySQL 的數據落地。

**7.2 監控資源權限**

* Namespace: rbac
* RBAC: 確保 Prometheus 有權限跨 Namespace 存取 rbac 內的 Service 端點。

**7.3 標籤匹配規則 (Label Selection)**

* ServiceMonitor Label: release: kube-prometheus
* Service Selector: app: rbac-service 或 app: mysql-exporter
