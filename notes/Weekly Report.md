## Short-term Goals

* **[Literature Review] 5G/6G Random Access Channel Attack, Detection, and Mitigation**
  * **Status:** IN PROGRESS
  * **Objective:** Conduct an in-depth study of this paper to systematically break down the mathematical analytical framework of RACH attack and mitigation mechanisms. The current phase will be focusing on analyzing key variables during attack initiation (e.g., Preamble Collision Probability, System Resource Depletion Rate) and core parameter settings in the detection model (e.g., Detection Threshold, False Alarm Rate). This establishes a quantitative theoretical foundation for implementing defense mechanisms in the OAI environment.

* **[Environment Setup] OAI 5G SA End-to-End Connection Testing**
  * **Status:** COMPLETED
  * **Objective:** Complete the deployment of the 5G Data Plane tunnel between gNB and UE in the WSL2 environment and produce standardized technical documentation.

---
### [Week of 2026-04-27 ~ 2026-05-03]

**Total Time Spent:** ~12.5 Hours

**【 2026-04-27 】 OAI Official Tutorial Deployment & Initial Testing (約 3 小時)**
* 依照 OAI 官方教學文檔，完整進行 5G 核心網與基站/終端的基礎編譯與安裝。
* **測試結果與瓶頸**：僅能成功建立終端設備 (UE) 與基地台 (gNB) 之間的底層連線，但無法順利生成虛擬網卡 (oaitun_ue1)，導致無法進行後續的 IP 配發與資料平面 (Data Plane) 測試。

**【 2026-04-30 】 First Phase Debugging & Command Adjustment (約 3 小時)**
* 針對虛擬網卡與 IP 分配失敗之問題進行首次除錯。
* 重新檢視並調整 gNB 與 UE 端的啟動指令與相關設定檔參數。
* **測試結果與瓶頸**：成功突破網卡限制並取得 IP，但在進行資料平面 Ping 測試時，遭遇 100% Packet Loss 的路由異常。

**【 2026-05-03 】 Final Troubleshooting & System Validation (約 6.5 小時)**
**13:30–16:00: OAI System Compilation & Core Network Troubleshooting**
* 調整 Windows `.wslconfig` 資源分配 (24GB RAM)，解決 `make` 過程中的 OOM 崩潰問題。
* 修改 gNB 設定檔的 `NETWORK_INTERFACES`，排除 `/26` 網段廣播位址 (`.191`) 陷阱，成功解決 100% Packet Loss 異常。
* 解決因 WSL2 背景資源排程導致的實體層時序崩潰 (`Gap in writing to USRP`)。

**16:00–18:00: E2E Connection Testing & Stability Verification**
* 指令列強制覆寫 (`--uicc0.imsi`) UE 註冊號碼，解決 `Registration reject` 錯誤。
* 執行 Ping 穿透測試，確認平均延遲 (RTT) 穩定維持於 ~9.6ms，網路抖動 (mdev) ~1.8ms。

**19:00–20:00: Technical Documentation**
**[OAI 5G SA 網路環境建置與測試手冊](https://github.com/jtfinn35/Research-Note/blob/main/notes/OAI%205G%20SA%20build-up%20new.md)**
