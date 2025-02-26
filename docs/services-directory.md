# src/services/ ディレクトリの詳細分析

このドキュメントでは、Roo Code拡張機能の`src/services/`ディレクトリに含まれる各サブディレクトリとその機能について詳細に説明します。

## 概要

`src/services/`ディレクトリは、Roo Code拡張機能の中核的なサービスを提供するコンポーネントを含んでいます。各サブディレクトリは特定のサービス領域に焦点を当てています：

- `browser/`: ブラウザ操作とURL内容取得のためのサービス
- `checkpoints/`: コードのチェックポイント（スナップショット）管理サービス
- `glob/`: ファイルパターンマッチングサービス
- `mcp/`: Model Context Protocol (MCP) サーバー管理サービス
- `ripgrep/`: コード検索サービス
- `tree-sitter/`: 構文解析サービス

## browser/ サブディレクトリ

### 概要
`browser/`サブディレクトリは、Puppeteerを使用したブラウザ操作とURL内容取得のためのサービスを提供します。

### 主要ファイル
- `BrowserSession.ts`: ブラウザセッションを管理するクラス
- `UrlContentFetcher.ts`: URLからコンテンツを取得してMarkdownに変換するクラス

### 主要機能

#### BrowserSession
```typescript
export class BrowserSession {
  // ...
  async launchBrowser(): Promise<void> { /* ... */ }
  async closeBrowser(): Promise<BrowserActionResult> { /* ... */ }
  async navigateToUrl(url: string): Promise<BrowserActionResult> { /* ... */ }
  async click(coordinate: string): Promise<BrowserActionResult> { /* ... */ }
  async type(text: string): Promise<BrowserActionResult> { /* ... */ }
  async scrollDown(): Promise<BrowserActionResult> { /* ... */ }
  async scrollUp(): Promise<BrowserActionResult> { /* ... */ }
}
```

このクラスは、Puppeteerを使用してブラウザセッションを管理します。主に以下の機能を提供します：
- Chromiumブラウザの起動と終了
- ブラウザでのアクション実行（ナビゲーション、クリック、タイプ、スクロールなど）
- スクリーンショットの取得
- コンソールログの収集
- ページの安定性の確認

#### UrlContentFetcher
```typescript
export class UrlContentFetcher {
  // ...
  async launchBrowser(): Promise<void> { /* ... */ }
  async closeBrowser(): Promise<void> { /* ... */ }
  async urlToMarkdown(url: string): Promise<string> { /* ... */ }
}
```

このクラスは、URLからコンテンツを取得してMarkdownに変換します。主に以下の機能を提供します：
- Chromiumブラウザの起動と終了
- URLからHTMLコンテンツの取得
- HTMLのクリーンアップ（不要な要素の削除）
- HTMLからMarkdownへの変換

## checkpoints/ サブディレクトリ

### 概要
`checkpoints/`サブディレクトリは、コードのチェックポイント（スナップショット）を管理するためのサービスを提供します。Gitを使用して、コードの状態を保存・復元する機能を実装しています。

### 主要ファイル
- `CheckpointServiceFactory.ts`: チェックポイントサービスのファクトリークラス
- `constants.ts`: チェックポイント関連の定数
- `index.ts`: エクスポート用のインデックスファイル
- `LocalCheckpointService.ts`: ローカルGitリポジトリを使用したチェックポイントサービス
- `ShadowCheckpointService.ts`: シャドウGitリポジトリを使用したチェックポイントサービス
- `types.ts`: チェックポイント関連の型定義

### 主要機能

#### CheckpointServiceFactory
```typescript
export class CheckpointServiceFactory {
  public static create<T extends CreateCheckpointServiceFactoryOptions>(options: T): CheckpointServiceType<T> {
    // ...
  }
}
```

このクラスは、チェックポイントサービスを作成するためのファクトリーです。指定されたストラテジー（"local"または"shadow"）に基づいて、適切なチェックポイントサービスのインスタンスを返します。

#### LocalCheckpointService
```typescript
export class LocalCheckpointService implements CheckpointService {
  // ...
  public async saveCheckpoint(message: string) { /* ... */ }
  public async restoreCheckpoint(commitHash: string) { /* ... */ }
  public async getDiff({ from, to }: { from?: string; to?: string }) { /* ... */ }
}
```

このクラスは、ローカルのGitリポジトリを使用してチェックポイントを管理します。主に以下の機能を提供します：
- Gitリポジトリの初期化と設定
- チェックポイントの保存（現在の状態をコミットとして保存）
- チェックポイントの復元（特定のコミットの状態に戻す）
- 差分の取得
- メインブランチの復元

このサービスは、2つのブランチを使用しています：
- メインブランチ（通常の操作用）
- 隠しブランチ（チェックポイントの保存用）

#### ShadowCheckpointService
```typescript
export class ShadowCheckpointService implements CheckpointService {
  // ...
  public async saveCheckpoint(message: string) { /* ... */ }
  public async restoreCheckpoint(commitHash: string) { /* ... */ }
  public async getDiff({ from, to }: { from?: string; to?: string }) { /* ... */ }
}
```

このクラスは、シャドウGitリポジトリを使用してチェックポイントを管理します。主に以下の機能を提供します：
- シャドウGitリポジトリの初期化と設定
- チェックポイントの保存（現在の状態をコミットとして保存）
- チェックポイントの復元（特定のコミットの状態に戻す）
- 差分の取得
- ネストされたGitリポジトリの一時的な無効化と再有効化

このサービスは、元のワークスペースとは別の場所（シャドウディレクトリ）にGitリポジトリを作成し、そのリポジトリのworktreeを元のワークスペースに設定することで、元のワークスペースの状態を追跡します。

## mcp/ サブディレクトリ

### 概要
`mcp/`サブディレクトリは、Model Context Protocol (MCP) サーバーとの通信を管理するためのサービスを提供します。MCPは、AIモデルが外部リソースやツールにアクセスするためのプロトコルです。

### 主要ファイル
- `McpHub.ts`: MCPサーバーとの通信を管理するクラス
- `McpServerManager.ts`: MCPサーバーのシングルトンインスタンスを管理するクラス

### 主要機能

#### McpHub
```typescript
export class McpHub {
  // ...
  getServers(): McpServer[] { /* ... */ }
  async readResource(serverName: string, uri: string): Promise<McpResourceResponse> { /* ... */ }
  async callTool(
    serverName: string,
    toolName: string,
    toolArguments?: Record<string, unknown>,
  ): Promise<McpToolCallResponse> { /* ... */ }
}
```

このクラスは、MCPサーバーとの通信を管理します。主に以下の機能を提供します：
- MCPサーバーの設定ファイルの読み込みと監視
- MCPサーバーへの接続と切断
- サーバーの状態管理（接続中、切断、エラーなど）
- ツールとリソースのリストの取得
- ツールの呼び出しとリソースの読み取り
- サーバーの有効/無効の切り替え
- 「常に許可」設定の管理

#### McpServerManager
```typescript
export class McpServerManager {
  // ...
  static async getInstance(context: vscode.ExtensionContext, provider: ClineProvider): Promise<McpHub> { /* ... */ }
  static unregisterProvider(provider: ClineProvider): void { /* ... */ }
  static notifyProviders(message: any): void { /* ... */ }
  static async cleanup(context: vscode.ExtensionContext): Promise<void> { /* ... */ }
}
```

このクラスは、MCPサーバーのシングルトンインスタンスを管理します。主に以下の機能を提供します：
- MCPサーバーのシングルトンインスタンスの取得と管理
- プロバイダーの登録と登録解除
- すべての登録済みプロバイダーへの通知
- シングルトンインスタンスのクリーンアップ

## まとめ

`src/services/`ディレクトリは、Roo Code拡張機能の中核的なサービスを提供するコンポーネントを含んでいます。特に、ブラウザ操作、チェックポイント管理、MCPサーバー管理などの機能は、Roo Codeの高度な機能を実現するために不可欠です。

これらのサービスにより、Roo Codeは以下のような機能を実現しています：
- ブラウザ操作とWebコンテンツの取得
- コードのチェックポイント（スナップショット）の作成と復元
- 外部ツールやリソースへのアクセス（MCP経由）

これらの機能は、Roo Codeがユーザーに対して高度なコーディング支援を提供するための基盤となっています。