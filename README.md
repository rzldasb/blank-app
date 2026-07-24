# GL840 1号機・2号機 運用・保守マニュアル

| 項目 | 内容 |
| --- | --- |
| 文書番号 | GL840-OPS-JA-001 |
| 版 | 第2.0版 |
| 更新日 | 2026年7月24日 |
| 対象 | LS研ラボ培養監視システム 1号機・2号機 |
| 基準時刻 | 日本時間（JST） |

---

## 現行構成（2026-07-24）

1号機と2号機は、同じ共有実装を使用する独立した2インスタンス構成です。

```text
/home/kenkyu01/Documents/IoT_shared/gl840_core
    └─ 収集、RGB、MQTT outbox、ローカルAPI、履歴、CSV、Dashboardの共通実装

/home/kenkyu01/Documents/IoT_test
    └─ 1号機固有の設定、DB、証明書、起動ラッパー

/home/kenkyu01/Documents/IoT_test2
    └─ 2号機固有の設定、DB、証明書、起動ラッパー
```

| 項目 | 1号機 | 2号機 |
| --- | --- | --- |
| ローカルAPI | `127.0.0.1:2100` | `127.0.0.1:2101` |
| Dashboard | `8501` | `8502` |
| 測定制御サービス | `gl840-worker.service` | `gl840-console2.service` |
| 画面サービス | `gl840-dashboard.service` | `gl840-dashboard2.service` |
| 正式DB | `IoT_test/data/gl840_console.db` | `IoT_test2/data/gl840_console2.db` |

4つのsystemdサービスはOS起動時に起動しますが、測定制御サービスは必ず待機状態で開始します。サービスの「起動済み」またはsystemdの`active`は、操作を受け付けられることを示すだけで、センサー測定中またはAWS送信中という意味ではありません。

測定は、Dashboardで対象チャンネルまたはRGBセンサーを選択し、AWS送信を有効にする確認欄を選択したうえで、「測定開始」を押した場合に限って始まります。自動測定開始は無効です。測定停止後のMQTT outbox処理は最長13秒で終了し、未送信データが残っていてもAWS IoTとの接続を切断して待機状態へ戻ります。

## 1. 本書の目的

本書は、GL840 1号機・2号機を安全に使用し、測定データを失わずに運用・保守するための手順をまとめたものです。

対象者は次のとおりです。

- 日常の測定を行う利用者
- Dashboardで測定値や履歴を確認する利用者
- CSVを出力する利用者
- Linux、systemd、SQLite、ネットワークを保守する担当者

本書では、画面上の操作を「測定」で統一します。サービスやデータベース内部の技術用語が必要な場合のみ、その名称を併記します。

## 2. 最重要ルール

> **測定中に測定サービスを再起動しないでください。**  
> 再起動すると、進行中の測定は正常終了ではなく「中断」として記録されます。必ずDashboardの「測定停止」で停止してから保守作業を行ってください。

> **1号機と2号機でRGBセンサーを同時に使用しないでください。**  
> 現在は1台のM5 RGBモジュールを共用しています。システム側に排他制御はないため、両方で同時に有効にすると、同じRGB値が別々の測定へ保存される可能性があります。

> **サービス稼働中に起動スクリプトを手動で実行しないでください。**  
> 正式な起動方法はsystemdです。二重起動すると、SQLite、ポート、AWS IoTのクライアントIDやトピックが競合します。

> **AWS送信待ちデータを削除しないでください。**  
> SQLiteは測定記録であると同時に、AWS IoTへ送信するための永続キューです。ネットワーク障害時もデータはローカルに残ります。待機中はAWSへ接続しません。未送信データは、利用者が次の測定を明示的に開始してAWS送信を確認した後に再送されます。

## 3. システム構成

### 3.1 構成一覧

| 項目 | 1号機 | 2号機 |
| --- | --- | --- |
| プロジェクト | `/home/kenkyu01/Documents/IoT_test` | `/home/kenkyu01/Documents/IoT_test2` |
| 共有実装 | `/home/kenkyu01/Documents/IoT_shared/gl840_core` | 同左 |
| 測定サービス | `gl840-worker.service` | `gl840-console2.service` |
| Dashboard | `gl840-dashboard.service` | `gl840-dashboard2.service` |
| Dashboardポート | `8501` | `8502` |
| ローカル制御API | `127.0.0.1:2100` | `127.0.0.1:2101` |
| 正式DB | `data/gl840_console.db` | `data/gl840_console2.db` |
| 主設定 | `.env` | `.env` |
| 保守ロック | `.maintenance.env` | `.maintenance.env` |
| チャンネル設定 | `data/channel_config.json` | `data/channel_config.json` |
| 測定サービス起動入口 | `scripts/run_backend.sh` | `scripts/run_backend.sh` |
| Dashboard起動入口 | `scripts/run_dashboard.sh` | `scripts/run_dashboard.sh` |

各プロジェクト内のPythonモジュールと`run_backend.sh`／`run_dashboard.sh`は、インスタンス固有の設定を読み込み、共有実装を呼び出すための薄いラッパーです。共有機能を修正する場合は、両プロジェクトへ同じコードを複製せず、`IoT_shared/gl840_core`を更新します。

現在、プロジェクト直下に正式な`run.sh`は存在しません。`scripts/`内の起動スクリプトはsystemdから使用されるため、サービス稼働中に手動実行しないでください。

### 3.2 データの流れ

```text
ブラウザー
   └─→ Streamlit Dashboard
          └─→ localhost API
                 └─→ 共有実装 gl840_core
                        ├─→ GL840／M5 RGB
                        ├─→ インスタンス固有SQLite
                        └─→ MQTT outbox ─→ AWS IoT Core
```

- Dashboardからの測定開始・停止は、各号機のlocalhost APIを経由します。
- 2100番・2101番APIはサーバー内部だけで使用し、LANやインターネットへ公開しません。
- GL840とRGBセンサーは独立して取得されます。
- 片方で通信エラーが起きても、取得できた側のデータは保存されます。
- AWS IoTへ送信できない場合も、測定データはSQLiteに残ります。
- サービスが`active`でも、測定中とは限りません。測定状態はDashboardで確認してください。

## 4. Dashboardへの接続

同じネットワーク内のPCから、次のURLを開きます。

| システム | URL |
| --- | --- |
| 1号機 | `http://<ホストのIPアドレス>:8501` |
| 2号機 | `http://<ホストのIPアドレス>:8502` |

現在のホストIPが`192.168.1.201`の場合の例：

```text
http://192.168.1.201:8501
http://192.168.1.201:8502
```

ローカルでのヘルスチェック：

```bash
curl -fsS http://127.0.0.1:8501/_stcore/health
curl -fsS http://127.0.0.1:8502/_stcore/health
curl -fsS http://127.0.0.1:2100/api/status
curl -fsS http://127.0.0.1:2101/api/status
```

Dashboardのヘルスチェックは正常時に`ok`を返します。APIは測定状態を含むJSONを返します。

`127.0.0.1:2100`および`127.0.0.1:2101`を使用するコマンドは、Dashboardを開いているクライアントPCではなく、IoTサーバー本体の端末で実行してください。

## 5. 測定を開始する

### 5.1 測定前チェック

測定を始める前に、次を確認します。

- GL840の電源が入っている。
- GL840がLANに接続されている。
- 使用するセンサーとGL840チャンネルの配線が一致している。
- Dashboardの「測定制御サービス」と「画面サービス」が「起動済み」になっている。
- 「測定状態」が「待機中」になっている。
- 「AWS送信隔離」に未確認のメッセージがない。
- RGBを使用する場合、M5の電源とWi-Fi接続が正常である。
- RGBを使用するシステムが1号機か2号機のどちらか一方に決まっている。
- もう一方のシステムでRGBが選択されていない。

### 5.2 測定開始手順

1. Dashboardの「測定操作」を開きます。
2. 「バッチID」を入力します。
3. 「測定チャンネル」で使用するCHを選択します。
4. RGBを使用する場合だけ「RGBセンサーを使用する」をオンにします。
5. 必要に応じて「担当者」「株」「培地」「メモ」を入力します。
6. 「測定開始と同時にAWS送信を有効にすることを確認しました。」を選択します。
7. 「測定開始」を1回押します。
8. 「測定中：測定ID（AWS送信有効）」が表示されることを確認します。
9. 「モニタリング」を開き、最初の測定値と「最終測定時刻」が更新されることを確認します。

「測定開始を受け付けました」という表示は、開始処理の完了を意味しません。「測定中」へ変わり、最初の測定値が保存されたことを確認してください。

バッチIDは、両システムで共通して扱いやすいように、64文字以内の英数字、ハイフン、アンダースコアを推奨します。

現在の標準測定間隔は約30秒です。測定開始直後は、次の案内が表示されることがあります。

```text
測定を開始しました。最初の測定値を待っています。
```

最初の測定値が保存されるまで、開始ボタンを繰り返し押さないでください。

「起動済み」は、Dashboardから操作を受け付けられる状態を示します。「測定中」という意味ではありません。通常の待機中（測定停止後、最長13秒のoutbox排出処理中を除く）は、AWS送信待ち件数にかかわらず、センサー測定、AWS IoTへの接続、AWSへの送信を開始しません。

測定停止後はoutboxの排出処理を行いますが、処理時間には上限があります。現行設定は`STOP_WAIT_TIMEOUT_SEC=13`で、13秒以内に送信できなかった行はSQLiteへ安全に保持し、AWS接続を切断します。待機中に再接続することはなく、残った行は利用者が次の測定を明示的に開始してAWS送信を確認した後に再送されます。

### 5.3 両号機共通の制御方式

1号機と2号機は、いずれもDashboardから各号機のローカルAPIを呼び出して測定を制御します。SQLiteを測定開始・停止の命令キューとして使用しません。

- Dashboardサービスが起動済みでも、測定制御サービスが正常とは限りません。
- 1号機で開始できない場合は、`http://127.0.0.1:2100/api/status`を確認してください。
- 2号機で開始できない場合は、`http://127.0.0.1:2101/api/status`を確認してください。
- チャンネルもRGBも選択していない場合は開始できません。
- AWS送信の確認がない開始要求は拒否されます。
- 同時に複数の測定を開始することはできません。
- APIの制御要求はメモリー内だけで処理され、サービス再起動後に古い開始要求が実行されることはありません。
- バッチIDには英数字、ハイフン、アンダースコアを使用してください。

## 6. 測定を停止する

1. 「測定操作」を開きます。
2. 「測定停止」を1回押します。
3. 停止受付の案内を確認します。
4. 測定状態が「完了」または「待機中」になるまで待ちます。
5. 停止後のoutbox処理が完了するか、最長13秒の排出時間が終了し、AWS接続が切断されたことを画面または`/api/status`で確認します。
6. 必要に応じて「CSV出力」からデータを保存します。

13秒の上限時点でAWS送信待ちが残っていても異常終了ではありません。データはSQLiteに残り、待機中にAWSへ再接続することはありません。測定停止によって、保存済みデータや送信待ちデータが削除されることはありません。

次の操作は避けてください。

- 測定停止直後にサービスを再起動する
- 停止ボタンを連打する
- 測定中に電源を切る
- 測定中にデータベースを移動する

## 7. モニタリング

### 7.1 表示条件

「モニタリング」では、次の条件を選択できます。

| 項目 | 内容 |
| --- | --- |
| 表示する測定 | 「最新」または測定ID |
| 最新の表示期間 | 1、6、12、24、72、168時間 |
| 初期表示期間 | 6時間 |
| 過去の測定 | 測定開始から終了まで |
| 最大表示点数 | 100～5,000点 |
| 初期表示点数 | 2,000点 |
| 自動更新 | 初期状態はオフ |
| 更新間隔 | 2～60秒、初期値5秒 |

表示点数を制限しても、SQLite内の測定データは削除されません。グラフ表示だけが間引かれます。全データが必要な場合はCSVを使用してください。

### 7.2 表示される状態

上段には次の6項目が表示されます。

1. 測定ID
2. 正常チャンネル
3. RGBセンサー
4. AWS送信待ち
5. AWS送信隔離
6. 最終測定時刻

グラフは次の3種類です。

- 現在値ゲージ
- 時系列グラフ
- 面グラフ

RGBは次の名称で表示されます。

- RGB（赤）
- RGB（緑）
- RGB（青）
- RGB（クリア）

### 7.3 現在値の考え方

「最新」表示では、測定中で、かつ一定時間以内に取得された値だけを現在値として表示します。

測定していない場合：

```text
現在は測定を行っていません。過去の測定値は現在値として表示されません。
```

最終測定から基準時間を超えた場合、現在値は非表示になります。現行の判定基準は90秒です。

次の値は正常値として扱われません。

- 空欄
- `----`
- 非数値
- チャンネルがオフ
- 最新値の状態が正常でない

過去の正常値を現在値として再利用することはありません。過去の結果を確認する場合は、測定IDを明示的に選択してください。

### 7.4 正常チャンネルが少ない場合

1. GL840本体で対象チャンネルがオンになっているか確認します。
2. 配線とチャンネル番号を確認します。
3. GL840のWeb画面で値が表示されているか確認します。
4. 対象機の測定制御サービスのログを確認します。
5. 空欄や非数値の場合は、センサー、変換器、入力レンジを確認します。

## 8. 共有RGBセンサー

### 8.1 現行構成

1号機と2号機は、同じM5 RGBモジュールを使用します。

| 項目 | 現行設定 |
| --- | --- |
| URL | `http://192.168.1.209/rgb` |
| タイムアウト | 3秒 |
| 電源 | 独立電源 |
| 接続 | Wi-Fi |

### 8.2 使用ルール

1. 測定前に、RGBモジュールを測定対象へ設置します。
2. 1号機または2号機のどちらで記録するか決めます。
3. 選んだ側だけで「RGBセンサーを使用する」をオンにします。
4. もう一方では必ずオフにします。
5. 測定中にRGBモジュールを別の対象へ移動しないでください。
6. 移動が必要な場合は、いったん測定を停止し、メモへ変更内容を記録してから新しい測定を開始します。

### 8.3 動作確認

M5へ直接接続して確認：

```bash
curl -fsS --max-time 5 http://192.168.1.209/rgb
```

各号機のバックエンド経由で確認：

```bash
curl -fsS http://127.0.0.1:2100/api/rgb_test
curl -fsS http://127.0.0.1:2101/api/rgb_test
```

正常時は、JSON内の`ok`と`sensor`が`true`で、`r`、`g`、`b`、`c`が取得できます。

RGBだけがエラーの場合でも、GL840のデータ取得は継続します。

## 9. 現行チャンネル対応

1号機と2号機では、チャンネル配置が同一ではありません。測定前に必ず対象機の対応表を確認してください。

| 用途 | 1号機 | 2号機 |
| --- | --- | --- |
| Glucose A | CH01 | CH01 |
| Glucose B | CH02 | CH07 |
| pH A | CH04 | CH03 |
| pH B | CH07 | CH08 |
| DO A | CH05 | CH04 |
| DO B | CH08 | CH09 |

その他のチャンネルは`CHxx`として登録されています。

配線を変更した場合は、次のファイルも更新し、変更日時と担当者を記録してください。

```text
/home/kenkyu01/Documents/IoT_test/data/channel_config.json
/home/kenkyu01/Documents/IoT_test2/data/channel_config.json
```

チャンネル名だけを見て判断せず、GL840本体の物理配線も確認してください。

## 10. CSV出力

1号機と2号機は、`IoT_shared/gl840_core/csv_export.py`の同じCSV実装を使用します。列順、日時変換、絞り込み、スナップショット方式は共通で、出力元DBだけが号機ごとに分かれます。

- CSVは固定列で、CH01～CH20、RGB、測定メタデータ、MQTT送信状態を同じ形式で出力します。
- 日時条件はJSTで入力し、CSVにはエポック秒、UTC、JSTの時刻列を出力します。
- 文字コードはBOM付きUTF-8です。
- 全件出力は作成開始時点のスナップショットを、分割して読み出します。
- 表計算ソフトで数式として解釈され得る文字列は、安全な形式へ変換されます。

### 10.1 Dashboardから出力する

「CSV出力」では次の条件を指定できます。

- バッチ
- 測定ID
- 種別：データ、イベント、すべて
- プレビュー件数：100、500、1,000、5,000
- 開始日・開始時刻
- 終了日・終了時刻

日時は日本時間として処理されます。

操作手順：

1. 絞り込み条件を指定します。
2. 「対象件数」を確認します。
3. 画面プレビューを確認します。
4. 少量確認では「プレビューCSVをダウンロード」を使用します。
5. 全データが必要な場合は「全件CSVを作成」を押します。
6. 作成後に「全件CSVをダウンロード」を押します。

注意事項：

- プレビュー件数は全件数ではありません。
- 絞り込み条件を変更した場合は、全件CSVを作成し直してください。
- 全件CSVは作成開始時点のデータを対象とします。
- 作成後に追加された測定値は、そのCSVには含まれません。
- 全件CSVは一時ファイルとして作成されるため、作成後は速やかにダウンロードしてください。
- 両システムとも一時ファイルに`/tmp`を使用します。空き容量も確認してください。

### 10.2 コマンドラインから出力する

1号機：

```bash
/home/kenkyu01/Documents/IoT_test/.venv/bin/python \
  /home/kenkyu01/Documents/IoT_test/tools/export_gl840_sql_to_csv_v3.py \
  --out /安全な保存先/gl840_1_export.csv
```

2号機：

```bash
/home/kenkyu01/Documents/IoT_test2/.venv/bin/python \
  /home/kenkyu01/Documents/IoT_test2/tools/export_gl840_sql_to_csv_v3.py \
  --out /安全な保存先/gl840_2_export.csv
```

`/安全な保存先/`は実際の保存先へ置き換え、書込み可能なディレクトリを先に作成してください。

利用可能な条件を確認：

```bash
/home/kenkyu01/Documents/IoT_test/.venv/bin/python \
  /home/kenkyu01/Documents/IoT_test/tools/export_gl840_sql_to_csv_v3.py \
  --help

/home/kenkyu01/Documents/IoT_test2/.venv/bin/python \
  /home/kenkyu01/Documents/IoT_test2/tools/export_gl840_sql_to_csv_v3.py \
  --help
```

通常の保守では`--sql`を使用せず、`--batch-id`、`--run-id`、`--start-time`、`--end-time`などの標準条件を使用してください。

## 11. 日常点検

### 11.1 測定前

- [ ] 対象機のDashboardへ接続できる
- [ ] 測定制御サービスが起動済み
- [ ] 測定状態が待機中
- [ ] GL840の電源とネットワークが正常
- [ ] チャンネル配線と画面上の選択が一致
- [ ] RGB使用先が一方だけに決まっている
- [ ] AWS送信隔離メッセージがない
- [ ] ディスク空き容量が十分
- [ ] ホスト時刻が正しい

### 11.2 測定中

- [ ] 最終測定時刻が定期的に更新されている
- [ ] 正常チャンネル数が想定どおり
- [ ] RGB使用時はRGB状態が正常
- [ ] 現在値に不自然な欠損がない
- [ ] AWS送信待ちが増え続けていない

### 11.3 測定後

- [ ] Dashboardから測定停止を行った
- [ ] 測定状態が完了または待機中
- [ ] 履歴から測定の開始・終了範囲を確認した
- [ ] 必要なCSVを保存した
- [ ] バッチID、担当者、株、培地、メモを記録した
- [ ] 異常があれば保守記録へ残した

## 12. 定期保守

### 毎日

- サービス状態を確認する
- Dashboardの最終測定時刻を確認する
- AWS送信待ち・隔離件数を確認する
- ディスク空き容量を確認する

### 毎週

- 測定サービスのエラーログを確認する
- SQLiteの`PRAGMA quick_check`を実施する
- DBバックアップを作成する
- バックアップ側も`PRAGMA quick_check`で検査する
- M5 RGBの直接応答を確認する
- チャンネル名と物理配線の対応を確認する

### 毎月

- バックアップを別の場所に試験的に復元し、読み取れることを確認する
- AWS IoT証明書の期限とファイル権限を確認する
- `.env`と秘密鍵の権限が`600`であることを確認する
- OS時刻同期を確認する
- DB、`/tmp`、バックアップ先の容量増加を確認する

## 13. systemdサービス管理

### 13.1 状態確認

1号機：

```bash
systemctl status gl840-worker.service --no-pager
systemctl status gl840-dashboard.service --no-pager
systemctl is-enabled gl840-worker.service gl840-dashboard.service
curl -fsS http://127.0.0.1:2100/api/status
```

2号機：

```bash
systemctl status gl840-console2.service --no-pager
systemctl status gl840-dashboard2.service --no-pager
systemctl is-enabled gl840-console2.service gl840-dashboard2.service
curl -fsS http://127.0.0.1:2101/api/status
```

### 13.2 ログ確認

直近100行：

```bash
journalctl -u gl840-worker.service -n 100 --no-pager
journalctl -u gl840-dashboard.service -n 100 --no-pager
journalctl -u gl840-console2.service -n 100 --no-pager
journalctl -u gl840-dashboard2.service -n 100 --no-pager
```

リアルタイム表示：

```bash
journalctl -fu gl840-worker.service
journalctl -fu gl840-console2.service
```

終了するには`Ctrl+C`を押します。

### 13.3 安全な再起動

#### Dashboardだけを変更した場合

Dashboardだけの再起動は、進行中の測定を中断しません。

```bash
sudo systemctl restart gl840-dashboard.service
sudo systemctl restart gl840-dashboard2.service
```

#### 測定制御サービスを変更した場合

両号機とも同じ手順です。対象機だけを操作してください。

1. Dashboardから測定を停止します。
2. 対象APIの`status`が`idle`で、AWS接続が切断済みであることを確認します。
3. 対象機の測定制御サービスを再起動します。
4. 対象APIの`status`が`idle`であることを確認します。
5. Dashboardの変更も含む場合だけ、対象機のDashboardを再起動します。

1号機：

```bash
curl -fsS http://127.0.0.1:2100/api/status
sudo systemctl restart gl840-worker.service
curl -fsS http://127.0.0.1:2100/api/status
sudo systemctl restart gl840-dashboard.service
```

2号機：

```bash
curl -fsS http://127.0.0.1:2101/api/status
sudo systemctl restart gl840-console2.service
curl -fsS http://127.0.0.1:2101/api/status
sudo systemctl restart gl840-dashboard2.service
```

### 13.4 緊急停止

Dashboardから停止できない場合に限り、故障している対象機の測定サービスだけを停止します。次の2つを同時に実行しないでください。

1号機：

```bash
sudo systemctl stop gl840-worker.service
```

2号機：

```bash
sudo systemctl stop gl840-console2.service
```

この方法では進行中の測定が「中断」になります。ただし、停止前に保存済みのデータは残ります。

## 14. 設定ファイル

### 14.1 主設定

```text
/home/kenkyu01/Documents/IoT_test/.env
/home/kenkyu01/Documents/IoT_test2/.env
```

主な設定分類：

| 分類 | 設定例 |
| --- | --- |
| GL840 | `GL840_IP`、`GL840_WEB_PORT`、`GL840_WEB_TIMEOUT` |
| RGB | `M5_RGB_ENABLED`、`M5_RGB_URL`、`M5_RGB_TIMEOUT` |
| AWS IoT | `AWS_ENDPOINT`、`CLIENT_ID`、`TOPIC_DATA`、`TOPIC_EVENT` |
| 証明書 | `PATH_TO_ROOT_CA`、`PATH_TO_PRIVATE_KEY`、`PATH_TO_CERTIFICATE` |
| 測定周期 | `COLLECTION_INTERVAL_SEC`、`RETRY_AFTER_ERROR_SEC` |
| 停止後の送信 | `STOP_WAIT_TIMEOUT_SEC` |
| MQTT outbox | `OUTBOX_FLUSH_INTERVAL_SEC`、`OUTBOX_FLUSH_BATCH_SIZE`、再試行設定 |
| 現在値判定 | `STALE_READ_ALERT_SEC` |
| データベース | `DB_PATH` |
| チャンネル | `CH_XX_LABEL`、`CH_XX_UNIT` |
| インスタンス | `DEVICE_NUMBER`、`WEB_HOST`、`WEB_PORT`、`STREAMLIT_PORT`、`GL840_BACKEND_URL` |
| サービス | `BACKEND_SERVICE_NAME`、`DASHBOARD_SERVICE_NAME` |
| 安全制御 | `WORKER_AUTO_START`、`MEASUREMENT_START_ENABLED`、`MQTT_ENABLED` |
| CSV | `EXPORT_FILE_PREFIX` |

重要事項：

- `.env`を変更した場合は、測定を停止してから、対象機の測定サービスとDashboardの両方を再起動します。
- 測定サービスに関する設定変更は、必ず測定停止後に行います。
- 両号機の`WORKER_AUTO_START`は`false`を維持してください。共有ランタイムは自動開始要求を実行しませんが、設定値も安全側に固定します。
- OS起動時にsystemdサービスが起動しても、測定は自動開始されません。
- 通常運用時は`MEASUREMENT_START_ENABLED=true`、`MQTT_ENABLED=true`とし、実際の測定開始とAWS送信はDashboardでの明示的な確認後にだけ許可します。
- 両号機の`STOP_WAIT_TIMEOUT_SEC`は現在`13`です。停止後のAWS接続を無制限に継続させないため、理由なく延長しないでください。
- `PUBLISH_QOS`は`1`を維持してください。

### 14.2 保守ロック（`.maintenance.env`）

長時間の保守や実機接続試験中に、誤操作による測定開始とAWS送信を確実に防ぐ場合は、対象プロジェクト直下の`.maintenance.env`を使用します。このファイルは対象機の測定制御サービスだけに読み込まれ、もう一方の号機には影響しません。

保守ロックの内容：

```text
MEASUREMENT_START_ENABLED=false
MQTT_ENABLED=false
```

設定手順：

1. Dashboardから対象機の測定を停止します。
2. 停止後のoutbox処理が完了するか、最長13秒の排出時間が終了し、AWS接続が切断されたことを確認します。
3. 対象機のプロジェクト直下へ、上記2行の`.maintenance.env`を権限`600`で作成します。
4. 対象機の測定制御サービスだけを再起動します。
5. `/api/status`で`measurement_start_enabled`と`mqtt_enabled`がともに`false`であることを確認します。
6. Dashboardで「保守モード」と表示され、測定開始ボタンが無効であることを確認します。

作成例（1号機）：

```bash
umask 077
printf '%s\n' \
  'MEASUREMENT_START_ENABLED=false' \
  'MQTT_ENABLED=false' \
  > /home/kenkyu01/Documents/IoT_test/.maintenance.env
sudo systemctl restart gl840-worker.service
curl -fsS http://127.0.0.1:2100/api/status
```

2号機では保存先を`/home/kenkyu01/Documents/IoT_test2/.maintenance.env`、サービスを`gl840-console2.service`、APIを`127.0.0.1:2101`へ読み替えます。

保守終了時：

1. `.maintenance.env`を、日時を含む重複しない名前へ退避します。削除せず、保守記録とともに一定期間保存してください。
2. 対象機の測定制御サービスを再起動します。
3. `/api/status`で`status`が`idle`、`measurement_start_enabled`と`mqtt_enabled`がともに`true`、`mqtt_connected`が`false`であることを確認します。
4. Dashboardで待機中であることを確認します。確認のために測定を開始する必要はありません。

`.maintenance.env`を作成・退避しただけでは、起動中のサービス設定は変わりません。必ず対象機の測定制御サービスを再起動してください。

### 14.3 チャンネル設定

```text
/home/kenkyu01/Documents/IoT_test/data/channel_config.json
/home/kenkyu01/Documents/IoT_test2/data/channel_config.json
```

JSON内のラベルと単位は、`.env`の初期値より優先されます。

編集時の注意：

- JSON形式を壊さない
- CH番号を1～20の範囲にする
- 物理配線と一致させる
- 測定中に変更しない
- 変更前ファイルを保存する
- 変更後は測定を停止した状態で、対象機の測定サービスとDashboardの両方を再起動する
- 再起動後にDashboardと測定データの両方で表示を確認する

## 15. データベースとMQTT outbox

### 15.1 正式データベース

```text
/home/kenkyu01/Documents/IoT_test/data/gl840_console.db
/home/kenkyu01/Documents/IoT_test2/data/gl840_console2.db
```

データベースには次の情報が含まれます。

- 測定データ
- 測定開始・停止イベント
- バッチと測定ID
- 担当者、株、培地、メモ
- MQTT送信状態
- MQTT再試行情報
- 隔離されたメッセージ

両号機ともSQLiteのWALモードを使用しています。稼働中に`.db`本体だけを単純コピーすると、未反映データが欠落する可能性があります。

### 15.2 整合性確認

1号機：

```bash
/home/kenkyu01/Documents/IoT_test/.venv/bin/python -c \
'import sqlite3,sys; c=sqlite3.connect("file:"+sys.argv[1]+"?mode=ro",uri=True); print(c.execute("PRAGMA quick_check").fetchone()[0]); c.close()' \
/home/kenkyu01/Documents/IoT_test/data/gl840_console.db
```

2号機：

```bash
/home/kenkyu01/Documents/IoT_test2/.venv/bin/python -c \
'import sqlite3,sys; c=sqlite3.connect("file:"+sys.argv[1]+"?mode=ro",uri=True); print(c.execute("PRAGMA quick_check").fetchone()[0]); c.close()' \
/home/kenkyu01/Documents/IoT_test2/data/gl840_console2.db
```

正常時は`ok`と表示されます。

### 15.3 AWS送信待ち

MQTT outboxの動作は両号機で共通です。

- 測定制御サービスの起動直後は待機状態で、AWS IoTへ接続しません。
- DashboardでAWS送信を確認して測定を開始した場合だけ、outboxの送信が許可されます。以前の測定で残った送信待ちデータも、この明示的な開始後に再送されます。
- 測定中は、保存したデータとイベントをQoS 1で送信し、確認できた行だけを送信済みにします。
- 測定停止後は、終了イベントをoutboxへ保存し、最大`STOP_WAIT_TIMEOUT_SEC`秒だけ排出処理を継続します。現行値は13秒です。
- 13秒以内にoutboxが空になれば、直ちにAWS接続を切断します。
- 13秒を過ぎても未送信行が残る場合は、送信許可を解除してAWS接続を切断します。待機中に自動再接続しません。
- 残った行はSQLiteへ保持され、次回の明示的な測定開始後に再送されます。

AWS送信待ちが増えた場合：

1. DBを削除しないでください。
2. 測定サービスのログを確認します。
3. ネットワーク、DNS、AWS IoT endpointを確認します。
4. 証明書パス、権限、AWS IoT policyを確認します。
5. `run_id`とメッセージIDを確認し、どの測定に属するデータかを特定します。
6. 原因復旧後、次の測定を明示的に開始したときに再送されることを確認します。再送を目的とした試験測定を行う場合は、対象と実施時刻を記録し、保守担当者の承認を得てください。

待機中は、AWS費用を抑えるためにAWS IoTへ接続しません。未送信データを手動でDBから削除したり、`uploaded`を変更したりしないでください。

通常のネットワーク障害だけで、メッセージが隔離へ移されることはありません。

### 15.4 AWS送信隔離

隔離済み件数が増えた場合：

1. Dashboardの「隔離されたMQTTメッセージ」を開きます。
2. `最終エラー`とメッセージ内容を確認します。
3. 測定サービスのログを保存します。
4. 内容を確認せず、DB行を削除・変更しないでください。
5. 再送が必要な場合は、原因とメッセージIDを記録したうえで保守担当者が判断します。

主な原因は、ローカルoutbox内のJSON破損など、再送を繰り返しても改善しないデータです。

## 16. バックアップ

### 16.1 バックアップ対象

最低限、次を保管します。

- 1号機と2号機それぞれの正式SQLite DB
- 1号機と2号機それぞれの`.env`
- 1号機と2号機それぞれの`data/channel_config.json`
- 1号機と2号機それぞれの`certs/`
- 共有実装`/home/kenkyu01/Documents/IoT_shared`
- 各プロジェクトの起動ラッパー、`scripts/`、`systemd/`
- 各プロジェクトの`requirements.txt`
- 各仮想環境の`pip freeze`出力

`.env`、秘密鍵、DBは機密情報です。アクセス制限または暗号化された保存先を使用してください。

共有実装は1回のバックアップで構いませんが、インスタンス固有のDB、設定、証明書は号機ごとに分けて保管してください。

### 16.2 推奨DBバックアップ

この環境には`sqlite3`コマンドがないため、PythonのSQLite backup APIを使用します。測定停止中の実施を推奨します。

最初に、次の3つのパスを対象機に合わせて設定します。

```bash
python_exe="/home/kenkyu01/Documents/IoT_test/.venv/bin/python"
source_db="/home/kenkyu01/Documents/IoT_test/data/gl840_console.db"
backup_db="/安全な保存先/gl840_console_YYYYMMDD_HHMMSS.db"
```

2号機の場合：

```bash
python_exe="/home/kenkyu01/Documents/IoT_test2/.venv/bin/python"
source_db="/home/kenkyu01/Documents/IoT_test2/data/gl840_console2.db"
backup_db="/安全な保存先/gl840_console2_YYYYMMDD_HHMMSS.db"
```

`/安全な保存先/`は、実際のアクセス制限されたバックアップ先に置き換えます。保存先ディレクトリを先に作成し、既存ファイルと重複しない名前を指定してください。

次の共通処理で、バックアップと整合性確認を行います。

```bash
(
set -e
umask 077

"$python_exe" - "$source_db" "$backup_db" <<'PY'
import sqlite3
import sys
from pathlib import Path

source_path = Path(sys.argv[1])
backup_path = Path(sys.argv[2])

if backup_path.exists():
    raise SystemExit(f"保存先が既に存在するため中止しました: {backup_path}")
if not backup_path.parent.is_dir():
    raise SystemExit(f"保存先ディレクトリが存在しません: {backup_path.parent}")

source = sqlite3.connect(f"file:{source_path}?mode=ro", uri=True)
destination = sqlite3.connect(backup_path)
try:
    source.backup(destination)
    result = destination.execute("PRAGMA quick_check").fetchone()[0]
    if result != "ok":
        raise SystemExit(f"バックアップの整合性確認に失敗しました: {result}")
    print(f"バックアップ完了: {backup_path}")
finally:
    destination.close()
    source.close()
PY

chmod 600 "$backup_db"
sha256sum "$backup_db"
)
```

バックアップ後：

1. バックアップDBへ`PRAGMA quick_check`を実行します。
2. `ok`を確認します。
3. ファイル権限が`600`であることを確認します。
4. ファイルサイズとチェックサムを記録します。
5. 可能であれば別の物理媒体にも複製します。

### 16.3 単純コピーを使用する場合

単純コピーは、次の条件をすべて満たす場合だけ使用します。

1. Dashboardから測定を停止済み
2. 測定状態が待機中
3. 対象機のDashboardと測定サービスを停止済み
4. DB本体と関連する`-wal`、`-shm`の扱いを理解している

両号機とも、稼働中の`.db`本体だけをコピーすることは禁止します。

### 16.4 自動バックアップについて

現在、自動バックアップ機能や自動保存期間は設定されていません。バックアップは保守計画に従って明示的に実施してください。

## 17. 復元

復元は保守担当者が行います。

1. 復元対象とバックアップ日時を確認します。
2. Dashboardから測定を停止します。
3. 測定状態が待機中であることを確認します。
4. Dashboardを停止します。
5. 測定サービスを停止します。
6. 現行DB、`-wal`、`-shm`を削除せず、別の安全な場所へ退避します。
7. 復元元DBへ`PRAGMA quick_check`を実行します。
8. 正式DB位置へ配置します。
9. 所有者を`kenkyu01:kenkyu01`にします。
10. DB権限を`600`または運用で定めた権限にします。
11. 測定サービスを開始します。
12. 対象機の`/api/status`を確認します。1号機は2100番、2号機は2101番です。
13. Dashboardを開始します。
14. 履歴、AWS送信待ち、最終測定時刻を確認します。

異なる時点のDB本体と`-wal`、`-shm`を混在させてはいけません。

復元元の日時より後に取得したデータは、復元DBには含まれません。現行DBの退避は必須です。

## 18. 容量と時刻の確認

```bash
df -h /home/kenkyu01/Documents
df -h /tmp
du -h /home/kenkyu01/Documents/IoT_test/data/gl840_console.db
du -h /home/kenkyu01/Documents/IoT_test2/data/gl840_console2.db
timedatectl
```

注意事項：

- DBの自動削除・自動縮小はありません。
- 独自SQLで古い行を削除しないでください。
- 正式DBへ手動で`VACUUM`を実行しないでください。
- ディスク不足時は測定開始を見合わせてください。
- 両システムの全件CSVでは`/tmp`も使用します。

## 19. 更新とテスト

### 19.1 更新前

1. 測定を停止します。
2. 待機中を確認します。
3. 14.2節の保守ロックを対象機へ設定します。
4. DB、設定、変更対象コードをバックアップします。
5. 変更内容と復旧方法を記録します。

`IoT_shared/gl840_core`の変更は両号機へ同時に影響します。共有実装を変更する場合は、1号機・2号機の両方を作業対象として停止・保守ロック・確認してください。

### 19.2 テスト

共有実装：

```bash
cd /home/kenkyu01/Documents/IoT_shared
/home/kenkyu01/Documents/IoT_test/.venv/bin/python \
  -m compileall -q gl840_core tests
/home/kenkyu01/Documents/IoT_test/.venv/bin/python \
  -m unittest discover -s tests -v
```

1号機インスタンス：

```bash
cd /home/kenkyu01/Documents/IoT_test
.venv/bin/python -m compileall -q gl840_lab tests tools
.venv/bin/python -m unittest discover -s tests -v
```

2号機インスタンス：

```bash
cd /home/kenkyu01/Documents/IoT_test2
.venv/bin/python -m compileall -q gl840_lab tests tools
.venv/bin/python -m unittest discover -s tests -v
```

共有実装を変更した場合は、共有テストと両インスタンステストをすべて実行します。テストは一時SQLite DBを使用し、正式DBを書き換えません。テスト件数は変更により増減するため、最終結果が`OK`であることを確認してください。

自動テストが成功しても、実機のGL840、M5 RGB、Wi-Fi、AWS IoTへの接続までは保証されません。再起動後は、各HTTPヘルスチェックと実機通信も確認してください。

### 19.3 unitファイルを変更した場合

最初に、変更したプロジェクト内unitを検証します。

```bash
systemd-analyze verify \
  /home/kenkyu01/Documents/IoT_test/systemd/gl840-worker.service \
  /home/kenkyu01/Documents/IoT_test/systemd/gl840-dashboard.service \
  /home/kenkyu01/Documents/IoT_test2/systemd/gl840-console2.service \
  /home/kenkyu01/Documents/IoT_test2/systemd/gl840-dashboard2.service
```

検証が成功した後、変更したunitだけを`/etc/systemd/system/`へ配置します。次の4コマンドを無条件にすべて実行せず、変更対象だけを選んでください。

```bash
sudo install -m 0644 \
  /home/kenkyu01/Documents/IoT_test/systemd/gl840-worker.service \
  /etc/systemd/system/gl840-worker.service

sudo install -m 0644 \
  /home/kenkyu01/Documents/IoT_test/systemd/gl840-dashboard.service \
  /etc/systemd/system/gl840-dashboard.service

sudo install -m 0644 \
  /home/kenkyu01/Documents/IoT_test2/systemd/gl840-console2.service \
  /etc/systemd/system/gl840-console2.service

sudo install -m 0644 \
  /home/kenkyu01/Documents/IoT_test2/systemd/gl840-dashboard2.service \
  /etc/systemd/system/gl840-dashboard2.service

sudo systemctl daemon-reload
```

その後、13.3節の安全な順序に従って変更対象のサービスを再起動し、`systemctl status`とHTTPヘルスチェックを確認します。unitを変更していない場合、この作業は不要です。

## 20. セキュリティ

- `.env`と秘密鍵の権限は`600`を維持します。
- 秘密鍵を画面表示、メール送信、チャットに貼り付けないでください。
- `.env`、`certs/`、DB、`archive/device-onboarding/`をGitへ登録しないでください。
- `archive/`は正式サービスでは使いませんが、秘密情報を含むため機密扱いです。
- 1号機の2100番APIと2号機の2101番APIはlocalhost専用です。外部公開しないでください。
- Streamlit Dashboardには、現行構成ではアプリ独自の認証やTLSがありません。
- 8501番・8502番は、信頼できるLAN、VPN、ファイアウォール内だけで利用してください。
- インターネットへ直接公開しないでください。
- OSやPythonパッケージを無計画に一括更新しないでください。
- `requirements.txt`は完全なロックファイルではないため、更新前に復旧手段を用意してください。

証明書情報の確認例：

```bash
openssl x509 -in /証明書のパス/certificate.pem.crt -noout -subject -issuer -dates
```

秘密鍵の内容を確認する必要はありません。

## 21. よくある障害と対応

| 症状 | 確認 | 対応 |
| --- | --- | --- |
| Dashboardが開かない | Dashboardサービス、8501/8502、ログ | Dashboardだけを再起動する |
| 1号機／2号機で開始できない | 1号機は`2100/api/status`、2号機は`2101/api/status`、測定制御サービスのログ | バックエンドを復旧してから再操作する |
| 測定開始ボタンが無効 | `measurement_start_enabled`、`.maintenance.env`、測定状態 | 保守中なら正常。保守完了後に14.2節の手順でロックを解除する |
| サービスは起動済みだが測定されない | 測定状態 | 待機中なら正常。確認欄を選択してDashboardから開始する |
| 開始後も値が出ない | 最終測定時刻、GL840応答、ログ | 電源、LAN、IP、Web画面、解析エラーを確認する |
| 現在値が消えた | 測定状態、最終測定からの経過時間 | 90秒超なら意図した非表示。通信を確認する |
| 正常チャンネルが0 | GL840のオン・オフ、空欄、非数値 | 配線、入力レンジ、チャンネル設定を確認する |
| RGBが取得できない | M5電源、Wi-Fi、URL、`/rgb` | 設置先と選択機を確認し、直接応答を試す |
| AWS送信待ちが増える | `run_id`、AWS接続、DNS、証明書、ネットワーク | DBを保持する。停止後は最長13秒で切断され、次回の明示的な測定開始後に再送される |
| AWS送信隔離が増える | Dashboard詳細、最終エラー、ログ | 削除せず、破損原因とメッセージIDを調査する |
| CSV作成に失敗する | 対象件数、保存先、`/tmp`、Dashboardログ、対象機のバックエンドログ | 空き容量を確保し、条件を確認して再作成する |
| DB読み取りエラー | ファイル存在、所有者、容量、quick_check | DBを交換せず、まず原因を記録・調査する |
| `database is locked` | 二重起動、手動スクリプト、長時間処理 | 重複プロセスを確認し、`-wal`や`-shm`は削除しない |
| 再起動後に測定が「中断」と表示される | 再起動時刻とログ | 保存済みデータを確認し、新しい測定を開始する |
| 時刻がずれる | `timedatectl` | タイムゾーンをJSTに設定し、NTP同期を有効にしてから測定する |

## 22. GL840通信確認

1号機の現行IPを使用する例：

```bash
curl -fsS --max-time 10 \
  'http://192.168.1.174/digital.cgi?chgrp=0'
```

2号機の現行IPを使用する例：

```bash
curl -fsS --max-time 10 \
  'http://192.168.1.66/digital.cgi?chgrp=0'
```

HTTP応答があっても、HTML構造が変わると測定サービス側で解析できない場合があります。必ずサービスログとDashboardの正常チャンネル数も確認してください。

## 23. 禁止事項

- 測定中の測定サービス再起動
- 測定中のOS再起動・電源断
- 1号機・2号機でのRGB同時使用
- systemd稼働中の`run_backend.sh`、`run_dashboard.sh`手動実行
- `archive/`内の旧プログラム起動
- 1号機と2号機のDB、証明書、クライアントID、トピック、完全な`.env`の相互コピー
- 正式DBの削除、直接更新、行削除
- 両号機DBの`-wal`、`-shm`ファイル削除
- AWS送信待ち行の手動`uploaded=1`変更
- プレビューCSVをSQLiteバックアップの代わりに使用すること
- 稼働中のDB本体だけを単純コピー
- 異なる時点のDBと`-wal`、`-shm`の混在
- サービス稼働中の`.env`、`certs/`、`data/`、`.venv/`、`scripts/`の移動
- サービス稼働中の`IoT_shared`の移動・削除
- 8501番、8502番、2100番、2101番の無防備なインターネット公開
- 秘密鍵の共有やGit登録
- 復旧手段のないOS・Python一括更新
- 本番機での無条件な`pip install --upgrade`

## 24. 保守作業記録

保守後は、少なくとも次を記録してください。

| 項目 | 記録内容 |
| --- | --- |
| 作業日時 | 日本時間 |
| 対象 | 1号機／2号機 |
| 担当者 | 氏名 |
| 測定状態 | 作業前／作業後 |
| 変更内容 | コード、設定、unit、証明書など |
| バックアップ | 保存先、ファイル名、チェックサム |
| テスト結果 | 実行コマンド、OK／NG |
| サービス状態 | 各unitの状態 |
| MQTT状態 | 送信待ち件数、隔離済み件数 |
| DB確認 | `PRAGMA quick_check`結果 |
| 備考 | 障害、復旧、未解決事項 |

## 25. クイックリファレンス

### 1号機

```bash
systemctl status gl840-worker.service gl840-dashboard.service --no-pager
journalctl -u gl840-worker.service -n 100 --no-pager
curl -fsS http://127.0.0.1:2100/api/status
curl -fsS http://127.0.0.1:8501/_stcore/health
```

### 2号機

```bash
systemctl status gl840-console2.service gl840-dashboard2.service --no-pager
journalctl -u gl840-console2.service -n 100 --no-pager
curl -fsS http://127.0.0.1:2101/api/status
curl -fsS http://127.0.0.1:8502/_stcore/health
```

### 共有RGB

```bash
curl -fsS --max-time 5 http://192.168.1.209/rgb
```

### ディスク

```bash
df -h /home/kenkyu01/Documents
df -h /tmp
```

## 26. 新しい装置を追加する場合

新しい号機は、共有実装を複製せず、独立したインスタンスを1つ追加する形で構築します。既存プロジェクト全体のコピーは、DB、証明書、クライアントID、トピックの重複を招くため禁止します。

### 26.1 分離が必要な項目

| 項目 | 方針 |
| --- | --- |
| 共有コード | `/home/kenkyu01/Documents/IoT_shared/gl840_core`をそのまま使用する |
| プロジェクト | 新しい固有ディレクトリを作成する |
| APIポート | `127.0.0.1`上の未使用ポートを割り当てる |
| Dashboardポート | 未使用ポートを割り当てる |
| systemd | 測定制御用とDashboard用に固有の2サービス名を付ける |
| GL840 | 固有IPアドレスと接続条件を設定する |
| SQLite | 空の固有DBから開始し、他号機のDBをコピーしない |
| AWS IoT | 固有のクライアントID、トピック、証明書、ポリシーを使用する |
| チャンネル | 実配線に合わせた固有の`channel_config.json`を作成する |
| CSV | 固有の`EXPORT_FILE_PREFIX`を設定する |
| RGB | 共有M5を使用する場合は、既存号機との同時使用を禁止する |

### 26.2 `.env`で必ず固有化する設定

- `DEVICE_NUMBER`
- `GL840_IP`
- `WEB_PORT`
- `STREAMLIT_PORT`
- `GL840_BACKEND_URL`
- `BACKEND_SERVICE_NAME`
- `DASHBOARD_SERVICE_NAME`
- `DB_PATH`
- `CLIENT_ID`
- `TOPIC_DATA`
- `TOPIC_EVENT`
- 証明書3点のパス
- `EXPORT_FILE_PREFIX`

`WEB_HOST`は`127.0.0.1`のままとし、ローカルAPIを外部公開しません。`WORKER_AUTO_START=false`も必須です。

### 26.3 導入手順の概要

1. 号機番号、プロジェクト名、APIポート、Dashboardポート、サービス名を台帳で予約します。
2. `/home/kenkyu01/Documents/IoT_shared/templates/`の手順と環境変数テンプレートを使用し、既存号機から秘密情報を含まない最小限のラッパー、起動スクリプト、`requirements.txt`だけを基に新しいプロジェクトを作成します。
3. 新しい`.env`、空のDB領域、チャンネル設定、専用証明書を準備し、`IoT_shared/constraints.txt`を制約ファイルとして新しい仮想環境へ依存パッケージを導入します。
4. 新しい測定制御サービスとDashboardサービスのunitを作成します。
5. `.maintenance.env`を設定し、測定開始とAWS送信を無効にした状態で導入します。
6. 共有テスト、新インスタンステスト、`systemd-analyze verify`を実行します。
7. APIとDashboardのヘルスチェックを行い、`status=idle`、`auto_start=false`、`mqtt_connected=false`を確認します。
8. GL840通信、チャンネルラベル、履歴表示、CSV列を確認します。
9. AWS IoT側のクライアントID、トピック、ポリシーを別担当者と相互確認します。
10. 保守ロックを解除し、承認された試験測定を1回だけ明示的に開始します。
11. 停止後、outboxが空になるか13秒の排出上限に達した時点で、AWS接続が切断されることを確認します。
12. 構成表、バックアップ手順、監視、保守記録を更新してから本運用へ移行します。

共有実装の変更だけで新しい号機固有の条件を吸収できない場合は、まず設定項目または薄いラッパーでの対応を検討します。号機ごとに`gl840_core`を分岐させると、修正漏れと動作差異が生じるため避けてください。

---

本書と実際の設定が異なる場合は、測定を開始せず、保守担当者へ確認してください。
