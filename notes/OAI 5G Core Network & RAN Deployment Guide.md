# OAI 5G Core Network & RAN Deployment Guide

本文件紀錄了部署 OpenAirInterface (OAI) 5G 核心網與 RAN 的完整步驟。

---

## 步驟 1：下載核心網部署檔並啟動 Docker

### 執行指令
```bash
git clone [https://gitlab.eurecom.fr/oai/oai-cn5g-fed.git](https://gitlab.eurecom.fr/oai/oai-cn5g-fed.git)
cd oai-cn5g-fed/docker-compose
python3 core-network.py --type OAI-OAI-OAI --scenario 1 --capture /tmp/oai_pcap.pcap
```

### Details
5G 核心網由很多微服務（AMF、SMF、UPF、NRF 等）組成。我們利用 Docker 把這些服務以「貨櫃」的形式一次性跑起來。同時，Docker 會在電腦裡建立一張專屬的「虛擬網卡」，作為總公司的內部網路。

---

## 步驟 2：下載並編譯 OAI RAN 原始碼

### 執行指令
```bash
git clone [https://gitlab.eurecom.fr/oai/openairinterface5g.git](https://gitlab.eurecom.fr/oai/openairinterface5g.git)
cd openairinterface5g
source oaienv
cd cmake_targets
./build_oai -I -w SIMU --gNB --nrUE
```

### Details
OAI 的基地台和手機是由幾百萬行的 C/C++ 寫成的。這個步驟是去安裝所有必要的依賴庫 (-I)，並且告訴編譯器：「請幫我打造出一台 5G 基地台 (--gNB)、一支 5G 手機 (--nrUE)，並且包含純軟體射頻模擬器 (-w SIMU)」。編譯完成後會得到兩個執行檔：`nr-softmodem` 和 `nr-uesoftmodem`。

---

## 步驟 3：修改基地台設定檔 (.conf)

### 說明
打開 gNB 的設定檔，找到 **NETWORK_INTERFACES** 區塊，將裡面的 `GNB_IPV4_ADDRESS_FOR_NG_AMF` 等 IP，改為與步驟 1 中 Docker 網卡相同的網段。

### Details
這步驟是在完成網路橋接。基地台本身只負責發射電波，它必須知道核心網的 IP 位址在哪裡。 IP 正確基地台才能成功建立 NGAP 連線（控制面，向 AMF 註冊）與 GTP-U 隧道（資料面，把手機的資料送給 UPF）。

---

## 步驟 4：啟動基地台 (gNB)

### 執行指令
```bash
cd openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O [設定檔路徑] --gNBs.[0].min_rxtxtime 6 --rfsim --rfsimulator.options server
```

### 說明
這行指令正式把基地台開起來。它會做三件事：
1. **向核心網報到** (NGSetupRequest)。
2. **啟動 5G 協定棧**：啟動實體層 (PHY) 到無線資源控制層 (RRC) 的所有 5G 協定棧。
3. **開啟射頻模擬器**：開啟 `--rfsim`，並把自己設定為 Server，對著空氣發佈同步訊號 (SSB)，等待手機連線。

---

## 步驟 5：啟動虛擬手機 (nrUE)

### 執行指令
```bash
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --uicc0.imsi 001010000000001 --rfsim --rfsimulator.serveraddr 127.0.0.1
```

### Details
啟動手機的 5G 晶片模擬。手機啟動後會：
1. 尋找基地台發出的同步訊號。
2. 發起隨機存取 (Random Access, Msg1~Msg4)。
3. 成功後，將 SIM 卡資訊 (imsi) 透過基地台傳給核心網進行身分認證。
4. 認證通過後，核心網會配發一個 IP 給這支手機，完成 PDU Session 建立。這時畫面上就會開始瘋狂跳動 MAC 層的排程資訊。

---

## 步驟 6：資料傳輸測試 (Ping)

### 執行指令
```bash
ping -I oaitun_ue1 8.8.8.8
```

### 詳細說明 (Details)
驗證這條 5G 網路是不是真的能傳資料。強制封包走核心網發給手機的那張專屬虛擬網卡 (oaitun_ue1)。如果 Ping 得通，代表資料成功穿越了：
**UE -> (射頻模擬) -> gNB -> (GTP-U) -> UPF (核心網) -> 外網**。
