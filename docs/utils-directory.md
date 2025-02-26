# src/utils/ ディレクトリの詳細分析

このドキュメントでは、Roo Code拡張機能の`src/utils/`ディレクトリに含まれる各ファイルとその機能について詳細に説明します。

## 概要

`src/utils/`ディレクトリは、Roo Code拡張機能全体で使用される共通のユーティリティ関数とクラスを提供します。これらのユーティリティは、ファイルシステム操作、パス操作、Git操作、シェル操作、コスト計算、ロギングなど、様々な機能をサポートしています。

## ファイル一覧

- `cost.ts`: API呼び出しのコスト計算に関するユーティリティ
- `fs.ts`: ファイルシステム操作に関するユーティリティ
- `git.ts`: Git操作に関するユーティリティ
- `path.ts`: パス操作に関するユーティリティ
- `shell.ts`: シェル操作に関するユーティリティ
- `single-completion-handler.ts`: 単一の補完処理を行うためのユーティリティ
- `sound.ts`: サウンド再生に関するユーティリティ
- `xml-matcher.ts`: XMLタグをマッチングするためのユーティリティ
- `logging/`: ロギング機能に関するユーティリティ

## 詳細説明

### cost.ts

このファイルには、API呼び出しのコスト計算に関するユーティリティ関数が含まれています。

#### 主要機能

```typescript
export function calculateApiCost(
  modelInfo: ModelInfo,
  inputTokens: number,
  outputTokens: number,
  cacheCreationInputTokens?: number,
  cacheReadInputTokens?: number,
): number
```

この関数は、モデル情報、入力トークン数、出力トークン数、キャッシュ関連のトークン数に基づいてAPIコストを計算します。キャッシュの書き込みコスト、キャッシュの読み取りコスト、基本入力コスト、出力コストを合計して総コストを返します。

```typescript
export const parseApiPrice = (price: any) => (price ? parseFloat(price) * 1_000_000 : undefined)
```

この関数は、API価格を解析します。価格が存在する場合は、浮動小数点数に変換して100万倍します。

### fs.ts

このファイルには、ファイルシステム操作に関するユーティリティ関数が含まれています。

#### 主要機能

```typescript
export async function createDirectoriesForFile(filePath: string): Promise<string[]>
```

この関数は、指定されたファイルパスに必要なすべてのサブディレクトリを作成し、後で削除するためにそれらを配列に収集します。

```typescript
export async function fileExistsAtPath(filePath: string): Promise<boolean>
```

この関数は、指定されたパスが存在するかどうかを確認します。

### git.ts

このファイルには、Git操作に関するユーティリティ関数が含まれています。

#### 主要機能

```typescript
async function checkGitRepo(cwd: string): Promise<boolean>
```

この関数は、指定されたディレクトリがGitリポジトリかどうかを確認します。

```typescript
async function checkGitInstalled(): Promise<boolean>
```

この関数は、Gitがインストールされているかどうかを確認します。

```typescript
export async function searchCommits(query: string, cwd: string): Promise<GitCommit[]>
```

この関数は、指定されたクエリに基づいてGitコミットを検索します。ハッシュまたはメッセージでコミットを検索し、最大10件の結果を返します。

```typescript
export async function getCommitInfo(hash: string, cwd: string): Promise<string>
```

この関数は、指定されたハッシュのコミット情報を取得します。コミット情報、統計情報、差分を別々に取得し、それらを組み合わせて返します。

```typescript
export async function getWorkingState(cwd: string): Promise<string>
```

この関数は、作業ディレクトリの状態を取得します。ステータスと差分を取得し、それらを組み合わせて返します。

### path.ts

このファイルには、パス操作に関するユーティリティ関数が含まれています。

#### 主要機能

```typescript
function toPosixPath(p: string)
```

この関数は、Windowsのバックスラッシュをフォワードスラッシュに変換します。

```typescript
export function arePathsEqual(path1?: string, path2?: string): boolean
```

この関数は、異なるプラットフォーム間でパスを比較します。パスを正規化し、Windowsでは大文字と小文字を区別しないように比較します。

```typescript
function normalizePath(p: string): string
```

この関数は、パスを正規化します。`./`や`../`セグメントを解決し、重複するスラッシュを削除し、パス区切り文字を標準化します。

```typescript
export function getReadablePath(cwd: string, relPath?: string): string
```

この関数は、読みやすいパス形式に変換します。現在の作業ディレクトリとの相対パスを計算し、適切な形式で返します。

```typescript
export const toRelativePath = (filePath: string, cwd: string) => { /* ... */ }
```

この関数は、絶対パスを相対パスに変換します。

### shell.ts

このファイルには、シェル操作に関するユーティリティ関数が含まれています。

#### 主要機能

```typescript
function getWindowsTerminalConfig()
```

この関数は、Windows用のターミナル設定を取得します。

```typescript
function getMacTerminalConfig()
```

この関数は、macOS用のターミナル設定を取得します。

```typescript
function getLinuxTerminalConfig()
```

この関数は、Linux用のターミナル設定を取得します。

```typescript
function getWindowsShellFromVSCode(): string | null
```

この関数は、VSCodeの設定からWindows用のシェルパスを取得します。

```typescript
function getMacShellFromVSCode(): string | null
```

この関数は、VSCodeの設定からmacOS用のシェルパスを取得します。

```typescript
function getLinuxShellFromVSCode(): string | null
```

この関数は、VSCodeの設定からLinux用のシェルパスを取得します。

```typescript
function getShellFromUserInfo(): string | null
```

この関数は、os.userInfo()からシェルを取得します。

```typescript
function getShellFromEnv(): string | null
```

この関数は、環境変数からシェルを取得します。

```typescript
export function getShell(): string
```

この関数は、上記の関数を使用して適切なシェルを取得します。VSCodeの設定、userInfo()、環境変数の順に試し、それでも見つからない場合はデフォルトのシェルを返します。

### single-completion-handler.ts

このファイルには、単一の補完処理を行うためのユーティリティ関数が含まれています。

#### 主要機能

```typescript
export async function singleCompletionHandler(apiConfiguration: ApiConfiguration, promptText: string): Promise<string>
```

この関数は、設定されたAPIを使用してプロンプトを強化します。Clineインスタンスやタスク履歴を作成せずに、APIの補完機能のみを使用する軽量な代替手段です。

### sound.ts

このファイルには、サウンド再生に関するユーティリティ関数が含まれています。

#### 主要機能

```typescript
export const isWAV = (filepath: string): boolean
```

この関数は、ファイルがWAVファイルかどうかを判定します。

```typescript
export const setSoundEnabled = (enabled: boolean): void
```

この関数は、サウンドの有効/無効を設定します。

```typescript
export const setSoundVolume = (newVolume: number): void
```

この関数は、サウンドの音量を設定します。

```typescript
export const playSound = (filepath: string): void
```

この関数は、サウンドファイルを再生します。サウンドが有効で、ファイルがWAVファイルであり、最小再生間隔を超えている場合にのみ再生します。

### xml-matcher.ts

このファイルには、XMLタグをマッチングするためのユーティリティクラスが含まれています。

#### 主要機能

```typescript
export class XmlMatcher<Result = XmlMatcherResult>
```

このクラスは、XMLタグをマッチングするためのクラスです。テキストストリームからXMLタグを抽出し、マッチしたコンテンツと非マッチのコンテンツを区別します。

### logging/ ディレクトリ

このディレクトリには、ロギング機能に関するユーティリティが含まれています。

#### 主要ファイル

- `CompactLogger.ts`: コンパクトなロガー実装
- `CompactTransport.ts`: ログトランスポート実装
- `index.ts`: ロギングシステムのメインエントリーポイント
- `types.ts`: ロギングシステムのコア型定義

#### 主要機能

```typescript
export const logger = process.env.JEST_WORKER_ID !== undefined ? new CompactLogger() : noopLogger
```

デフォルトのロガーインスタンス。通常の操作ではCompactLoggerを使用し、Jestテスト環境ではnoopLoggerに切り替えます。

```typescript
export interface ILogger {
  debug(message: string, meta?: LogMeta): void
  info(message: string, meta?: LogMeta): void
  warn(message: string, meta?: LogMeta): void
  error(message: string | Error, meta?: LogMeta): void
  fatal(message: string | Error, meta?: LogMeta): void
  child(meta: LogMeta): ILogger
  close(): void
}
```

ロガー実装のインターフェース。デバッグ、情報、警告、エラー、致命的なエラーのログレベルをサポートし、子ロガーの作成とクローズ機能を提供します。

## まとめ

`src/utils/`ディレクトリは、Roo Code拡張機能全体で使用される共通のユーティリティ関数とクラスを提供しています。これらのユーティリティは、ファイルシステム操作、パス操作、Git操作、シェル操作、コスト計算、ロギングなど、様々な機能をサポートしています。

これらのユーティリティは、Roo Codeの他のコンポーネントから広く使用され、コードの重複を減らし、一貫した動作を保証するのに役立っています。特に、ファイルシステム操作、パス操作、ロギングなどの機能は、拡張機能の多くの部分で使用されています。