# タスク実行フロー

このドキュメントでは、Roo Code拡張機能におけるタスク実行のフローについて説明します。タスク実行フローは、ユーザーの入力からタスク完了までの一連のプロセスを表します。

## 1. タスク実行フローの概要

Roo Codeのタスク実行フローは、以下の主要なステップで構成されています：

1. **タスクの開始**: ユーザーがタスクを入力し、AIモデルとの対話が開始されます
2. **タスクループの実行**: AIモデルが繰り返しタスクを分析し、ツールを使用して進捗します
3. **ツールの実行**: AIモデルがタスクを完了するために必要なツールを実行します
4. **タスクの完了**: タスクが完了し、結果がユーザーに表示されます

## 2. タスクの開始

タスク実行フローは、ユーザーがタスクを入力することから始まります。

### 2.1 ユーザー入力の処理

1. ユーザーがタスクを入力すると、WebViewからClineProviderに通知されます
2. `ClineProvider.initClineWithTask`メソッドが呼び出されます
3. 新しいClineインスタンスが作成され、タスクとオプションの画像が渡されます

```typescript
// ClineProvider.ts
public async initClineWithTask(task?: string, images?: string[]) {
    await this.clearTask();
    const {
        apiConfiguration,
        customModePrompts,
        diffEnabled,
        checkpointsEnabled,
        fuzzyMatchThreshold,
        mode,
        customInstructions: globalInstructions,
        experiments,
    } = await this.getState();

    const modePrompt = customModePrompts?.[mode] as PromptComponent;
    const effectiveInstructions = [globalInstructions, modePrompt?.customInstructions].filter(Boolean).join("\n\n");

    this.cline = new Cline({
        provider: this,
        apiConfiguration,
        customInstructions: effectiveInstructions,
        enableDiff: diffEnabled,
        enableCheckpoints: checkpointsEnabled,
        fuzzyMatchThreshold,
        task,
        images,
        experiments,
    });
}
```

### 2.2 Clineインスタンスの初期化

1. Clineコンストラクタが呼び出され、必要なサービスとマネージャーが初期化されます
2. `startTask`メソッドが呼び出され、タスクの実行が開始されます

```typescript
// Cline.ts
private async startTask(task?: string, images?: string[]): Promise<void> {
    // 会話履歴の初期化
    this.clineMessages = [];
    this.apiConversationHistory = [];
    await this.providerRef.deref()?.postStateToWebview();

    await this.say("text", task, images);
    this.isInitialized = true;

    let imageBlocks: Anthropic.ImageBlockParam[] = formatResponse.imageBlocks(images);
    await this.initiateTaskLoop([
        {
            type: "text",
            text: `<task>\n${task}\n</task>`,
        },
        ...imageBlocks,
    ]);
}
```

## 3. タスクループの実行

タスクが開始されると、AIモデルとの対話ループが開始されます。

### 3.1 タスクループの開始

1. `initiateTaskLoop`メソッドが呼び出され、ユーザーのコンテンツが渡されます
2. `recursivelyMakeClineRequests`メソッドが呼び出され、AIモデルとの対話が開始されます

```typescript
// Cline.ts
private async initiateTaskLoop(userContent: UserContent): Promise<void> {
    let nextUserContent = userContent;
    let includeFileDetails = true;
    while (!this.abort) {
        const didEndLoop = await this.recursivelyMakeClineRequests(nextUserContent, includeFileDetails);
        includeFileDetails = false; // 最初の呼び出しでのみファイル詳細を含める

        if (didEndLoop) {
            break;
        } else {
            nextUserContent = [
                {
                    type: "text",
                    text: formatResponse.noToolsUsed(),
                },
            ];
            this.consecutiveMistakeCount++;
        }
    }
}
```

### 3.2 AIモデルとの対話

1. `recursivelyMakeClineRequests`メソッドでは、環境の詳細を取得し、APIリクエストを準備します
2. `attemptApiRequest`メソッドが呼び出され、AIモデルにリクエストが送信されます
3. レスポンスがストリーミングされ、処理されます

```typescript
// Cline.ts
async recursivelyMakeClineRequests(
    userContent: UserContent,
    includeFileDetails: boolean = false,
): Promise<boolean> {
    // 環境の詳細を取得
    const [parsedUserContent, environmentDetails] = await this.loadContext(userContent, includeFileDetails);
    userContent = parsedUserContent;
    userContent.push({ type: "text", text: environmentDetails });

    await this.addToApiConversationHistory({ role: "user", content: userContent });

    // APIリクエストを試行
    const stream = this.attemptApiRequest(previousApiReqIndex);
    let assistantMessage = "";
    
    // レスポンスのストリーミングと処理
    for await (const chunk of stream) {
        // チャンクの処理
        switch (chunk.type) {
            case "text":
                assistantMessage += chunk.text;
                this.assistantMessageContent = parseAssistantMessage(assistantMessage);
                this.presentAssistantMessage();
                break;
            // その他のケース
        }
    }
    
    // ユーザーの応答を待機
    await pWaitFor(() => this.userMessageContentReady);
    
    // 次のリクエストを再帰的に呼び出す
    const recDidEndLoop = await this.recursivelyMakeClineRequests(this.userMessageContent);
    return recDidEndLoop;
}
```

## 4. アシスタントのメッセージの処理

AIモデルからのレスポンスは、テキストコンテンツとツール使用に分割されて処理されます。

### 4.1 メッセージの解析

1. `parseAssistantMessage`関数がアシスタントのメッセージを解析します
2. メッセージはテキストコンテンツとツール使用に分割されます

```typescript
// parse-assistant-message.ts
export function parseAssistantMessage(assistantMessage: string) {
    let contentBlocks: AssistantMessageContent[] = [];
    
    // メッセージを解析してコンテンツブロックに分割
    // テキストコンテンツとツール使用を識別
    
    return contentBlocks;
}
```

### 4.2 メッセージの表示

1. `presentAssistantMessage`メソッドがコンテンツブロックを処理します
2. テキストコンテンツはユーザーに表示されます
3. ツール使用は実行されます

```typescript
// Cline.ts
async presentAssistantMessage() {
    // 現在のコンテンツブロックを取得
    const block = cloneDeep(this.assistantMessageContent[this.currentStreamingContentIndex]);

    switch (block.type) {
        case "text":
            // テキストコンテンツを表示
            await this.say("text", content, undefined, block.partial);
            break;
        case "tool_use":
            // ツールの実行
            switch (block.name) {
                case "read_file":
                    // ファイル読み込みツールの実行
                    break;
                case "write_to_file":
                    // ファイル書き込みツールの実行
                    break;
                // その他のツール
            }
            break;
    }
    
    // 次のコンテンツブロックの処理
    this.currentStreamingContentIndex++;
    if (this.currentStreamingContentIndex < this.assistantMessageContent.length) {
        this.presentAssistantMessage();
    }
}
```

## 5. ツールの実行

AIモデルはタスクを完了するために様々なツールを使用します。

### 5.1 ツールの種類

Roo Codeは以下のようなツールを提供しています：

- **ファイル操作**: `read_file`, `write_to_file`, `apply_diff`, `search_files`, `list_files`
- **コード解析**: `list_code_definition_names`
- **ブラウザ操作**: `browser_action`
- **コマンド実行**: `execute_command`
- **MCP操作**: `use_mcp_tool`, `access_mcp_resource`
- **対話**: `ask_followup_question`
- **タスク管理**: `attempt_completion`, `switch_mode`, `new_task`

### 5.2 ツールの実行フロー

1. AIモデルがツールの使用を決定します
2. ツールの使用がXMLタグ形式で出力されます
3. `presentAssistantMessage`メソッドがツールの使用を検出し、対応するメソッドを呼び出します
4. ツールの実行結果がユーザーに表示されます
5. ユーザーからの承認または拒否を待機します
6. 承認された場合、ツールが実行され、結果がAIモデルに返されます

```typescript
// Cline.ts (例: read_fileツールの実行)
case "read_file": {
    const relPath: string | undefined = block.params.path;
    const sharedMessageProps: ClineSayTool = {
        tool: "readFile",
        path: getReadablePath(cwd, removeClosingTag("path", relPath)),
    };
    
    try {
        if (block.partial) {
            // 部分的なツール使用の処理
            const partialMessage = JSON.stringify({
                ...sharedMessageProps,
                content: undefined,
            } satisfies ClineSayTool);
            await this.ask("tool", partialMessage, block.partial).catch(() => {});
            break;
        } else {
            // 完全なツール使用の処理
            if (!relPath) {
                // 必須パラメータの検証
                this.consecutiveMistakeCount++;
                pushToolResult(await this.sayAndCreateMissingParamError("read_file", "path"));
                break;
            }
            this.consecutiveMistakeCount = 0;
            const absolutePath = path.resolve(cwd, relPath);
            const completeMessage = JSON.stringify({
                ...sharedMessageProps,
                content: absolutePath,
            } satisfies ClineSayTool);
            
            // ユーザーの承認を待機
            const didApprove = await askApproval("tool", completeMessage);
            if (!didApprove) {
                break;
            }
            
            // ツールの実行
            const content = await extractTextFromFile(absolutePath);
            pushToolResult(content);
            break;
        }
    } catch (error) {
        await handleError("reading file", error);
        break;
    }
}
```

## 6. タスクの完了

タスクは、AIモデルが`attempt_completion`ツールを使用するか、ユーザーがタスクをキャンセルすると完了します。

### 6.1 attempt_completionツールの使用

1. AIモデルがタスクの完了を判断し、`attempt_completion`ツールを使用します
2. タスクの結果がユーザーに表示されます
3. オプションでコマンドが実行され、結果が表示されます

```typescript
// Cline.ts
case "attempt_completion": {
    const result: string | undefined = block.params.result;
    const command: string | undefined = block.params.command;
    
    try {
        if (!result) {
            // 必須パラメータの検証
            this.consecutiveMistakeCount++;
            pushToolResult(await this.sayAndCreateMissingParamError("attempt_completion", "result"));
            break;
        }
        this.consecutiveMistakeCount = 0;

        // 結果の表示
        await this.say("completion_result", result, undefined, false);
        
        // コマンドの実行（オプション）
        if (command) {
            const didApprove = await askApproval("command", command);
            if (didApprove) {
                const [userRejected, execCommandResult] = await this.executeCommandTool(command!);
                if (!userRejected) {
                    commandResult = execCommandResult;
                }
            }
        }
        
        // ユーザーからのフィードバックを待機
        const { response, text, images } = await this.ask("completion_result", "", false);
        if (response === "yesButtonClicked") {
            // タスク完了
            break;
        }
        
        // フィードバックの処理
        await this.say("user_feedback", text ?? "", images);
        // フィードバックをAIモデルに返す
        break;
    } catch (error) {
        await handleError("completing task", error);
        break;
    }
}
```

### 6.2 タスクのキャンセル

1. ユーザーがタスクをキャンセルすると、`cancelTask`メソッドが呼び出されます
2. 実行中のプロセスが中止され、リソースが解放されます

```typescript
// ClineProvider.ts
async cancelTask() {
    if (this.cline) {
        const { historyItem } = await this.getTaskWithId(this.cline.taskId);
        this.cline.abortTask();

        // タスクの中止を待機
        await pWaitFor(
            () =>
                this.cline === undefined ||
                this.cline.isStreaming === false ||
                this.cline.didFinishAbortingStream ||
                this.cline.isWaitingForFirstChunk,
            {
                timeout: 3_000,
            },
        ).catch(() => {
            console.error("Failed to abort task");
        });

        if (this.cline) {
            this.cline.abandoned = true;
        }

        // タスク履歴からタスクを再開
        await this.initClineWithHistoryItem(historyItem);
    }
}
```

## 7. タスクの状態管理

タスクの状態は、Clineインスタンスとグローバルステートで管理されます。

### 7.1 会話履歴の管理

1. APIとの会話履歴は`apiConversationHistory`配列に保存されます
2. UIメッセージは`clineMessages`配列に保存されます
3. これらの履歴はディスクに保存され、タスクの再開に使用されます

```typescript
// Cline.ts
private async addToApiConversationHistory(message: Anthropic.MessageParam) {
    const messageWithTs = { ...message, ts: Date.now() };
    this.apiConversationHistory.push(messageWithTs);
    await this.saveApiConversationHistory();
}

private async addToClineMessages(message: ClineMessage) {
    this.clineMessages.push(message);
    await this.saveClineMessages();
}
```

### 7.2 チェックポイント機能

1. チェックポイント機能が有効な場合、タスクの進捗状況が保存されます
2. ユーザーはチェックポイントに戻ることができます

```typescript
// Cline.ts
public async checkpointSave({ isFirst }: { isFirst: boolean }) {
    if (!this.checkpointsEnabled) {
        return;
    }

    try {
        const service = await this.getCheckpointService();
        const strategy = service.strategy;
        const version = service.version;

        const commit = await service.saveCheckpoint(`Task: ${this.taskId}, Time: ${Date.now()}`);
        const fromHash = service.baseHash;
        const toHash = isFirst ? commit?.commit || fromHash : commit?.commit;

        if (toHash) {
            await this.providerRef.deref()?.postMessageToWebview({ type: "currentCheckpointUpdated", text: toHash });

            const checkpoint = { isFirst, from: fromHash, to: toHash, strategy, version };
            await this.say("checkpoint_saved", toHash, undefined, undefined, checkpoint);
        }
    } catch (err) {
        this.providerRef.deref()?.log("[checkpointSave] disabling checkpoints for this task");
        this.checkpointsEnabled = false;
    }
}
```

## 8. エラー処理

タスク実行中のエラーは適切に処理され、ユーザーに通知されます。

### 8.1 ツール実行エラーの処理

1. ツールの実行中にエラーが発生すると、`handleError`関数が呼び出されます
2. エラーメッセージがユーザーに表示され、AIモデルに返されます

```typescript
// Cline.ts
const handleError = async (action: string, error: Error) => {
    const errorString = `Error ${action}: ${JSON.stringify(serializeError(error))}`;
    await this.say(
        "error",
        `Error ${action}:\n${error.message ?? JSON.stringify(serializeError(error), null, 2)}`,
    );
    pushToolResult(formatResponse.toolError(errorString));
}
```

### 8.2 APIリクエストエラーの処理

1. APIリクエスト中にエラーが発生すると、ユーザーに通知されます
2. ユーザーはリトライするか、タスクをキャンセルするかを選択できます

```typescript
// Cline.ts
try {
    // APIリクエストの試行
} catch (error) {
    const { response } = await this.ask(
        "api_req_failed",
        error.message ?? JSON.stringify(serializeError(error), null, 2),
    );
    if (response !== "yesButtonClicked") {
        throw new Error("API request failed");
    }
    await this.say("api_req_retried");
    yield* this.attemptApiRequest(previousApiReqIndex);
    return;
}
```

## 9. まとめ

Roo Codeのタスク実行フローは、ユーザー入力の処理から始まり、AIモデルとの対話、ツールの実行、そしてタスクの完了までの一連のプロセスを含みます。このフローは、ユーザーとAIモデルの間の効果的な対話を可能にし、複雑なタスクを段階的に実行することができます。

主要なコンポーネントは以下の通りです：

1. **ClineProvider**: ウェブビューとの通信を担当し、Clineインスタンスを管理します
2. **Cline**: タスクの実行を担当し、AIモデルとの通信、ツールの実行、状態管理を行います
3. **ツール**: タスクを完了するために使用される様々な機能を提供します
4. **状態管理**: タスクの状態、会話履歴、チェックポイントを管理します

このアーキテクチャにより、Roo Codeは柔軟で拡張可能なタスク実行フローを実現しています。