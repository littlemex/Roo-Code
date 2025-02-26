# Roo Code リポジトリ構造

このドキュメントでは、Roo Code VSCode 拡張機能のリポジトリ構造を詳細に説明します。

## ディレクトリ構造

```
/home/coder/Roo-Code/
├── .changeset/          # Changesets 設定（バージョン管理とリリースノート生成用）
├── .git/                # Git リポジトリ
├── .github/             # GitHub 関連設定（ワークフロー、テンプレートなど）
├── .husky/              # Husky 設定（Git フック）
├── .vscode/             # VSCode 設定
├── assets/              # アセットファイル（アイコンなど）
├── audio/               # 音声ファイル
├── cline_docs/          # Cline 関連ドキュメント
├── docs/                # ドキュメント（このプロジェクトで作成中）
├── src/                 # メインソースコード
│   ├── __mocks__/       # テスト用モック
│   ├── activate/        # 拡張機能のアクティベーション関連コード
│   ├── api/             # API 関連の実装
│   ├── core/            # コア機能の実装
│   ├── exports/         # エクスポート定義
│   ├── integrations/    # VSCode との統合機能
│   ├── services/        # 様々なサービス実装
│   ├── shared/          # 共有ユーティリティ
│   ├── test/            # テストコード
│   ├── utils/           # ユーティリティ関数
│   └── extension.ts     # 拡張機能のエントリーポイント
└── webview-ui/          # ウェブビューのフロントエンド実装
    ├── .storybook/      # Storybook 設定
    ├── public/          # 静的ファイル
    └── src/             # フロントエンドのソースコード
        ├── __mocks__/   # テスト用モック
        ├── __tests__/   # テスト
        ├── components/  # UI コンポーネント
        ├── context/     # React コンテキスト
        ├── lib/         # ライブラリ
        ├── stories/     # Storybook ストーリー
        └── utils/       # ユーティリティ関数
```

## 主要ディレクトリの説明

### src/

メインのソースコードディレクトリです。VSCode 拡張機能のバックエンド部分を実装しています。

#### src/activate/

拡張機能のアクティベーション関連のコードが含まれています。VSCode 拡張機能が起動したときに実行される処理を定義しています。

- `handleUri.ts`: URI ハンドラーの実装
- `index.ts`: アクティベーションのメインロジック
- `registerCodeActions.ts`: コードアクション登録
- `registerCommands.ts`: コマンド登録
- `registerTerminalActions.ts`: ターミナルアクション登録

#### src/api/

AI プロバイダー（Anthropic, OpenAI など）との通信を担当するコードが含まれています。

- `providers/`: 各 AI プロバイダーの実装
  - `anthropic.ts`: Anthropic API クライアント
  - `bedrock.ts`: AWS Bedrock API クライアント
  - `openai.ts`: OpenAI API クライアント
  - その他多数のプロバイダー実装
- `transform/`: API レスポンスの変換処理

#### src/core/

拡張機能のコア機能を実装しています。

- `Cline.ts`: メインのクライアントクラス
- `CodeActionProvider.ts`: コードアクションプロバイダー
- `EditorUtils.ts`: エディタユーティリティ
- `mode-validator.ts`: モードバリデーション
- `assistant-message/`: アシスタントメッセージ処理
- `config/`: 設定管理
- `diff/`: 差分処理
- `mentions/`: メンション処理
- `prompts/`: プロンプト管理
- `sliding-window/`: スライディングウィンドウ実装
- `webview/`: ウェブビュー関連

#### src/integrations/

VSCode との統合機能を実装しています。

- `diagnostics/`: 診断機能
- `editor/`: エディタ統合
- `misc/`: その他の統合機能
- `terminal/`: ターミナル統合
- `theme/`: テーマ統合
- `workspace/`: ワークスペース統合

#### src/services/

様々なサービスを実装しています。

- `browser/`: ブラウザ関連サービス
- `checkpoints/`: チェックポイント機能
- `glob/`: ファイルグロブ処理
- `mcp/`: Model Context Protocol 実装
- `ripgrep/`: Ripgrep 統合
- `tree-sitter/`: Tree-sitter 統合

#### src/shared/

共有ユーティリティを提供しています。

- `api.ts`: API 関連の共有定義
- `ExtensionMessage.ts`: 拡張機能メッセージ定義
- `HistoryItem.ts`: 履歴アイテム定義
- `WebviewMessage.ts`: ウェブビューメッセージ定義
- その他多数の共有ユーティリティ

#### src/utils/

ユーティリティ関数を提供しています。

- `cost.ts`: コスト計算
- `fs.ts`: ファイルシステム操作
- `git.ts`: Git 操作
- `path.ts`: パス操作
- `shell.ts`: シェル操作
- `sound.ts`: 音声操作
- `logging/`: ロギング機能

### webview-ui/

ウェブビューのフロントエンド実装です。React を使用して UI を構築しています。

#### webview-ui/src/components/

UI コンポーネントを実装しています。

#### webview-ui/src/context/

React コンテキストを定義しています。

#### webview-ui/src/lib/

ライブラリコードを提供しています。

#### webview-ui/src/utils/

フロントエンド用のユーティリティ関数を提供しています。

## 設定ファイル

- `package.json`: プロジェクトの依存関係と設定
- `tsconfig.json`: TypeScript の設定
- `esbuild.js`: ビルド設定
- `.eslintrc.json`: ESLint の設定
- `.prettierrc.json`: Prettier の設定