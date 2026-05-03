# Weekly Report

> **Student**: 黃仁廷 (JTFinn)
> **Advisor**: 鄭教授 (Prof. Ray-Guang Cheng)

| 日期 (Date) | 本週進度與除錯 (Studying Notes) | 產出與參考文件 (Documents) | 下週計畫 (Studying Plan) |
| :--- | :--- | :--- | :--- |
| **2026-05-03** | **[核心目標]**<br>完成 OAI 5G SA 虛擬環境架設，並成功打通 gNB 與 UE 的端到端 (E2E) 連線。<br><br>**[關鍵技術突破與除錯]**<br>1. **WSL2 OOM 崩潰**: 修改 Windows `.wslconfig` 重新分配 24GB RAM 與 24 Cores。<br>2. **廣播位址陷阱**: 排除 `/26` 網段限制，將 gNB 綁定至合法 IP (`.129`)，解決 100% Packet Loss。<br>3. **實體層時序崩潰**: 解決因背景資源排程導致的 `errno(14)` 與 `Gap in writing to USRP` 斷線問題。<br>4. **IMSI 註冊失敗**: 透過指令列強制覆寫 (`--uicc0.imsi`) 匹配核心網 MySQL 資料庫。 | 👉 **[OAI 5G SA 網路環境建置與測試手冊 (v1.0)](https://github.com/jtfinn35/Research-Note/blob/main/notes/OAI%205G%20SA%20build-up%20new.md)** | **[推進至 UORA 基礎測試]**<br>1. 使用 `iperf3` 在目前的 5G 隧道中進行極限頻寬 (Throughput) 測試。<br>2. 透過 Wireshark 側錄封包，觀察 Buffer Status Report (BSR) 觸發時機與相關排程參數。 |
