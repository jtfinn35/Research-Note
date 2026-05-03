# Weekly Report (2026-05-03)

> **Student**: 黃仁廷 (JTFinn)
> **Advisor**: 鄭教授 (Prof. Ray-Guang Cheng)
> **Project**: 5G/6G NTN UORA 排程機制研究

---

## Studying Notes 

本週的核心目標為完成 OAI 5G SA (Standalone) 虛擬環境的架設，並成功打通 gNB 與 UE 的端到端 (End-to-End) 連線，為後續的 UORA (Uplink OFDMA-based Random Access) 實驗建立穩定的測試平台。

### 關鍵技術突破與除錯 (Troubleshooting)
在 WSL2 環境下編譯與運行 OAI 時，克服了以下關鍵技術挑戰：
1. **WSL2 資源限制導致 OOM**：修改 Windows `.wslconfig` 重新分配 24GB RAM 與 24 Cores，解決 `make` 編譯過程中的記憶體崩潰問題。
2. **5G 網路路由廣播位址陷阱**：排除 OAI 預設 `/26` 網段 (`192.168.70.191`) 的廣播位址限制，成功將 gNB 綁定至合法 IP (`.129`)，解決資料平面 (Data Plane) 100% Packet Loss 的問題。
3. **實體層時序崩潰 (CPU Starvation)**：確認並解決因 WSL2 背景資源排程導致的 `Gap in writing to USRP` 與 `errno(14)` 斷線問題。
4. **IMSI 註冊失敗**：透過指令列強制覆寫 (`--uicc0.imsi`) UE 的 IMSI 號碼，使其與 Docker 核心網內的 MySQL 資料庫匹配。

### 📚 詳細技術文件 (Official SOP)
本週已將上述架設流程、參數配置與除錯細節，整理成符合實驗室規範的技術指南：
👉 **[OAI 5G SA 網路環境建置與測試手冊 (v1.0)](OAI_5G_SA_build-up_new.md)** 

---
