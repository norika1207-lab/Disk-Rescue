# 04 打包與驗證管線

## 為什麼救回來要立刻打包成 tar

救回的檔案樹通常有海量小檔（node_modules、.git、快取、依賴）。海量小檔在 exFAT 上寫入極慢（每個小檔都要更新配置表），而且 exFAT 的配置單元大（常見 256KB），幾 KB 的小檔也吃掉一整個單元，空間爆增。

解法：分目錄打包成 tar，排掉可重建的垃圾，只留真正的原創內容。

```
EXC="--exclude=node_modules --exclude=.git --exclude=gitpacks \
     --exclude=.venv --exclude=venv --exclude=site-packages \
     --exclude=__pycache__ --exclude=Pods --exclude=DerivedData --exclude=build"
tar cf - $EXC "專案目錄" | (cd /目的地 && tar xf -)
```

## 驗證：不要相信「指令沒報錯」

打包和搬移的每一步都要用實測數據驗，不能看 exit code 0 就當成功。

- **tar 結構完整**：`tar -tf x.tar | wc -l` 逐條讀出，確認沒截斷。
- **逐檔 byte 比對**：搬移後比對來源與目的的檔案 byte 數（`stat -f%z`），一個都不能差。
- **內容完整**：抽樣算 SHA256 對比；壓縮檔用 `gzip -t`（完整 CRC 檢查，不只看 header）、`tar -tf`、zstd 串流解。

## 中文檔名的比對陷阱（重要）

`tar -tf` 輸出非 ASCII 檔名時會轉成八進制轉義（例如 `\351\201\270`），而 `find` 輸出的是 UTF-8。直接用 `comm` / `diff` 比對兩者會出現大量「假不符」，讓你以為掉了檔。

正解：用 Python 的 `tarfile` 模組讀 tar 內容、用 `os.walk` 讀磁碟，兩邊都做 Unicode NFC 正規化再比對：

```python
import tarfile, os, unicodedata
def norm(s): return unicodedata.normalize("NFC", s)
tar_set  = {norm(m.rstrip("/")) for m in tarfile.open(p).getnames()}
disk_set = {norm(os.path.relpath(...)) for ... in os.walk(target)}
only_in_tar  = tar_set  - disk_set
only_in_disk = disk_set - tar_set
```

實戰中一度因為 `tar -tf` 八進制轉義，誤報「46 個檔漏了」，改用 Python tarfile 後證明零真漏。
