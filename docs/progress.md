# Roo Code リポジトリ解析プロジェクト

## プロジェクト概要

このプロジェクトでは、VSCode プラグインである Roo Code の実装を解析し、以下の内容を文書化します：

1. リポジトリのファイル構造
2. 各ファイルが持つメソッドとその機能
3. クラス間の関係（mermaid 記法で図示）

すべての文書は `/home/coder/Roo-Code/docs` ディレクトリに作成します。

## 進捗状況

### 2025-02-26

#### 午前

- プロジェクト開始
- リポジトリの基本構造を確認
- 主要ファイルの内容を確認（extension.ts, Cline.ts など）
- docs/progress.md ファイルを作成
- 以下のドキュメントを作成:
  - [repository-structure.md](repository-structure.md): リポジトリ構造の詳細な説明
  - [core-classes.md](core-classes.md): コアクラスとその関係の説明
  - [extension-activation.md](extension-activation.md): 拡張機能のアクティベーションプロセスの説明
  - [cline-class.md](cline-class.md): Cline クラスの詳細な説明
  - [api-providers.md](api-providers.md): API プロバイダーの実装の説明
  - [webview-ui.md](webview-ui.md): Webview UI の実装の説明
  - [mcp-implementation.md](mcp-implementation.md): MCP の実装の説明
  - [class-diagram.md](class-diagram.md): クラス図（mermaid 記法）

#### 午後

- 「次のステップ」で計画した未分析のディレクトリの構造を確認:
  - src/integrations/ ディレクトリ（diagnostics, editor, misc, terminal, theme, workspace サブディレクトリを含む）
  - src/services/ ディレクトリ（browser, checkpoints, glob, mcp, ripgrep, tree-sitter サブディレクトリを含む）
  - src/utils/ ディレクトリ（cost.ts, fs.ts, git.ts, path.ts, shell.ts, single-completion-handler.ts, sound.ts, xml-matcher.ts など）
- 特定の機能に関連するディレクトリの内容を確認:
  - チェックポイント機能（src/services/checkpoints/）
  - ターミナル統合（src/integrations/terminal/）
  - ブラウザ統合（src/services/browser/）
- 以下の追加ドキュメントを作成:
  - [integrations-directory.md](integrations-directory.md): src/integrations/ ディレクトリの詳細な分析
  - [services-directory.md](services-directory.md): src/services/ ディレクトリの詳細な分析
  - [utils-directory.md](utils-directory.md): src/utils/ ディレクトリの詳細な分析
  - [checkpoints-feature.md](checkpoints-feature.md): チェックポイント機能の詳細な分析
  - [terminal-integration.md](terminal-integration.md): ターミナル統合の詳細な分析
  - [browser-integration.md](browser-integration.md): ブラウザ統合の詳細な分析
- ユースケースの分析と文書化:
  - [task-execution-flow.md](task-execution-flow.md): タスクの実行フロー
  - [tool-execution-flow.md](tool-execution-flow.md): ツールの実行フロー
  - [error-handling.md](error-handling.md): エラー処理

### 2025-02-27

#### 予定

- 拡張機能の設定とカスタマイズの文書化:
  - extension-settings.md: 設定オプション
  - custom-modes.md: カスタムモード
  - custom-tools.md: カスタムツール

## 作成したドキュメント一覧

1. **repository-structure.md**
   - リポジトリの全体構造
   - 各ディレクトリの役割と内容
   - 主要な設定ファイルの説明

2. **core-classes.md**
   - 主要なクラスの説明
   - クラス間の関係
   - 主要なデータフロー

3. **extension-activation.md**
   - 拡張機能のアクティベーションプロセス
   - コマンドの登録
   - コードアクションの登録
   - ターミナルアクションの登録

4. **cline-class.md**
   - Cline クラスの詳細な説明
   - 主要なプロパティとメソッド
   - タスク管理、メッセージ管理、ツール実行などの機能

5. **api-providers.md**
   - API ハンドラーの説明
   - サポートされているプロバイダー（Anthropic, OpenAI, Google など）
   - フォーマット変換
   - モデル情報とコスト計算

6. **webview-ui.md**
   - Webview UI の実装
   - React コンポーネント
   - 状態管理
   - VSCode との通信

7. **mcp-implementation.md**
   - Model Context Protocol (MCP) の実装
   - MCP サーバーの管理
   - ツールとリソースの提供
   - Cline クラスでの MCP の使用

8. **class-diagram.md**
   - 全体のクラス図
   - コアモジュールのクラス図
   - API プロバイダーのクラス図
   - MCP 関連のクラス図
   - チェックポイント関連のクラス図
   - ウェブビュー関連のクラス図

9. **integrations-directory.md**
   - src/integrations/ ディレクトリの詳細な分析
   - diagnostics サブディレクトリ（診断機能）
   - editor サブディレクトリ（エディタ統合）
   - terminal サブディレクトリ（ターミナル統合）
   - workspace サブディレクトリ（ワークスペース統合）

10. **services-directory.md**
    - src/services/ ディレクトリの詳細な分析
    - browser サブディレクトリ（ブラウザ統合）
    - checkpoints サブディレクトリ（チェックポイント機能）
    - glob サブディレクトリ（ファイルパターンマッチング）
    - ripgrep サブディレクトリ（コード検索）
    - tree-sitter サブディレクトリ（構文解析）

11. **utils-directory.md**
    - src/utils/ ディレクトリの詳細な分析
    - ファイルシステム操作（fs.ts）
    - Git 操作（git.ts）
    - パス操作（path.ts）
    - シェル操作（shell.ts）
    - ロギング機能（logging/）

12. **checkpoints-feature.md**
    - チェックポイント機能の詳細な分析
    - CheckpointServiceFactory, LocalCheckpointService, ShadowCheckpointService の役割と関係
    - チェックポイントの保存と復元のフロー

13. **terminal-integration.md**
    - ターミナル統合の詳細な分析
    - TerminalManager, TerminalProcess, TerminalRegistry の役割と関係
    - ターミナルコマンドの実行と出力の取得

14. **browser-integration.md**
    - ブラウザ統合の詳細な分析
    - BrowserSession, UrlContentFetcher の役割と関係
    - ブラウザ操作のフロー

15. **task-execution-flow.md**
    - タスクの実行フロー
    - ユーザー入力からタスク完了までのフロー
    - タスクの状態管理
    - タスクの再開と中断の処理

16. **tool-execution-flow.md**
    - ツールの実行フロー
    - ツールの登録と実行のメカニズム
    - ツール実行結果の処理
    - ツールの種類と特性

17. **error-handling.md**
    - エラー処理のメカニズム
    - エラーの種類と処理方法
    - エラーメッセージの表示
    - エラーからの回復

## 主要なクラスと機能

### ClineProvider

`src/core/webview/ClineProvider.ts` に定義されています。

VSCode のウェブビュープロバイダーを実装したクラスで、拡張機能のメインビューを提供します。Cline インスタンスを管理し、ウェブビューとの通信を担当します。

### Cline

`src/core/Cline.ts` に定義されています。

拡張機能のコア機能を実装するクラスで、AI モデルとの通信、ツールの実行、メッセージの管理などを担当します。

### ApiHandler

`src/api/index.ts` で定義されているインターフェースです。

AI プロバイダー（Anthropic, OpenAI など）との通信を抽象化するインターフェースです。

### McpHub

`src/services/mcp/McpHub.ts` に定義されています。

Model Context Protocol (MCP) サーバーとの通信を管理するクラスです。

## 次のステップ

1. 拡張機能の設定とカスタマイズの文書化
   - **extension-settings.md**: 設定オプション
     - 利用可能な設定項目とその効果
     - 設定の保存と読み込み
   - **custom-modes.md**: カスタムモード
     - モードの定義と登録
     - モード切り替えのメカニズム
   - **custom-tools.md**: カスタムツール
     - ツールの定義と登録
     - ツールの実行と結果の処理

## 調査メモ

### 拡張機能の設定に関する調査状況

- `src/core/config/ConfigManager.ts`: API設定の管理を担当するクラス
- `src/core/config/CustomModesManager.ts`: カスタムモードの管理を担当するクラス
- `src/core/config/CustomModesSchema.ts`: カスタムモードのスキーマ定義
- `package.json`: 拡張機能の設定オプションの定義（`contributes.configuration`セクション）
- `src/extension.ts`: 拡張機能のアクティベーション時に設定を読み込む処理

次回は、これらのファイルを詳細に分析し、拡張機能の設定とカスタマイズに関するドキュメントを作成する予定です。

## まとめ

Roo Code は VSCode の拡張機能として実装されており、AI モデルを使用してコーディングタスクを支援します。拡張機能は複数のモジュールに分かれており、それぞれが特定の機能を担当しています。主要なクラスは Cline で、AI モデルとの通信、ツールの実行、メッセージの管理などを担当しています。

拡張機能は複数の AI プロバイダーをサポートしており、共通のインターフェースを通じてこれらのプロバイダーと通信します。また、Model Context Protocol (MCP) を使用して、AI モデルが外部リソースやツールにアクセスできるようにしています。

UI は React を使用して実装されており、VSCode の Webview API を通じて表示されます。UI は複数のコンポーネントに分割され、React コンテキストを使用して状態を管理しています。

現在、リポジトリの基本構造と主要なクラスの分析が完了しており、17のドキュメントが作成されています。最初の8つのドキュメントでは、リポジトリ構造、コアクラス、拡張機能のアクティベーション、Clineクラス、APIプロバイダー、Webview UI、MCP実装、クラス図について詳細に説明しています。

さらに、src/integrations/、src/services/、src/utils/ディレクトリの詳細な分析と、チェックポイント機能、ターミナル統合、ブラウザ統合の詳細な分析を行い、6つの追加ドキュメントを作成しました。

また、タスク実行フロー、ツール実行フロー、エラー処理に関する3つのドキュメントを作成し、ユースケースの分析と文書化を行いました。

次のステップでは、拡張機能の設定とカスタマイズに関する文書化を行う予定です。これにより、Roo Code の実装の全体像を把握し、各コンポーネントの役割と関係を明確にすることができます。