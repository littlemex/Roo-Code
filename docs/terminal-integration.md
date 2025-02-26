# ターミナル統合の詳細分析

このドキュメントでは、Roo Code拡張機能のターミナル統合機能について詳細に説明します。ターミナル統合は、VSCodeのターミナル機能と連携し、コマンドの実行や出力の取得を可能にする重要なコンポーネントです。

## 概要

ターミナル統合機能は、`src/integrations/terminal/`ディレクトリに実装されています。この機能により、Roo Codeは以下のことが可能になります：

1. VSCodeのターミナルを作成・管理する
2. ターミナルでコマンドを実行する
3. コマンドの出力をリアルタイムで取得する
4. 実行中のコマンドの状態を監視する
5. ターミナルの出力を後から取得する

## アーキテクチャ

ターミナル統合機能は、以下の主要なクラスで構成されています：

1. **TerminalManager**: ターミナルの作成と管理を担当
2. **TerminalProcess**: ターミナルでのコマンド実行と出力処理を担当
3. **TerminalRegistry**: ターミナルの登録と管理を担当

これらのクラスの関係と役割について詳しく見ていきましょう。

## TerminalManager

`TerminalManager`クラスは、ターミナルの作成、コマンドの実行、出力の取得などの高レベル機能を提供します。

### 主要メソッド

```typescript
runCommand(terminalInfo: TerminalInfo, command: string): TerminalProcessResultPromise
```
指定されたターミナルでコマンドを実行します。このメソッドは、コマンドの実行を表す`TerminalProcess`オブジェクトを返します。このオブジェクトは、Promiseとして扱うこともできます（`TerminalProcessResultPromise`）。

```typescript
async getOrCreateTerminal(cwd: string): Promise<TerminalInfo>
```
指定された作業ディレクトリに対応するターミナルを取得または作成します。既存のターミナルを再利用するか、新しいターミナルを作成します。

```typescript
getTerminals(busy: boolean): { id: number; lastCommand: string }[]
```
ビジー状態または非ビジー状態のターミナルのリストを取得します。

```typescript
getUnretrievedOutput(terminalId: number): string
```
指定されたターミナルの未取得の出力を取得します。

```typescript
isProcessHot(terminalId: number): boolean
```
指定されたターミナルのプロセスがホット（最近出力があった）かどうかを確認します。

```typescript
async getTerminalContents(commands = -1): Promise<string>
```
ターミナルの内容を取得します。指定されたコマンド数分の内容を含めることができます。

### シェル統合

`TerminalManager`は、VSCodeのシェル統合APIを使用して、ターミナルコマンドの実行と出力の取得を行います。シェル統合APIが利用できない場合は、従来の`sendText`メソッドにフォールバックします。

```typescript
// シェル統合APIを使用したコマンド実行
if (terminal.shellIntegration && terminal.shellIntegration.executeCommand) {
  const execution = terminal.shellIntegration.executeCommand(command)
  const stream = execution.read()
  // ストリームから出力を読み取る
}
```

## TerminalProcess

`TerminalProcess`クラスは、ターミナルでのコマンド実行と出力処理を担当します。このクラスは`EventEmitter`を継承しており、コマンドの出力を行単位でイベントとして発行します。

### 主要メソッド

```typescript
async run(terminal: vscode.Terminal, command: string)
```
指定されたターミナルでコマンドを実行します。シェル統合APIが利用可能な場合は、それを使用してコマンドを実行し、出力をストリームとして取得します。

```typescript
continue()
```
プロセスの継続を指示します。これにより、プロセスはイベントの発行を停止し、Promiseが解決されます。

```typescript
getUnretrievedOutput(): string
```
未取得の出力を取得します。

### 出力処理

`TerminalProcess`は、コマンドの出力を処理するための複数のメソッドを提供しています：

```typescript
private emitIfEol(chunk: string)
```
チャンクに行末が含まれている場合、行単位でイベントを発行します。

```typescript
private emitRemainingBufferIfListening()
```
リスニング中の場合、残りのバッファをイベントとして発行します。

```typescript
removeLastLineArtifacts(output: string)
```
出力の最後の行からアーティファクト（`%`、`$`、`#`、`>`など）を削除します。

### ホット状態の管理

`TerminalProcess`は、プロセスのホット状態（最近出力があった状態）を管理します。これは、APIリクエストを遅延させるために使用されます。

```typescript
// ホット状態の設定
this.isHot = true
if (this.hotTimer) {
  clearTimeout(this.hotTimer)
}
this.hotTimer = setTimeout(
  () => {
    this.isHot = false
  },
  isCompiling ? PROCESS_HOT_TIMEOUT_COMPILING : PROCESS_HOT_TIMEOUT_NORMAL,
)
```

## TerminalRegistry

`TerminalRegistry`クラスは、ターミナルの登録と管理を担当します。このクラスは、ターミナルの作成、取得、更新、削除などの機能を提供します。

### 主要メソッド

```typescript
static createTerminal(cwd?: string | vscode.Uri | undefined): TerminalInfo
```
新しいターミナルを作成し、登録します。

```typescript
static getTerminal(id: number): TerminalInfo | undefined
```
指定されたIDのターミナル情報を取得します。

```typescript
static updateTerminal(id: number, updates: Partial<TerminalInfo>)
```
指定されたIDのターミナル情報を更新します。

```typescript
static removeTerminal(id: number)
```
指定されたIDのターミナルを削除します。

```typescript
static getAllTerminals(): TerminalInfo[]
```
すべてのターミナルのリストを取得します。

### TerminalInfo インターフェース

`TerminalInfo`インターフェースは、ターミナルの情報を表します：

```typescript
export interface TerminalInfo {
  terminal: vscode.Terminal
  busy: boolean
  lastCommand: string
  id: number
}
```

## ターミナル出力の取得

ターミナル統合機能には、ターミナルの出力を取得するための2つの方法があります：

1. **リアルタイム取得**: `TerminalProcess`のイベントを使用して、コマンドの出力をリアルタイムで取得します。
2. **後から取得**: `getLatestTerminalOutput`関数または`TerminalManager.getTerminalContents`メソッドを使用して、ターミナルの内容を後から取得します。

### getLatestTerminalOutput

`get-latest-output.ts`ファイルに定義されている`getLatestTerminalOutput`関数は、アクティブなターミナルの最新の出力を取得します：

```typescript
export async function getLatestTerminalOutput(): Promise<string>
```

この関数は、以下の手順で動作します：

1. 元のクリップボード内容を保存
2. ターミナルの全内容を選択
3. 選択内容をクリップボードにコピー
4. 選択を解除
5. クリップボードからターミナル内容を取得
6. コマンド区切りのクリーンアップ
7. 元のクリップボード内容を復元

## ターミナルコマンドの実行フロー

ターミナルコマンドの実行フローは以下の通りです：

1. `TerminalManager.runCommand`メソッドが呼び出される
2. 指定されたターミナルの`busy`フラグが`true`に設定される
3. 新しい`TerminalProcess`インスタンスが作成される
4. シェル統合APIが利用可能な場合は、それを使用してコマンドが実行される
5. コマンドの出力がストリームとして取得され、行単位でイベントとして発行される
6. コマンドが完了すると、`completed`イベントが発行される
7. ターミナルの`busy`フラグが`false`に設定される

## エラー処理

ターミナル統合機能には、堅牢なエラー処理メカニズムが組み込まれています：

1. **シェル統合の検出**: シェル統合APIが利用できない場合は、従来の`sendText`メソッドにフォールバックします。
2. **タイムアウト処理**: シェル統合APIの初期化を待つためのタイムアウト処理が実装されています。
3. **エラーイベント**: `TerminalProcess`は、エラーが発生した場合に`error`イベントを発行します。

## 実装の詳細

### シェル統合APIの拡張

VSCodeのシェル統合APIを使用するために、型定義を拡張しています：

```typescript
declare module "vscode" {
  interface Window {
    onDidStartTerminalShellExecution?: (
      listener: (e: any) => any,
      thisArgs?: any,
      disposables?: vscode.Disposable[],
    ) => vscode.Disposable
  }
}

type ExtendedTerminal = vscode.Terminal & {
  shellIntegration?: {
    cwd?: vscode.Uri
    executeCommand?: (command: string) => {
      read: () => AsyncIterable<string>
    }
  }
}
```

### 出力のクリーンアップ

ターミナルの出力には、ANSIエスケープシーケンスやVSCodeのカスタムシーケンスなど、様々なアーティファクトが含まれています。`TerminalProcess`は、これらのアーティファクトを削除するための複数のメソッドを提供しています：

```typescript
// ANSIエスケープシーケンスの削除
data = stripAnsi(data)

// VSCodeのカスタムシーケンスの削除
const vscodeSequenceRegex = /\x1b\]633;.[^\x07]*\x07/g
const lastMatch = [...data.matchAll(vscodeSequenceRegex)].pop()
if (lastMatch && lastMatch.index !== undefined) {
  data = data.slice(lastMatch.index + lastMatch[0].length)
}

// 非人間可読文字の削除
lines[0] = lines[0].replace(/[^\x20-\x7E]/g, "")
```

### PromiseとEventEmitterの組み合わせ

`TerminalProcess`は、PromiseとEventEmitterの両方の機能を提供するために、特別な`mergePromise`関数を使用しています：

```typescript
export function mergePromise(process: TerminalProcess, promise: Promise<void>): TerminalProcessResultPromise {
  const nativePromisePrototype = (async () => {})().constructor.prototype
  const descriptors = ["then", "catch", "finally"].map(
    (property) => [property, Reflect.getOwnPropertyDescriptor(nativePromisePrototype, property)] as const,
  )
  for (const [property, descriptor] of descriptors) {
    if (descriptor) {
      const value = descriptor.value.bind(promise)
      Reflect.defineProperty(process, property, { ...descriptor, value })
    }
  }
  return process as TerminalProcessResultPromise
}
```

## まとめ

ターミナル統合機能は、Roo Code拡張機能の重要なコンポーネントであり、VSCodeのターミナル機能と連携してコマンドの実行や出力の取得を可能にします。`TerminalManager`、`TerminalProcess`、`TerminalRegistry`の3つの主要なクラスが連携して、ターミナルの作成、コマンドの実行、出力の処理を行います。

シェル統合APIを使用することで、コマンドの出力をリアルタイムで取得できるようになり、ユーザーエクスペリエンスが向上しています。また、堅牢なエラー処理メカニズムにより、シェル統合APIが利用できない場合でも適切に対処できるようになっています。

ターミナル統合機能は、Roo Codeの他の機能（特にAIによるコード生成や編集）と連携して、ユーザーがコマンドラインツールを使用する際の支援を提供します。