# 作業履歴

## 現在の状態 (毎セッション末尾に上書き更新 / 20行以内)

- **稼働中**:
  - Node0: Proxmox VE 9.2.2 (NVMe単体 ZFS rpool、管理IP `192.168.10.150/24`、`node0.ghome.local`)
  - ネットワーク: RTX830 + SWX2110P-8G 投入済み、WLX222 VAP1/VAP4 接続確認済み
  - vfio-pci バインド: `02:00.0`(USB3.1) / `02:00.1`(SATA 6TB側) → グループ14をVE2へ渡す準備完了・reboot後確認済み
- **中途半端な状態**:
  - VE2 (TrueNAS SCALE) 未構築。ISO (25.10.4) は `local` に取得・照合済み
  - VLAN20/25 の観察・一時許可が継続中 (VLAN20→10 は pass-log、VLAN25 の80/NTPは一時許可)
- **次の一手 (最大3件)**:
  1. `docs/disks.md` 作成 (全ディスクのシリアル/役割表 — VM作成前に確定)
  2. VE2 (TrueNAS SCALE) 構築 → 6TB HDD認識確認
  3. VLANアウェアブリッジ (vmbr) 設定
- **注意中の問題 (最大3件)**:
  1. **UPS未導入** — 本番投入前に必須 (7/20 実停電あり、正弦波必須)
  2. **PBSクォーラム** — 2ノードでQDevice未手当て
  3. **記録と実機の乖離(要確認)** — リポジトリの `network/rtx830/` は `192.168.11.0/24`・AP=WSR-3200AX4S・IPoE MAP-E だが、`plan/02-network.md` は `192.168.10.0/24`・AP=WLX222。どちらが現行か未確認

---

## 2026-07-22

### やったこと
- plan 一式 (CLAUDE.md / plan.md / plan/01〜06) と本 worklog を `g-hub` リポジトリに取り込み、構築の正本として運用する体制を確立
- `.gitignore` を追加 (WireGuard鍵・トークン・実データ系を除外)

### 決めたこと
- ドキュメントの正本を `g-hub` リポジトリ内に置く (アップロード参照ではなくGit管理下で回す)。
  - 却下案: 別リポジトリ (homelab専用) 分離 → 当面は既存 `g-hub` に集約する方が参照が一元化されると判断
- 作業分担: **実機操作はユーザー本人**が行い、Claude はインフラエンジニアとして手順設計・レビュー・破壊的操作の安全確認・ドキュメント整備を担当 (Claudeは実機コマンドを代行しない)

### 未解決・次回やること
- `docs/disks.md` 作成のため `lsblk -o NAME,SERIAL,MODEL,SIZE,TYPE,WWN` と `ls -l /dev/disk/by-id/` の出力待ち
- 上記「記録と実機の乖離」— `network/rtx830/` config と `plan/02-network.md` の突き合わせ。推測で直さずユーザー確認

### 実機の状態
- 稼働中: Node0 (Proxmox VE 9.2.2)、ネットワーク一式
- 停止中/未構築: VE1〜VE6 すべて未構築
- 中途半端: vfio-pciバインドは投入済みだがパススルー先のVE2が未作成
