# 作業履歴

## 現在の状態 (毎セッション末尾に上書き更新 / 20行以内)

- **稼働中**:
  - Node0: Proxmox VE 9.2.2 (NVMe単体 ZFS rpool、**管理IP `192.168.20.150/24` VLAN20**、`node0.Ghome.local`)
  - ネットワーク: RTX830 + SWX2110P-8G 投入済み、WLX222 VAP1/VAP4 接続確認済み
  - **vmbr0 = VLANアウェアブリッジ化完了 (2026-07-23)**。ホスト管理をVLAN20へ移設・検証済み
  - vfio-pci バインド: `02:00.0`(USB3.1) / `02:00.1`(SATA `43c8` 全4ポート) → グループ14をVE2へ渡す準備完了・reboot後確認済み
- **確定した設計 (2026-07-23, 案4)**:
  - SATAは単一4ポートコントローラで分割不可 → `43c8` 全ポートをVE2(TrueNAS)へ。**6TB=データプール / SATA SSD=SSDプール**、**VMディスクは全てNVMe(rpool)集約**。vfio現設定のまま。HBA(案1)/ディスク単位PT(案2)/ホストZFS(案3)は却下 (詳細 03-proxmox.md)
  - NVMe 256GBは当面据置 (相場高騰中)。シンプロビジョニングで薄く運用、逼迫かつ相場緩和でM2_1を1TB換装
- **中途半端な状態**:
  - VE2 (TrueNAS SCALE) 未構築。ISO (25.10.4) は `local` に取得・照合済み
  - VLAN20/25 の観察・一時許可が継続中 (VLAN20→10 は pass-log、VLAN25 の80/NTPは一時許可)
- **次の一手 (最大3件)**:
  1. VE2 (TrueNAS SCALE, VMID=200) 構築 → q35/OVMF, boot 32GB(NVMe), RAM 8GB, `hostpci0: 0000:02:00,pcie=1`, net0 `tag=20` → 6TB/SSD認識確認 → disks.mdへ実シリアル反映
  2. VE1構築 (Frigate+Immich, GTX1650) → TrueNAS NFS連携
  3. ホストDNSを `192.168.20.254` に向いているか確認 (要 cat /etc/resolv.conf)
- **注意中の問題 (最大3件)**:
  1. **UPS未導入** — 本番投入前に必須 (7/20 実停電あり、正弦波必須)
  2. **PBSクォーラム** — 2ノードでQDevice未手当て
  3. **DNS向き先** — `.20.254`(RTX VLAN20側)であること。`.10.1`向けだと将来 `10212`→reject でDNS断

---

## 2026-07-23 (2) VLANアウェア化・ホストVLAN20移設

### やったこと
- **vmbr0をVLANアウェアブリッジ化し、ホスト管理IFをVLAN10(`192.168.10.150`)→VLAN20(`192.168.20.150`)へ移設** (レベルC)。物理コンソール確保のうえ、VLAN10退避路を残す2段階移行で実施→検証完了
- 検証: `vmbr0.20`=192.168.20.150/24, default via .254, `.254`/github.com ping 0% loss, MacBook(VLAN10)→GUI `https://192.168.20.150:8006` 到達, `/etc/hosts` も.20.150へ
- RTX830権威config(2026-07-23)を確認 → `192.168.10.x`・VLAN20 `.254`・`10110 pass` を実機で確認。**ネットワーク乖離issueクローズ** (リポジトリ `network/rtx830` の.11が旧)
- `configs/network/interfaces.node0` を新構成に更新。plan/03-proxmox に完了記録

### 決めたこと
- 案B (ホストもVLAN20へ) を採用。理由: サーバ類をVLAN20で明確に分離する構想
- 移行は2段階 (dual-home→検証→VLAN10除去)。却下: 一発移設は未実証のVLAN20へ飛ぶ博打のため

### 未解決・次回やること
- `/etc/resolv.conf` のDNSが `.20.254` を向いているか確認 (`.10.1`向けは将来断のリスク)
- VE2 (TrueNAS 200) 構築

### 実機の状態
- Node0管理: VLAN20 `192.168.20.150`。vmbr0=VLANアウェア稼働中
- 未構築: VE1〜VE6

## 2026-07-23 ストレージ方式(案4)確定

### やったこと
- ストレージ配置を実機事実に基づき再設計し、**案4に確定**。`plan/03-proxmox.md`・`01-hardware.md` を改訂、`docs/disks.md` を新規作成
- 発見: ホストからNVMeしか見えない → 6TB側SATA `43c8` をvfioに渡しており、**同じ`43c8`配下のSATA SSDも一緒に不可視**。当初「SSDはFCH `09:00.2`側でホスト温存」の前提が誤り (B450M Pro4はSATA単一4ポート、`7901`に物理コネクタ無し)

### 決めたこと (案4)
- **`43c8` 全4ポートを丸ごとVE2(TrueNAS)へパススルー** (現vfio設定のまま)。6TB=データプール、SATA SSD=SSDプール、**VMディスクは全てNVMe(rpool)集約**
- 却下: 案1 HBA (PCIE3占有→RTX3060締め出し+出費)、案2 ディスク単位PT (TrueNAS公式非推奨)、案3 ホストZFS+Cockpit (TrueNAS GUIを捨てる)
- NVMe容量: 相場高騰中につき**買わない**。シンプロビジョニングで256GB薄運用、実効70-75%超+相場緩和でM2_1を1TB換装 (M2_2はSATA専用で増設不可、換装が唯一の拡張路)

### 未解決・次回やること
- VE2(TrueNAS)構築 → PCIデバイスとしてグループ14追加 → 6TB/SSD認識 → disks.mdへ実シリアル反映
- TrueNAS boot diskはNVMe上に小容量 (16GB程度) で作成

### 実機の状態
- 稼働中: Node0 (Proxmox VE 9.2.2)、ネットワーク一式
- 未構築: VE1〜VE6 すべて。vfioバインド済みだがパススルー先VE2未作成
- 不可視: 6TB HDD・SATA SSD (共に`43c8`配下=vfio) → TrueNAS内で確認予定

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
