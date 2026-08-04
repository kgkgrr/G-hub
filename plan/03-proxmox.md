# 03. Proxmox / 仮想化構成 — 【進行中】

← [インデックスに戻る](../plan.md)

> **現在アクティブな分冊。** セッションではまずここを読む。

---

## VM / コンテナ構成

| VM | 内容 | 状態 |
|---|---|---|
| VE1 | Frigate + Immich、GTX1650 GPUパススルー (VMID=100, hostname=`Ghome`, IP=`192.168.20.160`) | 構築中: VM作成・GPU passthrough・Debian 13インストール完了、SSH到達確認済み。Docker/NFS/Immich未 (手順: [`docs/ve1-immich-build.md`](../docs/ve1-immich-build.md)) |
| VE2 | **TrueNAS SCALE VM** (VMID=200)。6TB HDDを**ディスク単位パススルー**(案4撤回→案2)。プール`tank`+データセット3つ(`pic_tank`/`cam_tank`=NFS,`doc_tank`=SMB)・共有設定まで完了 | **ストレージ層完成**(NFS許可先はVE1確定IPへ絞り込み待ち) |
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
| FCH SATA (CPU側) | `1022:7901` (`09:00.2`) | 20 | `ahci` のままだが**物理コネクタ未配線** (1ポート幽霊・link down)。ホスト用SATAとしては使えない |
| USB (CPU側, KB/マウス) | `1022:145f` (`08:00.3`) | 18 | **ホスト側に温存** |

- グループ14を渡すと、そこに同居するUSB 3.1ポート群もホストから失われる。ただしマウス・キーボードはグループ18 (`08:00.3`) 側に接続されているため、ホスト操作に支障はない
- vfio-pciバインド設定は投入・再起動後の確認済み (下記「IOMMU / パススルー」参照)

### ⚠️ 案4は撤回 (2026-07-23夜) → 案2 (ディスク単位パススルー) へ

**下記「案4(コントローラ単位PT)」は実機で不可能と判明し撤回した。** IOMMUグループ14の実態が記録と異なり、チップセットPCIeスイッチ+配下の両NICを含んでいた (`docs/iommu-groups.md`)。VE2へ渡してリセットした瞬間に管理NICごと落ちてホストがハングした (事故: `docs/worklog.md` 2026-07-23(3))。
**採用: 案2** = vfioを使わずホストが6TB/SSDを直接持ち、`qm set 200 -scsiX /dev/disk/by-id/ata-...` でVE2(TrueNAS)へディスク単位パススルー。TrueNAS GUI維持・GPUのPCIE3温存。SSD/NVMe集約の考え方は案4から引き継ぐ。(本セクションは案2動作確認後に正式改訂する)

> **⚠️ 注意 (2026-07-24 実測)**: `qm set -scsiX` でのディスク割り当ては**必ず `serial=` を明示指定すること**。指定しないとゲスト側でシリアルが空(`None`)になり、TrueNAS等シリアル重複チェックを行うストレージOSでプール作成時に `topology: Disks have duplicate serial numbers` エラーになる。VE2では `scsi0`(ローカルzvolブート)に `serial=TN200BOOT`、`scsi1`(6TB by-id)に `serial=WD-WX42D369CEFE`(実シリアル)を付与して解決した。VE1等で同様のディスク追加を行う際も同じ手順を踏む (詳細: `docs/worklog.md` 2026-07-24(4))

### ストレージ配置の改訂 — SSDもVE2へ、VMディスクはNVMe集約 (2026-07-23) 【※案4=撤回済み・下記は経緯記録】

**当初「SATA SSDはホスト側 (`09:00.2`) に温存しVE1のVMディスクにする」と想定していたが、実機で成立しないことが判明したため配置を改訂した。**

- **実機の事実**: B450M Pro4のSATAは**単一コントローラ4ポート (`43c8`)** で、物理SATAコネクタは全てこの `43c8` に配線されている。`09:00.2` (`7901`) には物理コネクタが無い。つまり `43c8` をパススルーするとホストが使えるSATAポートは**ゼロ**になり、SSDをホスト側に確保できない (SSD/6TBの両方が `43c8` 配下=パススルー側)。
- **決定 (案4)**: SSDをホストに引き剥がすのを諦め、**`43c8` 全4ポートを丸ごとVE2 (TrueNAS) へパススルー**する。これは**現行のvfio設定そのまま**であり再設定不要。
  - **6TB HDD** → TrueNASのデータプール (録画・写真)
  - **SATA SSD 256GB** → TrueNAS配下の**SSDプール** (アプリ/高速データセット用。死蔵しない)
  - **全VM/LXCのブート・アプリディスク** → **NVMe (rpool) に集約**
- **却下した代替案**:
  - *案1 HBA増設*: TrueNASは正攻法になるが、HBAが**PCIE3を占有しRTX3060 (ローカルLLM) を締め出す** + 出費。「先走った投資をしない」方針に反する
  - *案2 ディスク単位パススルー*: TrueNAS公式非推奨 (SMART/ZFS直アクセス制限)。案4なら正攻法のコントローラPTが取れるので不採用
  - *案3 ホストZFS + Cockpit*: 低メンテだが**TrueNAS GUIを捨てる**。GUIを残したい要件で不採用
- **案4の利点**: 正攻法のコントローラ単位PT (=実HBA相当の直アクセス・SMART/ZFS完全) を**HBAを買わずに**実現。TrueNAS GUI維持・PCIE3温存 (RTX3060の道)・追加投資ゼロ。
- **代償と対策**: VMローカルストレージがNVMe 256GB 1本に集中する。**ZFSシンプロビジョニング**前提なら実使用は当面100〜110GB程度で収まる見込み (Win11実体〜40GB等)。膨張しやすいImmich派生データはTrueNASのSSDプールへNFSで逃がせる (**ブート不可・データのみ**。TrueNAS自身がVMのため循環依存を避ける)。逼迫 (実効70〜75%超) かつ**NVMe相場が緩んだ**タイミングでM2_1を1TB級へ換装する (M2_2はSATA M.2専用かつSATA3_3と排他のため増設に使えず、換装が唯一の拡張路)。

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

### 完了 (2026-07-23)

VLANアウェア化＋ホスト管理のVLAN20移設を実施・検証済み。物理コンソール確保のうえ、VLAN10退避路を残す2段階移行で無停止に近い形で完了。

- `vmbr0` = VLANアウェアブリッジ (`bridge-vlan-aware yes` / `bridge-vids 2-4094`、ポート `nic1`)
- ホスト管理IF = `vmbr0.20` = **`192.168.20.150/24`、GW `192.168.20.254`** (旧 `192.168.10.150` から移設)
- `/etc/hosts` も `.20.150` へ更新済み。MacBook(VLAN10)→GUI `https://192.168.20.150:8006` 到達確認 (RTX `10110 pass` + 戻り `10212`/dynamic)
- 実機の権威RTX config (2026-07-23) は `192.168.10.x`・VLAN20=`.254` で plan/02-network と一致。リポジトリ旧 `network/rtx830`(192.168.11) が古いことが確定 (乖離issueクローズ)
- 現行 interfaces は `configs/network/interfaces.node0` に保存
- VMをVLANに載せる時は net デバイスに `tag=<vid>` を付ける (例: TrueNAS(VE2)=`tag=20`)
- 注意: ホストDNSは `192.168.20.254` (RTXのVLAN20側) を向けること。`.10.1` 向けだと将来 `10212`→reject 時にDNS断

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

## データセット設計 (VE2 TrueNAS) と Immich/Postgres配置 (2026-07-24)

### 前提
- 家族4人想定。当面はスマホを持つ夫婦2人のみ運用。子供がスマホを持つ時期に合わせて拡張する
- Immichは写真+動画を扱う
- Immich以外の用途: 家計簿等の一般ドキュメント、監視カメラ(Frigate)のイベント検知録画(当初1台→将来3台)

### TrueNAS (6TBプール) 側データセット — プール名・データセット名確定 (2026-07-24)

**プール名: `tank`**(単騎5.46TiB, Stripe, 暗号化なし=ローカル運用のため不要と判断)

| データセット | 用途 | recordsize | compression | snapshot | 共有方式 |
|---|---|---|---|---|---|
| `tank/pic_tank` | Immichの写真+動画の実体ファイル (`UPLOAD_LOCATION`) | 1M | lz4 | 日次〜週次・少世代 (誤削除復旧が目的) | NFS→VE1 |
| `tank/cam_tank` | Frigateのイベント検知録画クリップ | 1M | lz4 | 不要〜最小限 (Frigate自身が保持期間を管理) | NFS→VE1 |
| `tank/doc_tank` | 家計簿等の一般ドキュメント。当面は夫婦共有の単一データセット。子供が端末を持つ時期に `tank/doc_tank/<name>` を子データセットとして追加する拡張パスを想定 | デフォルト(128K) | lz4 | 日次・多世代 (復旧価値が最も高い) | SMB |

### 家族共有はデータセット分割ではなくアプリ機能で実現 (決定)
- Immichの写真・動画をデータセットレベルで家族ごとに分割**しない**。Immichは内部でユーザーIDごとにアップロードを自動整理するため、物理分割は不要かつ非効率
- 共有は Immich の**共有アルバム** (子供の写真・家族写真など特定テーマ) と **Partner機能** (ライブラリ全体の相互閲覧、夫婦間向け) で実現する
- 子供がスマホを持つまでは、親のいずれかがアップロードし共有アルバムに集約する運用とする。専用アカウント発行は端末保有後に着手

### Postgres (Immichメタデータ) の配置 — SanDisk SSD (LVM-thin) に決定・ストレージ作成完了 (2026-07-24)
- **却下**: NVMe(rpool)配置 → 容量を圧迫する。NFS(6TBプール)配置 → Postgresの信頼性・ロック問題があり非推奨
- **採用**: `docs/disks.md` 記載のSanDisk SSD 240GB (`154778407406`) をホスト側で **LVM-thinストレージ化**(Proxmoxストレージ名 `ssd-thin`, Volume Group `ssd-thin`, 235.12GB)し、VE1に専用の追加ディスクとしてアタッチする。VE1内でこのディスクをマウントし、docker-composeの `DB_DATA_LOCATION` をそこへ向ける
- VE1のOS/ブートディスクはNVMe上に薄く維持 (既定方針通り)。写真・動画本体はTrueNASのNFS (`tank/pic_tank`)、DBだけSSDという3層構成になる
- **⚠️ 注意 (2026-07-24 実測)**: ProxmoxのLVM-Thinpool作成ダイアログの「候補ディスク一覧」はパーティション/FS検出の有無だけで絞り込んでおり、**稼働中VMが使用中のディスクでも「未使用」として選択肢に出ることがある**(今回6TB=VE2使用中のディスクが候補に出た)。ダイアログの選択肢を鵜呑みにせず、**必ずシリアル番号で対象を照合してから選択すること**。詳細: `docs/worklog.md` 2026-07-24(6)

### 未実施
- VE1構築自体が未着手 (Frigate+Immichコンテナ)。`ssd-thin`はまだ未アタッチ
- VE1構築時に `ssd-thin` からPostgres用ディスクを切り出してアタッチ
- **構築手順のドラフトを作成** ([`docs/ve1-immich-build.md`](../docs/ve1-immich-build.md))。GPU IOMMUグループ実測→nouveauブラックリスト(レベルC)→VM作成→GPU passthrough→OS/Docker/nvidia-container-toolkit→NFS/SSDマウント→Immich docker-compose起動、の順。docker-compose雛形は `configs/immich/`。**実機実行はユーザー本人が行う** (`docs/worklog.md` 2026-07-22の役割分担)

---

## Hermes Agent サンドボックス (VE6)

- スペック: 2 vCPU / 2〜4GB RAM
- ネットワーク: VLAN25 (静的IP)。**アウトバウンドHTTPS + DNS(RTX宛)のみ許可**
- **未決定**: 使用プラットフォーム (Telegram / Slack 等)。決定後、そのエンドポイントをファイアウォールで許可リスト化する
- VE6構築完了後、VLAN25の一時許可 (ポート80 / NTP) を削除する ([`02-network.md`](02-network.md) 参照)
