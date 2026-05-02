# OAI 5G SA (Standalone) 網路環境建置與測試手冊

> **文件版本** v1.0
> **作者**: 黃仁廷 (JTFinn)
> **建立日期**: 2026-05-03

### 目錄 (Table of Contents)
1. [專案概述 (Executive Summary)](#1-專案概述-executive-summary)
2. [系統架構與網路拓樸 (Architecture & Topology)](#2-系統架構與網路拓樸-architecture--topology)
3. [測試環境與前置準備 (Prerequisites)](#3-測試環境與前置準備-prerequisites)
4. [安裝與操作步驟 (Step-by-Step Guide)](#4-安裝與操作步驟-step-by-step-guide)
5. [關鍵設定檔說明 (Configuration)](#5-關鍵設定檔說明-configuration)
6. [測試與驗證 (Verification)](#6-測試與驗證-verification)
7. [常見問題與排除 (Troubleshooting)](#7-常見問題與排除-troubleshooting)
8. [參考文獻 (References)](#8-參考文獻-references)

---

### 1. 專案概述 (Executive Summary)
本文件旨在提供 OpenAirInterface (OAI) 5G Standalone (SA) 模式的標準安裝與測試流程。透過 WSL2 環境建置 OAI 核心網 (CN5G) 與無線接取網 (RAN)，並使用 RF Simulator 完成基站 (gNB) 與終端設備 (UE) 的端到端 (E2E) 模擬連線測試，為後續導入 PRACH 攻擊與防禦演算法建立基礎測試平台。

### 2. 系統架構與網路拓樸 (Architecture & Topology)
本環境採純軟體模擬，包含以下三個主要節點，所有連線皆於本機 (Localhost/127.0.0.1) 內網進行：
*   **OAI-CN5G (5G Core Network)**：透過 Docker Compose 部署，包含 AMF, SMF, UPF, NRF 等核心網路功能 (NFs)。
*   **OAI gNB (基地台)**：透過 OAI RAN 原始碼編譯，啟用 `nr-softmodem` 並使用 RF Simulator 取代實體 USRP 硬體介面。
*   **OAI nrUE (終端設備)**：透過 OAI RAN 原始碼編譯，啟用 `nr-uesoftmodem` 模擬 5G SA 終端設備，建立 `oaitun_ue1` 虛擬網卡與核心網進行資料傳輸。

### 3. 測試環境與前置準備 (Prerequisites)

為確保 OAI 系統編譯與執行之穩定性，避免大型 C++ 專案平行編譯時發生 OOM (Out of Memory) 崩潰，本專案針對 Host 硬體進行了環境調優：

*   **實體硬體 (Host Machine)**：Acer Predator PHN16-72 (32GB RAM, 32 Logical Cores)
*   **虛擬化環境**：Windows Subsystem for Linux (WSL2) v2.6.3
    *   效能調優策略：由於 WSL2 預設會將記憶體限制為實體 RAM 的 50% (16GB)。若貿然執行 `make -j $(nproc)` (32 執行緒平行編譯)，極易耗盡記憶體導致程序被強制終止 (Killed)。因此已透過 Windows 端 `%userprofile%\.wslconfig` 重新分配資源：
        ```ini
        [wsl2]
        memory=24GB
        processors=24
        swap=8GB
        ```
*   **作業系統**：Ubuntu 22.04 LTS (Kernel 6.6.87)
*   **容器化配置調優 (Docker Compose)**：
    *   由於 OAI 官方提供的 `docker-compose.yaml` 預設時區為法國巴黎 (`TZ=Europe/Paris`)。為確保後續 6G/NTN 排程延遲 (Scheduling Latency) 實驗的 Log 時間戳記與本機環境一致，已將核心網所有 NFs (如 MySQL, AMF, SMF 等) 的環境變數修改為台灣標準時間：
        
```yaml
        environment:
            - TZ=Asia/Taipei
```

---

### 4. 安裝與操作步驟 (Step-by-Step Guide)

### 4.1 安裝 Docker (5G 核心網運行環境)
```bash
sudo apt update
sudo apt install -y git net-tools putty ca-certificates curl

# 加入 Docker 官方 GPG 密鑰與軟體源
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
git clone [https://gitlab.eurecom.fr/oai/openairinterface5g.git](https://gitlab.eurecom.fr/oai/openairinterface5g.git) ~/openairinterface5g
cd ~/openairinterface5g
git checkout develop

# 執行自動建置腳本
cd ~/openairinterface5g/cmake_targets
./build_oai -I
sudo apt install -y libforms-dev libforms-bin

# 同時編譯 gNB 與 nrUE
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

---

## 5. 關鍵設定檔說明 (Configuration)
本環境採 OAI 官方預設配置，未對核心網與基地台參數進行客製化修改。
*   **gNB 設定檔路徑**：`targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf`
*   **頻段與頻寬**：Band 78, FR1, 106 PRB。
*   **UE 參數**：使用預設 IMSI `001010000000001` 以符合 Docker 核心網中資料庫之預設註冊用戶。

---

## 6. 測試與驗證 (Verification)
此章節驗證 5G 端到端連線之各節點狀態。

### 6.1 核心網路 (CN5G) 狀態檢查
```bash
cd ~/oai-cn5g
docker compose pull
docker compose up -d
docker ps
```
**預期結果 (Expected Output)**：
> ```text
> [jtfinn@DESKTOP-4VFQQL9:~/oai-cn5g$ docker compose up -d
[+] up 11/11
 ✔ Network oai-cn5g-public-net Created                                                                              0.0s
 ✔ Container oai-nrf           Started                                                                              1.1s
 ✔ Container ims               Started                                                                              1.1s
 ✔ Container mysql             Started                                                                              1.1s
 ✔ Container oai-ext-dn        Started                                                                              1.0s
 ✔ Container oai-udr           Started                                                                              1.1s
 ✔ Container oai-udm           Started                                                                              1.2s
 ✔ Container oai-ausf          Started                                                                              1.3s
 ✔ Container oai-amf           Started                                                                              1.4s
 ✔ Container oai-smf           Started                                                                              1.6s
 ✔ Container oai-upf           Started                                                                              2.0s
jtfinn@DESKTOP-4VFQQL9:~/oai-cn5g$ docker ps
CONTAINER ID   IMAGE                                     COMMAND                  CREATED          STATUS                            PORTS                                                   NAMES
542be273748c   oaisoftwarealliance/oai-upf:develop       "sh /openair-upf/bin…"   9 seconds ago    Up 7 seconds (health: starting)   2152/udp, 8805/udp, 5342-5344/tcp                       oai-upf
2f77bd3c5ebb   oaisoftwarealliance/oai-smf:develop       "/openair-smf/bin/oa…"   9 seconds ago    Up 7 seconds (health: starting)   80/tcp, 5342-5344/tcp, 8080/tcp, 9090/tcp, 8805/udp     oai-smf
726b505a8e92   oaisoftwarealliance/oai-amf:develop       "/openair-amf/bin/oa…"   9 seconds ago    Up 7 seconds (health: starting)   80/tcp, 5342-5344/tcp, 8080/tcp, 9090/tcp, 38412/sctp   oai-amf
b9cec451c60d   oaisoftwarealliance/oai-ausf:develop      "/openair-ausf/bin/o…"   9 seconds ago    Up 7 seconds (health: starting)   80/tcp, 5342-5344/tcp, 8080/tcp                         oai-ausf
47e78be0a35b   oaisoftwarealliance/oai-udm:develop       "/openair-udm/bin/oa…"   9 seconds ago    Up 7 seconds (health: starting)   80/tcp, 5342-5344/tcp, 8080/tcp                         oai-udm
c04d9a0461fa   oaisoftwarealliance/oai-udr:develop       "/openair-udr/bin/oa…"   9 seconds ago    Up 8 seconds (health: starting)   80/tcp, 8080/tcp                                        oai-udr
f3b50def7523   mysql:9.6                                 "docker-entrypoint.s…"   10 seconds ago   Up 8 seconds (health: starting)   3306/tcp, 33060/tcp                                     mysql
1b09b64608ec   oaisoftwarealliance/oai-nrf:develop       "/openair-nrf/bin/oa…"   10 seconds ago   Up 8 seconds (health: starting)   80/tcp, 5342-5344/tcp, 8080/tcp                         oai-nrf
2594e32d475f   oaisoftwarealliance/trf-gen-cn5g:latest   "/bin/bash -c ' ip r…"   10 seconds ago   Up 8 seconds (healthy)                                                                    oai-ext-dn
a49d1d94cefd   oaisoftwarealliance/ims:latest            "asterisk -fp"           10 seconds ago   Up 8 seconds (healthy)                                                                    ims
]
> ```

### 6.2 啟動基站 (gNB) 與終端 (nrUE) 連線
**啟動 gNB (Window 1)**：
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim --rfsimulator.[0].serveraddr server
```

**啟動 UE (Window 2)**：
```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --uicc0.imsi 001010000000001 --rfsim --rfsimulator.serveraddr 127.0.0.1 --SA
```
**預期結果 (Expected Output)**：
> ```text
> [請在此貼上 UE 終端機顯示 RRC Connection Setup Complete，或是分配到 IP 的綠字 Log 截圖]
> ```

### 6.3 資料平面 (Data Plane) Ping 測試
連線建立後，確認虛擬網卡 `oaitun_ue1` 是否能連至外部網路。
```bash
# 確認網卡 IP
ip addr | grep oaitun

# Ping 測試
ping -I oaitun_ue1 8.8.8.8
```
**預期結果 (Expected Output)**：
> ```text
> [請在此貼上你 ping 8.8.8.8 成功收到封包回覆的 Log 截圖]
> ```

---

## 7. 常見問題與排除 (Troubleshooting)

*   **Issue 1: 啟動 UE 時無法與核心網連線 / RRC 逾時**
    *   **Root Cause**: 啟動 `nr-uesoftmodem` 時未指定網路架構模式，導致系統可能預設尋找 NSA 環境之 4G eNB。
    *   **Solution**: 必須於啟動指令最後方明確加入 `--SA` 參數，強制設備以 5G Standalone 模式運作。

## 8. 參考文獻 (References)
*   [OAI 5G Core Network Deployment Tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_CN5G.md)
*   [OAI 5G SA nrUE Execution Tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_OAI_nrUE.md?ref_type=heads)
```
