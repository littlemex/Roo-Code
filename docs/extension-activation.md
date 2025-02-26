# Roo Code 拡張機能のアクティベーションプロセス

このドキュメントでは、Roo Code VSCode 拡張機能のアクティベーションプロセスについて説明します。

## アクティベーションのエントリーポイント

拡張機能のエントリーポイントは `src/extension.ts` ファイルの `activate` 関数です。この関数は VSCode によって拡張機能が有効化されたときに呼び出されます。

```typescript
export function activate(context: vscode.ExtensionContext) {
    extensionContext = context
    outputChannel = vscode.window.createOutputChannel("Roo-Code")
    context.subscriptions.push(outputChannel)
    outputChannel.appendLine("Roo-Code extension activated")

    // デフォルトコマンドを設定から取得
    const defaultCommands = vscode.workspace.getConfiguration("roo-cline").get<string[]>("allowedCommands") || []

    // グローバル状態の初期化
    if (!context.globalState.get("allowedCommands")) {
        context.globalState.update("allowedCommands", defaultCommands)
    }

    const sidebarProvider = new ClineProvider(context, outputChannel)

    context.subscriptions.push(
        vscode.window.registerWebviewViewProvider(ClineProvider.sideBarId, sidebarProvider, {
            webviewOptions: { retainContextWhenHidden: true },
        }),
    )

    registerCommands({ context, outputChannel, provider: sidebarProvider })

    // 差分ビュー用のコンテンツプロバイダーを登録
    const diffContentProvider = new (class implements vscode.TextDocumentContentProvider {
        provideTextDocumentContent(uri: vscode.Uri): string {
            return Buffer.from(uri.query, "base64").toString("utf-8")
        }
    })()

    context.subscriptions.push(
        vscode.workspace.registerTextDocumentContentProvider(DIFF_VIEW_URI_SCHEME, diffContentProvider),
    )

    context.subscriptions.push(vscode.window.registerUriHandler({ handleUri }))

    // コードアクションプロバイダーを登録
    context.subscriptions.push(
        vscode.languages.registerCodeActionsProvider({ pattern: "**/*" }, new CodeActionProvider(), {
            providedCodeActionKinds: CodeActionProvider.providedCodeActionKinds,
        }),
    )

    registerCodeActions(context)
    registerTerminalActions(context)

    return createClineAPI(outputChannel, sidebarProvider)
}
```

## アクティベーションの主要ステップ

1. **出力チャンネルの作成**:
   - `vscode.window.createOutputChannel("Roo-Code")` を使用して出力チャンネルを作成
   - 拡張機能のログ出力に使用される

2. **デフォルト設定の取得**:
   - `vscode.workspace.getConfiguration("roo-cline")` を使用して設定を取得
   - `allowedCommands` 設定を取得し、グローバル状態に保存

3. **サイドバープロバイダーの作成と登録**:
   - `new ClineProvider(context, outputChannel)` でプロバイダーを作成
   - `vscode.window.registerWebviewViewProvider` で登録
   - `retainContextWhenHidden: true` オプションで非表示時もコンテキストを保持

4. **コマンドの登録**:
   - `registerCommands` 関数を呼び出してコマンドを登録
   - サイドバーのボタン、エディタのコンテキストメニュー、ターミナルのコンテキストメニューなどのコマンドを登録

5. **差分ビュー用のコンテンツプロバイダーの登録**:
   - `vscode.TextDocumentContentProvider` インターフェースを実装したクラスを作成
   - `vscode.workspace.registerTextDocumentContentProvider` で登録
   - 差分ビューの左側（元のコンテンツ）を表示するために使用

6. **URI ハンドラーの登録**:
   - `vscode.window.registerUriHandler` で URI ハンドラーを登録
   - 外部からの URI リクエストを処理するために使用

7. **コードアクションプロバイダーの登録**:
   - `vscode.languages.registerCodeActionsProvider` でコードアクションプロバイダーを登録
   - すべてのファイル（`{ pattern: "**/*" }`）に対してコードアクションを提供

8. **コードアクションとターミナルアクションの登録**:
   - `registerCodeActions` と `registerTerminalActions` 関数を呼び出して登録
   - エディタとターミナルの特定のアクションを登録

9. **API の作成と返却**:
   - `createClineAPI` 関数を呼び出して API を作成
   - 他の拡張機能が Roo Code の機能を利用するための API を提供

## コマンドの登録

`registerCommands` 関数（`src/activate/registerCommands.ts`）では、以下のようなコマンドが登録されます：

- `roo-cline.plusButtonClicked`: 新しいタスクを作成
- `roo-cline.mcpButtonClicked`: MCP サーバーを表示
- `roo-cline.promptsButtonClicked`: プロンプトを表示
- `roo-cline.historyButtonClicked`: 履歴を表示
- `roo-cline.popoutButtonClicked`: エディタで開く
- `roo-cline.settingsButtonClicked`: 設定を表示
- `roo-cline.helpButtonClicked`: ドキュメントを表示
- `roo-cline.openInNewTab`: 新しいタブで開く
- `roo-cline.explainCode`: コードを説明
- `roo-cline.fixCode`: コードを修正
- `roo-cline.improveCode`: コードを改善
- `roo-cline.addToContext`: コンテキストに追加

## コードアクションの登録

`registerCodeActions` 関数（`src/activate/registerCodeActions.ts`）では、エディタのコードアクションが登録されます。これにより、選択したコードに対して特定のアクションを実行できます。

## ターミナルアクションの登録

`registerTerminalActions` 関数（`src/activate/registerTerminalActions.ts`）では、ターミナルのアクションが登録されます。これにより、ターミナルの出力に対して特定のアクションを実行できます：

- `roo-cline.terminalAddToContext`: ターミナルの内容をコンテキストに追加
- `roo-cline.terminalFixCommand`: コマンドを修正
- `roo-cline.terminalExplainCommand`: コマンドを説明

## 拡張機能の非アクティブ化

`deactivate` 関数は拡張機能が無効化されたときに呼び出されます：

```typescript
export async function deactivate() {
    outputChannel.appendLine("Roo-Code extension deactivated")
    // MCP サーバーマネージャーのクリーンアップ
    await McpServerManager.cleanup(extensionContext)
}
```

この関数では、以下の処理が行われます：

1. 出力チャンネルにログを出力
2. `McpServerManager.cleanup` を呼び出して MCP サーバーをクリーンアップ