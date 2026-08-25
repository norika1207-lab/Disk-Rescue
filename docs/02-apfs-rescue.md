# 02 APFS 壞碟唯讀救援

## 問題

APFS container 的 spaceman 損壞，macOS 掛載失敗、Finder 讀不到。付費救援軟體要價不低，且目標是全程唯讀、不花錢。

## 走過的死路（記錄下來避免重蹈）

- **TSK（The Sleuth Kit）**：`fls -P apfs -B <block>` 可以列目錄，但 `icat` / `tsk_recover` 讀內容會撞 spaceman 崩潰（Abort trap 6）。
- **apfs-fuse 直接 mount**：讀到損壞的 btree 節點會 segfault / 死迴圈。

## 有效解法：自寫 extractor 走 apfs-fuse 的 library

clone sgan81/apfs-fuse，但不用它的 mount，而是直接 link 它的 ApfsLib，自寫一個遞迴 extractor：

- 用 ApfsLib 的 `ListDirectory` / `GetInode` / `ReadFile` / `GetAttribute` 遞迴唯讀 extract。
- 每個檔案 try/catch 包起來，讀到壞檔就跳過、不整個崩潰。
- 支援續傳：目標已存在且大小相符就跳過。
- 處理壓縮檔（DecompressFile）與 symlink。

關鍵細節：
- `ReadFile` 要用 record 的 `private_id`，不是 inode number。
- 常數：`APFS_UF_COMPRESSED=0x20`、`INODE_HAS_UNCOMPRESSED_SIZE=0x00040000`、`DREC_TYPE_MASK=0x000F`、`DT_DIR=4 / DT_REG=8 / DT_LNK=10`、`ROOT_DIR_INO_NUM=2`。
- cmake 4.x 拒絕舊的 VERSION 宣告，加 `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`。

## 更穩的路：先做完整 raw 映像，再從映像 extract

與其反覆對著半死的實體碟讀（每次壞區都可能卡住），不如先用 ddrescue 把整顆碟做成一個完整的 bit-level 映像，之後所有 extract 都對映像做：

```
ddrescue -n /dev/sdX disk.img mapfile      # 第一遍：no-scrape 快速過，壞區標記跳過
ddrescue -d /dev/sdX disk.img mapfile      # 後續：只補未讀成功的區
```

`-n`（no-scrape）先快速把好區全撈下來，壞區在 mapfile 標記，之後再針對壞區細修。mapfile 讓整個過程可中斷、可續傳。

macOS 對 USB 碟的 I/O timeout 不可調，遇到壞磁區會進 kernel 長時間重試 → 進程卡在 U 狀態（uninterruptible sleep）連 kill -9 都殺不掉，只能拔 USB。Linux 可調 `/sys/block/sdX/device/timeout` 且能 raw 讀 `/dev/sdX` 不觸發 APFS 掛載，所以把碟帶到 Linux 跑 ddrescue 明顯順很多。實測 Linux 上一遍過、0 bad sectors。

## 判斷紀律

過程中一度把「讀得慢/偶爾卡」誤判成「碟物理半死」，一路加嚴 watchdog、把探針 timeout 設到 4 秒，結果把好區也誤標成壞。後來的事實（ddrescue 82MB/s、0 錯誤；rescue probe 直讀就救回 91GB）證明碟是健康的。教訓：**別用過嚴的 timeout 把「慢」誤判成「死」。**
