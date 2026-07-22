# 03. Proxmox / 仮想化構成 — 【進行中】

← [インデックスに戻る](../plan.md)

> **現在アクティブな分冊。** セッションではまずここを読む。

---

## VM / コンテナ構成

| VM | 内容 | 状態 |
|---|---|---|
| VE1 | Frigate + Immich、GTX1650 GPUパススルー | 未着手 |
| VE2 | **TrueNAS SCALE VM に確定** (下記参照)。6TB HDDをコントローラ単位でパススルー | 構築中 |
| VE3 | Windows 11 (TPM仮想化必要) | 未着手 |
| VE4 | Pi-hole + SYSLOG (LXC, 特権)。将来Avahi(mDNSリフレクター)も同居 | 未着手 |
| VE5 | 開発用Linux | 未着手 |
| VE6 | Hermes Agent サンドボックス (2 vCPU / 2〜4GB RAM / VLAN25) | 未着手 |

---

## ストレージ層 (VE2) の方式 — TrueNAS SCALE VM に確定 (2026-07-21)

**IOMMUグループ実測の結果、6TB HDDのSATAコントローラ (`02:00.1`) がUSB 3.1コントローラ (`02:00.0`) と同一グループ14に同居しており、ディスク単位で分離できない。よってコントローラ単位パススルー + TrueNAS SCALE VM を採用する。**

### 経緯
当初は「ディスク単位で済むなら Proxmoxネイティブ NFS (ホストZFS + 特権LXC) を優先、コントローラ単位が必要なら TrueNAS VMへ回帰」という条件分岐で保留していた ([`06-principles.md`](06-principles.md))。今回まさに後者の条件が成立した。

なお、RAMには余裕がある (48GB) ため、TrueNAS VMのオーバーヘッドは許容範囲。TrueNASの管理GUI・SMART監視・スナップショット管理を活かせる利点もあり、「消極的な妥協」ではなく積極的な採用と位置づける。

### パススルー構成 (確定)

| デバイス | ID | IOMMUグループ | 扱い |
|---|---|---|---|
| USB 3.1 xHCI | `1022:43d5` (`02:00.0`) | 14 | VE2へパススルー (グループ道連れ) |
| SATA (6TB HDD側) | `1022:43c8` (`02:00.1`) | 14 | VE2へパススルー (本命) |
| FCH SATA (CPU側) | `1022:7901` (`09:00.2`) | 20 | **ホスト側に温存** (`ahci`) |
| USB (CPU側, KB/マウス) | `1022:145f` (`08:00.3`) | 18 | **ホスト側に温存** |

- グループ14を渡すと、そこに同居するUSB 3.1ポート群もホストから失われる。ただしマウス・キーボードはグループ18 (`08:00.3`) 側に接続されているため、ホスト操作に支障はない
- vfio-pciバインド設定は投入・再起動後の確認済み (下記「IOMMU / パススルー」参照)

---

## IOMMU / パススルー

### 進捗 (2026-07-21)

- [x] BIOS設定 (IOMMU有効化 / Secure Boot無効 / CSM無効) 済み。AMD環境では `amd_iommu=on` を明示せずともBIOS有効ならIOMMUグループは生成された (22グループ 0〜21を確認)
- [x] Proxmox VE 9.2.2 インストール済み (NVMe単体 ZFS rpool)
- [x] IOMMUグループ確認済み → 6TB HDDのSATAコントローラはグループ14 (USB 3.1と同居) と判明 → **コントローラ単位パススルーに決定**
- [x] vfio-pciバインド設定 投入・確認済み (下記)
- [ ] `docs/disks.md` / `docs/iommu-groups.md` への正式記録 (未。TrueNAS構築前後で作成する)
- [ ] VE2 (TrueNAS SCALE) 構築 → 6TB HDD認識確認
- [ ] VLANアウェアブリッジ設定 → VE1構築 → NFS連携

### 投入済みのカーネル/モジュール設定

**`/etc/kernel/cmdline`** (systemd-boot、`proxmox-boot-tool refresh` で反映):
```
root=ZFS=rpool/ROOT/pve-1 boot=zfs amd_iommu=on iommu=pt
```

**`/etc/modprobe.d/vfio.conf`**:
```
options vfio-pci ids=1022:43d5,1022:43c8
softdep ahci pre: vfio-pci
softdep xhci_pci pre: vfio-pci
```
- `softdep` 行が**必須だった**。これがないと、初期化の早い `ahci`/`xhci_hcd` が先にデバイスを掴んでしまい、後からロードされる `vfio-pci` が奪い返せず、`Kernel driver in use` が `vfio-pci` にならない。詳細は [`06-principles.md`](06-principles.md)

**`/etc/modules`** (末尾に追記):
```
vfio
vfio_iommu_type1
vfio_pci
```

反映は `update-initramfs -u -k all && proxmox-boot-tool refresh` → reboot。再起動後、`lspci -nnk -s 02:00.0` / `02:00.1` が `vfio-pci`、`09:00.2` が `ahci` であることを確認済み (BIOSメモリ再設定を挟んだ後も維持を再確認済み)。

### 残存リスク・今後の注意

- `pcie_acs_override=downstream` は**今回は不要だった** (グループ14がそのまま渡せる粒度だったため)。将来他デバイスで粒度問題が出た場合のみ検討。分離保証を無効化する回避策なので安易に使わない (CLAUDE.md レベルC)
- GTX1650をVE1にパススルーする際は、別途ホスト側でGPUドライバ (nouveau等) のブラックリスト化が必要 (未実施)

---

## ネットワーク (Proxmox側)

**Node0はRTX830に直結**しているため、Proxmox側でVLANタグを自分で付ける必要がある。

- `vmbr0` を **VLANアウェアブリッジ**として構成
- 管理IF (VLAN20) と VE6 (VLAN25) のタグを Node0 自身が付与する
- **この設定を壊すと管理アクセスが全滅する。** 変更は物理コンソールを確保できるタイミングでのみ実施 (CLAUDE.md レベルC)
- IPは静的設定 (方式①)。詳細は [`02-network.md`](02-network.md)

---

## バックアップ設計

| Tier | 内容 |
|---|---|
| Tier 1 | ZFSスナップショット (`zfs-auto-snapshot` または cron) |
| Tier 2 | (検討中) |
| Tier 3 | Node2 (PBS) → 6TB USB HDD |

### PBSクラスタのクォーラム問題
2ノード構成ではクォーラムが成立しない。第三の投票者として軽量なQDevice (既存ノード上のVM、またはRaspberry Pi) を用意する必要がある。**未着手**。

---

## 外部公開戦略

| 対象 | 方式 |
|---|---|
| Immich | **Cloudflare Tunnel + Cloudflare Access**。専用LXCで動かし、VE1本体からは分離する |
| Proxmox / PBS 管理画面 | **WireGuard VPN (RTX830) 経由のみ。インターネットへ絶対に公開しない** |

---

## Hermes Agent サンドボックス (VE6)

- スペック: 2 vCPU / 2〜4GB RAM
- ネットワーク: VLAN25 (静的IP)。**アウトバウンドHTTPS + DNS(RTX宛)のみ許可**
- **未決定**: 使用プラットフォーム (Telegram / Slack 等)。決定後、そのエンドポイントをファイアウォールで許可リスト化する
- VE6構築完了後、VLAN25の一時許可 (ポート80 / NTP) を削除する ([`02-network.md`](02-network.md) 参照)
