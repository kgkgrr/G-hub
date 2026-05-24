# サイネージ端末 セットアップ TODO

対象機: ARROWS Tab Q7310 / Windows 11 / 8GB RAM

## 1. Sophia Script（デブロート）
- [ ] PowerShell を管理者で起動
- [ ] `irm script.sophi.app | iex` を実行
- [ ] GUI で以下を適用
  - SysMain（Superfetch）停止
  - Windows Search 停止
  - テレメトリ・診断データ送信 無効化
  - Xbox 関連サービス 無効化
  - 不要なスタートアップアプリ 無効化

## 2. 手動サービス停止（services.msc）
- [ ] Print Spooler → 無効
- [ ] Fax → 無効
- [ ] Connected User Experiences and Telemetry → 無効
- [ ] Windows Update → 手動（止めすぎ注意）

## 3. Assigned Access（キオスクアカウント）
- [ ] 設定 → アカウント → 他のユーザー → キオスクの設定
- [ ] 新規キオスクアカウントを作成
- [ ] アプリとして Microsoft Edge を割り当て

## 4. signage.html をローカルに配置
- [ ] `C:\signage\signage.html` に配置
- [ ] Edge キオスク起動コマンドを設定
  ```
  msedge.exe --kiosk "file:///C:/signage/signage.html" --edge-kiosk-type=fullscreen
  ```

## 5. 動作確認
- [ ] アイドルメモリが 2GB 前後に収まっていること
- [ ] タッチでナビボタンが反応すること
- [ ] 15秒ごとに自動ローテーションすること
- [ ] 60秒ホールド後に自動復帰すること
- [ ] 時計が常時表示されていること
