# RBAC Jenkins Pipeline README

本文件說明 **rbac-frontend** 及 **rbac-backend** 專案的 Jenkins CI/CD Pipeline 設計與執行流程，涵蓋程式碼拉取、前端編譯、程式碼掃描、Docker 映像建置以及 Kubernetes 部署。

---

## 一、Pipeline 概述

本 CI/CD 文件同時涵蓋 **前端（rbac-frontend）** 與 **後端（rbac-service）** 兩條 Jenkins Pipeline，兩者共用 Kubernetes Jenkins Agent 架構，但在編譯、掃描與產出物上有所差異。

此 Pipeline 採用 **Jenkins Declarative Pipeline**，並透過 **Kubernetes Plugin** 動態建立 Agent Pod 來執行各階段任務。

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
