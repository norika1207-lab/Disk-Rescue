# 03 誤格式化後的 block 直讀救援

## 情境

一顆碟被快速格式化（quick format）。快速格式化只重寫檔案系統的目錄結構，實際資料區塊大多還在原地沒被覆蓋。只要能拿到「哪個檔案的資料在哪些磁區」的對照表，就能從原始磁區把檔案一個個重建出來。

## 拿到 block map

用檔案掃描/救援軟體對這顆碟做一次深度掃描（掃描本身唯讀），它會建出一份資料庫，記錄每個檔案 → 起始磁區 → 大小。這份掃描資料通常是一個 SQLite 檔，裡面有三張關鍵表：

- `files`：`_id, name, path(→paths.index), size, block, hasFilePlaces, deletedState`
- `paths`：`index, path`
- `filePlaces`：`file(→files._id), block, count`

（不同軟體欄位名不同，原理一樣：檔案對應到磁區清單。）

## 從磁區重建檔案

核心邏輯（Python，唯讀開 `/dev/rdiskN`）：

- `block` 是 512-byte 磁區編號。
- 若 `hasFilePlaces=0` → 檔案連續：從 `block*512` 讀 `size` bytes。
- 若 `hasFilePlaces=1` → 檔案分散：照 `filePlaces` 的每個 (block, count) 區段（count 單位是 512-byte 磁區），依序讀出、拼接，最後截斷到 `size`。

```python
import os
dev = os.open("/dev/rdisk7", os.O_RDONLY)   # 唯讀原始裝置
# 對每個檔案：
os.lseek(dev, block * 512, 0)
data = os.read(dev, count * 512)            # 大檔要分段 64MB 讀
# 分散檔：依 _id 順序把各段接起來，最後 truncate 到 size
```

## 三個關鍵防呆

1. **只讀對的碟**：動手前確認目標裝置真的是那顆被格式化的碟（比對容量、讀開頭 magic），絕不寫入。
2. **驗證重建結果**：每個檔案重建後驗 magic（gzip `1f8b`、SQLite header、tar…），不只看大小。
3. **斷點續傳**：已存在且大小相符就跳過，讓過程可中斷。

## 為什麼有些檔救不回

`deletedState` 標記現存檔 vs 已刪片段；連續小於一個配置單元的極小檔可能 `block=NULL`（沒有位置資訊）救不回。實戰中失敗的多是幾百 KB 以下的中繼小檔，大檔幾乎全數可救。

## raw device 讀取注意

raw 裝置讀取必須 512-byte 對齊，否則 `Invalid argument`（errno 22）。讀的 offset 與長度都對齊到 512 的倍數。
