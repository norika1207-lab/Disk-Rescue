# Disk Rescue — 外接碟資料救援與重組實戰紀錄

這個 repo 記錄一次橫跨多顆外接碟的資料救援與重組工程：從一顆 APFS 檔案系統損壞、Finder 讀不到的 SSD，到一次誤下格式化指令毀掉另一顆 4TB 工作碟，再到把全部資料救回、驗證、重新打包、清理、歸位的完整過程。

重點不在「用了什麼付費軟體」，而在「不付費、唯讀、可中斷、可續傳、每一步都拿得出證據」的方法論。所有技巧都在 macOS / Linux 命令列可重現。

## 目錄

- [01 災難時間線](docs/01-disaster-timeline.md) — 兩起事故怎麼發生、各階段怎麼收拾
- [02 APFS 壞碟唯讀救援](docs/02-apfs-rescue.md) — spaceman 損壞繞過、apfs-fuse 自寫 extractor、ddrescue 完整映像
- [03 誤格式化後 block 直讀救援](docs/03-block-direct-read.md) — 用檔案掃描器的 block map 從原始磁區重建檔案
- [04 打包與驗證管線](docs/04-pack-and-verify.md) — tar 排垃圾打包、逐檔 byte/checksum 驗證、中文檔名比對陷阱
- [05 三碟重組](docs/05-three-disk-reorg.md) — 格式化、rsync 搬移、解 tar、清 AppleDouble、資料夾歸位
- [06 工具與技巧總表](docs/06-tools-and-techniques.md)
- [08 多重異地備份](docs/08-multi-site-backup.md) — 唯一副本做成多份、跨機/跨內網跳板、每份驗證、長傳輸監控
- [07 工程教訓](docs/07-lessons.md) — exFAT 特性、認碟鐵則、監控與判斷紀律

## 一句話總結

硬碟故障與人為誤操作都可救，前提是：先觀測再動手、只做唯讀與可中斷的操作、每一步都用實測數據驗證，不靠印象、不憑猜測。

---

作者：織田紀香（Norika Oda）· https://norika.tw
