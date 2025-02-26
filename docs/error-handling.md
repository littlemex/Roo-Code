# エラー処理

このドキュメントでは、Roo Code拡張機能におけるエラー処理の仕組みについて説明します。エラー処理は、タスク実行中に発生する様々な問題を検出し、適切に対応するための重要なコンポーネントです。

## 1. エラー処理の概要

Roo Codeのエラー処理は、以下の主要なカテゴリで構成されています：

1. **APIリクエストエラー**: AIモデルとの通信中に発生するエラー
2. **ツール実行エラー**: ツールの実行中に発生するエラー
3. **パラメータ検証エラー**: ツールのパラメータが不足または無効な場合のエラー
4. **ファイル操作エラー**: ファイルの読み書き中に発生するエラー
5. **ブラウザ操作エラー**: ブラウザの操作中に発生するエラー
6. **コマンド実行エラー**: システムコマンドの実行中に発生するエラー
7. **MCP操作エラー**: MCPサーバーとの通信中に発生するエラー

各カテゴリのエラーは、ユーザーに適切に通知され、AIモデルに返されて次のステップを決定するために使用されます。

## 2. APIリクエストエラー

AIモデルとの通信中に発生するエラーは、`attemptApiRequest`メソッドで処理されます。

### 2.1 エラーの種類

APIリクエストエラーには、以下のような種類があります：

- **接続エラー**: APIサーバーに接続できない
- **認証エラー**: APIキーが無効または期限切れ
- **レート制限エラー**: APIリクエストの制限に達した
- **サーバーエラー**: APIサーバーの内部エラー
- **タイムアウトエラー**: リクエストがタイムアウトした

### 2.2 エラー処理フロー

APIリクエストエラーの処理フローは以下の通りです：

1. エラーが発生すると、ユーザーに通知されます
2. ユーザーはリトライするか、タスクをキャンセルするかを選択できます
3. リトライを選択した場合、APIリクエストが再試行されます
4. 自動リトライが有効な場合、指数バックオフを使用して自動的に再試行されます

```typescript
// Cline.ts
async *attemptApiRequest(previousApiReqIndex: number, retryAttempt: number = 0): ApiStream {
    // ...
    try {
        // 最初のチャンクを待機してエラーを検出
        this.isWaitingForFirstChunk = true;
        const firstChunk = await iterator.next();
        yield firstChunk.value;
        this.isWaitingForFirstChunk = false;
    } catch (error) {
        if (alwaysApproveResubmit) {
            const errorMsg = error.message ?? "Unknown error";
            const baseDelay = requestDelaySeconds || 5;
            const exponentialDelay = Math.ceil(baseDelay * Math.pow(2, retryAttempt));
            const finalDelay = Math.max(exponentialDelay, rateLimitDelay);

            // 指数バックオフを使用したカウントダウンタイマーを表示
            for (let i = finalDelay; i > 0; i--) {
                await this.say(
                    "api_req_retry_delayed",
                    `${errorMsg}\n\nRetry attempt ${retryAttempt + 1}\nRetrying in ${i} seconds...`,
                    undefined,
                    true,
                );
                await delay(1000);
            }

            await this.say(
                "api_req_retry_delayed",
                `${errorMsg}\n\nRetry attempt ${retryAttempt + 1}\nRetrying now...`,
                undefined,
                false,
            );

            // 再帰的な呼び出しでリトライカウントを増やす
            yield* this.attemptApiRequest(previousApiReqIndex, retryAttempt + 1);
            return;
        } else {
            const { response } = await this.ask(
                "api_req_failed",
                error.message ?? JSON.stringify(serializeError(error), null, 2),
            );
            if (response !== "yesButtonClicked") {
                // ユーザーがリトライを拒否した場合
                throw new Error("API request failed");
            }
            await this.say("api_req_retried");
            // 再帰的な呼び出しでリトライ
            yield* this.attemptApiRequest(previousApiReqIndex);
            return;
        }
    }
    // ...
}
```

### 2.3 ストリーミングエラーの処理

APIリクエストのストリーミング中にエラーが発生した場合、以下のように処理されます：

1. ストリーミングが中断されます
2. 現在のタスクが中止されます
3. エラーメッセージがユーザーに表示されます
4. タスク履歴からタスクが再開されます

```typescript
// Cline.ts
try {
    // ストリーミングの処理
    for await (const chunk of stream) {
        // チャンクの処理
    }
} catch (error) {
    if (!this.abandoned) {
        this.abortTask(); // タスクを中止
        await abortStream(
            "streaming_failed",
            error.message ?? JSON.stringify(serializeError(error), null, 2),
        );
        const history = await this.providerRef.deref()?.getTaskWithId(this.taskId);
        if (history) {
            await this.providerRef.deref()?.initClineWithHistoryItem(history.historyItem);
        }
    }
} finally {
    this.isStreaming = false;
}
```

## 3. ツール実行エラー

ツールの実行中に発生するエラーは、各ツールの実行メソッドで処理されます。

### 3.1 エラーの検出

ツール実行エラーは、try-catchブロックで検出されます。

```typescript
// Cline.ts
try {
    // ツールの実行
    const content = await extractTextFromFile(absolutePath);
    pushToolResult(content);
} catch (error) {
    await handleError("reading file", error);
    break;
}
```

### 3.2 エラーの処理

ツール実行エラーは、`handleError`関数で処理されます。

```typescript
// Cline.ts
const handleError = async (action: string, error: Error) => {
    const errorString = `Error ${action}: ${JSON.stringify(serializeError(error))}`;
    await this.say(
        "error",
        `Error ${action}:\n${error.message ?? JSON.stringify(serializeError(error), null, 2)}`,
    );
    pushToolResult(formatResponse.toolError(errorString));
};
```

### 3.3 エラーメッセージのフォーマット

エラーメッセージは、`formatResponse.toolError`関数でフォーマットされます。

```typescript
// responses.ts
export const formatResponse = {
    toolError: (error?: string) => `The tool execution failed with the following error:\n<error>\n${error}\n</error>`,
    // その他のフォーマット関数
};
```

## 4. パラメータ検証エラー

ツールのパラメータが不足または無効な場合、エラーが発生します。

### 4.1 必須パラメータの検証

各ツールの実行前に、必須パラメータが存在するかが検証されます。

```typescript
// Cline.ts
if (!relPath) {
    this.consecutiveMistakeCount++;
    pushToolResult(await this.sayAndCreateMissingParamError("read_file", "path"));
    break;
}
```

### 4.2 パラメータ検証エラーの処理

パラメータ検証エラーは、`sayAndCreateMissingParamError`メソッドで処理されます。

```typescript
// Cline.ts
async sayAndCreateMissingParamError(toolName: ToolUseName, paramName: string, relPath?: string) {
    await this.say(
        "error",
        `Roo tried to use ${toolName}${
            relPath ? ` for '${relPath.toPosix()}'` : ""
        } without value for required parameter '${paramName}'. Retrying...`,
    );
    return formatResponse.toolError(formatResponse.missingToolParameterError(paramName));
}
```

### 4.3 パラメータ検証エラーメッセージのフォーマット

パラメータ検証エラーメッセージは、`formatResponse.missingToolParameterError`関数でフォーマットされます。

```typescript
// responses.ts
export const formatResponse = {
    missingToolParameterError: (paramName: string) =>
        `Missing value for required parameter '${paramName}'. Please retry with complete response.\n\n${toolUseInstructionsReminder}`,
    // その他のフォーマット関数
};
```

## 5. ファイル操作エラー

ファイルの読み書き中に発生するエラーは、各ファイル操作メソッドで処理されます。

### 5.1 ファイル読み込みエラー

ファイルの読み込み中にエラーが発生した場合、以下のように処理されます：

```typescript
// Cline.ts
try {
    const content = await extractTextFromFile(absolutePath);
    pushToolResult(content);
} catch (error) {
    await handleError("reading file", error);
    break;
}
```

### 5.2 ファイル書き込みエラー

ファイルの書き込み中にエラーが発生した場合、以下のように処理されます：

```typescript
// Cline.ts
try {
    await this.diffViewProvider.update(
        everyLineHasLineNumbers(newContent) ? stripLineNumbers(newContent) : newContent,
        true,
    );
    // ファイルの保存
    const { newProblemsMessage, userEdits, finalContent } = await this.diffViewProvider.saveChanges();
    // 結果の処理
} catch (error) {
    await handleError("writing file", error);
    await this.diffViewProvider.reset();
    break;
}
```

### 5.3 ファイル差分適用エラー

ファイルの差分適用中にエラーが発生した場合、以下のように処理されます：

```typescript
// Cline.ts
const diffResult = (await this.diffStrategy?.applyDiff(
    originalContent,
    diffContent,
    parseInt(block.params.start_line ?? ""),
    parseInt(block.params.end_line ?? ""),
)) ?? {
    success: false,
    error: "No diff strategy available",
};
if (!diffResult.success) {
    this.consecutiveMistakeCount++;
    const currentCount = (this.consecutiveMistakeCountForApplyDiff.get(relPath) || 0) + 1;
    this.consecutiveMistakeCountForApplyDiff.set(relPath, currentCount);
    const errorDetails = diffResult.details
        ? JSON.stringify(diffResult.details, null, 2)
        : "";
    const formattedError = `Unable to apply diff to file: ${absolutePath}\n\n<error_details>\n${
        diffResult.error
    }${errorDetails ? `\n\nDetails:\n${errorDetails}` : ""}\n</error_details>`;
    if (currentCount >= 2) {
        await this.say("error", formattedError);
    }
    pushToolResult(formattedError);
    break;
}
```

## 6. ブラウザ操作エラー

ブラウザの操作中に発生するエラーは、`browser_action`ツールの実行メソッドで処理されます。

### 6.1 ブラウザ起動エラー

ブラウザの起動中にエラーが発生した場合、以下のように処理されます：

```typescript
// Cline.ts
try {
    await this.browserSession.launchBrowser();
    browserActionResult = await this.browserSession.navigateToUrl(url);
} catch (error) {
    await this.browserSession.closeBrowser();
    await handleError("executing browser action", error);
    break;
}
```

### 6.2 ブラウザ操作エラー

ブラウザの操作中にエラーが発生した場合、以下のように処理されます：

```typescript
// Cline.ts
try {
    switch (action) {
        case "click":
            browserActionResult = await this.browserSession.click(coordinate!);
            break;
        // その他のアクション
    }
} catch (error) {
    await this.browserSession.closeBrowser();
    await handleError("executing browser action", error);
    break;
}
```

### 6.3 ブラウザクリーンアップ

エラーが発生した場合でも、ブラウザセッションは適切にクリーンアップされます：

```typescript
// Cline.ts
try {
    // ブラウザ操作
} catch (error) {
    await this.browserSession.closeBrowser(); // ブラウザセッションを終了
    await handleError("executing browser action", error);
    break;
}
```

## 7. コマンド実行エラー

システムコマンドの実行中に発生するエラーは、`executeCommandTool`メソッドで処理されます。

### 7.1 コマンド実行エラーの検出

コマンド実行エラーは、ターミナルマネージャーによって検出されます：

```typescript
// Cline.ts
const process = this.terminalManager.runCommand(terminalInfo, command);

process.once("no_shell_integration", async () => {
    await this.say("shell_integration_warning");
});

try {
    await process;
} catch (error) {
    await handleError("executing command", error);
    break;
}
```

### 7.2 コマンド出力の処理

コマンドの出力は、成功した場合でもエラーメッセージを含む可能性があります。出力は適切に処理され、AIモデルに返されます：

```typescript
// Cline.ts
const output = truncateOutput(lines.join("\n"), terminalOutputLineLimit);
const result = output.trim();

if (completed) {
    return [false, `Command executed.${result.length > 0 ? `\nOutput:\n${result}` : ""}`];
} else {
    return [
        false,
        `Command is still running in the user's terminal.${
            result.length > 0 ? `\nHere's the output so far:\n${result}` : ""
        }\n\nYou will be updated on the terminal status and new output in the future.`,
    ];
}
```

## 8. MCP操作エラー

MCPサーバーとの通信中に発生するエラーは、`use_mcp_tool`と`access_mcp_resource`ツールの実行メソッドで処理されます。

### 8.1 MCPツール実行エラー

MCPツールの実行中にエラーが発生した場合、以下のように処理されます：

```typescript
// Cline.ts
try {
    const toolResult = await this.providerRef
        .deref()
        ?.getMcpHub()
        ?.callTool(server_name, tool_name, parsedArguments);

    const toolResultPretty =
        (toolResult?.isError ? "Error:\n" : "") +
            toolResult?.content
                .map((item) => {
                    if (item.type === "text") {
                        return item.text;
                    }
                    if (item.type === "resource") {
                        const { blob, ...rest } = item.resource;
                        return JSON.stringify(rest, null, 2);
                    }
                    return "";
                })
                .filter(Boolean)
                .join("\n\n") || "(No response)";
    await this.say("mcp_server_response", toolResultPretty);
    pushToolResult(formatResponse.toolResult(toolResultPretty));
} catch (error) {
    await handleError("executing MCP tool", error);
    break;
}
```

### 8.2 MCPリソースアクセスエラー

MCPリソースへのアクセス中にエラーが発生した場合、以下のように処理されます：

```typescript
// Cline.ts
try {
    const resourceResult = await this.providerRef
        .deref()
        ?.getMcpHub()
        ?.readResource(server_name, uri);
    const resourceResultPretty =
        resourceResult?.contents
            .map((item) => {
                if (item.text) {
                    return item.text;
                }
                return "";
            })
            .filter(Boolean)
            .join("\n\n") || "(Empty response)";
    await this.say("mcp_server_response", resourceResultPretty);
    pushToolResult(formatResponse.toolResult(resourceResultPretty));
} catch (error) {
    await handleError("accessing MCP resource", error);
    break;
}
```

## 9. 連続エラーの処理

AIモデルが連続してエラーを発生させる場合、特別な処理が行われます。

### 9.1 連続エラーカウンターの管理

連続エラーは、`consecutiveMistakeCount`変数でカウントされます：

```typescript
// Cline.ts
if (!relPath) {
    this.consecutiveMistakeCount++;
    pushToolResult(await this.sayAndCreateMissingParamError("read_file", "path"));
    break;
}
this.consecutiveMistakeCount = 0; // エラーがない場合はリセット
```

### 9.2 連続エラーの処理

連続エラーが一定回数に達すると、ユーザーにフィードバックを求めることができます：

```typescript
// Cline.ts
if (this.consecutiveMistakeCount >= 3) {
    const { response, text, images } = await this.ask(
        "mistake_limit_reached",
        this.api.getModel().id.includes("claude")
            ? `This may indicate a failure in his thought process or inability to use a tool properly, which can be mitigated with some user guidance (e.g. "Try breaking down the task into smaller steps").`
            : "Roo Code uses complex prompts and iterative task execution that may be challenging for less capable models. For best results, it's recommended to use Claude 3.7 Sonnet for its advanced agentic coding capabilities.",
    );
    if (response === "messageResponse") {
        userContent.push(
            ...[
                {
                    type: "text",
                    text: formatResponse.tooManyMistakes(text),
                } as Anthropic.Messages.TextBlockParam,
                ...formatResponse.imageBlocks(images),
            ],
        );
    }
    this.consecutiveMistakeCount = 0;
}
```

### 9.3 ツール固有の連続エラーカウンター

一部のツールでは、ツール固有の連続エラーカウンターが使用されます：

```typescript
// Cline.ts
if (!diffResult.success) {
    this.consecutiveMistakeCount++;
    const currentCount = (this.consecutiveMistakeCountForApplyDiff.get(relPath) || 0) + 1;
    this.consecutiveMistakeCountForApplyDiff.set(relPath, currentCount);
    // エラーの処理
    if (currentCount >= 2) {
        await this.say("error", formattedError);
    }
    pushToolResult(formattedError);
    break;
}
```

## 10. エラーメッセージの表示

エラーメッセージは、ユーザーに適切に表示されます。

### 10.1 エラーメッセージの種類

エラーメッセージには、以下の種類があります：

- **一般エラー**: `error`タイプのメッセージ
- **ツールエラー**: `toolError`関数でフォーマットされたメッセージ
- **パラメータエラー**: `missingToolParameterError`関数でフォーマットされたメッセージ
- **APIエラー**: `api_req_failed`タイプのメッセージ

### 10.2 エラーメッセージの表示方法

エラーメッセージは、`say`メソッドを使用してユーザーに表示されます：

```typescript
// Cline.ts
await this.say(
    "error",
    `Error ${action}:\n${error.message ?? JSON.stringify(serializeError(error), null, 2)}`,
);
```

### 10.3 エラーメッセージのフォーマット

エラーメッセージは、適切にフォーマットされてユーザーに表示されます：

```typescript
// responses.ts
export const formatResponse = {
    toolError: (error?: string) => `The tool execution failed with the following error:\n<error>\n${error}\n</error>`,
    missingToolParameterError: (paramName: string) =>
        `Missing value for required parameter '${paramName}'. Please retry with complete response.\n\n${toolUseInstructionsReminder}`,
    tooManyMistakes: (feedback?: string) =>
        `You seem to be having trouble proceeding. The user has provided the following feedback to help guide you:\n<feedback>\n${feedback}\n</feedback>`,
    // その他のフォーマット関数
};
```

## 11. エラーからの回復

エラーが発生した場合、システムは適切に回復を試みます。

### 11.1 APIリクエストエラーからの回復

APIリクエストエラーが発生した場合、以下の回復方法があります：

- **自動リトライ**: 指数バックオフを使用して自動的に再試行
- **ユーザー承認リトライ**: ユーザーの承認を得て再試行
- **タスクの中止**: ユーザーがタスクをキャンセルすることを選択

### 11.2 ツール実行エラーからの回復

ツール実行エラーが発生した場合、以下の回復方法があります：

- **エラーの通知**: エラーメッセージがユーザーとAIモデルに通知される
- **別のアプローチ**: AIモデルが別のアプローチを試みる
- **ユーザーのフィードバック**: ユーザーがフィードバックを提供し、AIモデルが対応する

### 11.3 タスクの中止と再開

深刻なエラーが発生した場合、タスクを中止して再開することができます：

```typescript
// Cline.ts
async abortTask(isAbandoned = false) {
    if (isAbandoned) {
        this.abandoned = true;
    }

    this.abort = true;

    this.terminalManager.disposeAll();
    this.urlContentFetcher.closeBrowser();
    this.browserSession.closeBrowser();

    if (this.isStreaming && this.diffViewProvider.isEditing) {
        await this.diffViewProvider.revertChanges();
    }
}
```

## 12. まとめ

Roo Codeのエラー処理は、タスク実行中に発生する様々な問題を検出し、適切に対応するための重要なコンポーネントです。エラー処理の主要な特徴は以下の通りです：

1. **包括的なエラー検出**: 様々な種類のエラーを検出し、適切に処理します
2. **ユーザーへの通知**: エラーメッセージがユーザーに適切に表示されます
3. **AIモデルへのフィードバック**: エラー情報がAIモデルに返され、次のステップを決定するために使用されます
4. **回復メカニズム**: エラーからの回復を試みるための様々なメカニズムが提供されます
5. **連続エラーの処理**: AIモデルが連続してエラーを発生させる場合、特別な処理が行われます

このアーキテクチャにより、Roo Codeは堅牢なエラー処理を実現し、タスク実行の信頼性を向上させています。