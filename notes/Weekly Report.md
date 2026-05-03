## Short-term Goals

1. **[OAI 5G SA 端到端環境建置與連線測試](https://github.com/jtfinn35/Research-Note/blob/main/notes/OAI%205G%20SA%20build-up%20new.md)**
   * **Status:** COMPLETED
   * **Objective:** 在 WSL2 環境下編譯 OAI 核心網與基地台，並透過 RF Simulator 成功建立 gNB 與 UE 的 5G 資料隧道 (Data Plane)，為後續 UORA 實驗建立基準環境。

2. **[UORA 與 BSR 排程機制文獻探討與實作準備]**
   * **Status:** IN PROGRESS
   * **Objective:** 透過 iperf3 測試環境吞吐量，並研讀 OAI 原始碼中 MAC 層 BSR (Buffer Status Report) 的觸發條件，確立下一步 UORA 演算法切入點。

---

## Daily Log (2026-05-03)

**Total Time Spent:** ~6.5 Hours

**13:30–16:00: OAI 系統編譯與核心網除錯 (Troubleshooting)**
* 調整 Windows `.wslconfig` 資源分配 (24GB RAM)，解決 `make` 過程中的 OOM (Out of Memory) 崩潰問題。
* 修改 gNB 設定檔的 `NETWORK_INTERFACES`，排除 `/26` 網段廣播位址 (`.191`) 陷阱，成功解決資料平面 100% Packet Loss 錯誤。
* 確認並解決因 WSL2 背景資源排程導致的實體層時序崩潰 (`Gap in writing to USRP` 與 `errno(14)`)。

**16:00–18:00: 端到端連線測試與穩定度驗證**
* 透過指令列強制覆寫 (`--uicc0.imsi`) UE 註冊號碼，解決與核心網 MySQL 資料庫不匹配造成的 `Registration reject` 問題。
* 執行長時間 Ping 穿透測試，驗證 5G 隧道穩定度，確認平均延遲 (RTT) 維持在 ~9.6ms，網路抖動 (mdev) 極低約 1.8ms。

**19:00–20:00: [SOP 技術文件撰寫與成果彙整](https://github.com/jtfinn35/Research-Note/blob/main/notes/OAI%205G%20SA%20build-up%20new.md)**
* 將今日的除錯過程與環境架設指令，依據實驗室 User Guide 規範，彙整為完整的 Markdown 技術報告。
