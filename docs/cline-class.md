# Cline クラスの詳細解説

このドキュメントでは、Roo Code の中核となる `Cline` クラスについて詳細に解説します。

## 概要

`Cline` クラスは `src/core/Cline.ts` に定義されており、Roo Code の主要な機能を実装しています。このクラスは AI モデルとの通信、ツールの実行、メッセージの管理、タスクの実行と管理などを担当します。

## クラス定義

```typescript
export class Cline {
    readonly taskId: string
    api: ApiHandler
    private terminalManager: TerminalManager
    private urlContentFetcher: UrlContentFetcher
    private browserSession: BrowserSession
    private didEditFile: boolean = false
    customInstructions?: string
    diffStrategy?: DiffStrategy
    diffEnabled: boolean = false
    fuzzyMatchThreshold: number = 1.0

    apiConversationHistory: (Anthropic.MessageParam & { ts?: number })[] = []
    clineMessages: ClineMessage[] = []
    private askResponse?: ClineAskResponse
    private askResponseText?: string
    private askResponseImages?: string[]
    private lastMessageTs?: number
    private consecutiveMistakeCount: number = 0
    private consecutiveMistakeCountForApplyDiff: Map<string, number> = new Map()
    private providerRef: WeakRef<ClineProvider>
    private abort: boolean = false
    didFinishAbortingStream = false
    abandoned = false
    private diffViewProvider: DiffViewProvider
    private lastApiRequestTime?: number
    isInitialized = false

    // チェックポイント関連
    checkpointsEnabled: boolean = false
    private checkpointService?: CheckpointService

    // ストリーミング関連
    isWaitingForFirstChunk = false
    isStreaming = false
    private currentStreamingContentIndex = 0
    private assistantMessageContent: AssistantMessageContent[] = []
    private presentAssistantMessageLocked = false
    private presentAssistantMessageHasPendingUpdates = false
    private userMessageContent: (Anthropic.TextBlockParam | Anthropic.ImageBlockParam)[] = []
    private userMessageContentReady = false
    private didRejectTool = false
    private didAlreadyUseTool = false
    private didCompleteReadingStream = false

    constructor({
        provider,
        apiConfiguration,
        customInstructions,
        enableDiff,
        enableCheckpoints,
        fuzzyMatchThreshold,
        task,
        images,
        historyItem,
        experiments,
        startTask = true,
    }: ClineOptions) {
        // 初期化処理
    }

    // その他のメソッド
}
```

## 主要なプロパティ

- `taskId`: タスクの一意の識別子
- `api`: AI モデルとの通信を担当する `ApiHandler` インスタンス
- `terminalManager`: ターミナルを管理する `TerminalManager` インスタンス
- `browserSession`: ブラウザセッションを管理する `BrowserSession` インスタンス
- `diffViewProvider`: 差分ビューを提供する `DiffViewProvider` インスタンス
- `apiConversationHistory`: AI モデルとの会話履歴
- `clineMessages`: ウェブビューに表示するメッセージ
- `checkpointsEnabled`: チェックポイント機能が有効かどうか
- `diffEnabled`: 差分機能が有効かどうか

## 主要なメソッド

### タスク管理

#### startTask

```typescript
private async startTask(task?: string, images?: string[]): Promise<void>
```

新しいタスクを開始します。タスクとオプションの画像を受け取り、AI モデルとの会話を開始します。

#### resumeTaskFromHistory

```typescript
private async resumeTaskFromHistory()
```

履歴からタスクを再開します。過去の会話履歴を読み込み、中断されたタスクを再開します。

#### initiateTaskLoop

```typescript
private async initiateTaskLoop(userContent: UserContent): Promise<void>
```

タスクループを開始します。ユーザーのコンテンツを AI モデルに送信し、レスポンスを処理します。

#### recursivelyMakeClineRequests

```typescript
async recursivelyMakeClineRequests(
    userContent: UserContent,
    includeFileDetails: boolean = false,
): Promise<boolean>
```

AI モデルにリクエストを送信し、レスポンスを処理するループを実行します。

#### abortTask

```typescript
async abortTask(isAbandoned = false)
```

実行中のタスクを中止します。

### メッセージ管理

#### ask

```typescript
async ask(
    type: ClineAsk,
    text?: string,
    partial?: boolean,
): Promise<{ response: ClineAskResponse; text?: string; images?: string[] }>
```

ウェブビューにメッセージを送信し、ユーザーからのレスポンスを待ちます。

#### say

```typescript
async say(
    type: ClineSay,
    text?: string,
    images?: string[],
    partial?: boolean,
    checkpoint?: Record<string, unknown>,
): Promise<undefined>
```

ウェブビューにメッセージを表示します。

#### presentAssistantMessage

```typescript
async presentAssistantMessage()
```

アシスタントのメッセージを処理し、適切なツールを実行します。

### ツール実行

#### executeCommandTool

```typescript
async executeCommandTool(command: string): Promise<[boolean, ToolResponse]>
```

コマンドを実行し、結果を返します。

#### attemptApiRequest

```typescript
async *attemptApiRequest(previousApiReqIndex: number, retryAttempt: number = 0): ApiStream
```

AI モデルにリクエストを送信し、レスポンスをストリーミングします。

### ファイル操作

Cline クラスには、以下のようなファイル操作メソッドが含まれています：

- ファイルの読み取り
- ファイルの書き込み
- 差分の適用
- ファイルの検索
- コード定義の一覧表示

これらの操作は `presentAssistantMessage` メソッド内で処理されます。

### チェックポイント管理

#### checkpointSave

```typescript
public async checkpointSave({ isFirst }: { isFirst: boolean })
```

現在の状態をチェックポイントとして保存します。

#### checkpointRestore

```typescript
public async checkpointRestore({
    ts,
    commitHash,
    mode,
}: {
    ts: number
    commitHash: string
    mode: "preview" | "restore"
})
```

指定されたチェックポイントに状態を復元します。

#### checkpointDiff

```typescript
public async checkpointDiff({
    ts,
    commitHash,
    mode,
}: {
    ts: number
    commitHash: string
    mode: "full" | "checkpoint"
})
```

チェックポイント間の差分を表示します。

## データフロー

1. **タスクの開始**:
   - `startTask` または `resumeTaskFromHistory` が呼び出される
   - `initiateTaskLoop` が呼び出される

2. **AI モデルとの通信**:
   - `recursivelyMakeClineRequests` が呼び出される
   - `attemptApiRequest` を使用して AI モデルにリクエストを送信
   - レスポンスがストリーミングされる

3. **レスポンスの処理**:
   - ストリーミングされたレスポンスが `parseAssistantMessage` で解析される
   - `presentAssistantMessage` が呼び出されて解析されたメッセージが処理される

4. **ツールの実行**:
   - `presentAssistantMessage` 内でツールの種類に応じた処理が実行される
   - ツールの結果が `userMessageContent` に追加される

5. **次のリクエスト**:
   - ツールの実行結果が AI モデルに送信される
   - AI モデルが次のステップを決定

## エラー処理

Cline クラスには、以下のようなエラー処理メカニズムが含まれています：

- `consecutiveMistakeCount`: 連続したエラーの回数を追跡
- `didRejectTool`: ユーザーがツールの実行を拒否したかどうか
- `abortStream`: ストリーミングを中止する関数

エラーが発生した場合、適切なエラーメッセージが表示され、必要に応じてタスクが中止されます。

## 状態管理

Cline クラスは、以下のような状態を管理しています：

- `apiConversationHistory`: AI モデルとの会話履歴
- `clineMessages`: ウェブビューに表示するメッセージ
- `userMessageContent`: ユーザーのメッセージ内容
- `assistantMessageContent`: アシスタントのメッセージ内容

これらの状態は、タスクの実行中に更新され、必要に応じてディスクに保存されます。

## まとめ

Cline クラスは Roo Code の中核となるクラスで、AI モデルとの通信、ツールの実行、メッセージの管理、タスクの実行と管理などを担当しています。このクラスは、ユーザーのタスクを実行するために必要なすべての機能を提供しています。