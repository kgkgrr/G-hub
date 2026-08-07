# Immich 初回一括投入 — iCloud / Google Photos (ドラフト)

← [worklog](worklog.md) / [ve1-immich-build.md](ve1-immich-build.md) / [plan/03-proxmox.md](../plan/03-proxmox.md)

> **この手順書の位置づけ**: `ve1-immich-build.md` と同じ役割分担。Claudeは手順設計・レビュー・ドキュメント整備を担当し、
> 実機(VE1)・作業端末(Mac)でのコマンド実行はユーザー本人が行う。以下はレビュー用のドラフト。
> パッケージ名・CLIオプションは記載時点の情報のため、実行前に各ツールの最新ドキュメントと差分がないか確認すること。

最終更新: 2026-08-07

---

## 前提

- 作業端末: **Mac**(写真.appでiCloud写真ライブラリと同期)
- iCloud側が「オリジナルをダウンロード」設定済みでMac上にフル実体があるか、iCloudクラウド上のみ(サムネのみローカル)かは**未確認**。9-1で切り分ける
- 投入先: Immich (`http://192.168.20.160:2283`)、実体は `tank/pic_tank` (NFS、2026-08-06時点で5.4TB利用可能)
- iCloudとGoogle Photosの両方に、同期済みで重複する写真が相当数含まれる可能性が高い

---

## Step 1: 現状確認 (iCloudの実体所在)

Mac の「写真」アプリ → 環境設定 → iCloud タブで **「このMacに元のフルサイズ写真をダウンロード」** が有効かを確認する。

- **有効(フル実体あり)**: そのままローカルライブラリからエクスポート可能。Step 2-A へ
- **無効(iCloud上のみ)**: エクスポート時に都度iCloudからダウンロードが走る (回線・時間がかかる)。設定を有効に変更してから数時間〜数日待ってフル同期させるか、`osxphotos` の `--download-missing` オプションでダウンロードしながらエクスポートする (Step 2-A の注記参照)

---

## Step 2-A: iCloud (写真.app) → フォルダへエクスポート

**推奨ツール: [`osxphotos`](https://github.com/RhetTbull/osxphotos)** (Mac用OSS CLI、`brew install osxphotos`)。
GUIの `File > Export > Export Unmodified Original(s)` でも可能だが、大量枚数では非現実的 (手動選択・進捗管理が困難)。`osxphotos` はライブラリ全体を自動でファイル書き出しでき、EXIF・アルバム構成・位置情報・キーワードを維持できる。

```bash
brew install osxphotos

# ライブラリ全体をアルバム構造を保ったまま書き出す例 (提案値、要調整)
osxphotos export ~/Desktop/icloud-export \
  --original-name \
  --album-folders \
  --download-missing \
  --skip-live-photos-mp4=false \
  --exiftool
```

- `--download-missing`: iCloud上のみで未ダウンロードの写真もその場で取得する (Step 1で「無効」だった場合の保険。時間がかかる)
- `--album-folders`: アルバム構造をディレクトリ階層として再現 (Immich CLI側で `--album-name` の代わりにディレクトリ名をアルバム名として使う運用と相性が良い)
- `--exiftool`: exiftoolを使ってEXIFメタデータの整合性を強化 (要 `brew install exiftool`)
- 書き出し先 (`~/Desktop/icloud-export` 等) はMacのローカルディスク容量を消費する。ライブラリ全量だと数百GB〜規模になりうるため、**事前にMac側の空き容量を確認すること**

---

## Step 2-B: Google Photos (Google Takeout) → 前処理

1. [Google Takeout](https://takeout.google.com/) で「Google フォト」のみを選択してエクスポート依頼 (zip分割サイズは扱いやすい単位、例: 10GB刻みを推奨)
2. ダウンロードしたzip群をMac上で展開する
3. **展開したフォルダをそのままImmichへ投入しない。** Google Takeoutは各写真に対応する `<ファイル名>.supplemental-metadata.json` サイドカーを生成し、**撮影日時等の正しいメタデータがEXIFではなくこのJSON側にしか無いケースがある**(特にスクリーンショット、編集後ファイル、モバイルアップロード動画)。EXIFのまま投入すると、Immichのタイムライン上で日付が「取り込み日」や不正な値になる不具合が起きやすい
4. **[Google Photos Takeout Helper (GPTH)](https://github.com/TheLastGimbus/GooglePhotosTakeoutHelper)** で前処理する:
   - JSONサイドカーの日時をEXIF/ファイルmtimeへ書き戻す
   - Live Photos (HEIC+MP4等のペア) の関連付けを維持
   - アルバムごとのシンボリックリンク/フォルダ復元
   - 重複排除

```bash
# GPTHはMac/Win/Linuxバイナリ配布。brewでは提供されていない場合、GitHub Releasesから取得
./gpth --input ~/Downloads/Takeout --output ~/Desktop/google-export --divide-to-dates
```

具体的なCLIオプションは配布時点のREleaseノートを確認 (バージョンでオプション名が変わりやすいツール)。

---

## Step 3: Immich CLI で一括アップロード

```bash
npm install -g @immich/cli
# もしくは都度: npx @immich/cli ...

immich login  # サーバーURLとAPIキーを対話入力 (下記参照)
immich upload --recursive --album-name "iCloud Import" ~/Desktop/icloud-export
immich upload --recursive --album-name "Google Photos Import" ~/Desktop/google-export
```

- **APIキーの取得**: Immich管理画面 → Account Settings → API Keys → 発行。**このキーは秘密情報。会話ログやGitに絶対貼らない/コミットしない** (`CLAUDE.md` §6)。`immich login` はキーをローカルの認証情報ファイルに保存するので、コマンドライン引数に直書きするより安全
- サーバー側でファイルの**チェックサム(SHA1)完全一致**による重複検知が働き、同一バイナリの再アップロードは自動スキップされる (Immich CLI/サーバーの既定動作)
- Immich CLIの正確なサブコマンド/オプション名は [公式ドキュメント](https://immich.app/docs/features/command-line-interface) をインストール時に確認すること (頻繁に更新される)

### 投入順序の推奨

1. **iCloud (Step 2-A) を先に投入**
2. 次に **Google Photos (Step 2-B) を投入**

理由: 同じ写真がGoogle Photos側で再圧縮/変換されていると、iCloud版とバイナリが異なりチェックサム一致による自動重複検知が効かない。iCloud(通常オリジナル品質)を先に確定させておき、後から入れるGoogle Photos側で「本当に新規のものだけ」を見分けやすくする

---

## Step 4: 投入後の重複レビュー

チェックサム一致以外の重複(見た目は同じだがバイナリが異なるもの)は自動では弾かれない。全バッチ投入完了後、Immich管理画面 **Administration → Duplicates** (Smart Search/CLIP類似度ベースの重複検出ジョブ)でレビューし、手動でマージ/削除する。**全件投入の途中では実施しない**(ジョブキューが埋まっている間は検出精度・速度が落ちるため、投入完了後に一度でよい)。

---

## Step 5: バッチ分割とジョブキュー負荷の管理

大量枚数 (数千〜数万規模) を一度に投げると、以下の理由から**バッチ分割を推奨**する。

- アップロードごとにサムネイル生成・Smart Search (CLIP embedding)・顔検出/認識のジョブがキューに積まれ、GTX1650 (VRAM 4GB) で順次処理される。一括投入するとキューが長時間(数日規模もありうる)詰まる
- `plan/04-gpu-ai.md` の調整方針: OOM/VRAM逼迫が出た場合は Administration → Settings → Machine Learning の同時実行数(concurrency)を1に制限する
- pic_tank側の空き容量(2026-08-06時点で5.4TB利用可能)を、大きいバッチの前に都度TrueNAS側で確認する習慣にする

**運用案**: 年別 or アルバム別にフォルダを分け、1バッチ数千枚程度に区切って投入 → Immich管理画面の **Jobs** ページでキューが捌けたことを確認 → 次バッチ、という進め方にする。

> **CLAUDE.mdとの関係**: 「30分以上かかる処理の開始」はレベルB2(説明+承認)に相当しうる規模。**最初の1バッチを投入する前に、概算の総枚数・容量とバッチ計画をすり合わせてから開始する**運用とする。

---

## 未実施

- Step 1〜5 とも設計のみ。実機(Mac / Immich管理画面)での実行はこれから
- iCloud側のダウンロード状況の確認 (Step 1) が最初の一手
- VLAN10(Mac)→VLAN20(VE1)のポート2283疎通は基本許可されている想定だが、実アップロード前に軽く確認しておく (`plan/02-network.md` のVLAN間フィルター参照)
