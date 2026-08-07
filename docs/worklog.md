# 作業履歴

## 現在の状態 (毎セッション末尾に上書き更新 / 20行以内)

- **稼働中**:
  - Node0: Proxmox VE 9.2.2、管理IP `192.168.20.150/24`(VLAN20、vmbr0アウェア化済み)
  - VE2 (TrueNAS SCALE、`.151`, hostname `Gnas`): **ストレージ層完成**。プール`tank`(6TB, Stripe) + `pic_tank`/`cam_tank`(NFS)・`doc_tank`(SMB)、共有・ACL設定済み
  - **🎉 VE1 (Immich、VMID=100, hostname `Ghome`, `.160`)**: Debian 13、GTX1650パススルー(グループ15単独確認済み)、Docker+nvidia-container-toolkit、NFS(`pic_tank`)+SSD(`ssd-thin`→`/mnt/ssd-pgdata/postgres`)連携、**Immich稼働中・初回アップロードでエンドツーエンドの動作確認済み**(`docs/ve1-immich-build.md`に詳細・つまずき集)
  - SATA SSD 240GBは`ssd-thin`(LVM-Thin, 235.12GB)化済み、VE1のPostgres用に32GB使用中
- **決定: Node2(バックアップノード)の構成 (2026-08-06)**: Dynabook R741、玄人志向2台挿しドックで6TB(新品・ZFS化しTrueNAS Replication先/`pic_tank`+`doc_tank`)+3TB(中古・PBSデータストア)を接続。**未構築、実運用優先のため着手は後回し (2026-08-07決定)**、詳細 `plan/01-hardware.md`
- **中途半端な状態**: VLAN20/25の観察・一時許可が継続中
- **次の一手 (最大3件)**:
  1. **Node0/VE1の自動復帰の堅牢化(目下最優先、2026-08-07)** — 実機投入手順は `docs/ve1-immich-build.md` Step 9 にチェックリスト化済み(9-0確認→9-1/9-2→9-4→9-5検証)。まず9-0の現状確認から
  2. NFS許可アドレスの絞り込み(暫定`192.168.20.0/24`→VE1確定IP`192.168.20.160`)
  3. iCloud/Google Photos初回一括投入(設計ドラフト完了 `docs/immich-bulk-import.md`、Step1のiCloudダウンロード状況確認から着手)
- **注意中の問題 (最大3件)**:
  1. **`tank`(写真含む)がオフホスト・バックアップ無し** — バックアップ(Node2)は実運用優先のため後回しと決定 (2026-08-07)。ディスク単体障害・盗難・火災に無防備な状態のまま運用・一括投入を進める点は既知のリスクとして許容
  2. **UPS未導入** — 本番投入前に必須 (7/20 実停電あり、正弦波必須)
  3. **PBSクォーラム** — 2ノードでQDevice未手当て(Node2構築後に対応)

---

## 2026-08-07 (2) 方針転換: バックアップ後回し・Node0自動復帰を最優先に

### やったこと
- ユーザーより「先に実運用までこぎ着けたいので、バックアップ構築(Node2)は後回しにする。目先はNode0の自動復帰が急務」との方針決定を受領
- `docs/ve1-immich-build.md` Step 9を実行チェックリストとして再構成。Node0全体の自動復帰を①Proxmox起動順序(VE2/VE1のonboot・startup) ②VE2(TrueNAS)側のNFSサービス自動起動設定 ③VE1内のsystemd依存関係(docker.service ⇔ NFSマウント)の3層に整理し、まずレベルA(読み取り専用)の現状確認(9-0: `qm config`のonboot/startup、TrueNAS GUIのNFSサービス`Start Automatically`)から始める順序に組み替えた
- `plan.md`次のステップ・`docs/worklog.md`現在の状態を、この優先順位変更に合わせて更新(Node2構築を後回しに変更、Node0/VE1自動復帰を最上位に)

### 決めたこと
- **Node2(バックアップ)構築は後回し**。実運用を軌道に乗せることを優先し、`tank`がオフホスト・バックアップ無しの状態のまま一括投入等を進めることを許容する(リスクは認識のうえでの意図的な優先順位判断、ユーザー指示による)
- **Node0/VE1自動復帰の実行順序を確定**: 9-0(現状確認、レベルA)→9-1/9-2(VE1内systemd、レベルB1)→9-4(Proxmox起動順序、レベルB1)→9-5(段階的reboot検証)

### 未解決・次回やること
1. (ユーザー側) 9-0の現状確認コマンド実行 (`qm config 100`/`qm config 200`のonboot/startup、TrueNAS GUIのNFSサービス自動起動設定)
2. 確認結果を踏まえてStep 9を実機投入
3. 自動復帰の検証完了後、NFS許可絞り込み → iCloud/Google Photos初回投入へ進む

### 実機の状態
- 変更なし(方針決定とドキュメント整備のみ)。VE1/VE2は稼働継続中

---

## 2026-08-07 Immich運用フェーズ着手: 再起動自動復旧・初回一括投入の設計

### やったこと
- ユーザーからImmichの運用フェーズ着手を依頼(①コンテナの自動復帰、②iCloud/Google Photosからの初回一括投入)。`plan.md`/`docs/worklog.md`/`docs/ve1-immich-build.md`を確認し前提を把握
- 作業端末はMac確定(iCloud側がフル実体ダウンロード済みかは未確認、ユーザー側で確認予定)と確認
- **①再起動時の自動復旧**: 現状 `docker compose` の `restart: unless-stopped` だけでは、dockerがNFSマウント完了前に起動した場合に `/mnt/nfs/pic_tank` がローカル空ディレクトリのままbind mountされる潜在リスク(サイレントな誤配置)を指摘。対策として `docs/ve1-immich-build.md` に **Step 9** を追加: (a) VE1ゲスト内 `docker.service` へ `RequiresMountsFor=` のsystemd drop-in、(b) `/etc/fstab` に `x-systemd.mount-timeout` 追加、(c) Proxmoxホスト側でVE2→VE1の起動順序・up-delay設定 (`qm set -onboot -startup`)、(d) 検証はVE1単体reboot→Node0全体rebootの段階実施を提案
- **②初回一括投入**: `docs/immich-bulk-import.md` を新規作成。iCloud側は`osxphotos`でのエクスポート(GUI手動選択は非現実的と判断)、Google Photos側はTakeout→**Google Photos Takeout Helper (GPTH)** でJSONサイドカーの日時をEXIF/mtimeへ前処理してからImmich CLI (`@immich/cli`)で投入する設計とした。投入順序はiCloud→Google Photos(チェックサム重複検知を効かせやすくするため)、投入後に管理画面Duplicatesで見た目重複をレビューする運用とした
- バッチ分割の必要性(GPU VRAM 4GB・ジョブキュー詰まり対策、`plan/04-gpu-ai.md`のconcurrency調整方針と接続)、および**Node2(バックアップ)未構築のまま大量の原本を投入するリスク**を指摘し、全量投入はNode2構築後を推奨する形で`plan.md`次のステップに反映

### 決めたこと
- iCloudエクスポートは `osxphotos` を第一候補とする(GUI Exportは大量枚数に非現実的)。**却下**: Apple公式データリクエスト(privacy.apple.com) — Macに写真アプリがあり同期済みなら不要、到着まで数日かかり形式も扱いにくい
- Google Photos側は生Takeoutをそのまま投入せず、**GPTHでの前処理を必須ステップ**とする。理由: JSONサイドカーにしかない正しい撮影日時をEXIFへ書き戻さないと、Immichのタイムライン上で日付が壊れるケースがある
- 自動復旧の堅牢化は、マウント未完了時に「ローカルに誤って書き込む」のではなく「起動させない」fail-safe方針を採用 (`CLAUDE.md`「わからない時は止まる」に準拠)
- **iCloud/Google Photosの全量投入は、Node2(バックアップ)構築完了後に開始することを推奨**として`plan.md`へ明記。少量のテストバッチ投入(動作確認目的)はNode2構築と並行して先行してよい、とした

### 未解決・次回やること
1. (ユーザー側) Macの「写真」アプリでiCloudオリジナルのダウンロード設定状況を確認
2. VE1再起動時の自動復旧(Step 9)を実機投入・検証(まずVE1単体rebootから)
3. Node2構築の進捗次第で、初回一括投入のテストバッチ開始時期を判断

### 実機の状態
- 変更なし(設計・ドキュメント整備のみ、実機操作は未実施)。VE1/VE2は稼働継続中

---

## 2026-08-06 (2) Storage Template見送り決定、Node2バックアップ構成を決定

### やったこと
- Immichの Storage Template機能について相談 → `library/`ではなく`upload/`配下にUUIDベースで保持する現行仕様を確認済みだったため、有効化のメリット(人間可読パス)とコスト(日付修正時のリネーム発生、Immich公式もリネーム依存を弱める傾向)を整理して提示
- worklog記載の`qm set 200 -scsi1 ...,backup=0`を再確認し、**`tank`(写真含む)がPBSバックアップ対象外で、同一ディスク上のZFSスナップショットのみで保護されている**ことを指摘。「単騎stripe、冗長はPBS」という過去の記述との食い違いを報告
- Node2(バックアップノード)の構成を相談。当初「6TB=PBS、3TB=写真レプリカ先」を提案したが、ユーザーの意向で**逆(6TB新品=写真レプリカ先、3TB中古=PBS)に決定**。物理接続は玄人志向の2台挿しHDDドッキングステーション

### 決めたこと
- **Storage Templateは無効のまま**。理由: `pic_tank`はNFSのみでSMB共有せず、家族が直接ファイルシステムを覗く運用を想定していない。バックアップもブロック/データセット単位(ZFSスナップショット・Replication)であり命名規則に依存しない。有効化のメリット(万一のDB損失時の手動サルベージ)は、この構成では優先度が低いと判断
- **Node2のストレージ構成**: 6TB(新品)→ZFSプール化しTrueNAS Replication Task受け先(`pic_tank`+`doc_tank`)、3TB(中古)→PBSデータストア。**却下**: 逆の割り当て(6TB=PBS、3TB=写真)。理由: より重要(撮り直し不可)なデータには信頼性の高い新品ドライブを割り当てるべきと判断。PBSの実際の必要容量(VM設定・ブートディスク)は小さく3TBで十分
- `cam_tank`(Frigate録画)はレプリケーション対象外のまま(容量・優先度の両面)

### 未解決・次回やること
1. Node2構築: Dynabook R741にPBSインストール → ドッキングステーションで6TB/3TB接続 → 6TBをZFSプール化 → TrueNAS側でReplication Task設定 → 3TBをPBSデータストア登録
2. Node2構築完了まで、`tank`のオフホスト・バックアップが無い状態が続く点を認識しておく

### 実機の状態
- 変更なし(設計討議のみ、実機操作は未実施)。Node2は依然未構築

---

## 2026-08-06 VE1 Immich初回アップロード動作確認完了

### やったこと
- Immich Web UI (`http://192.168.20.160:2283`) から`IMG_0473.JPG`をアップロード
- `find /mnt/nfs/pic_tank/library -type f` では実ファイルが見つからず一時混乱 → DB(`asset`テーブル)の`originalPath`を確認したところ`upload/<ownerId>/<checksum先頭2桁>/<次2桁>/<uuid>.拡張子`形式で、実際には`library/`ではなく`upload/`配下に原本が保持される仕様と判明。実ファイルパスとDBの`originalPath`が一致することを確認
- `asset`テーブルにチェックサム・解像度(1535×2729)・thumbhash等のメタデータが記録されていることを確認 → NFS書き込み・Postgres書き込み・サムネイル生成(ML)まで一通り正常動作

### つまづきと解決 (次回のため)
- **現行Immichのストレージレイアウトは`library/`ではなく`upload/<ownerId>/<checksumの先頭バイト2桁ずつ>/<uuid>.拡張子`**。ストレージテンプレート機能(旧来の`library/`整理)は既定で無効。「`library/`が空=異常」ではない点に注意(バージョンで仕様が変わりうるので、疑わしい時はDBの`originalPath`列で実体パスを確認するのが確実)

### 決めたこと
- Frigate統合は**カメラ機材の準備待ちのため保留**。着手時期は未定、次はNFS許可の絞り込みとVE1再起動テストを優先する

### 実機の状態
- 稼働中: Node0(Proxmox)、VE2(TrueNAS `.151`)、VE1(`Ghome`, `192.168.20.160`) — **Immich稼働中・エンドツーエンド動作確認済み**(アップロード→NFS保存→Postgresメタデータ→サムネイル生成)

---

## 2026-08-04 (4) 🎉 VE1構築 Step8完了・Immich起動成功

### やったこと
- `/opt/immich`に`configs/immich/docker-compose.yml`相当を配置。`.env`を`openssl rand -base64 24`でパスワード生成し作成
- **事故**: 構文確認のつもりで`docker compose config`を実行したところ、`.env`の変数が実際の値に展開され**DB_PASSWORDが会話ログに平文で出力**された。初回起動前(Postgres未初期化)だったため、直ちに`openssl rand`で新しいパスワードを再生成して`.env`を上書きし、実害なく収束
  - 1回目の再生成コマンドは`sed 's/.../.../ '`(区切り`/`)を使ったが、base64パスワードに`/`が含まれ`sed: unknown option to 's'`で失敗・実際には置換されていなかった。区切りを`#`に変えて再実行し成功を確認
- `docker compose up -d`→`ghcr.io/immich-app/postgres:14`が**存在しないタグ**でpull失敗。公式リポジトリ(`github.com/immich-app/immich`)の`docker/docker-compose.yml`・`docker/hwaccel.ml.yml`を直接取得して正しいイメージ参照(`postgres:14-vectorchord0.4.3-pgvectors0.2.0@sha256:...`、`valkey:9@sha256:...`)を確認・反映
- 再度`docker compose up -d`→全イメージpull成功・4コンテナ起動したが、**`database`コンテナがクラッシュループ**(`initdb: error: directory "/var/lib/postgresql/data" exists but is not empty`、`lost+found`が原因)。`DB_DATA_LOCATION`をマウントポイント直下(`/mnt/ssd-pgdata`)からサブディレクトリ(`/mnt/ssd-pgdata/postgres`)へ変更して解決
- 全コンテナ`Up`確認 → `http://192.168.20.160:2283`にアクセスし**管理者アカウント作成に成功** → `docker exec immich-immich-machine-learning-1 nvidia-smi`でMLコンテナからGTX1650(4096MiB)を確認

### つまづきと解決 (次回のため・重要)
- **`docker compose config`は`.env`の変数を実際の値に展開して表示する**。秘密情報が入った`.env`がある状態で構文だけ確認したい場合は`docker compose config --quiet`を使う。今回のように誤って平文出力してしまった場合、初回起動(初期化)前なら値を再生成するだけで実害なく収束できる
- **`sed`の区切り文字とbase64文字列の`/`は衝突する**。置換対象の値に何が入るか分からない場合は`#`等の衝突しにくい区切り文字を使う
- **Immichの公式Postgresイメージのタグは頻繁に変わる**(バージョン管理拡張の名称変更等)。ドラフト時点の値を鵜呑みにせず、導入時に`github.com/immich-app/immich`の`docker/docker-compose.yml`・`docker/hwaccel.ml.yml`を直接取得して照合するのが確実
- **PostgresのDB_DATA_LOCATIONはマウントポイント直下ではなくサブディレクトリを指定する**。ext4等の`lost+found`があると「非空」判定でinitdbが失敗し続ける

### 決めたこと
- `configs/immich/docker-compose.yml`・`immich.env.example`を2026-08-04時点の実機確認値(イメージ参照・DB_DATA_LOCATIONパス)に更新し、次回以降このつまずきを再現しないようにした

### 未解決・次回やること
1. 実際に写真をアップロードし、NFS(`pic_tank`)書き込み・Postgresメタデータ保存を確認
2. NFS許可アドレスの絞り込み(暫定サブネット→VE1単体IP)
3. VE1再起動テスト(NFS/SSDマウント・Docker自動復旧)
4. Frigate統合、Cloudflare Tunnel外部公開は別セッションで着手

### 実機の状態
- 稼働中: Node0(Proxmox)、VE2(TrueNAS `.151`)、**VE1(`Ghome`, `192.168.20.160`) — Immich稼働中**(`immich-server`/`immich-machine-learning`/`redis`/`database`、`http://192.168.20.160:2283`)。管理者アカウント作成済み。GPU(GTX1650)・NFS(`pic_tank`)・SSD(`ssd-thin`→`/mnt/ssd-pgdata/postgres`)すべて連携動作確認済み

---

## 2026-08-04 (3) VE1構築 Step7完了(SSD/ssd-thinディスクアタッチ)

### やったこと
- VE1を`shutdown -h now`で正常停止 → Node0側で`qm set 100 -scsi1 ssd-thin:32,serial=VE1PGDATA`でPostgres用32GBディスクを追加 → `qm start 100`
- ゲスト内`lsblk`で確認したところ、**起動時のディスク割り当てがインストール時と入れ替わっていた**(OS本体が`sdb`、新規追加の`VE1PGDATA`が`sda`という並び)。`lsblk -o NAME,SIZE,SERIAL`でシリアル照合し`/dev/sda`=`VE1PGDATA`と確定してから`mkfs.ext4`実行(取り違え防止)
- `/mnt/ssd-pgdata`にマウント、UUID(`17117cb0-c2fc-46f1-90cf-87b274d3386c`)で`/etc/fstab`恒久化

### つまづきと解決 (次回のため)
- **ゲスト内のディスク名(`/dev/sda`/`sdb`)は再起動を挟むと入れ替わりうる**。今回も実際に入れ替わったが、事前にシリアルで照合していたため実害なし。逆に言えば照合を省いていたら誤ってブートディスクをフォーマットする事故になり得た。**「推測で叩かない」原則はゲストOS内のディスク操作にも適用する**

### 未解決・次回やること
1. Step8: `configs/immich/`のdocker-compose.yml/.envを配置しImmich起動
2. NFS許可アドレスの絞り込み(暫定サブネット→VE1単体IP `192.168.20.160`)

### 実機の状態
- 稼働中: Node0(Proxmox)、VE2(TrueNAS `.151`)、VE1(`Ghome`, `192.168.20.160`) — GPU/Docker/NFS(`pic_tank`)/SSD(`/mnt/ssd-pgdata`)すべて準備完了。Immich本体は未起動

---

## 2026-08-04 (2) VE1構築 Step6完了(NFSマウント、root_squash対応)

### やったこと
- `showmount -e 192.168.20.151` で実際のエクスポートパス(`/mnt/tank/pic_tank`, `/mnt/tank/cam_tank`)を確認 → 手順書のプレースホルダと一致
- `/mnt/nfs/pic_tank`にマウント、`df -h`で5.4TB利用可能を確認、`/etc/fstab`に恒久化エントリ追加
- 書き込みテスト(`touch`)で`Permission denied` → `pic_tank`が`root:root,755`のためNFS root_squashでVE1のroot権限が無権限ユーザーへ格下げされていたと判明
- TrueNAS GUI (`Shares → Unix Shares (NFS) → pic_tank → Advanced Options`) で `Maproot User=root` / `Maproot Group=root` を設定 → 書き込みテスト成功

### つまづきと解決 (次回のため)
- **NFS root_squashはハマりどころ**。エクスポート先が`root:root`所有かつ非トラステッドクライアント想定の権限だと、クライアントのroot書き込みが弾かれる。単一トラステッドVMからのアクセスなら`Maproot User/Group=root`で解消できる。`plan/06-principles.md`に一般原則として記録
- `cam_tank`(Frigate用)も同様のroot_squash設定が将来必要になる可能性が高い(未対応、Frigate統合時に要確認)

### 未解決・次回やること
1. Step7: VE1をシャットダウンし、`ssd-thin`からPostgres用ディスクを切り出してアタッチ
2. Step8: Immich docker-compose起動
3. NFS許可アドレスの絞り込み(暫定サブネット→VE1単体IP)

### 実機の状態
- 稼働中: Node0(Proxmox)、VE2(TrueNAS `.151`)、VE1(`Ghome`, `192.168.20.160`) — GPU+Docker動作確認済み、`tank/pic_tank`をNFSマウント・書き込み確認済み。SSD/Immichは未設定

---

## 2026-08-04 VE1構築 Step5完了(GPU+Docker+nvidia-container-toolkit)

### やったこと
- SSH経由の作業に移行。まず`apt install -y linux-headers-$(uname -r)`未実施のまま`nvidia-driver`を入れた結果`nvidia-smi`が失敗 → `dkms status`で`added`止まり(ビルド未完了)と判明 → `linux-headers`導入+`dkms autoinstall`でビルドし直し、`nvidia-smi`でGTX1650(4096MiB)認識を確認
- Docker/nvidia-container-toolkit導入コマンドを実行したところ`curl`/`gpg`が無く失敗 → Step4の「インストール後の最小設定」(`curl gnupg2 ca-certificates nfs-common`)を未実施だったことが判明、導入して解決
- `docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi` でコンテナ内からGPU認識を確認

### つまづきと解決 (次回のため)
- **`nvidia-driver`をdkmsでビルドするには`linux-headers-$(uname -r)`が必須**。無いと`dkms status`が`added`のまま`installed`にならず、`nvidia-smi`が「ドライバと通信できない」エラーになる
- **手順書の「インストール後の最小設定」(curl等の導入)を飛ばして先のステップに進んでしまうと、後工程でまとめてエラーになる**。手順の順序通りに進めることの重要性を再確認

### 未解決・次回やること
1. Step6: NFS(`tank/pic_tank`)を`/mnt/nfs/pic_tank`へマウント。実際のエクスポートパスは`showmount -e 192.168.20.151`で確認する(手順書のパスは未確認のプレースホルダ)
2. Step7: `ssd-thin`からPostgres用ディスクを切り出しVE1へアタッチ
3. Step8: Immich docker-compose起動

### 実機の状態
- 稼働中: Node0(Proxmox)、VE2(TrueNAS `.151`)、VE1(`Ghome`, `192.168.20.160`) — Debian13、NVIDIAドライバ+Docker+nvidia-container-toolkit導入済み・GPU認識確認済み。NFS/SSD/Immichは未設定

---

## 2026-08-01 (3) VE1構築 Step4完了(Debian 13インストール、SSH到達確認)

### やったこと
- `qm set 100 -hostpci0 0000:07:00,pcie=1,x-vga=0` 実行、`qm config`で反映確認 (Step3完了)
- Debian netinst ISOをダウンロード → 実際は`current`が指す**Debian 13(trixie)13.6.0**だった(当初想定の12は既にoldstable)
- `vga=qxl`設定→`ide2`にISOアタッチ→`boot order=ide2;scsi0;net0`で起動したが、**q35+OVMFでCD-ROMがブート候補に出ずPXE/HTTPブートへ流れる**事象が発生
- CD-ROMを`ide2`から**`scsi1`**へ繋ぎ直し(`qm stop`→`-delete ide2`→`-scsi1 ...,media=cdrom`→`-boot order=scsi1;scsi0;net0`→`qm start`)で解決、インストーラ起動を確認
- ガイドパーティショニング(LVM無し、ディスク全体)、ソフトウェア選択(デスクトップ環境オフ、SSHサーバ+標準システムユーティリティのみ)でインストール実施
- インストール完了後、稼働中VMから`scsi1`をホットアンプラグしようとして`hotplug problem`で失敗 → **`-boot order`を`scsi0;net0`に変更していれば実害無し**と判断しそのまま続行
- reboot後、noVNCコンソールの表示が更新されておらず「インストーラが再起動した」ように見えて混乱 → `qm status`/`ping`など別経路で確認したところ実際は正常起動していた(コンソール描画の問題)。最終的に`qm stop`→`qm start`で確実にクリーン起動させ、ログインプロンプト到達を確認
- SSHログインを試みたところrootパスワードをユーザーが失念 → コンソールから`passwd root`で再設定
- SSHは`PermitRootLogin`のデフォルト(`prohibit-password`)によりパスワード認証拒否 → `sed`で`PermitRootLogin yes`に変更(1回目は`Rootlogin`と大文字小文字を打ち間違えてマッチ失敗、`/i`オプション付きで再実行し解決)→ `systemctl restart ssh` → SSHログイン成功

### つまづきと解決 (次回のため・重要)
- **q35+OVMFではide2のCD-ROMがブートエントリとして認識されないことがある。SCSIバス(`scsi1`等)に繋ぐ方が確実**
- **稼働中VMのブートディスク扱いのデバイスはホットアンプラグ拒否される。`-boot order`変更で優先度を下げれば実害なく次回起動に進める**。確実に外すにはVM完全停止後に削除
- **noVNCコンソールはCtrl+Vペースト不可**(左端のクリップボードパネル使用 or 直接キー入力)。**VM再起動を挟むと表示が古いフレームのまま固まることがある**ため、コンソールの見た目だけで状態判断せず`qm status`/`ping`等で裏取りする
- **手打ちコマンドは大文字小文字の打ち間違いに注意**。`sed`の置換パターンでは`/i`(case-insensitive)フラグを付けておくと事故が減る
- Debianのデフォルト`sshd_config`は`PermitRootLogin prohibit-password`相当でパスワードでのrootログインを拒否する。内部限定VLANでの運用のため`yes`に変更(外部公開はしない前提)

### 決めたこと
- VE1のホスト名は**`Ghome`**(ユーザーが意図的に選択、`ve1`ではない)
- VE1のIPは**`192.168.20.160/24`**で確定

### 未解決・次回やること
1. Step5: NVIDIAドライバ+Docker+nvidia-container-toolkit導入(ゲスト内)
2. Step6-7: NFS(`tank/pic_tank`)/SSD(`ssd-thin`)マウント
3. Step8: Immich docker-compose起動

### 実機の状態
- 稼働中: Node0(Proxmox)、VE2(TrueNAS `.151`)、**VE1(`Ghome`, `192.168.20.160`, VMID=100)** — Debian 13、SSH到達可能、GPU/Docker/NFS/Immichは未設定
- Proxmoxストレージ: `ssd-thin`(LVM-Thin, 235.12GB)まだ未アタッチ

---

## 2026-08-01 (2) VE1構築 Step2完了(VM作成)

### やったこと
- `qm create` でVE1本体を作成。VMID提案は101だったが、ユーザー判断で**100**に変更
- スペックは`docs/ve1-immich-build.md`の提案値をそのまま採用: q35/OVMF、`scsi0`=local-zfs 32GB(`serial=VE1BOOT`,discard=on,ssd=1)、`net0`=vmbr0 tag20、6コア/RAM12GB/cpu=host、qemu-guest-agent有効
- efidisk0・scsi0とも正常作成を確認 (`transferred ... 100.00%`のログ)

### 決めたこと
- VMIDは101→**100**に変更(ユーザー判断)。以降の手順書・記録は100で統一

### 未解決・次回やること
1. Step3: `qm set 100 -hostpci0 0000:07:00,pcie=1,x-vga=0` でGPU passthrough追加
2. Step4: Debian 12 netinstをアタッチしOSインストール
3. Step5以降: Docker/nvidia-container-toolkit→NFS/SSDマウント→Immich起動

### 実機の状態
- 稼働中: Node0(Proxmox)、VE2(TrueNAS `.151`、稼働中)
- VE1(VMID=100): 作成済み(ディスク・NIC設定のみ)、GPU未接続、OS未インストール、未起動

---

## 2026-08-01 VE1構築 Step0-1完了(GTX1650 IOMMU実測・vfio-pciバインド)

### やったこと
- Step0: `lspci -nnk`でGTX1650のBDF確認 (`07:00.0`=VGA `10de:1f82`、`07:00.1`=Audio `10de:10fa`)。IOMMUグループ15を実測し、道連れデバイスなし(この2件のみ)を確認 → `docs/iommu-groups.md`に記録
- Step1: `/etc/modprobe.d/vfio.conf`が存在しないことに気づき調査 → 2026-07-23の案4撤回時に`/root/backup/vfio.conf.20260724-000854`へ退避済みと判明(想定通り、復元不要と判断)。VE2(GNAS)を`qm shutdown 200`で正常停止 → `blacklist-nouveau.conf`(nouveau無効化)と`vfio.conf`(GTX1650の2ID `10de:1f82,10de:10fa`のみ。旧SATA/USB用IDは案4撤回済みのため含めず)を新規作成 → `update-initramfs -u -k all && proxmox-boot-tool refresh` → `reboot` → `07:00.0`/`07:00.1`とも`vfio-pci`確認 → `qm start 200`でVE2復帰

### つまづきと解決 (次回のため)
- 手順書のファイル内容(`blacklist nouveau`等)をコードブロックでそのまま提示したところ、ユーザーがシェルにペーストしコマンドとして実行を試みるミスが発生(実害なし、command not foundのみ)。**設定ファイルの中身を提示する際は必ず`cat > file <<'EOF' ... EOF`のヒアドキュメント形式で「実行すべきコマンド」として渡す**ことを徹底する
- ホストの`reboot`は全VM(VE2含む)を巻き込むため、**host reboot前に稼働中VMを正常停止する**ひと手間を追加した(ZFSは電源断でも無傷だった実績はあるが、正常停止の方が望ましい)

### 決めたこと
- 退避済みの旧`vfio.conf`(案4のSATA/USB コントローラ用ID)は**復元しない**。現在のVE2は案2(ディスク単位パススルー)で稼働しており、コントローラ単位パススルーの設定は不要かつ、復元するとグループ14事故の再発リスクがある

### 未解決・次回やること
1. Step2: VE1 VM作成 (`qm create 101`、RAM/コア数等の提案値をユーザーと確認)
2. Step3: `hostpci0`でGTX1650(グループ15、`0000:07:00`)をVE1へ追加
3. Step4-8: OS/Docker/nvidia-container-toolkit→NFS/SSDマウント→Immich起動

### 実機の状態
- 稼働中: Node0(Proxmox, VLAN20 `.150`)、VE2(TrueNAS `.151`, 再起動済み・稼働中)
- GTX1650(`07:00.0`/`07:00.1`): `vfio-pci`バインド済み、ホストからは使用不可(意図通り、VE1への割当待ち)
- 未構築: VE1(VM本体はまだ未作成)、VE3〜VE6

---

## 2026-07-31 VE1(Immich)構築手順ドラフト作成

### やったこと
- `worklog.md`/`plan/03-proxmox.md`を確認 → 前提条件(VE2ストレージ層・SSD LVM-thin化)が完了済みで、VE1構築が次の一手の筆頭になっていることを確認
- 作業ブランチ(`claude/ve1-immich-setup-297x6f`)が古いコミット(`49f8297`)から分岐しており、mainの最新9コミット(TrueNASプール作成・データセット/NFS/SMB共有・SSD LVM-thin化)を含んでいなかったため、`origin/main`へfast-forwardして最新化
- VE1構築手順のドラフト `docs/ve1-immich-build.md` を新規作成: GPU IOMMUグループ実測→nouveauブラックリスト(レベルC)→VM作成→GPU passthrough→OS/Docker/nvidia-container-toolkit→NFS(`tank/pic_tank`)/SSD(`ssd-thin`)マウント→Immich起動、の8ステップ
- Immichのdocker-compose雛形を作成: `configs/immich/docker-compose.yml`、`configs/immich/immich.env.example`(秘密情報は含まず、`.env`自体は`.gitignore`対象)
- `plan/03-proxmox.md`のVE1行・Postgres配置セクションから新規ドキュメントへリンクを追加

### 決めたこと
- **本セッションでは実機コマンドを実行しない**。2026-07-22に決めた役割分担(実機操作はユーザー本人、Claudeは手順設計・ドキュメント整備)を踏襲。このセッション自体もNode0実機にアクセスできないリモート環境のため、方針と実態が一致
- 手順書のスペック値(VM RAM/コア数、ディスク容量等)は**提案値**として明記し、確定値としては書かない。実機未確認のGTX1650のPCI IDやIOMMUグループも同様にプレースホルダとし、Step0の実測を経てから埋める設計とした(CLAUDE.md §2「推測で叩かない」)
- Frigateのdocker-compose統合は本ドラフトのスコープ外とし、後続タスクとして明記(ブランチ名`ve1-immich-setup`のスコープに合わせた)

### 未解決・次回やること
1. `docs/ve1-immich-build.md` Step0: GTX1650のIOMMUグループを実機で実測 (グループ15と推定、未確定)
2. Step1のnouveauブラックリスト化(レベルC)を承認のうえ実行 → reboot → Step2以降でVE1構築
3. NFS許可アドレスの絞り込み、Cloudflare Tunnel外部公開はVE1構築完了後

### 実機の状態
- 変更なし(ドキュメント整備のみ、実機操作は未実施)。稼働中: Node0(Proxmox, VLAN20 `.150`)、VE2(TrueNAS `.151`, プール`tank` Healthy・共有設定済み)。Proxmoxストレージ: `ssd-thin`(LVM-Thin, 235.12GB)未アタッチ。未構築: VE1〜VE6

---

## 2026-07-24 (6) 【ヒヤリハット】LVM-Thinpool作成ダイアログの罠、SSD LVM-Thin化完了

### やったこと
- Mac→`kenji`/`miho`でSMB接続テスト実施、`doc_tank`へのアクセス確認OK
- Proxmox GUI `Disks → LVM-Thin → Create: Thin Pool` でSSDをThinpool化しようとしたところ、**ディスク選択肢に6TB(`/dev/sdb`, VE2使用中)しか出ず、SSD(`/dev/sda`)が出ない**事象が発生。ユーザーが気づいてダイアログを開いたまま報告、実行前に停止できた(実害なし)
- 原因調査: `/dev/sda`(SanDisk)に旧NTFSパーティション(sda1/2/3)が残存しており、**Proxmoxはパーティション/FSが検出済みのディスクを候補から除外する**。一方`/dev/sdb`(6TB)は現在VE2が使用中にもかかわらず、ホスト側のパーティションテーブルとしては未検出のため**「未使用」として候補に出てしまう**
- 対処: ダイアログを一旦閉じ、`Node0 → Disks`で`/dev/sda`(シリアル`154778407406`で再確認)を選択→**「ディスクの消去」**でNTFS残骸をクリア→再度`Create: Thin Pool`を開くと`/dev/sda`が選択可能になり、Thinpool `ssd-thin`(Volume Group `ssd-thin`, 235.12GB)を作成・Proxmoxストレージとして登録

### つまづきと解決 (次回のため・重要)
- **ProxmoxのLVM-Thinpool作成ダイアログの「候補ディスク一覧」は安全装置として信用できない。** 既存のパーティション/FS情報の有無だけで候補を出し分けており、「実際に他のVMが使用中かどうか」は考慮されない。**稼働中のVM(特にTrueNASのようなディスクパススルー先)が使っているディスクでも平気で選択肢に出てくる。** → **今後同様の作業では、ダイアログの選択肢を鵜呑みにせず、必ずシリアル番号で対象を照合してから選択・実行する**(今回はこの原則を徹底していたおかげでヒヤリハットで済んだ)

### 決めたこと
- SSDのストレージ化方式は**LVM-Thin**(名前`ssd-thin`)で確定。ZFS-on-ZFSのオーバーヘッドを避けるための当初方針通り

### 未解決・次回やること
1. VE1構築時に`ssd-thin`からPostgres専用ディスクを切り出してアタッチ(まだ未実施、Thinpool作成のみ完了)

### 実機の状態
- 稼働中: Node0(Proxmox, VLAN20 `.150`)、VE2(TrueNAS `.151`, プール`tank` Healthy、共有設定済み・SMB接続確認済み)
- Proxmoxストレージ: `ssd-thin`(LVM-Thin, 235.12GB)追加済み、未アタッチ
- 未構築: VE1〜VE6

---

## 2026-07-24 (5) データセット作成・NFS/SMB共有設定完了

### やったこと
- `tank`直下に3データセット作成: `pic_tank`(汎用, recordsize=1M, lz4)、`cam_tank`(汎用, recordsize=1M, lz4)、`doc_tank`(SMBプリセット, recordsize=デフォルト128K, lz4)
  - 初回作成時に`cam_tank`→`pic_tank`の子、`doc_tank`→`cam_tank`の子という誤った入れ子になったため、空の状態で削除し`tank`直下に作り直して解決
- NFS共有作成: `pic_tank`・`cam_tank` を Unix Shares(NFS) で公開。許可ネットワークは暫定で`192.168.20.0/24`(VLAN20全体)。VE1未構築のためホスト単位への絞り込みは保留
- ローカルユーザー`kenji`・`miho`を作成(SMBアクセスのみ有効化。TrueNASアクセス/Full Adminは付与せず最小権限に)。ホームディレクトリは`doc_tank`配下に自動提案されたが、共有ドキュメント領域に個人フォルダが混在するのを避けるためデフォルトに変更
- 共有グループ`G-home`(GID 3002, Samba認証=有効)を作成し、`kenji`・`miho`を補助グループとして追加
- `doc_tank`のACLエディタで Owner Group を `root` → `G-home` に変更(オーナーは`kenji`のまま)、`group@`に修正(読み書き)権限を付与して保存
- SMB共有(`doc_tank`)を作成・有効化

### つまづきと解決 (次回のため)
- **ユーザー作成時、デフォルトで「TrueNASアクセス」がFull Adminで有効になっていた。** 家族の共有ファイルアクセス用アカウントなので、管理画面へのアクセスは不要と判断しチェックを外した(最小権限)。ユーザー追加時は毎回この項目を確認すること
- **グループ作成画面には「メンバー追加」欄が無い。** メンバー登録は各ユーザーの編集画面の「補助グループ」から行う(グループ側からの一括追加はできない)
- **ユーザー作成で自動生成される「プライベートグループ」(ユーザー名と同名, 例: `kenji`グループ)は正常な仕様。** 削除しようとしてもプライマリグループとして使用中のため削除できないが、これは想定通りで放置してよい。共有アクセス制御に使うのは別途作成した`G-home`グループのみ

### 未解決・次回やること
1. Macから`kenji`・`miho`両アカウントでSMB接続テスト(`smb://192.168.20.151/doc_tank`)
2. VE1構築後、NFS許可を`192.168.20.0/24`からVE1の確定IPへ絞り込み
3. SSD(240GB)のLVM-thin化 → VE1へPostgres専用ディスクとしてアタッチ

### 実機の状態
- 稼働中: Node0(Proxmox, VLAN20 `.150`)、VE2(TrueNAS `.151`, プール`tank` Healthy、データセット3つ+共有設定済み)
- ホスト保持・未使用: SSD 240GB(`sda`)
- 未構築: VE1〜VE6

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
