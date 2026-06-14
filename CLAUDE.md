# CLAUDE.md

## プロジェクト概要

**就活メモ帳（Job_Tracker）** — 就職活動で検討中の企業を記録・比較するためのウェブツール。
求人ごとに業界・勤務地・待遇・選考状況・複数観点の星評価・自由メモなどを残し、一覧／比較表で見比べる。

- フロントエンドは **単一の HTML ファイル**（インライン CSS + バニラ JS、ビルド工程なし）
- バックエンドは **Google Apps Script (GAS)** の Web アプリ（このリポジトリには**含まれない**）
- データの保存先は **Google スプレッドシート**。利用者ごとに 1 シートを割り当てる
- **ログイン制**（ユーザー名 + パスワード）。プライバシー保護のため、各利用者は自分のシートのみ閲覧・編集できる
- **編集ロック**機構あり。同じ企業エントリを 2 人（特に利用者と管理者）が同時に編集できないようにする
- 依存は CDN の Google Fonts（Zen Kaku Gothic New / Shippori Mincho）のみ。フレームワーク・パッケージマネージャなし
- UI 文言・コメントはすべて日本語。紙（paper/ink）をモチーフにしたデザイン

## ファイル構成

GitHub Pages が配信するのは `index.html`。それ以外は手動バックアップ的な世代スナップショット。

| ファイル | 役割 |
|---|---|
| `index.html` | 配信される最新かつ最も完全な実装。**実装・修正の対象はこれ** |
| `index15.html`, `index14.html`, `index13.html` | 比較的新しい世代スナップショット |
| `index.html2`〜`index.html12`, `index3.html` | 旧世代スナップショット（拡張子に番号が付くものと `indexN.html` 形式が混在） |
| `old`, `old2` | さらに初期のスナップショット |

> 命名規則は不統一（`index.htmlN` と `indexN.html` が混在）。**正本は常に `index.html`**。
> 新しい変更は `index.html` をベースに行い、公開もこのファイルへの反映で行う。

## アーキテクチャ

```
ブラウザ (index.html)  ──POST JSON──▶  GAS Web アプリ  ──▶  Google スプレッドシート
       │                                 (認証・ロック・通知を処理)   ├─ ユーザー管理シート
       │                                                              └─ 利用者ごとの企業データシート
       └─ localStorage: shushoku_gas_url, shushoku_session
```

### GAS 通信プロトコル

すべて `api(payload)` 経由（`index.html` 内）。`POST` で JSON を送り、JSON を受け取る。
- ログイン後のリクエストには **`token`**（`session.token`）が自動付与される
- 管理者が他利用者のシートを見ているときは **`sheetName`** も付与される
- レスポンスに `error:'auth'` が返ると自動ログアウト（セッション切れ扱い）

`action` 一覧:

| 区分 | action | 用途 |
|---|---|---|
| 認証 | `login` `{username, password}` | 成功で `{username, role, sheetName, token}` を返す |
| 企業データ | `getAll` / `save` `{data}` / `delete` `{id}` | 一覧取得・保存（新規/更新兼用）・削除 |
| 編集ロック | `lock` `{id}` / `unlock` `{id}` / `heartbeat` `{id}` | 排他制御。`lock` 競合時は `{error:'locked', lockedBy}` |
| 更新通知 | `getNotifications` / `markReadByUser` `{fromUser}` / `clearFlag` `{id}` | 管理者向けの「利用者が更新した」通知 |
| 利用者管理 | `getUsers` / `addUser` `{userData}` / `deleteUser` `{username}` | 管理者のみ |

### データモデル（企業エントリ）

```
{
  id,            // 生成: Date.now().toString(36)+random
  updatedAt,     // ミリ秒。一覧の「更新順」ソートに使用
  name,          // 企業名（必須）
  url,           // JSON 文字列。[{label, url}, ...] 形式（最大5件、URL_MAX）
  industry, location, employment, salary,
  status,        // '0'..'5'（STATUS_LABELS と対応）
  work, culture, support, memo,        // 自由記述メモ
  rWork, rCulture, rSupport, rOverall, // 星評価 1..5（0=未設定）
  priority,      // 'high' | 'mid' | 'low' | '0'
  updatedBy      // 最終更新者。本人以外なら一覧に「🔔 更新」バッジ
}
```

- `STATUS_LABELS = ['検討中','ボツ','書類選考中','面接中','内定','不採用・辞退']`（index 0〜5）
- `PRIORITY_LABELS = {high:'高', mid:'中', low:'低'}`

## ログイン・権限・プライバシー

- **ログイン**: ユーザー名 + パスワードを GAS に送り、検証は GAS 側で行う。成功すると `session`
  （`{username, role, sheetName, token}`）を `localStorage` に保存し、以降のリクエストに `token` を付ける
- **一般ユーザー（`role:'user'`）**: 自分の `sheetName` のシートのみ閲覧・編集。他人のデータは見えない
- **管理者（`role:'admin'`）**: 追加で以下が有効化される
  - 「管理」タブ — 利用者の追加（ユーザー名 / パスワード / シート名）・削除
  - 利用者切り替えドロップダウン（`admin-user-select`）— 任意の利用者のシートを開いて閲覧・編集
  - 通知ベル（🔔）— 利用者が企業を更新すると件数バッジが付く（`getNotifications` を 60 秒ごとにポーリング）
- パスワード・利用者名簿・各データは **GAS / スプレッドシート側**にあり、ブラウザには `token` とセッション情報のみ保持

## 編集ロック（排他制御）

利用者と管理者が同じエントリを同時編集して上書き合うのを防ぐ仕組み。

- `openForm(id)` でフォームを開く際にまず `lock` を要求。`{error:'locked'}` なら
  「現在『◯◯』が編集中です」とトーストして開かない
- 編集中は `heartbeat` を **30 秒ごと**に送り、ロックを維持（`startHeartbeat`/`stopHeartbeat`）
- 保存（`saveCompany`）・キャンセル（`unlockAndBack`）時に `unlock` を送ってロック解放
- ロックの有効期限・タイムアウト判定は GAS 側の責務（heartbeat が途絶えたら失効させる想定）

## 状態と永続化

- `localStorage` キー: `shushoku_gas_url`（GAS Web アプリ URL）、`shushoku_session`（ログインセッション）
- GAS URL はログイン画面の「⚙ GAS URL設定」からユーザーが入力（`https://script.google.com` で始まる検証あり）
- 楽観的更新ではなく、保存・削除は GAS のレスポンスを待ってからローカル `companies` 配列へ反映

## 開発上の注意

- ビルド・テスト・lint の仕組みはない。ブラウザで `index.html` を直接開いて動作確認する
- GAS バックエンドはこのリポジトリ外。プロトコル（`action` / レスポンス形）を変えるときは GAS 側も合わせて更新が必要
- innerHTML へ値を差し込む箇所が多いが、ユーザー入力は基本 `esc()`（`& < > "` をエスケープ）を通している。
  新規に値を埋め込むときも既存同様 `esc()` を必ず通すこと
- 画面はログイン画面 / メイン画面（一覧・フォーム・比較表・管理）の単一ページ切り替え（`.page.active`）。
  ページ遷移にはフリップ／フェードの CSS アニメーションを使用
- レスポンシブ対応（`@media(max-width:600px)` でフォームを 1 カラム化）。PC・スマホ両対応

## デプロイ

- `main` ブランチへの push で GitHub Pages が更新される（リポジトリ: `memotan/Job_Tracker`）。配信されるのは `index.html`
- GAS 側のコード変更は別途 GAS エディタでデプロイが必要（このリポジトリ外）
