# VE1 構築手順 — Frigate + Immich (ドラフト)

← [worklog](worklog.md) / [plan/03-proxmox.md](../plan/03-proxmox.md)

> **この手順書の位置づけ**: 実機操作はユーザー本人が行う運用ルール (`docs/worklog.md` 2026-07-22決定)。
> Claudeは手順設計・レビュー・破壊的操作の安全確認・ドキュメント整備を担当し、実機コマンドは代行しない。
> 以下はレビュー用のドラフト手順。**実行前に内容を確認し、特にレベルC/B2の項目は個別承認のうえで進めること** (`CLAUDE.md` §1)。

最終更新: 2026-07-31

---

## 前提条件チェックリスト (現状)

| 項目 | 状態 |
|---|---|
| VE2 (TrueNAS) ストレージ層: プール`tank`+データセット(`pic_tank`/`cam_tank`=NFS, `doc_tank`=SMB) | ✅ 完了 |
| SSD LVM-thin化 (`ssd-thin`, 235.12GB, Proxmoxストレージ登録済み) | ✅ 完了 (未アタッチ) |
| GTX1650のIOMMUグループ確認 (グループ15、単独=道連れなしを実測確認済み) | ✅ 完了 (2026-08-01、`docs/iommu-groups.md`) |
| GTX1650のホスト側ドライバ(nouveau)ブラックリスト化 | ✅ 完了 (2026-08-01、`07:00.0`/`07:00.1`とも`vfio-pci`確認済み) |
| VE1 VM本体 (VMID=100) 作成・GPU passthrough・Debian 13インストール | ✅ 完了 (2026-08-01、SSH到達確認済み) |
| VE1 GPU(nvidia-driver)+Docker+nvidia-container-toolkit | ✅ 完了 (2026-08-04) |
| VE1 NFSマウント(`tank/pic_tank`) + SSD(`ssd-thin`)アタッチ | ✅ 完了 (2026-08-04) |
| **Immich起動・管理者アカウント作成** | ✅ **完了 (2026-08-04)** |
| NFS許可アドレスの絞り込み (暫定 `192.168.20.0/24` → VE1確定IP `192.168.20.160` へ) | ⬜ 未対応 |

**この手順はVE1構築が「次の一手」の筆頭になった時点 (2026-07-24 worklog) を受けて作成した。前提条件 (TrueNASストレージ・SSD LVM-thin) は満たされている。**

---

## 全体設計 (再掲、根拠: `plan/03-proxmox.md` / `plan/04-gpu-ai.md`)

3層ストレージ構成:

| 層 | 内容 | 配置 |
|---|---|---|
| OS/ブート | VE1自体のシステムディスク | NVMe (`local-zfs`/rpool、シンプロビジョニング) |
| 写真・動画実体 (Immich `UPLOAD_LOCATION`) | `tank/pic_tank` | NFS → VE2(TrueNAS, `192.168.20.151`) |
| 監視カメラ録画 (Frigate、本手順の対象外・後続) | `tank/cam_tank` | NFS → VE2 |
| Postgres (Immichメタデータ、`DB_DATA_LOCATION`) | `ssd-thin` から切り出した追加ディスク | SATA SSD LVM-thin (ホスト直結) |

GPU: **GTX1650をVE1へまるごとパススルー** (VM分割・複数パススルーはしない)。VE1内でDocker (`nvidia-container-toolkit`) 経由でImmichの機械学習処理とFrigateの物体検出がGPUを共有する設計 (`plan/04-gpu-ai.md`)。Frigateはピーク時1GB前後、Immichはバースト1〜2GB程度で、4GB枠内での共存を想定。

**本手順書はImmich部分に絞る。** Frigateのdocker-compose統合は別途後続作業とする (branch scope: `ve1-immich-setup`)。

---

## Step 0: GTX1650のIOMMUグループ実測 (レベルA、承認不要)

VE2構築時にグループ14で事故 (`docs/iommu-groups.md`) があったため、**GTX1650をパススルーする前に必ず単独グループであることを実測で確認する。**

```bash
# GTX1650のBDFを確認 (plan上はグループ15、07:00.0/07:00.1と推定)
lspci -nnk | grep -A3 -i nvidia

# 該当グループの全デバイスを確認 (グループ番号は上記結果から読み替え)
for d in /sys/kernel/iommu_groups/15/devices/*; do
  echo "$d"; lspci -nnks "$(basename "$d")"
done
```

**確認したいこと**:
- グループにGTX1650のVGA機能とHDMI Audio機能の2つ**だけ**が含まれているか
- 管理NICやSATAコントローラ等、他の道連れデバイスがないか (グループ14事故の再発防止)

**✅ 2026-08-01実測確認済み**: グループ15は `07:00.0`(VGA)/`07:00.1`(Audio) の2件のみ、道連れデバイスなし。パススルー可能と判断し `docs/iommu-groups.md` に記録済み。Step 1 へ進んでよい。

---

## Step 1: nouveauブラックリスト + vfio-pciバインド (レベルC、要承認・reboot)

`CLAUDE.md` レベルC (カーネルパラメータ変更 + reboot)。**実行前に必ず承認を得ること。** VLANアウェアブリッジの回とは異なりロックアウトリスクは低いが、reboot前提のためCLAUDE.mdの分類上はCとして扱う。

### 変更前バックアップ (必須)

```bash
mkdir -p /root/backup
cp /etc/modprobe.d/vfio.conf /root/backup/vfio.conf.$(date +%Y%m%d)
```

### 追加設定

`/etc/modprobe.d/blacklist-nouveau.conf` (新規):
```
blacklist nouveau
options nouveau modeset=0
```

`/etc/modprobe.d/vfio.conf` (既存ファイルに追記。VE2用の `1022:43d5,1022:43c8` は変更しない):
```
options vfio-pci ids=1022:43d5,1022:43c8,10de:1f82,10de:10fa
softdep ahci pre: vfio-pci
softdep xhci_pci pre: vfio-pci
softdep nouveau pre: vfio-pci
softdep snd_hda_intel pre: vfio-pci
```
(`10de:1f82`=GTX1650 VGA, `10de:10fa`=HDMI Audio。2026-08-01実測確認済み、`docs/iommu-groups.md`参照。Audio機能は`snd_hda_intel`が先に掴むため`softdep`にも追加)

反映:
```bash
update-initramfs -u -k all
proxmox-boot-tool refresh
reboot
```

### 確認 (reboot後)

```bash
lspci -nnk -s 07:00.0
lspci -nnk -s 07:00.1
# 両方とも "Kernel driver in use: vfio-pci" になっていること
```

**✅ 2026-08-01実施・確認済み**: VE2を`qm shutdown 200`で正常停止 → `blacklist-nouveau.conf`/`vfio.conf`(GTX1650のみ、旧SATA/USB分`1022:43d5,1022:43c8`は案4撤回済みのため含めず)を新規作成 → `update-initramfs -u -k all && proxmox-boot-tool refresh` → `reboot` → `07:00.0`/`07:00.1`とも`Kernel driver in use: vfio-pci`を確認。VE2は`qm start 200`で再起動。

---

## Step 2: VE1 VM作成 (レベルB1、データ無しの新規VM — 説明のうえ実行し事後報告)

VE2 (`qm create 200 ...` → OVMF/q35/qxl→GPUパススルー時はqxl不要) の実績を踏襲。**以下のスペックは提案値。実行前にRAM/コア配分を確認すること** (Node0全体48GB、VE2に8GB割当済み)。

```bash
qm create 100 \
  --name ve1-frigate-immich \
  --machine q35 \
  --bios ovmf \
  --efidisk0 local-zfs:1,efitype=4m,pre-enrolled-keys=0 \
  --scsihw virtio-scsi-single \
  --scsi0 local-zfs:32,serial=VE1BOOT,discard=on,ssd=1 \
  --net0 virtio,bridge=vmbr0,tag=20 \
  --cores 6 \
  --memory 12288 \
  --cpu host \
  --ostype l26 \
  --agent enabled=1
```

- `serial=VE1BOOT`: VE2で踏んだ「シリアル未指定→重複エラー」の教訓を踏襲 (`docs/worklog.md` 2026-07-24(4))
- `tag=20`: VLAN20 (サーバーセグメント)、VE2と同一
- ブートディスク32GB: OSのみ (写真本体はNFS、DBはSSD側)。Debian 12 + Docker運用なら妥当な下限値、要確認
- RAM 12GB / 6コア: Frigate検出+Immich ML+Postgres+Docker常駐分の見積もり提案。Node0はRyzen 2600 (6c/12t)のため、コアはホストと按分される点に留意

VMディスク作成後、**まずISOなしで設定を確認し** (`qm config 100`)、問題なければDebian 12 netinst ISOをアタッチしてインストールへ進む。

---

## Step 3: GPU passthrough追加 (レベルB1、停止中VMへの設定変更)

Step 0で確認したBDF (グループ全体) を指定。`x-vga=0` はGTX1650が唯一のGPUではなくVE1内部で完結させるため (Proxmoxコンソールはserial/VNCで代替)。

```bash
qm set 100 -hostpci0 0000:07:00,pcie=1,x-vga=0
```

- `0000:07:00` (関数省略) でVGA+Audio両方を1エントリで渡す (`02:00,pcie=1` 形式、VE2のグループ14と同じ書式)
- インストール中の初期画面確認が必要な場合は、一時的に `-vga std` を残しGPU認識後に外す運用も検討 (VE2でqxl経由インストールした実績と同様の切り分けが必要になる可能性)

**✅ 2026-08-01実施・確認済み**: `qm set 100 -hostpci0 0000:07:00,pcie=1,x-vga=0` 実行、`qm config 100` で反映確認済み。

---

## Step 4: ゲストOSインストール

**Debian (netinst、`current`が指す最新安定版)を使用。** 理由: `nvidia-container-toolkit` の公式サポート対象であり、Frigate/Immich公式ドキュメントでの動作実績が豊富。GUIは不要 (netinst、SSH有効化のみ)。

**✅ 2026-08-01実施・確認済み**: 実際にダウンロードされたのは **Debian 13 (trixie) 13.6.0**(`current`シンボリックリンクが指す現行安定版。当初想定の12は既にoldstable)。以下の点でつまずいたので記録する。

### つまずき1: q35+OVMFで`ide2`のCD-ROMがブート認識されない
`qm set 100 -ide2 local:iso/...,media=cdrom` + `-boot order='ide2;scsi0;net0'` で起動すると、OVMFのブートマネージャがPXE/HTTPネットワークブートまで試して**CD-ROMを一切ブート候補に出さない**。`qm config`上は正しく設定されているにもかかわらず認識されない、q35+OVMFの既知の挙動。
**対処**: CD-ROMを`ide2`ではなく**`scsi1`**(SCSIバス)に繋ぎ直したところ認識された。
```bash
qm stop 100
qm set 100 -delete ide2
qm set 100 -scsi1 local:iso/<ファイル名>,media=cdrom
qm set 100 -boot order='scsi1;scsi0;net0'
qm start 100
```

### つまずき2: インストール完了後、CD-ROMを外そうとしてもホットアンプラグ拒否
稼働中VMから`qm set 100 -delete scsi1`を実行すると `hotplug problem - can't unplug bootdisk 'scsi1'` で失敗する(その時点でまだブートディスク扱いのため)。**`-boot order`を先に`scsi0;net0`へ変更しておけば、CD-ROMが挿さったままでも次回起動は正しくディスクから立ち上がる**ため実害はない。確実に外したい場合は `qm stop 100` で完全停止してから削除する。

### つまずき3: noVNCコンソールへのペーストができない/表示が更新されないことがある
noVNCはCtrl+Vでの直接貼り付けに対応していない。コンソール左端の`>>`タブ(クリップボードパネル)を使うか、短いコマンドは直接キーボード入力する。また、VM再起動を挟むと画面表示が古いフレームのまま見えることがあるため、**コンソールの見た目だけで判断せず、`ping`や`qm status`など別経路で実際の状態を確認する**こと。

### 最終的なインストール設定
| 項目 | 値 |
|---|---|
| ホスト名 | `Ghome` (意図的に選択。ドメインは`Ghome.local`) |
| IPアドレス | `192.168.20.160/24` (静的、VLAN20) |
| ゲートウェイ/DNS | `192.168.20.254` |
| パーティション | Guided・ディスク全体・LVM無し (ESP 1GB / ext4 31.6GB / swap 1.8GB) |
| ソフトウェア選択 | デスクトップ環境オフ、`SSH サーバ`+`標準システムユーティリティ`のみ |

### つまずき4: SSHでrootログインできない (パスワード認証拒否)
Debianのデフォルト`sshd_config`は`PermitRootLogin prohibit-password`相当(コメントアウトされたデフォルト値)で、**パスワードでのrootログインを拒否**する。VLAN20内部限定運用のため許容し、以下で変更した。
```bash
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/i' /etc/ssh/sshd_config
systemctl restart ssh
```
(このVMは内部管理用途のみで外部公開しない前提。将来Immichを公開する際もCloudflare Tunnel経由でVE1本体は直接晒さない方針 `plan/03-proxmox.md`)

インストール後の最小設定:
```bash
apt update && apt install -y curl gnupg2 ca-certificates nfs-common
```

---

## Step 5: NVIDIAドライバ + Docker + nvidia-container-toolkit (ゲスト内)

**ゲストOS内で実行** (ホストではない。ホストのGPUドライバはStep 1でブラックリスト化済み)。

```bash
# NVIDIA driver (Proprietary、GTX1650=Turing世代なので公式ドライバ対応)
apt install -y nvidia-driver firmware-misc-nonfree
reboot

# 確認
nvidia-smi

# Docker
curl -fsSL https://get.docker.com | sh

# nvidia-container-toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
apt update && apt install -y nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker

# 確認 (コンテナからGPUが見えること)
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

> パッケージ名・リポジトリURLは2026-01時点の情報。実行時に [NVIDIA公式](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) の最新手順と差分がないか確認すること。

**✅ 2026-08-04実施・確認済み**: `docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi` でコンテナ内からGTX1650(4096MiB VRAM)を確認。つまずいた点:
- `curl`/`gnupg2`が未インストールだった(Step4の「インストール後の最小設定」を未実施だったため)。先に`apt install -y curl gnupg2 ca-certificates nfs-common`が必要
- `nvidia-driver`インストール直後は`nvidia-smi`が失敗。原因は**`linux-headers-$(uname -r)`が入っておらずdkmsがカーネルモジュールをビルドできていなかった**(`dkms status`が`added`止まり)。`apt install -y linux-headers-$(uname -r) && dkms autoinstall`でビルドし直して解決

---

## Step 6: NFSマウント (`tank/pic_tank`)

VE2側のNFS許可は暫定で `192.168.20.0/24` 全体を許可済み (`plan/03-proxmox.md`)。VE1のIP確定後、**VE1単体へ絞り込むこと** (次の一手 #2、未実施)。

```bash
mkdir -p /mnt/nfs/pic_tank
mount -t nfs 192.168.20.151:/mnt/tank/pic_tank /mnt/nfs/pic_tank -o vers=4

# 恒久化 (/etc/fstab)
echo "192.168.20.151:/mnt/tank/pic_tank /mnt/nfs/pic_tank nfs4 defaults,_netdev 0 0" >> /etc/fstab
```

NFSエクスポートパス (`/mnt/tank/pic_tank` 等) はTrueNAS側の実際のマウントポイントに合わせて確認すること (未確認、TrueNAS GUIの共有設定画面で要確認)。

**✅ 2026-08-04実施・確認済み**: `showmount -e 192.168.20.151` でエクスポートパス確認 (`/mnt/tank/pic_tank`, `/mnt/tank/cam_tank`ともに`192.168.20.0/24`許可)。マウント・`df -h`で5.4TB利用可能を確認・`/etc/fstab`恒久化まで完了。

**⚠️ つまずき: NFS root_squashで書き込み拒否**。`pic_tank`は`root:root, 755`権限のため、VE1のroot権限がNFS越しに無権限ユーザーへ格下げされ`touch`が`Permission denied`になった。**TrueNAS GUI (`Shares → Unix Shares (NFS) → pic_tank → Advanced Options`) で `Maproot User=root` / `Maproot Group=root` を設定して解決**(VLAN20内部限定・単一トラステッドVMからのアクセスのため許容)。書き込みテストで確認済み。

---

## Step 7: Postgres用ディスク (`ssd-thin`) の切り出し・アタッチ

**ホスト側 (レベルB1、停止中VMへの設定変更)**:
```bash
qm set 100 -scsi1 ssd-thin:32,serial=VE1PGDATA
```
容量32GBはPostgresメタデータ用途としては余裕のある提案値。要確認。

**ゲスト内**: 新規ディスクを認識後、**`lsblk -o NAME,SIZE,SERIAL`でシリアル(`VE1PGDATA`)を確認してから**フォーマット・マウントする(デバイス名`/dev/sdX`の対応は再起動で入れ替わりうるため、思い込みで叩かない)。

```bash
lsblk -o NAME,SIZE,SERIAL  # VE1PGDATAのシリアルで対象デバイスを特定
mkfs.ext4 /dev/sdX  # 上で確認した実際のデバイス名に置き換える
mkdir -p /mnt/ssd-pgdata
mount /dev/sdX /mnt/ssd-pgdata
blkid /dev/sdX  # UUID取得
echo "UUID=<取得したUUID> /mnt/ssd-pgdata ext4 defaults 0 2" >> /etc/fstab
```

> **恒久化は必ずUUIDを使う**(`/dev/sdX`は再起動で変わりうる)。

**✅ 2026-08-04実施・確認済み**: `qm set 100 -scsi1 ssd-thin:32,serial=VE1PGDATA` でアタッチ。ゲスト内`lsblk`では**インストール時と`sda`/`sdb`の対応が入れ替わっていた**(起動時のOS本体ディスクが`sdb`に、新規追加した`VE1PGDATA`が`sda`になっていた)。シリアル(`lsblk -o NAME,SIZE,SERIAL`)で照合し`/dev/sda`と確定 → `mkfs.ext4`→マウント→UUID(`17117cb0-c2fc-46f1-90cf-87b274d3386c`)で`/etc/fstab`恒久化。既存のroot/EFI/swapエントリも元々UUID指定だったため、デバイス名の入れ替わり自体は無害だった。

---

## Step 8: Immich docker-compose 起動

`configs/immich/docker-compose.yml` と `configs/immich/immich.env.example` を参照。

```bash
mkdir -p /opt/immich && cd /opt/immich
# configs/immich/docker-compose.yml の内容をそのまま配置
# .env は immich.env.example を元に、DB_PASSWORD は必ずVE1上で生成する (会話ログ等に貼らない)
cat > .env <<EOF
UPLOAD_LOCATION=/mnt/nfs/pic_tank
DB_DATA_LOCATION=/mnt/ssd-pgdata/postgres
TZ=Asia/Tokyo
IMMICH_VERSION=release
DB_USERNAME=immich
DB_DATABASE_NAME=immich
DB_PASSWORD=$(openssl rand -base64 24)
EOF
chmod 600 .env

mkdir -p /mnt/ssd-pgdata/postgres  # マウントポイント直下は使えない (下記つまずき参照)
docker compose up -d
```

起動確認: `http://<VE1のIP>:2283` にアクセスし、初期管理者アカウント作成画面が出ることを確認。

**✅ 2026-08-04実施・確認済み**: VE1で管理者アカウント作成まで完了。以下のつまずきがあった。

### つまずき1: `docker compose config`が秘密情報を平文表示してしまう
YAML構文だけ確認するつもりで`docker compose config`を実行したところ、`.env`の変数が実際の値に展開されて**DB_PASSWORDが平文で出力された**(会話ログ等に残るリスク)。この場合は**新しいパスワードを再生成してから初回起動する**(初回起動前ならPostgres初期化前なので無害に差し替え可能)。以降、構文だけ確認したい場合は `docker compose config --quiet` (エラー時のみ出力) を使う。

### つまずき2: `sed`の区切り文字とbase64パスワードの`/`が衝突
`openssl rand -base64`が生成する文字列には`/`が含まれることがあり、`sed 's/.../.../ '`(区切りが`/`)で置換しようとすると`unknown option to 's'`で失敗する。**区切り文字を`#`など衝突しない文字にする**(`sed "s#...#...#"`)。

### つまずき3: 用意したイメージタグが古く`postgres:14`が見つからない
Immichの公式Postgresイメージはタグ体系が頻繁に変わる(バージョン管理拡張の名称変更等)。ドラフト作成時点の`ghcr.io/immich-app/postgres:14`は導入時には既に存在しなかった。**公式リポジトリから直接最新のcompose定義を取得して照合する**のが確実。
```bash
curl -fsSL https://raw.githubusercontent.com/immich-app/immich/main/docker/docker-compose.yml -o /tmp/immich-official-compose.yml
grep -B2 -A2 "image:" /tmp/immich-official-compose.yml
curl -fsSL https://raw.githubusercontent.com/immich-app/immich/main/docker/hwaccel.ml.yml -o /tmp/immich-hwaccel-ml.yml
cat /tmp/immich-hwaccel-ml.yml  # GPU(cuda)ブロックの照合用
```
2026-08-04時点の正しい値は `configs/immich/docker-compose.yml` に反映済み。

### つまずき4: Postgresが`initdb: error: directory ... exists but is not empty`でクラッシュループ
`DB_DATA_LOCATION`をマウントポイント直下(`/mnt/ssd-pgdata`)に向けていたため、ext4の`lost+found`がありinitdbが「空でない」と判定して失敗し続けた。**マウントポイント直下ではなく、その下にサブディレクトリ(`/mnt/ssd-pgdata/postgres`)を作ってそこを指定**して解決。

---

## 起動後チェックリスト

- [x] `docker compose ps` で全コンテナ `healthy`/`Up` (2026-08-04確認済み)
- [x] `docker exec immich-immich-machine-learning-1 nvidia-smi` でGPUが見えること (2026-08-04確認済み、4096MiB認識)
- [x] Immich Web UIで管理者アカウント作成 (2026-08-04完了)
- [ ] Immich Web UIから写真アップロード → `/mnt/nfs/pic_tank` に実ファイルが書き込まれることを確認
- [ ] Postgresデータが `/mnt/ssd-pgdata/postgres` に書き込まれていることを確認 (`du -sh /mnt/ssd-pgdata/postgres`)
- [ ] VE1再起動後もNFS/SSDマウント・docker-composeが自動復旧すること (`/etc/fstab` + Docker再起動ポリシーの動作確認)
- [ ] `nvidia-smi` のVRAM使用量を実測し `plan/04-gpu-ai.md` の見積もり (Frigate導入後3GB前後) と照合、`docs/` へ記録

---

## 未実施・後続タスク

1. NFS許可アドレスをVE1確定IPへ絞り込み (VE2/TrueNAS側の作業)
2. Cloudflare Tunnel + Access でのImmich外部公開 (専用LXC、`plan/03-proxmox.md` の外部公開戦略)
3. Frigateのdocker-compose統合 (別セッション、GPU共有の実測を踏まえて追加)
4. ZFSスナップショット/PBSバックアップ対象へVE1を追加
