** 1. 安裝 Docker (為 5G 核心網做準備)
核心網 (CN5G) 是透過 Docker 容器來運作的，首先需要安裝 Docker 與相關套件：

Bash
sudo apt update
sudo apt install -y git net-tools putty ca-certificates curl

# 加入 Docker 官方 GPG 密鑰與軟體源
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安裝 Docker 套件
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 將當前使用者加入 docker 群組 (避免每次都要打 sudo)
sudo usermod -a -G docker $(whoami)
注意： 執行完上述指令後，建議先登出再重新登入，或直接輸入 newgrp docker 讓群組權限生效。

2. 下載並啟動 OAI 5G 核心網 (CN5G)
取得官方寫好的 Docker Compose 設定檔，並把核心網架設起來：

Bash
# 下載並解壓縮設定檔
wget -O ~/oai-cn5g.zip https://gitlab.eurecom.fr/oai/openairinterface5g/-/archive/develop/openairinterface5g-develop.zip?path=doc/tutorial_resources/oai-cn5g
unzip ~/oai-cn5g.zip
mv ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g/doc/tutorial_resources/oai-cn5g ~/oai-cn5g
rm -r ~/openairinterface5g-develop-doc-tutorial_resources-oai-cn5g ~/oai-cn5g.zip

# 進入資料夾，拉取映像檔並啟動核心網
cd ~/oai-cn5g
docker compose pull
docker compose up -d
3. 安裝 UHD 驅動 (OAI 編譯相依套件)
即使我們只用 RF Simulator 而不接實體 USRP 天線，OAI 底層的編譯依然依賴 UHD 函式庫，必須先從源碼編譯它：

Bash
# 安裝相依套件
sudo apt install -y autoconf automake build-essential ccache cmake cpufrequtils doxygen ethtool g++ git inetutils-tools libboost-all-dev libncurses-dev libusb-1.0-0 libusb-1.0-0-dev libusb-dev python3-dev python3-mako python3-numpy python3-requests python3-scipy python3-setuptools python3-ruamel.yaml

# 從源碼編譯 UHD
git clone https://github.com/EttusResearch/uhd.git ~/uhd
cd ~/uhd
git checkout v4.8.0.0
cd host
mkdir build && cd build
cmake ../
make -j $(nproc)
sudo make install
sudo ldconfig
sudo uhd_images_downloader
4. 下載與編譯 OAI 基地台 (gNB) 與手機端 (nrUE)
接下來編譯 OAI 主程式。這步驟會花費較多時間，指令中的 -j 或 --ninja 會盡可能調用你的 CPU 多核心來加速：

Bash
# 取得 OAI 源碼 (使用 develop 分支)
git clone https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
cd ~/openairinterface5g
git checkout develop

# 安裝 OAI 特定的系統相依套件
cd ~/openairinterface5g/cmake_targets
./build_oai -I

# 安裝示波器 (nrscope) 視覺化工具相依套件
sudo apt install -y libforms-dev libforms-bin

# 同時編譯 gNB 與 nrUE
cd ~/openairinterface5g/cmake_targets
./build_oai -w USRP --ninja --nrUE --gNB --build-lib "nrscope" -C
5. 執行 RF Simulator 模擬 (開啟三個終端機)
環境都準備好後，我們需要模擬端對端連線。請開啟三個獨立的 Terminal 視窗：

Terminal 1：確認核心網運行中
如果你前面的 docker compose up -d 沒有關閉，這個步驟可以略過。如果有重開機，請回到 ~/oai-cn5g 執行 docker compose up -d。

Terminal 2：啟動 OAI gNB (基地台)
進入編譯好的目錄，加上 --rfsim 參數讓它跑在模擬模式：

Bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-softmodem -O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf --gNBs.[0].min_rxtxtime 6 --rfsim
Terminal 3：啟動 OAI nrUE (手機)
等 gNB 啟動完畢（畫面開始穩定印出 frame 資訊時），在第三個視窗啟動模擬手機，同樣加上 --rfsim：

Bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
sudo ./nr-uesoftmodem -r 106 --numerology 1 --band 78 -C 3619200000 --uicc0.imsi 001010000000001 --rfsim
6. 端對端連線測試
當 UE 成功連接上 gNB 並且完成註冊到 5G 核心網後，作業系統會產生一個名為 oaitun_ue1 的虛擬網路介面（這代表手機的網卡）。

你可以開啟 Terminal 4，嘗試從這個虛擬手機介面 Ping 核心網的預設 IP (192.168.70.135)，如果有封包回應，代表你的 5G SA 模擬網路已經成功通了！

Bash
ping 192.168.70.135 -I oaitun_ue1
