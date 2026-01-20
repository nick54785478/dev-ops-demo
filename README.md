# 我的 RBAC 雲原生 CI/CD 與監控系統架構

  本文件詳述 **rbac-frontend** 與 **rbac-backend** 專案的自動化流程與監控體系。
  
  涵蓋了從環境初始化、**Jenkins Pipeline** 設計、**Kubernetes** 部署，到基於 **PLG Stack (Prometheus, Loki, Grafana)** 的可觀測性方案。

* rbac-service 程式碼: https://github.com/nick54785478/rbac-service
* rbac-frontend 程式碼: https://github.com/nick54785478/rbac-frontend

---

## 前置作業 (System Pre-requisites)

在啟動 CI/CD 流程前，必須準備好基礎設施與相關服務：

**1. 硬體資源與作業系統**

* 節點規劃：準備三台虛擬機 (VM)，均安裝 CentOS 作業系統。
* 網路配置：

| 主機名稱 | IP 位址                       | 角色與職責                   |
| --------- | ---------------------------- | ------------------------ |
| Master     | 192.168.68.39 | Kubernetes Control Plane (管理節點)      |
| Node      | 192.168.68.159  | Kubernetes Worker (應用執行節點) |
| Infra    | 192.168.68.16           | 基礎設施 (Harbor, GitLab, Jenkins, NFS)  |

**2. 核心組件安裝**

* **容器引擎 :** 三台主機均需安裝 **Docker**。
* **K8s 叢集 :** 以 **Docker** 為基礎環境，建立 Kubernetes 叢集。
* **基礎服務 (於 Infra 節點) :**
  * GitLab：原始碼管理中心(**需將程式碼放置於該 GitLab 容器**)。
  * Harbor：私有 Docker Image 倉庫。
  * Jenkins：自動化執行引擎 (CI/CD)。
  * NFS Server：提供 K8s 持久化儲存 (PV/PVC)。
 
**3. K8s 內部服務初始化**

* MySQL 8.0 主從叢集：透過 StatefulSet 部署。
* SonarQube：程式碼品質掃描平台。
* 監控套件：安裝 kube-prometheus-stack (含 Prometheus Operator)。

---

## Jenkins 核心 Pipeline 流程

本架構採用 Jenkins Declarative Pipeline，並透過 Kubernetes Plugin 動態建立 Agent Pod，實現彈性的資源調度。

**1. Kubernetes Agent 結構**
當 Pipeline 啟動時，會動態生成包含以下 Container 的 Pod：

| Container | Image                        | 說明                  |
| --------- | ---------------------------- | ------------------------ |
| jnlp     | jenkins/inbound-agent | Jenkins Master/Agent 通訊      |
| node      | node:18-alpine | Angular 編譯與 SonarQube 掃描 |
| maven    | maven:3.8.5-openjdk-17       | Java/Spring Boot 編譯與掃描  |
| docker    | 192.168.68.16           | Docker Image 建置與 Push  |
| kubectl    | 192.168.68.16           | Kubernetes 部署作業  |


**2. Pipeline 執行階段**

**前端流程 (rbac-frontend) :**

1. Checkout: 從 GitLab 拉取代碼。

2. Angular Build: 安裝依賴並進行 Production 編譯。

3. SonarQube Scan: 執行靜態程式碼品質分析。
 
4. Docker Build & Push: 打包成 Image 並推送至 Harbor (192.168.68.16:80)。
 
5. K8s Deploy: 替換 YAML 標籤並更新 rbac Namespace 下的部署。

**後端流程 (rbac-service) :**

1. Checkout: 從 GitLab 拉取代碼。

2. Maven Build: 執行 mvn clean package 產出 JAR 檔。

3. SonarQube Scan: 針對 Java Bytecode 進行深層掃描。

4. Docker Build & Push: 建置後端服務容器鏡像。

5. K8s Deploy: 更新 Deployment 與 Ingress 設定。

---

## 監控與可觀測性架構 (Observability)

整合 **Prometheus**、**Loki**、**Grafana (PLG Stack)**，確保系統運行的透明度。

**1. 架構組件 :**
* 指標 (Metrics)：Prometheus 透過 ServiceMonitor 自動發現應用與 MySQL 指標。
* 日誌 (Logs)：Loki 收集所有容器標準輸出 (stdout/stderr)。
* 展示 (Visualization)：Grafana 整合數據源，提供即時儀表板。
* 持久化 (Storage)：所有監控數據均落地於 NFS Server (192.168.68.16)。

**2. 監控核心設定 :**
* MySQL 監控：使用 mysqld-exporter 採集 QPS、連接數與 Buffer Pool。
* 應用監控：Java 服務開啟 /actuator/prometheus 端點。
* 數據留存：Prometheus 設定 15 天留存期，上限 18GB。

## 系統架構流向示意
 
開發者 Push Code $\rightarrow$ Jenkins 觸發 $\rightarrow$ 動態 Pod 建置/掃描/打包 $\rightarrow$ 推送至 Harbor $\rightarrow$ 部署至 K8s $\rightarrow$ Prometheus/Loki 監控 $\rightarrow$ NFS 存儲

**快速維護指令**
* 查看部署狀態：kubectl get pods -n rbac
* 檢查監控目標：透過 Prometheus UI 確認 Targets 是否為 UP。
* 查看日誌：於 Grafana Explore 頁面選取 Loki 資料源。

