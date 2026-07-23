# ディスク台帳 — シリアルと役割の対応表

> **破壊的操作 (pool作成 / wipe / mkfs / zpool destroy 等) の前に必ずこの表で対象を照合する。**
> デバイスパス (`/dev/sdX`) は再起動で変わりうる。破壊的操作では**シリアル**で対象を特定すること (CLAUDE.md §2-4)。

最終更新: 2026-07-23

---

## 現状の可視性メモ

- SATAコントローラ `43c8` (全4ポート) を **vfio-pci でVE2 (TrueNAS) へパススルー中**のため、**そこに繋がる 6TB HDD と SATA SSD はProxmoxホストからは見えない**。
- よってこの2本のシリアルは **TrueNAS VM 内で `Storage → Disks` または `lsblk -o NAME,SERIAL,MODEL` から確定させる**。下表の「(要TrueNAS内確認)」は構築時に実測値で上書きする。
- ホストから確定できるのは NVMe (rpool) のみ。

---

## 台帳

| 役割 | 種別/容量 | シリアル | モデル | by-id (host) | 接続 | 扱い |
|---|---|---|---|---|---|---|
| Proxmox OS (rpool) | NVMe 256GB | `PHHH92530115256B` | INTEL SSDPEKKW256G8 | `nvme-INTEL_SSDPEKKW256G8_PHHH92530115256B` | M2_1 (CPU直結) | **ホスト専用。全VM/LXCのブート・アプリも同居。絶対にwipe/destroy禁止** |
| 録画・写真の実体データ | HDD 6TB | `WD-WX42D369CEFE` (要TrueNAS内確認) | (要確認) | — (vfio配下で不可視) | SATA3 (`43c8`) | VE2パススルー。TrueNASデータプール |
| アプリ/高速データ | SATA SSD 256GB | `154778407406` (要TrueNAS内確認) | (要確認) | — (vfio配下で不可視) | SATA3 (`43c8`) | VE2パススルー。TrueNAS SSDプール |

---

## TrueNAS内でのプール作成時の取り違え防止 (最重要)

TrueNASで pool を作る際、**6TB HDD と 256GB SSD を容量とシリアルの両方で必ず確認**してから対象を選ぶこと。

- データプール (録画/写真) = **6TB** 側 (`WD-...`)
- SSDプール = **256GB** 側 (`154778407406`)
- 容量が一桁違う (6TB vs 256GB) ので取り違えは起きにくいが、**シリアル照合を省略しない**。

> TrueNAS構築後、両ディスクの正式なシリアル・モデルをこの表に反映し「(要TrueNAS内確認)」を消すこと。
