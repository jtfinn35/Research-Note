# OAI 5G SA (Standalone) Deployment and Testing Manual

![Version](https://img.shields.io/badge/version-v1.0-blue.svg)
![Status](https://img.shields.io/badge/status-verified-brightgreen.svg)
![OS](https://img.shields.io/badge/OS-Ubuntu_22.04-orange.svg)

> **Author**: 黃仁廷 (JTFinn)
> **Date Created**: 2026-05-03

> ⚠️ **Note**
> All tests in this manual are based on the OAI `develop` branch. Since open-source projects are updated frequently, cloning the repository at a different time might result in discrepancies in core network configuration formats (e.g., YAML) or compilation parameters. Before executing this SOP, please ensure your hardware has at least 24GB of available RAM.

### Table of Contents
1. [Executive Summary](#1-executive-summary)
2. [Architecture and Topology](#2-architecture-and-topology)
3. [Prerequisites](#3-prerequisites)
4. [Step-by-Step Guide](#4-step-by-step-guide)
5. [Configuration](#5-configuration)
6. [Verification](#6-verification)
7. [Troubleshooting](#7-troubleshooting)
8. [References](#8-references)

---

### 1. Executive Summary
This document aims to provide a standardized installation and testing procedure for the OpenAirInterface (OAI) 5G Standalone (SA) mode. It covers the deployment of the OAI Core Network (CN5G) and Radio Access Network (RAN) within a WSL2 environment, utilizing the RF Simulator to achieve an End-to-End (E2E) simulated connection between the Base Station (gNB) and the User Equipment (UE). This establishes a foundational testing platform for the subsequent implementation of PRACH attack and mitigation algorithms.

### 2. Architecture and Topology
This environment utilizes a pure software simulation consisting of the following three main nodes. All connections are established internally via Localhost (127.0.0.1):

*   **OAI-CN5G (5G Core Network)**: Deployed via Docker Compose, comprising essential Network Functions (NFs) such as AMF, SMF, UPF, and NRF.
*   **OAI gNB (Base Station)**: Compiled from the OAI RAN source code. It runs the `nr-softmodem` executable and utilizes the RF Simulator to replace physical USRP hardware interfaces.
*   **OAI nrUE (User Equipment)**: Compiled from the OAI RAN source code. It runs the `nr-uesoftmodem` executable to simulate a 5G SA terminal, establishing the `oaitun_ue1` virtual network interface for data transmission with the core network.

### 3. Prerequisites
To ensure the stability of the OAI system during compilation and execution, and to prevent Out of Memory (OOM) crashes during the parallel compilation of massive C++ projects, the environment of the host machine has been specifically tuned:

*   **Host Machine**: Acer Predator PHN16-72 (32GB RAM, 32 Logical Cores).
*   **Virtualization Environment**: Windows Subsystem for Linux (WSL2) v2.6.3.
    *   **Performance Tuning Strategy**: By default, WSL2 restricts memory allocation to 50% of the physical RAM (16GB). Executing `make -j $(nproc)` (32-thread parallel compilation) under these conditions easily exhausts available memory, resulting in killed processes. Therefore, resources were explicitly reallocated via the Windows `%userprofile%\.wslconfig` file:
        ```ini
        [wsl2]
        memory=24GB
        processors=24
        swap=8GB
        ```
*   **Operating System**: Ubuntu 22.04 LTS (Kernel 6.6.87).
*   **Containerization Configuration (Docker Compose)**:
    *   The default timezone in the official OAI `docker-compose.yaml` is set to Paris (`TZ=Europe/Paris`). To ensure that the log timestamps align with the local environment for subsequent 6G/NTN Scheduling Latency experiments, the environment variables for all core NFs (e.g., MySQL, AMF, SMF) were modified to Taiwan Standard Time:
        ```yaml
        environment:
           - TZ=Asia/Taipei
        ```

---

### 4. Step-by-Step Guide

#### 4.1 Install Docker (5G Core Network Runtime Environment)
```bash
sudo apt update
sudo apt install -y git net-tools putty ca-certificates curl

# Add Docker's official GPG key and repository
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo \"${UBUNTU_CODENAME:-$VERSION_CODENAME}\") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 4.2 取得 OAI 5G 核心網 (CN5G) 設定檔
```bash
wget -O ~/oai-cn5g.zip [https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g](https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g)
unzip ~/oai-cn5g.zip
mv ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g/doc/tutorial_resources/oai-cn5g ~/oai-cn5g
rm -r ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g ~/oai-cn5g.zip
```

### 4.3 安裝 UHD 驅動 (OAI 編譯相依套件)
雖然使用 RF Simulator，但編譯仍相依 UHD 函式庫，需採外部編譯 (Out-of-source build) 保持原始碼目錄乾淨：
```bash
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml

git clone [https://github.com/EttusResearch/uhd.git](https://github.com/EttusResearch/uhd.git) ~/uhd
cd ~/uhd && git checkout v4.8.0.0
cd host && mkdir build && cd build
cmake ../
make -j $(nproc)
sudo make install
sudo ldconfig
sudo uhd_images_downloader
```

### 4.4 編譯 OAI 基地台 (gNB) 與手機端 (nrUE)
```bash
# 1. 取得原始碼並切換至 develop 分支
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
cd ~/openairinterface5g
git checkout develop

# 2. 安裝自動建置腳本所需的相依套件
cd ~/openairinterface5g/cmake_targets
./build_oai -I
sudo apt install -y libforms-dev libforms-bin

# 3. 執行平行編譯 (包含 gNB, nrUE 與 RF Simulator)
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

---

## 5. 關鍵設定檔說明與修改 (Configuration)
本環境主要基於 OAI 官方預設配置，但為了配合 WSL2 Docker 網段的路由規則，必須針對基地台 (gNB) 設定檔進行一處關鍵修改，避免回程封包被系統當作廣播封包丟棄。

*   **gNB 設定檔路徑**：`targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf`
*   **頻段與頻寬**：Band 78, FR1, 106 PRB。
*   **UE 參數**：使用預設 IMSI `001010000000001` 以符合 Docker 核心網中資料庫之預設註冊用戶。

**【修改：廣播位址陷阱排除】**
使用文字編輯器 (`nano`) 打開上述 gNB 設定檔，尋找 `NETWORK_INTERFACES` 區塊：
將原本預設的 `192.168.70.191` (此為 /26 網段之廣播位址) **修改為合法的 Host IP，例如 `192.168.70.129`**。
若未修改，雖能成功建線，但資料平面 (Ping) 將出現 100% 遺失。

---

## 6. 測試與驗證 (Verification)
此章節驗證 5G 端到端連線之各節點狀態。

### 6.1 核心網路 (CN5G) 狀態檢查與防火牆設定
在啟動前，為避免 WSL2 的本機防火牆阻擋 GTP-U 轉發封包，請先執行以下指令開放轉發權限：
```bash
sudo iptables -P FORWARD ACCEPT
```

接著啟動 OAI 核心網容器：
```bash
cd ~/oai-cn5g
docker compose pull
docker compose up -d
docker ps
```
**預期結果 (Expected Output)**：
<img width="1807" height="516" alt="螢幕擷取畫面 2026-05-03 003342" src="https://github.com/user-attachments/assets/2bdeb6ad-36e6-4d11-9583-101ab6c4aedb" />

### 6.2 啟動基站 (gNB) 與終端 (nrUE) 連線
**啟動 gNB (Window 1)**：
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim --rfsimulator.[0].serveraddr server
```

**啟動 nrUE (Window 2)**：
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf --sa -r 106 --numerology 1 --band 78 -C 3619200000 --ssb 516 --rfsim --rfsimulator.serveraddr 127.0.0.1 --uicc0.imsi 001010000000001
```
**預期結果 (Expected Output)**：

#### 6.2.1 gNB 基站端連線確認

> <img width="898" height="59" alt="image" src="https://github.com/user-attachments/assets/4629739f-117f-453f-ad29-7117171fcc8d" />
>
> <br>
> 基地台端成功分配 RNTI 並完成 RRC 連線建立。

#### 6.2.2 nrUE 終端端連線程序

> <img width="1306" height="1205" alt="image" src="https://github.com/user-attachments/assets/beefdc4c-508a-4257-8faa-7dea86b90a68" />
>
> <br>
> 手機端完整之信令流程，涵蓋從實體層同步至 NAS 層註冊完成。

### 6.3 資料平面 (Data Plane) Ping 測試
在 5G 控制平面成功配發 IP 後，接著須驗證資料平面 (Data Plane) 的端到端 (End-to-End) 連通性。本節將透過指定 UE 的虛擬網卡 oaitun_ue1，測試封包是否能順利穿越 5G 核心網 (UPF)，並成功抵達外部資料網路 (oai-ext-dn，IP: 192.168.70.135)。
```bash

# 基礎連通性測試
ping -c 4 -I oaitun_ue1 192.168.70.135

# 穩定度與基準測試
ping -c 100 -I oaitun_ue1 192.168.70.135
```
**預期結果 (Expected Output)**：

#### 6.3.1 基礎連通性測試

> <img width="1198" height="291" alt="image" src="https://github.com/user-attachments/assets/e5d1d4b6-8544-406d-a7f0-4aa802a018c9" />
>
> <br> 
> 此步驟為初步的連通性驗證 (Hello World 測試)。測試結果顯示 4 個 ICMP 封包皆成功循原路回傳，封包遺失率 (Packet Loss) 為 0%，且平均往返延遲 (avg RTT) 約為 10.1 ms。這證明 UE、gNB 與 5G 核心網之間的 GTP-U 隧道已成功建立，且無遭受路由阻擋。

#### 6.3.2 穩定度與基準測試

><img width="1195" height="802" alt="image" src="https://github.com/user-attachments/assets/bf1c3c69-019b-4dff-8756-fae7df84161c" />
>
> <br>
>為確保後續 MAC 層排程 (如 UORA 與 BSR 機制) 延遲分析的準確性，此步驟針對環境進行壓力與穩定度基準測試 (Baseline Test)。觀察連續封包的回傳數據，平均延遲穩定維持在 9.6 ms 左右，且代表網路抖動 (Jitter) 的 mdev 數值極低，僅約 1.8 ms。0% 的遺失率與極低的抖動，證實此 OAI WSL2 虛擬環境在長時間運行下具備高度的可靠性與時序穩定性。

---

## 7. 常見問題與排除 (Troubleshooting)

在建置或運行過程中若遇到報錯，請對照下表進行問題排除：

| Issue 現象 | Root Cause (根本原因) | Solution (解決方案) |
| :--- | :--- | :--- |
| **gNB 啟動閃退**<br>`Assertion (b->th.nbAnt != 0) failed!` | UE 未使用 `ue.conf` 初始化天線，或指令漏打 `--sa` 參數。 | 完整複製 6.2 節的 nrUE 啟動指令，確保 `-O .../ue.conf` 與 `--sa` 同時存在。 |
| **UE 無法取得 IP**<br>`Registration reject` | UE 嘗試使用的 IMSI 不存在於核心網資料庫。 | 於 UE 啟動指令尾端加入 `--uicc0.imsi 001010000000001` 覆寫預設號碼。 |
| **UE 運行中斷線**<br>`Gap in writing to USRP`<br>`errno(14)` | WSL2 資源排程導致 CPU 卡頓，破壞了 5G 實體層極度敏感的時間軸。 | 關閉 Windows 背景吃資源的大型軟體 (如瀏覽器、防毒)，釋放 CPU 後重跑。 |
| **UPF 容器狂重啟**<br>`Exit Code 1` | `develop` 分支的 UPF 已改用 YAML，若套用舊版 `.conf` 會無法啟動。 | 檢查 `docker-compose.yaml`，確保 UPF 掛載路徑為 `/openair-upf/etc/config.yaml`。 |
| **Ping `10.0.0.1` 失敗**<br>`100% Packet Loss` | OAI UPF 安全規則預設無視 (Drop) 打給自己的 ICMP 封包 (此為假警報)。 | 改為「穿透測試」，直接 Ping 外部伺服器 (如 `192.168.70.135`)。 |
| **UE 最後一秒閃退**<br>`[CONFIG] unknown option: --sa` | 參數放置順序不當，觸發 OAI 參數解析器的 Bug。 | 確保 `--sa` 參數緊接在 `-O .../ue.conf` 之後執行。 |


## 8. 參考文獻 (References)
*   [OAI 5G Core Network Deployment Tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_CN5G.md)
*   [OAI 5G SA nrUE Execution Tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_nrUE.md?ref_type=heads)
