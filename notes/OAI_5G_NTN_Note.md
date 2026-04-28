## 1. OAI 系統架構概念

在 OAI 的實驗環境中，主要分為以下三個核心部分：

* **OAI CN5G (Core Network)**：5G 核心網。負責用戶管理、IP 分配、安全認證等。通常於 **Docker** 容器中運作。
* **OAI gNB (Next Generation NodeB)**：5G 基地台。負責無線電波的發射與接收，是連接 UE 與 CN 的橋樑。
* **OAI nrUE (NR User Equipment)**：5G 終端設備（如 5G 手機）。在研究中是模擬與測試的主角。

---

## 2. 5G 協定棧 (Protocol Stack) 深度解析

### L3 - 無線資源控制層
* **RRC (Radio Resource Control)**：負責基地台與手機間的連線建立、狀態管理（Idle/Connected）及換手 (Handover)。

5G 協定與傳統 IP 協定的最大差異在於 **L2 (Data Link Layer)**。

### L2 - 無線連接層 (重點子層)
| 協定層 | 全名 | 主要功能 |
| :--- | :--- | :--- |
| **SDAP** | Service Data Adaptation Protocol | **階級制度 (QoS)**：根據服務類型（如視訊、下載）貼上標籤，分配不同優先權。 |
| **PDCP** | Packet Data Convergence Protocol | **瘦身與加密**：標頭壓縮 (ROHC) 以節省頻寬，並進行資料加密與完整性保護。 |
| **RLC** | Radio Link Control | **切碎與重組**：根據訊號品質將 IP 封包切割成適當大小，並負責遺失碎片的重傳 (ARQ)。 |
| **MAC** | Medium Access Control | **排程與資源分配**：5G 採「絕對獨裁」機制，由 gNB 決定誰能在哪毫秒使用哪個頻率。 |

### L1 - 實體層
* **PHY (Physical Layer)**：利用 **OFDM** (正交頻分多工) 技術，將數位位元 (0/1) 轉換為電磁波信號。

---

## 3. 資源請求與排程機制

在 5G 中，手機不能隨意發射訊號，必須獲得 gNB 的授權：

1. **傳統 BSR (Buffer Status Report)**：手機主動告知 gNB：「我有資料要傳，請給我資源」。
2. **隨機存取程序 (RACH)**：當手機連 BSR 的通道都沒有時，需在公共頻道發起請求。
   * **UORA (Uplink OFDMA-based Random Access)**：進階的隨機存取機制，常用於 6G 或衛星網路研究。

---

## 4. 非地面網路 (NTN) 的挑戰

衛星通訊（如 LEO 低軌衛星）環境與地面網路截然不同：
* **極高延遲**：訊號來回衛星與地面站的 RTT (Round Trip Time) 極長，傳統 BSR 機制會導致資源分配效率低下。
* **都卜勒頻移 (Doppler Shift)**：衛星高速移動會導致無線電頻率偏移，需精密的同步演算法補償。

---

## 5. 常用術語與工具

* **SA (Standalone)**：純 5G 架構（5G 基地台 + 5G 核心網），目前 OAI 的預設模式。
* **NSA (Non-Standalone)**：5G 基地台需依賴 4G 核心網運作。
* **SDR / USRP**：軟體定義無線電設備，用於發射實體無線電波。
* **RFsimulator**：**重要工具**。在沒有實體天線的情況下，於軟體層面模擬無線訊號傳遞，適合在 WSL2/Linux 上初步開發。

---

## 6. 實作指令參考 (Cheat Sheet)

### 系統與進程管理
```bash
# 查看所有進程的詳細資訊 (包含 PID、UID 等)
ps ael

# 強制刪除特定進程 (PID 為進程 ID)
sudo kill -9 <PID>

# 切換為超級使用者 (Root)
sudo -i
