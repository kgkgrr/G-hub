# ディスク台帳 — シリアルと役割の対応表

> **破壊的操作 (pool作成 / wipe / mkfs / zpool destroy 等) の前に必ずこの表で対象を照合する。**
> デバイスパス (`/dev/sdX`) は再起動で変わりうる。破壊的操作では**シリアル / by-id** で対象を特定すること (CLAUDE.md §2-4)。

最終更新: 2026-07-24 (全ディスク実シリアル確定)

---

## 構成方針 (案2)

案4(コントローラ単位PT)は不可能と判明 (`iommu-groups.md`)。**案2=ディスク単位パススルー**を採用:
- **6TB** → ホストが保持しつつ `qm set 200 -scsi1 /dev/disk/by-id/...` で VE2(TrueNAS) へ渡す
- **SATA SSD** → **ホスト側 Proxmox ストレージ**として使用 (VE1のVMディスク等)。TrueNASには渡さない
- **NVMe** → OS(rpool) + 全VMブート

---

## 台帳 (実測確定値)

| 役割 | 種別/容量 | シリアル | モデル | by-id | 接続/扱い |
|---|---|---|---|---|---|
| Proxmox OS + 全VMブート | NVMe 238.5G | `PHHH92530115256B` | INTEL SSDPEKKW256G8 | `nvme-INTEL_SSDPEKKW256G8_PHHH92530115256B` | M2_1。**rpool。wipe/destroy厳禁** |
| 録画・写真の実体データ | HDD 5.5T (6TB) | `WD-WX42D369CEFE` | WDC WD60EFPX-68C5ZN0 | `ata-WDC_WD60EFPX-68C5ZN0_WD-WX42D369CEFE` (wwn `0x50014ee2c1f2e586`) | SATA。ホストでは `sdb`。**VE2へscsi1でディスクPT**。VE2内では `sda`(5.46TiB)。TrueNASデータプール |
| VE1 VMディスク等 (ホスト) | SATA SSD 223.6G (240GB) | `154778407406` | SanDisk SDSSDA240G | `ata-SanDisk_SDSSDA240G_154778407406` (wwn `0x5001b44f15180dee`) | SATA。ホストでは `sda`。**ホスト側ストレージ用**(要wipe→ZFS/LVM化。フォーマット許可済み) |

---

## 取り違え防止メモ (最重要)

- **ホスト視点**: `sda`=240GB SSD(ホスト用) / `sdb`=6TB(VE2へPT) / `nvme0n1`=OS。**名前が紛らわしいので必ずシリアルで確認**
- **VE2(TrueNAS)視点**: 渡した6TBは **`sda`=5.46TiB** として見える (32GBブート=`sdb`)。**プール作成は5.46TiBの方**
- SSD(`sda`, `154778407406`)を**ホストでZFS/LVM化する時**は、6TB(`sdb`)と絶対に取り違えない。容量(240GB vs 6TB)＋シリアルの二重確認
