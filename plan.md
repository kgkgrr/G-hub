# ホームラボ構築計画 (Proxmox VE) — インデックス

最終更新: 2026-08-06

> **このファイルは索引と現況のみ。** 詳細は `plan/` 以下の各分冊を参照。
> セッション開始時はこのファイルを読み、必要な分冊だけを追加で開くこと。

---

## 目的

- iCloud代替の自己ホスト写真管理 (Immich)
- 自宅監視カメラ録画 (Frigate)
- ローカルAIによるファイル/書類整理 (Ollama)
- ネットワークセグメンテーション (VLAN)
- 自律AIエージェントのサンドボックス実行環境 (Hermes)

**通底する方針**: 低メンテナンス性を優先し、アーキテクチャは実測・検証を経てから確定させる (先走った投資をしない)。

---

## 分冊一覧

| 分冊 | 内容 | 状態 | 参照頻度 |
|---|---|---|---|
| [`plan/01-hardware.md`](plan/01-hardware.md) | Node0/Node2のハード構成、ストレージ、電源、UPS検討 | 確定 (メモリDDR4-2400確定) | 中 |
| [`plan/02-network.md`](plan/02-network.md) | VLAN設計、RTX830/スイッチ/AP設定、VLAN間フィルター | **完了** | 低 (アーカイブ的) |
| [`plan/03-proxmox.md`](plan/03-proxmox.md) | VM構成、IOMMU/パススルー、ストレージ設計、外部公開 | **進行中** | **高** |
| [`plan/04-gpu-ai.md`](plan/04-gpu-ai.md) | GPU配分方針、Ollama/モデル選定、Coral TPU | 一部保留 | 中 |
| [`plan/05-backlog.md`](plan/05-backlog.md) | 保留・将来検討事項 | — | 低 |
| [`plan/06-principles.md`](plan/06-principles.md) | 学び・原則、用語メモ | 継続追記 | 中 |

**運用ルール**: 分冊が肥大化したらさらに分割する。この索引ファイルは常に1画面に収まる長さを保つ。

---

## 現在の進捗 (2026-08-04)

### 完了

- **ネットワークフェーズ**: VLAN設計確定、RTX830 + SWX2110P-8G への設定投入・実機config確認済み。WLX222のVAP1(メイン)/VAP4(IoT)接続確認済み。**vmbr0のVLANアウェア化・ホスト管理のVLAN20移設も完了・検証済み**
- **VLAN間フィルター**: 投入済み。VLAN20→VLAN10は`pass-log`で観察中、VLAN25のポート80/NTPは一時許可中
- Node0のPCIE1に2.5G NIC装着済み。電源をCorsair CX450 (450W / 80+ Bronze) と確認
- **Proxmox VE 9.2.2 インストール完了** (NVMe単体 ZFS rpool、管理IP `192.168.20.150/24`(VLAN20)、hostname `node0.Ghome.local`)。管理NICはマザーオンボード側 (nic1)。nic0はNode2直結用に温存
- **IOMMU実測完了・ストレージパススルー方式を確定**: 当初のコントローラ単位パススルー(案4)は事故で撤回、**ディスク単位パススルー(案2)**へ移行。6TB HDDをVE2(TrueNAS)へ、SATA SSDはホスト側LVM-thin化 ([`03-proxmox.md`](03-proxmox.md))
- **メモリ安定性問題を解決**: 4枚48GB構成で DDR4-2933 が不安定 → **DDR4-2400 固定** で約11時間・4周・Errors:0を確認 ([`01-hardware.md`](01-hardware.md))
- **VE2 (TrueNAS SCALE 25.10.4) 構築完了**: 管理IP `192.168.20.151`、hostname `Gnas`。6TBプール`tank`作成、データセット`pic_tank`/`cam_tank`(NFS)・`doc_tank`(SMB)、共有・ACL設定まで完了
- **SATA SSD (240GB) のLVM-thin化完了**: Proxmoxストレージ`ssd-thin`(235.12GB)として登録
- **🎉 VE1 (Immich) 構築完了・動作確認済み**: VMID=100, hostname `Ghome`, IP `192.168.20.160`。GTX1650をIOMMUグループ15単独で確認・パススルー、Debian 13、Docker+nvidia-container-toolkit、NFS(`tank/pic_tank`)・SSD(`ssd-thin`)連携、Immich起動・管理者アカウント作成・**初回写真アップロードでNFS/Postgres書き込みをエンドツーエンドで確認済み** ([`docs/ve1-immich-build.md`](docs/ve1-immich-build.md))

### 次のステップ (この順序を守る)

1. NFS許可アドレスをVE1確定IP(`192.168.20.160`)へ絞り込み、VE1再起動後の自動復旧確認
2. VE1へFrigateを統合 (カメラ機材の準備待ちのため保留中、着手時期未定)
3. Immichの外部公開 (Cloudflare Tunnel + Access、専用LXC)
4. VE3 (Windows 11) / VE4 (Pi-hole LXC) / VE5 (開発用Linux) / VE6 (Hermesサンドボックス) の構築

### 未着手・保留

- ゲスト用VAP、VAP2/VAP3 (サーバー管理/Hermes用無線)
- Node0↔Node2 の 2.5G NIC直結バックアップリンク
- **UPS選定 (優先度↑)** — 構築中に実際の停電を経験。中古市場を継続ウォッチ
- 10Gスイッチ (中古の出物待ち)
- PBSクォーラム対応 (QDevice未手当て)

---

## 解決済みの記載修正

- **2026-07-21**: VLAN10のサブネットを `192.168.11.0/24` と誤記していた箇所を `192.168.10.0/24` (GW `192.168.10.1`) に修正。フィルタールール群は当初から正しく、実機動作への影響なし。転記誤りのみ
- **2026-07-21**: DHCPスコープの範囲を修正。**全VLAN共通で `.2`〜`.100` (99アドレス)**。`.101` 以降は静的IP・MACバインド用の予約帯
