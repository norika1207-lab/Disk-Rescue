# 01 災難時間線

## 事故一：APFS SSD 檔案系統損壞

一顆 500GB APFS 外接 SSD，硬體正常（磁碟工具讀得到裝置），但 Finder 掛載不了、看不到檔案。診斷是 APFS 的 spaceman（space manager，空間管理結構）損壞：`APFSSpacemanCIB: invalid object type`。

關鍵洞察：spaceman 管的是「哪些空間可用」，而讀既有檔案靠的是 fs-tree 和 omap（object map），這兩個結構獨立於 spaceman。所以只要繞過 spaceman，檔案內容仍讀得到 → 救援可行，而且可以全程唯讀。

## 事故二：誤格式化 4TB 工作碟

救援中途，一條帶 hardcoded 磁碟編號的格式化指令（`select disk 2` / `diskpart`）在外接碟拔插後磁碟編號重編，指到了另一顆 4TB TOSHIBA 工作碟，把它快速格式化。那顆碟裝著某批重要資料的唯一副本，且來源在下載後已刪、沒有第二份。

這起事故的根因與教訓見 [07 工程教訓](07-lessons.md)：**永遠用卷標認碟，絕不 hardcode 磁碟編號；拔插後磁碟編號會重編，動手前一定重新以 `diskutil list external physical` 對齊。**

## 各階段收拾順序

1. APFS 壞碟 → 唯讀救出全部資料（見 02）
2. 誤格式化的 4TB → 用檔案掃描器的 block map 做 block 直讀救回（見 03）
3. 救回的資料 → 排垃圾打包成 tar、逐檔驗證（見 04）
4. 三顆碟重新分工、格式化清理、資料歸位（見 05）

整個過程的鐵律：先確保資料在另一顆「可讀」的碟上且驗證過，才動任何格式化或刪除。
