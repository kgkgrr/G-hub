# 作業履歴

## 現在の状態 (毎セッション末尾に上書き更新 / 20行以内)

- **稼働中**:
  - Node0: Proxmox VE 9.2.2 (NVMe単体 ZFS rpool、**管理IP `192.168.20.150/24` VLAN20**、`node0.Ghome.local`)
  - ネットワーク: RTX830 + SWX2110P-8G 投入済み、WLX222 VAP1/VAP4 接続確認済み
  - **vmbr0 = VLANアウェアブリッジ化完了 (2026-07-23)**。ホスト管理をVLAN20へ移設・検証済み。DNS=`.20.254`確認済み
- **設計の重大変更 (2026-07-23夜, 案4→案2)**:
  - **案4(コントローラ単位PT)は実機で不可能と判明・撤回**。IOMMUグループ14は「USB+SATA」ではなく**チップセットPCIeスイッチ+配下全デバイス(両NIC含む)**。VE2へ渡した瞬間、管理NIC(`05:00.0`)ごとリセットされ**ホストがハング**(事故発生)。詳細 `docs/iommu-groups.md`
  - **案2(ディスク単位パススルー)へ移行**: vfio解除→ホストが6TB/SSD直認識→`qm set 200 -scsiX /dev/disk/by-id/...` でVE2へ。TrueNAS GUIは維持。NVMe集約・256GB据置方針は不変
- **VE2 (TrueNAS SCALE 25.10.4)**: **インストール完了・ネットワーク疎通OK**。管理 `192.168.20.151` (VLAN20)、hostname `Gnas`、admin=`truenas_admin`。6TBを`scsi1`でディスクPT済み(VE2内では`sda`5.46TiB)。VGA=qxl(std/OVMFで砂嵐→qxlで解決)、serial0あり
- **中途半端な状態**:
  - VE2: **プール`tank`作成完了**(5.46TiB, Stripe, Healthy, 暗号化なし)。**データセット未作成**、共有(NFS/SMB)未設定
  - SSD(`sda`,240GB)はホスト側で未使用(要wipe→ストレージ化、VE1用)
  - VLAN20/25 の観察・一時許可が継続中
- **次の一手 (最大3件)**:
  1. TrueNAS GUIで `tank/pic_tank`(Immich),`tank/cam_tank`(Frigate),`tank/doc_tank`(書類) データセット作成(recordsize/compression設定は`plan/03-proxmox.md`参照)→NFS/SMBエクスポート
  2. SSD(240GB)をホストで**LVM-thin化** → VE1にPostgres専用ディスクとしてアタッチ
  3. VE1構築 (Frigate+Immich, GTX1650) → NFS(写真/録画)・SSD(DB)接続
- **注意中の問題 (最大3件)**:
  1. **UPS未導入** — 本番投入前に必須 (7/20 実停電あり、正弦波必須)
  2. **PBSクォーラム** — 2ノードでQDevice未手当て
  3. **案4事故の教訓** — このボードはIOMMUグループが粗く、SATA/NIC分離不可。PCIパススルーは慎重に (GPUのグループ15は要再確認)

---

## 2026-07-24 (4) 【解決】ディスクのシリアル未伝播でプール作成失敗→serial付与で解決、プール`tank`作成完了

### やったこと
- TrueNAS GUIでプール作成を試行 → `エラー: topology / Disks have duplicate serial numbers: None (sda, sdb)` で失敗
- 原因調査: VE2のSCSIディスク定義(`scsi0`=ローカルzvolブート32G, `scsi1`=6TB by-idパススルー)に`serial=`パラメータが無く、ゲスト(TrueNAS)側で両ディスクとも空シリアル(`None`)として認識され、TrueNASの重複ディスク安全チェックに引っかかっていた
- 対処: VE2を`qm shutdown 200`で停止→`qm set`で両ディスクに`serial=`を追加 (`scsi0`→`serial=TN200BOOT`任意値、`scsi1`→`serial=WD-WX42D369CEFE`実シリアル)→`qm start 200`で起動
- TrueNAS側で両ディスクのシリアルが別々に表示されることを確認 → プール作成再試行 → **成功**。プール`tank`(1×STRIPE, 1×5.46TiB HDD, Healthy, 暗号化なし)

### つまづきと解決 (次回のため)
- **Proxmoxでディスクを`by-id`パススルーしても、ゲストにはデフォルトでシリアルが伝わらない。** ローカルzvolディスクも同様にシリアル未設定だと空扱いになる。**TrueNAS(や一部のストレージOS)は「シリアルが同一(=空も同一とみなす)のディスクでVDEVを組もうとしていないか」を検証するため、シリアル未設定だとプール作成時にtopologyエラーで弾かれる。** → **VMのディスクには必ず`serial=`を明示指定する**運用とする(今後VE1等でも同様の構成をする場合は要注意)

### 決めたこと
- VE2のディスクシリアル運用: 物理ディスク(6TB)は`docs/disks.md`記載の実シリアルをそのまま指定。ローカルzvol(ブート等、物理シリアルが無いもの)は任意の識別しやすい文字列を指定する方針とする

### 未解決・次回やること
1. TrueNAS GUIで `tank/pic_tank`,`tank/cam_tank`,`tank/doc_tank` データセット作成 → NFS/SMBエクスポート設定
2. SSD(240GB)のLVM-thin化 → VE1へPostgres専用ディスクとしてアタッチ
3. VE1構築(Frigate+Immich)着手

### 実機の状態
- 稼働中: Node0(Proxmox, VLAN20 `.150`)、VE2(TrueNAS `.151`, **プール`tank` Healthy稼働中**、データセット未作成)
- ホスト保持・未使用: SSD 240GB(`sda`)

---

## 2026-07-24 (3) プール名・データセット名を確定

### 決めたこと
- **プール名は `tank`**(単騎5.46TiB, Stripe, 暗号化なし=ローカル運用のため不要と判断)
- データセット名は用途プレフィックス方式で確定: `tank/pic_tank`(Immich写真+動画), `tank/cam_tank`(Frigate録画), `tank/doc_tank`(書類)。当初ユーザーから「pic_tank/doc_tank/cam_tankを独立プールに」という案が出たが、**物理ディスクが1本(5.46TiB)のためプールは1つしか作れない**ことを説明し、1プール+3データセット構成に合意
- `plan/03-proxmox.md` のデータセット設計表を確定名で更新済み(プレースホルダの`pool/*`から`tank/*`へ)

### 未解決・次回やること
- TrueNAS GUIでの実際のプール作成(`tank`)・3データセット作成・NFS/SMBエクスポート設定(未実施)

---

## 2026-07-24 (2) データセット設計・Immich共有方針・Postgres配置を決定

### やったこと
- 6TBプール上のデータセット構成を設計 (`pool/immich` / `pool/frigate` / `pool/docs`)。用途ごとの recordsize・compression・snapshot方針を整理し `plan/03-proxmox.md` に記録

### 決めたこと
- **家族間の写真共有はデータセット分割ではなくImmichのアプリ機能で実現**。共有アルバム(子供の写真・家族写真)とPartner機能(夫婦間の全体共有)を使う。データセットをユーザー単位で切るのは非効率と判断し却下
- **`pool/docs` は当面夫婦共有の単一データセット**。子供が端末を持つ時期に `pool/docs/<name>` を子データセットとして追加する拡張方針とし、今から先回りして分割しない
- **Postgres(Immichメタデータ)はSanDisk SSD(240GB, LVM-thin化)に配置**。却下: NVMe配置(容量圧迫)、NFS(6TBプール)配置(Postgresの信頼性・ロック問題で非推奨)。VE1のOS/ブートはNVMe、写真動画本体はTrueNAS NFS、DBのみSSDという3層構成に確定

### 未解決・次回やること
1. TrueNAS GUIで6TBプール作成 → 上記3データセット作成 → NFS/SMBエクスポート設定
2. ProxmoxホストでSSD(`154778407406`)をLVM-thin化 → VE1へ追加ディスクとしてアタッチ(実行前にシリアルで6TBとの取り違え防止を再確認)
3. VE1構築自体は未着手。Immich docker-composeの `UPLOAD_LOCATION`(NFS)/`DB_DATA_LOCATION`(SSD)設定はVE1構築時に反映

### 実機の状態
- 変更なし(設計のみ、実機操作は未実施)。稼働中: Node0(Proxmox, VLAN20 `.150`)、VE2(TrueNAS `.151`, プール未作成)。ホスト保持・未使用: SSD 240GB(`sda`)

---

## 2026-07-24 VE2(TrueNAS) 案2で構築完了・ネットワークまで疎通

### やったこと
- vfio.conf退避→initramfs更新→reboot でホスト復旧。`02:00.1`=ahci、6TB(`sdb`)/SSD(`sda`)がホスト可視化。**全ディスク実シリアル確定→disks.md更新**
- 6TBをVE2へディスクPT: `qm set 200 -scsi1 /dev/disk/by-id/ata-WDC_WD60EFPX-68C5ZN0_WD-WX42D369CEFE,backup=0`
- VE2起動→**TrueNAS SCALE 25.10.4 インストール成功**。認証=Administrative user `truenas_admin`(パスワード設定済み・ユーザー管理)
- TrueNASコンソールで静的IP設定: interface `enp6s18` の aliases に `192.168.20.151/24`、GW/DNS `192.168.20.254`、hostname `Gnas`
- **GUIログイン確認済み** (`https://192.168.20.151`, truenas_admin)

### つまづきと解決 (次回のため)
- **OVMF+std VGAでインストーラが砂嵐** → `qm set 200 -vga qxl` で解決 (cirrus/nomodeset/シリアルも代替として有効)。serial0も追加済み
- **シリアルコンソールでスペースキーが効かず**ディスク選択できない → VGA(qxl)に切替で解決
- **「ホストにMacからping不可」で焦ったが原因はMacのWiFiがIoT SSID(VLAN30)を自動接続**していただけ。VLAN30→VLAN20はFW遮断。ホストは終始正常。→ **切り分け時はまずクライアント側のVLANを疑う**
- **TrueNASのゲートウェイ設定でunreachableエラー** → interfaceのIP(alias)を先に適用してからGWを入れる順序。IPはaliasesに`x.x.x.x/24`形式で入れる

### 決めたこと
- SSD(240GB)は案2の副産物として**ホスト側ストレージに回す**(TrueNASには渡さない)。NVMe逼迫の受け皿+VE1ディスク。フォーマット許可済み

### 次回やること (別チャットへ引き継ぎ)
1. **TrueNAS GUIで6TBプール作成**: `https://192.168.20.151` → Storage → Create Pool。**対象はVE2内で`sda`=5.46TiBの方**(32GB=`sdb`はブート、選ばない)。単騎=stripe警告は承知の上でOK(冗長はPBS)。プール名は要決定
2. データセット設計 (Immich写真用/LLM書類用等) → NFS共有設定 → VE1から利用
3. SSD(`sda`,240GB,`154778407406`)をホストでwipe→ZFS/LVM化 (破壊操作: 6TB`sdb`と取り違え厳禁)

### 実機の状態
- 稼働中: Node0(Proxmox, VLAN20 `.150`)、VE2(TrueNAS `.151`, プール未作成)
- ホスト保持・未使用: SSD 240GB(`sda`)
- 未構築: VE1,VE3〜VE6

## 2026-07-23 (3) 【事故】案4パススルーでホストハング → 案2へ移行

### やったこと
- VE2(200)作成 (q35/OVMF/SecureBoot無効/host CPU/8GB/32GBブート/net tag20) → `hostpci0: 0000:02:00,pcie=1` 追加 → `qm start 200` で**ホストが完全ハング**(SSH/ping全断)
- 物理コンソールで復旧。`journalctl -b -1` で原因特定 → `hostpci0`削除、vfio.conf退避してreboot準備

### 原因 (確定)
- **IOMMUグループ14の実態がplan記録と違った**。記録は「USB(02:00.0)+SATA(02:00.1)」だけだったが、実際は**チップセットPCIeスイッチ(02:00.2)+配下の両NIC(04:00.0=2.5G, 05:00.0=オンボード1G=管理NIC)**を含む8デバイス
- グループごとVE2へ渡してリセット → 管理NIC(05:00.0)ごと落ちてホスト死。詳細 `docs/iommu-groups.md`

### 決めたこと
- **案4(コントローラ単位PT)を撤回**。B450M Pro4ではグループが粗すぎてSATAとNICを分離不可
- **案2(ディスク単位パススルー)を採用**。vfioを外しホストが直接ディスクを持ち、`qm set -scsi /dev/disk/by-id/` でVE2へ。TrueNAS GUI維持・GPUのPCIE3温存
- 却下: ACSオーバーライドでグループ細分化 → 分離保証を捨てる上NICを巻き込む構造でリスク過大 / HBA(案1) → PCIE3も同スイッチ配下の懸念+出費

### 未解決・次回やること
- reboot→NIC復旧＆ディスク可視化確認→disks.md確定→VE2へ6TBディスクPT→TrueNASインストール
- GPU(グループ15と推定)がVE1に単独パススルー可能か、VE1着手前に実機再確認 (同じ轍を踏まない)

### 実機の状態
- Node0: 事故から復旧作業中。hostpci0削除済み、vfio.conf退避済み、reboot待ち
- VE2(200): シェル作成のみ・未インストール

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
