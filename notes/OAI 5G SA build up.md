## 1. 安裝 Docker ( 5G 核心網)

核心網 (CN5G) 是透過 Docker 容器來運作的，首先需要安裝 Docker 與相關套件：

```Bash
# 安裝基本網路工具
sudo apt update
sudo apt install -y git net-tools putty ca-certificates curl

# 加入 Docker 官方 GPG 密鑰 (防偽認證)
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 將 Docker 官方加入軟體源 (避免下載時系統去Ubuntu的的舊倉庫抓，而是去 Docker 官方伺服器抓最新版本)
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安裝完整 Docker 套件
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**注意： 執行完上述指令後，執行 reboot 讓系統重新啟動**

---

## 2. 取得 OAI 5G 核心網 (CN5G) 設定檔

取得官方寫好的 Docker Compose 設定檔：

```Bash
# 下載並解壓縮設定檔
wget -O ~/oai-cn5g.zip https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g
unzip ~/oai-cn5g.zip
mv ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g/doc/tutorial_resources/oai-cn5g ~/oai-cn5g
rm -r ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g ~/oai-cn5g.zip
```
---

## 3. 安裝 UHD 驅動 (OAI 編譯相依套件)

即使只用 RF Simulator 而不接實體 USRP 天線，OAI 底層的編譯依然依賴 UHD 函式庫，必須先從源碼編譯：

```Bash
# 安裝相依套件 (Dependencies)
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml

# 取得原始碼與選定版本
git clone https://github.com/EttusResearch/uhd.git ~/uhd
cd ~/uhd
git checkout v4.8.0.0

# 執行外部編譯 (Out-of-source build)
# 為了隔離建構產物 (Build artifacts，如 .o 目的檔、CMake 快取) 與原始碼樹 (Source tree)，保持版本控制目錄的乾淨
cd host
mkdir build && cd build
  
# 建構系統生成 (Build System Generation)
cmake ../

# 編譯與連結 (Compilation & Linking)
# $(nproc) ：輸出系統目前的邏輯核心數
# -j (Jobs) ：指示 GNU Make 啟動平行編譯 (Parallel Compilation)，藉由多執行緒最大化 CPU 利用率，縮短建構時間
make -j $(nproc)
  

# 安裝與動態連結器快取更新與韌體下載
sudo make install
sudo ldconfig
sudo uhd_images_downloader
```

---

## 4. 編譯 OAI 基地台 (gNB) 與手機端 (nrUE)
編譯 OAI 主程式。

```Bash
# 取得 OAI 主程式原始碼並切換至 develop 分支
# develop 分支通常為整合最新功能（Features）與錯誤修復（Bug Fixes）的主要開發分支
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
cd ~/openairinterface5g
git checkout develop

# 安裝 OAI 特定的系統相依套件
cd ~/openairinterface5g/cmake_targets
./build_oai -I

# 安裝示波器 (nrscope) 視覺化工具相依套件
sudo apt install -y libforms-dev libforms-bin

# 同時編譯 gNB 與 nrUE
# --nrUE：編譯 5G NR 使用者設備（User Equipment）的軟體定義模組 (nr-uesoftmodem)
# --gNB：編譯 5G NR 基地台（gNodeB）的軟體定義模組 (nr-softmodem)
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
```

---

## 5. 啟動5G核心網 (CN5G)

```Bash
# 啟動核心網並於背景執行，避免運作日誌 (log) 洗版 Terminal 視窗
cd ~/oai-cn5g
docker compose pull
docker compose up -d
```

---

## 6. 執行 RF Simulator 模擬連線

```Bash
# 啟動基地台 (gNB)
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim --rfsimulator.[0].serveraddr server

# 執行手機端 (nrUE)
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --uicc0.imsi 001010000000001 --rfsim --rfsimulator.serveraddr 127.0.0.1

# 透過 5G 模擬手機的虛擬網卡，測試是否能連上外部的網際網路
ping -I oaitun_ue1 8.8.8.8

# 完成後關閉 5G 核心網
cd ~/oai-cn5g
docker compose down

# 確認沒有任何基地台或手機於背景執行
pgrep -l nr-softmodem
pgrep -l nr-uesoftmodem

# 確認射頻通訊埠 (4043) 關閉
sudo ss -tuln | grep 4043

# 確認 Docker 核心網關閉
docker ps | grep oai
---
