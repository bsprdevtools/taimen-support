# 対面サポート支援システム — システム仕様書

> 属人化防止のための技術ドキュメント。
> このシステムの保守・改修を行う開発者（Claude Code含む）が、全容を把握できることを目的とする。

---

## 1. システム概要

### 目的
シルバー人材（以下「シル人」）が対面サポートで記録したExcelデータを、iPad上で閲覧・検索・分析するためのWebアプリ。

### 重要な前提
- **データの唯一の情報源はシル人のExcel**。このシステムはデータを作成・編集しない（読み取り専用ビューアー）
- データ更新は遠山が `update_taimen.py` を手動実行して行う（自動同期なし）
- iPad Safari（対面サポート現場）での利用が主目的

### 構成
```
シル人Excel（暗号化.xlsx）
  ↓ taimen_support_manager.py（Mac GUIアプリ）
taimen_support_data.csv
  ↓ encrypt_taimen_data.py（AES-256-GCM暗号化）
data.enc + verify.enc
  ↓ update_taimen.py（git push）
GitHub Pages（bsprdevtools/taimen-support）
  ↓ iPad Safari でアクセス
index.html がfetch → 復号 → 表示
```

---

## 2. ファイル構成

### GitHubリポジトリ（~/taimen-support/）
| ファイル | 役割 |
|---------|------|
| `index.html` | システム本体（単一HTML、CSS+JS内包） |
| `data.enc` | 暗号化済みCSVデータ（Base64） |
| `verify.enc` | ログイン検証用トークン（Base64） |
| `SYSTEM_SPEC.md` | 本ドキュメント |
| `GUIDE_FOR_STAFF.md` | シル人向け説明書 |

### 管理ツール（Documents/works/システム管理ツール/）
| ファイル | 役割 |
|---------|------|
| `taimen_support_manager.py` | Excel→CSV変換GUIアプリ |
| `encrypt_taimen_data.py` | CSV→暗号化（data.enc生成） |
| `update_taimen.py` | 暗号化→git push のワンクリック実行 |
| `taimen_support_data.csv` | 蓄積データ本体（平文） |
| `backups/` | CSV自動バックアップ（最新10世代） |

---

## 3. 暗号化仕様

### アルゴリズム
- **AES-256-GCM**（認証付き暗号化）
- 鍵導出: **PBKDF2-SHA256**（パスコード → 256bit鍵）

### パラメータ
| 項目 | 値 |
|------|-----|
| パスコード | 6桁数字（`258013`） |
| PBKDF2 salt | `taimen_support_v4_salt_2026`（固定） |
| PBKDF2 iterations | 100,000 |
| IV | 12バイト（ランダム、暗号化のたびに再生成） |
| GCMタグ | 16バイト（暗号文末尾に付加） |

### data.enc のバイナリ構造
```
Base64( [IV 12byte] [暗号文 + GCMタグ 16byte] )
```

### ログイン検証フロー
1. ユーザーが6桁パスコード入力
2. パスコード → PBKDF2 → AES鍵導出
3. `CONFIG.verifyData`（HTMLに埋め込み済み）を復号
4. 復号結果が `TAIMEN_SUPPORT_VERIFIED_OK` と一致すれば認証成功
5. 認証後、`data.enc` をfetch → 同じ鍵で復号 → CSV表示

### セキュリティ機能
- パスコード5回失敗 → 30秒ロックアウト（カウントダウン表示）
- セッション管理: `sessionStorage` にフラグ保存（タブ閉じで無効化）

---

## 4. データ構造

### CSV列定義（14列）
| 列名 | 型 | 説明 |
|------|-----|------|
| No | 数値 | Excel内の行番号 |
| 日付 | YYYY-MM-DD | 対応日 |
| 開始時間 | HH:MM | 対応開始 |
| 終了時間 | HH:MM | 対応終了 |
| 対応時間 | 文字列 | 所要時間 |
| UserID | 文字列 | てくポユーザーID |
| 機種 | 文字列 | 端末名（例: iPhone 13, AQUOS sense8） |
| 性別 | 文字列 | 男性/女性 |
| 相談内容 | 文字列 | 相談の詳細 |
| 対応内容 | 文字列 | 対応の詳細 |
| 特記事項 | 文字列 | 備考 |
| 担当 | 文字列 | シル人の名前（複数名はカンマ区切り） |
| 場所 | 文字列 | 対応場所（例: 南口総合事務所） |
| インポート日 | YYYY-MM-DD HH:MM:SS | CSVへの取込日時 |

### 無効UserIDの扱い
以下の値はCSVパース時に空文字に変換される（来訪カウントの対象外）:
`未確認`, `確認漏れ`, `失念しました`, `ー`, `未登録`

### 自動カテゴリ分類
相談内容のテキストからキーワードマッチで自動分類（約30カテゴリ）。
カテゴリはCSVに保存されず、表示時にリアルタイム判定。

---

## 5. UI構成（5タブ）

### ホーム
- 統計サマリー: 全レコード数、**今回の対応**（直近対応日の件数）、ユニークユーザー数、エスカレ件数
- 上位カテゴリ（クイックフィルター）
- 最近の対応（直近5件）
- 週サマリー（過去8週の件数グラフ）

### カルテ
- UserIDで検索 → 個人の対応履歴一覧
- クイック検索: UserID入力で即座にカルテ表示
- リピーター一覧（来訪回数2回以上）
- 最近閲覧したカルテ履歴（`localStorage`に保存）
- 類似事例の自動提示

### 検索
- 3モード: UserID検索 / 機種検索 / キーワード検索
- フィルター: カテゴリ、担当、期間（今回/今週/今月/日付指定）、エスカレ有無
- 50件ごとのページング

### 統計
- 期間切替: **今回** / 今週 / 今月 / 全期間 / 期間指定
- グラフ: 月別推移、カテゴリ分布、エスカレ件数、新規vsリピーター、場所別、対応時間分布、デバイス別、担当者別、曜日ヒートマップ

### 設定
- レコード件数の表示
- PWAインストール案内

---

## 6. データフロー（週次更新手順）

### Step 1: シル人からExcelを受領
- パスワード付き`.xlsx`（パスワード: `Tekupo2025sjc`）
- メールまたはチャットで受領

### Step 2: Excel → CSV変換
```bash
cd ~/Documents/works/システム管理ツール
python taimen_support_manager.py
```
GUIが起動 → 「Excelインポート」→ ファイル選択 → CSVに追記

### Step 3: 暗号化 → GitHub push
```bash
python update_taimen.py
```
このコマンドが以下を自動実行:
1. CSVバックアップ作成
2. `encrypt_taimen_data.py` で暗号化
3. `data.enc`, `verify.enc`, `index.html` を git add/commit/push

### Step 4: iPadで確認
ブラウザをリロード → 最新データが自動反映

---

## 7. 技術的な判断記録

### なぜ単一HTMLか
- iPad Pro 1st gen（2015年、iOS 16）で安定動作させるため
- GitHub Pagesでの配信に最適（ビルド不要）
- オフライン対応しやすい（PWA化可能）

### なぜlocalStorage同期を撤去したか
- データ更新は遠山の手動実行のみ（週1回）
- 毎回サーバーからfetchすれば常に最新（キャッシュ不整合の心配なし）
- localStorageの5MB制限に抵触するリスクを排除

### なぜクライアント側復号か
- GitHub Pagesは静的ホスティング（サーバーサイド処理不可）
- パスコードはネットワークに送信されない（端末内で完結）
- AES-256-GCM + PBKDF2 100K iterationsで十分な強度

### アイコン方式
- Lucide Icons（インラインSVG）、CDN不要
- SF Symbols風デザイン（filled/outline切替、stroke-width 1.8）
- タブはiOS風の固定ボトムバー（backdrop-filter blur）

---

## 8. 既知の制約・注意点

- **iOS 16 Safari互換**: `let`/`const`を使わず`var`で統一、アロー関数不使用
- **Excelの列位置がずれるとデータ壊れる**: `taimen_support_manager.py`の列マッピング（行2以降、列1〜12）はシル人のExcelフォーマット固定前提
- **担当者名の区切り**: カンマ（`,`）と全角読点（`、`）の両方に対応済み
- **パスコード変更時**: `encrypt_taimen_data.py`で再暗号化 → `verify.enc`再生成 → HTMLの`CONFIG.verifyData`を更新
