# 架構與維護

```text
Spotlight → Apple 捷徑 → frontend.py → AppKit 原生視窗
                              ↓ JSON／命令列參數
                       music_downloader.py
                              ↓
                 SQLite 佇列 ← launchd worker
                              ↓
                  yt-dlp → ffmpeg → ffprobe
                              ↓
                  桌面／音樂／分類／MP3
```

## 原生介面與背景工作

`native_ui.swift` 編譯為私有 AppKit helper，使用 JSON stdin 傳遞資料；URL、分類不拼接為 Shell 指令。`frontend.py` 負責網址、首數、分類與提交摘要。`progress_status.py` 解析資料庫與目前下載 Log，為原生進度視窗提供狀態。

前端不直接掃描 Desktop，而是讀取工具私有分類快取；worker 定期刷新。關閉視窗只停止顯示，不取消下載。

launchd 在使用者登入後啟動單一 worker，以 FIFO 佇列處理任務。檔案鎖與 SQLite 交易保護重複提交、多入口及中斷恢復；逐首失敗不停止清單其餘項目。

## 成功記錄與檔案發布

下載完成後先以 ffprobe 驗證正時長 MP3，再計算 SHA-256 與發布。Video ID、檔案位置與雜湊由 SQLite 共用；記錄存在但檔案不再有效時，worker 可重新處理。

同名檔以 ID／序號區分，原子發布不覆寫既有檔案。檔案供應器不支援硬連結時使用排他重新命名備援。無論何種去重結果，都不以相似標題刪除既有音樂。

`yt-dlp --download-archive` 未作為成功記錄的權威來源，因為文字 archive 無法單獨滿足刪檔恢復、多任務與檔案存在驗證。

## 測試

執行 README 的 unittest 指令。測試以臨時目錄與模擬外部程序驗證資料庫、檔案安全與前端邊界；另以自產短音源測試真正 yt-dlp／ffmpeg 的 MP3、metadata 與封面流程。結果與限制見 TEST_REPORT.md。

Python 使用標準函式庫；外部相依為 Homebrew 工具與 Apple 原生框架。變更下載引擎、原生協定或安裝流程時，應使用隔離資料測試，避免以個人音樂庫作為測試環境。
