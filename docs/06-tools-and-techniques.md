# 06 工具與技巧總表

## 救援工具

| 工具 | 用途 | 備註 |
|---|---|---|
| ddrescue | raw 順序讀、做完整映像 | `-n` no-scrape 先快掃；mapfile 續傳；壞碟正解 |
| apfs-fuse (ApfsLib) | 讀 APFS，可自寫 extractor | 繞過 spaceman；link library 不用 mount |
| 檔案掃描器的 .scan DB | 取得檔案→磁區 block map | quick format 後 block 直讀重建的關鍵 |
| TSK (Sleuth Kit) | 列 APFS 目錄 | 讀內容會撞損壞崩潰，只堪列目錄 |
| dot_clean | 清 exFAT 上的 `._` 伴生檔 | `-m` 併回再刪，批次快 |
| rsync (openrsync) | 可續傳搬移 | macOS 內建不支援 `--info=progress2` |
| tar | 排垃圾打包 | 配合 `--exclude` 排依賴/快取 |
| Python tarfile | 權威比對 tar vs 磁碟 | 避開中文八進制轉義陷阱 |

## 觀測與判斷指令

| 目的 | 指令 |
|---|---|
| 認碟（別靠磁碟編號） | `diskutil list external physical` / `diskutil info /Volumes/卷標` |
| 看容量變化判進度 | `diskutil info` 取 Volume Used Space |
| 看某進程正在讀寫哪個檔 | `lsof -p <pid>` \| grep 掛載點 |
| 看進程是否卡在 I/O | `ps -o pid,stat,%cpu` — STAT 含 U = uninterruptible sleep |
| 檔案實際佔用磁區數 | `stat -f%b`（配 `-f%z` 邏輯大小比對配置單元浪費） |
| raw 唯讀讀裝置 | `os.open("/dev/rdiskN", os.O_RDONLY)`，512 對齊 |

## 跨 OS 技巧

- 壞碟在 macOS 卡 U 狀態殺不掉 → 帶到 Linux，可調 `/sys/block/sdX/device/timeout`、raw 讀不觸發自動掛載。
- WSL2 rootfs 是 `ext4.vhdx`，其註冊路徑（registry BasePath）要與檔案實際位置一致，否則 distro 起不來；同碟移動 vhdx 只改目錄項、瞬間完成。
- 透過 SSH 送 Windows PowerShell 指令有引號/編碼雷：用 base64 `-EncodedCommand`（先 `iconv -t UTF-16LE`），結果寫成 ASCII 檔再讀回，避免多層 shell 吃掉反斜線與中文亂碼。
