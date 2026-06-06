
# Folder History 拡張機能 仕様書

## 目的

VS Codeで開いたフォルダを日付付きで記録し、過去の履歴をリスト表示してクリックでエクスプローラや VS Code で開けるようにする。VS Codeの「最近使用した項目」ではタイムスタンプが取れず、件数も限られるという問題を解決する。

## 機能要件

### 1. 記録機能

- **トリガー**: VS Codeでフォルダ(ワークスペース)を開いた時
  - 起動時にすでにフォルダが開かれている場合
  - 起動後にフォルダが追加された場合(`onDidChangeWorkspaceFolders`)
- **記録粒度**: フォルダ単位、1日1件
  - 同じフォルダを同日に複数回開いても1件のみ
  - 別の日に再度開けば新しい記録として追加
- **記録項目**:
  - `date`: YYYY-MM-DD形式(ローカル日付)
  - `path`: フォルダのフルパス
  - `name`: フォルダ名(表示用)
- **複数フォルダ対応**: マルチルートワークスペースの場合、各フォルダを個別に記録
- **対象スキーム**: `file` スキームのフォルダのみ(リモート/仮想ワークスペースは対象外)

### 2. 表示機能

- **起動方法**:
  - アクティビティバーの専用ビュー(`folderHistory.sidebar`)に常駐表示
  - コマンドパレットから「Folder History: Show」で独立した WebView パネルを表示
- **UI**: VS Code内の WebView。データを inline で埋め込み、クライアント側でレンダリング(タブ・月・絞り込みの切替を extension 往復なしで反映)。状態(タブ・月選択・スターフィルタ)は `vscode.setState()` で永続化。
- **履歴タブ**:
  - リスト、新しい日付順。日付ごとに見出し(例: `2026-04-27 (月)`)、その下にフォルダ一覧
  - `date: null` のエントリは末尾の「日付不明」セクションにまとめて表示
  - 各行クリックでインラインメニューを開き、アクションを選択:
    - **新しいウィンドウで開く**(`vscode.openFolder`、`forceNewWindow`)
    - **現在のウィンドウで開く**(`vscode.openFolder`、`forceReuseWindow`)
    - **エクスプローラで開く**(`revealFileInOS`)
    - **フルパスをコピー**(`vscode.env.clipboard.writeText`)
  - フォルダが既に削除されている場合はエラー表示
  - 検索: テキストフィルタ(フォルダ名・パスで絞り込み)
- **ランキングタブ**:
  - 月セレクタ(◀ ▶ で前後月へ移動)で対象月を切替
  - その月に開いた**実日数**が多い順にフォルダをランキング表示
  - `date: null` の取込みエントリは集計対象外
  - 集計は月ごとに `Map<YYYY-MM, ranked[]>` でメモ化

### 3. スター機能

- 各履歴行の先頭に ★/☆ アイコンを常時表示。クリックでフォルダ単位の「お気に入り」を切替(楽観 UI のためレスポンスは即時)
- 絞り込み入力欄の横の **★ トグルボタン**で、スター付きフォルダのみに絞り込み(テキスト絞り込みと AND 条件)
- スターはパス単位で `stars: string[]` に保存

### 4. データ管理機能

- **ログファイルを開く**: コマンド「Folder History: Open Log File」で `history.json` をエディタで直接編集
- **エクスポート**: コマンド「Folder History: Export」で履歴とスターを JSON ファイルへ書き出し
- **インポート**: コマンド「Folder History: Import」で JSON ファイルから履歴とスターを取り込み(重複排除キー `date + path` で既存と重なるもの、既存スターはスキップ)
- **VS Code Recent List からの取込み**: コマンド「Folder History: Import from VS Code Recent List」で `state.vscdb` の `recentlyOpenedPathsList` を読み、**タイムスタンプなしの「過去に開いたことがある」エントリ**として `date: null` で取り込む(「日付不明」セクションに表示)。Windows 専用

## データ仕様

### 保存場所

```
%APPDATA%\Code\User\globalStorage\local.folder-history\history.json
```
(VS Codeの `globalStorageUri` で取得される拡張機能専用の永続領域。OS によりパスは異なる)

### フォーマット (version 2)

```json
{
  "version": 2,
  "entries": [
    {
      "date": "2026-04-27",
      "path": "C:\\projects\\kintone-plugin",
      "name": "kintone-plugin"
    },
    {
      "date": null,
      "path": "C:\\old\\some-project",
      "name": "some-project"
    }
  ],
  "stars": [
    "C:\\projects\\kintone-plugin"
  ]
}
```

- エントリの重複排除キー: `date + path` の組み合わせ
- `stars`: スター付きフォルダのパス配列(union でマージ)
- v1(`stars` なし)ファイルは読み込み時に `stars: []` を自動補完(後方互換あり)。保存時は常に v2 で書き出す
- ファイルアクセスは同期 I/O でシンプルに(履歴件数が爆発する想定はないため)

## 非機能要件

- **常駐なし**: VS Codeの起動中のみ動作
- **外部通信なし**: すべてローカル完結
- **依存ライブラリ**: VS Code拡張API(`vscode`モジュール)、Node.js標準モジュール(`fs`, `path`)のみ使用
- **対象OS**:
  - 表示系(開く/コピー)はクロスプラットフォーム対応(`revealFileInOS`・`vscode.openFolder`)
  - 「Import from VS Code Recent List」は Windows パス(`%APPDATA%`)を前提とした Windows 専用機能
- **VS Codeバージョン**: 1.80以上

## 想定外(やらないこと)

- 作業時間の計測(ActivityWatchの領域)
- ファイル単位の履歴
- カレンダー表示(リスト/ランキングのみ)
- クラウド同期(エクスポート/インポートによる手動移行は可能)
- 複数PC間での自動データ共有
- 自動的な古い履歴の削除(手動でJSONを編集、またはエクスポート/インポートで対応)

## ファイル構成

```
folder-history/
├── package.json          // 拡張機能マニフェスト
├── tsconfig.json
├── README.md
└── src/
    ├── extension.ts      // エントリポイント、記録処理、エクスポート/インポート
    ├── storage.ts        // history.jsonの読み書き、重複排除、スター管理
    ├── webview.ts        // リスト/ランキングUI(HTML生成)、パネル・サイドバー共通
    └── importer.ts       // state.vscdbからの取込み(Windows専用)
```

## ビルド・配布方法

- TypeScriptでコンパイル → `vsce package` で `.vsix` を生成
- `code --install-extension folder-history-<version>.vsix` でインストール
- マーケットプレイスへの公開はしない(ローカル利用前提)
