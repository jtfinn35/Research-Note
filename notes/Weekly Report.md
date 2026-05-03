## 🎯 Short-term Goals

* **[Literature Review] 5G/6G Random Access Channel Attack, Detection, and Mitigation**
  * **Status:** IN PROGRESS
  * **Objective:** 詳閱由鄭瑞光教授與陳奕全 (Richard) 學長共同發表的論文，深入探討 RACH 攻擊模式與防禦機制，為後續實作建立理論基礎。

* **[Environment Setup] OAI 5G SA End-to-End Connection Testing**
  * **Status:** COMPLETED
  * **Objective:** 在 WSL2 環境下完成 gNB 與 UE 的 5G 資料隧道 (Data Plane) 架設，產出標準化技術文件。

---

## 📅 Log History

### [Week of 2026-05-03]

**Total Time Spent:** ~6.5 Hours

**13:30–16:00: OAI System Compilation & Core Network Troubleshooting**
* 調整 Windows `.wslconfig` 資源分配 (24GB RAM)，解決 `make` 過程中的 OOM 崩潰問題。
* 修改 gNB 設定檔的 `NETWORK_INTERFACES`，排除 `/26` 網段廣播位址 (`.191`) 陷阱，成功解決 100% Packet Loss。
* 解決因 WSL2 背景資源排程導致的實體層時序崩潰 (`Gap in writing to USRP`)。

**16:00–18:00: E2E Connection Testing & Stability Verification**
* 指令列強制覆寫 (`--uicc0.imsi`) UE 註冊號碼，解決 `Registration reject`。
* 執行 Ping 穿透測試，確認平均延遲 (RTT) ~9.6ms，網路抖動 (mdev) ~1.8ms。

**19:00–20:00: Technical Documentation**
**[Deliverable]: [OAI 5G SA 網路環境建置與測試手冊](https://github.com/jtfinn35/Research-Note/blob/main/notes/OAI%205G%20SA%20build-up%20new.md)** 

---
