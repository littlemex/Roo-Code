# ツール実行フロー

このドキュメントでは、Roo Code拡張機能におけるツールの実行フローについて説明します。ツール実行フローは、AIモデルがタスクを完了するために使用するツールの登録、実行、結果の処理のプロセスを表します。

## 1. ツール実行フローの概要

Roo Codeのツール実行フローは、以下の主要なステップで構成されています：

1. **ツールの定義**: 利用可能なツールとその説明が定義されます
2. **ツールの選択**: AIモデルがタスクを分析し、適切なツールを選択します
3. **ツールの実行**: 選択されたツールが実行され、結果が生成されます
4. **結果の処理**: ツールの実行結果がAIモデルに返され、次のステップが決定されます

## 2. ツールの定義

Roo Codeは、様々なタスクを実行するための多くのツールを提供しています。

### 2.1 ツールの種類

ツールは以下のカテゴリに分類されます：

1. **ファイル操作ツール**:
   - `read_file`: ファイルの内容を読み取る
   - `write_to_file`: ファイルに内容を書き込む
   - `apply_diff`: 差分をファイルに適用する
   - `search_files`: ファイル内でテキストを検索する
   - `list_files`: ディレクトリ内のファイルを一覧表示する
   - `search_and_replace`: ファイル内のテキストを検索して置換する
   - `insert_content`: ファイルに内容を挿入する

2. **コード解析ツール**:
   - `list_code_definition_names`: ソースコードの定義名を一覧表示する

3. **ブラウザ操作ツール**:
   - `browser_action`: ブラウザを操作する（起動、クリック、スクロールなど）

4. **コマンド実行ツール**:
   - `execute_command`: システムコマンドを実行する

5. **MCP操作ツール**:
   - `use_mcp_tool`: MCPサーバーのツールを使用する
   - `access_mcp_resource`: MCPサーバーのリソースにアクセスする

6. **対話ツール**:
   - `ask_followup_question`: ユーザーに追加の質問をする

7. **タスク管理ツール**:
   - `attempt_completion`: タスクの完了を試みる
   - `switch_mode`: モードを切り替える
   - `new_task`: 新しいタスクを作成する

### 2.2 ツールの定義方法

各ツールは、`src/core/prompts/tools/`ディレクトリ内の個別のファイルで定義されています。例えば、`read_file`ツールは`read-file.ts`ファイルで定義されています。

ツールの定義には、以下の情報が含まれます：

- **名前**: ツールの一意の識別子
- **説明**: ツールの機能と使用方法の説明
- **パラメータ**: ツールが受け取るパラメータとその説明
- **使用例**: ツールの使用例

```typescript
// read-file.ts (例)
export function getReadFileDescription(args: ToolArgs): string {
    return `## read_file
Description: Request to read the contents of a file at the specified path. Use this when you need to examine the contents of an existing file you do not know the contents of, for example to analyze code, review text files, or extract information from configuration files. The output includes line numbers prefixed to each line (e.g. "1 | const x = 1"), making it easier to reference specific lines when creating diffs or discussing code. Automatically extracts raw text from PDF and DOCX files. May not be suitable for other types of binary files, as it returns the raw content as a string.
Parameters:
- path: (required) The path of the file to read (relative to the current working directory ${args.cwd})
Usage:
<read_file>
<path>File path here</path>
</read_file>

Example: Requesting to read frontend-config.json
<read_file>
<path>frontend-config.json</path>
</read_file>`;
}
```

### 2.3 ツールの登録

ツールは`src/core/prompts/tools/index.ts`ファイルで登録されています。このファイルでは、各ツールの説明関数がマップされ、モードに基づいて利用可能なツールが決定されます。

```typescript
// tools/index.ts
const toolDescriptionMap: Record<string, (args: ToolArgs) => string | undefined> = {
    execute_command: (args) => getExecuteCommandDescription(args),
    read_file: (args) => getReadFileDescription(args),
    write_to_file: (args) => getWriteToFileDescription(args),
    // その他のツール
};

export function getToolDescriptionsForMode(
    mode: Mode,
    cwd: string,
    supportsComputerUse: boolean,
    diffStrategy?: DiffStrategy,
    browserViewportSize?: string,
    mcpHub?: McpHub,
    customModes?: ModeConfig[],
    experiments?: Record<string, boolean>,
): string {
    const config = getModeConfig(mode, customModes);
    // モードに基づいて利用可能なツールを決定
    // ツールの説明を生成
    return `# Tools\n\n${descriptions.filter(Boolean).join("\n\n")}`;
}
```

## 3. ツールの選択

AIモデルは、タスクを分析し、適切なツールを選択します。

### 3.1 ツールの選択プロセス

1. AIモデルはタスクの要件を分析します
2. 利用可能なツールの中から、タスクに最適なツールを選択します
3. 選択したツールのパラメータを決定します
4. ツールの使用をXMLタグ形式で出力します

```
<tool_name>
<parameter1_name>value1</parameter1_name>
<parameter2_name>value2</parameter2_name>
...
</tool_name>
```

### 3.2 ツールの使用制限

モードに基づいて、AIモデルが使用できるツールは制限されます。例えば、「Architect」モードでは、ファイル編集ツールは使用できません。

```typescript
// mode-validator.ts
export function isToolAllowedForMode(
    toolName: ToolName,
    mode: Mode,
    customModes: ModeConfig[],
    toolOptions?: Record<string, boolean>,
): boolean {
    // モードに基づいてツールの使用が許可されているかを確認
    const config = getModeConfig(mode, customModes);
    
    // ツールが常に利用可能なツールリストに含まれているか確認
    if (ALWAYS_AVAILABLE_TOOLS.includes(toolName)) {
        return true;
    }
    
    // モードのグループに基づいてツールの使用が許可されているか確認
    for (const groupEntry of config.groups) {
        const groupName = getGroupName(groupEntry);
        const toolGroup = TOOL_GROUPS[groupName];
        if (toolGroup && toolGroup.tools.includes(toolName)) {
            // グループに制限がある場合は確認
            if (Array.isArray(groupEntry)) {
                const [_, restrictions] = groupEntry;
                // 制限に基づいてツールの使用が許可されているか確認
                // ...
            }
            return true;
        }
    }
    
    return false;
}
```

## 4. ツールの実行

AIモデルがツールの使用を決定すると、Clineクラスがツールを実行します。

### 4.1 ツールの使用の検出

1. AIモデルのレスポンスがストリーミングされます
2. `parseAssistantMessage`関数がレスポンスを解析し、ツールの使用を検出します
3. `presentAssistantMessage`メソッドがツールの使用を処理します

```typescript
// parse-assistant-message.ts
export function parseAssistantMessage(assistantMessage: string) {
    let contentBlocks: AssistantMessageContent[] = [];
    // メッセージを解析してコンテンツブロックに分割
    // テキストコンテンツとツール使用を識別
    
    for (let i = 0; i < assistantMessage.length; i++) {
        // ツールの使用を検出
        const possibleToolUseOpeningTags = toolUseNames.map((name) => `<${name}>`);
        for (const toolUseOpeningTag of possibleToolUseOpeningTags) {
            if (accumulator.endsWith(toolUseOpeningTag)) {
                // ツールの使用を検出
                currentToolUse = {
                    type: "tool_use",
                    name: toolUseOpeningTag.slice(1, -1) as ToolUseName,
                    params: {},
                    partial: true,
                };
                // パラメータの解析
                // ...
            }
        }
    }
    
    return contentBlocks;
}
```

### 4.2 ツールの実行プロセス

1. `presentAssistantMessage`メソッドがツールの種類に基づいて対応するメソッドを呼び出します
2. ツールのパラメータが検証されます
3. ユーザーの承認が必要な場合は、承認を待機します
4. ツールが実行され、結果が生成されます

```typescript
// Cline.ts
async presentAssistantMessage() {
    // 現在のコンテンツブロックを取得
    const block = cloneDeep(this.assistantMessageContent[this.currentStreamingContentIndex]);

    switch (block.type) {
        case "tool_use":
            // ツールの実行
            switch (block.name) {
                case "read_file": {
                    const relPath: string | undefined = block.params.path;
                    // パラメータの検証
                    if (!relPath) {
                        pushToolResult(await this.sayAndCreateMissingParamError("read_file", "path"));
                        break;
                    }
                    
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
                // その他のツール
            }
            break;
    }
}
```

### 4.3 ユーザーの承認プロセス

多くのツールは、実行前にユーザーの承認が必要です。承認プロセスは以下の通りです：

1. ツールの使用がユーザーに表示されます
2. ユーザーはツールの使用を承認または拒否します
3. 承認された場合、ツールが実行されます
4. 拒否された場合、ツールは実行されず、拒否メッセージがAIモデルに返されます

```typescript
// Cline.ts
const askApproval = async (type: ClineAsk, partialMessage?: string) => {
    const { response, text, images } = await this.ask(type, partialMessage, false);
    if (response !== "yesButtonClicked") {
        // 拒否された場合
        if (text) {
            await this.say("user_feedback", text, images);
            pushToolResult(
                formatResponse.toolResult(formatResponse.toolDeniedWithFeedback(text), images),
            );
        } else {
            pushToolResult(formatResponse.toolDenied());
        }
        this.didRejectTool = true;
        return false;
    }
    // 承認された場合
    if (text) {
        await this.say("user_feedback", text, images);
        pushToolResult(formatResponse.toolResult(formatResponse.toolApprovedWithFeedback(text), images));
    }
    return true;
};
```

## 5. 結果の処理

ツールの実行結果は、AIモデルに返され、次のステップが決定されます。

### 5.1 結果の形式

ツールの実行結果は、テキストまたはテキストと画像のブロックの配列として返されます。

```typescript
// responses.ts
export const formatResponse = {
    toolResult: (
        text: string,
        images?: string[],
    ): string | Array<Anthropic.TextBlockParam | Anthropic.ImageBlockParam> => {
        if (images && images.length > 0) {
            const textBlock: Anthropic.TextBlockParam = { type: "text", text };
            const imageBlocks: Anthropic.ImageBlockParam[] = formatImagesIntoBlocks(images);
            return [textBlock, ...imageBlocks];
        } else {
            return text;
        }
    },
    // その他のフォーマット関数
};
```

### 5.2 結果の返却

ツールの実行結果は、`userMessageContent`配列に追加され、AIモデルに返されます。

```typescript
// Cline.ts
const pushToolResult = (content: ToolResponse) => {
    this.userMessageContent.push({
        type: "text",
        text: `${toolDescription()} Result:`,
    });
    if (typeof content === "string") {
        this.userMessageContent.push({
            type: "text",
            text: content || "(tool did not return anything)",
        });
    } else {
        this.userMessageContent.push(...content);
    }
    // ツール結果が収集されたら、他のツールの使用を無視
    this.didAlreadyUseTool = true;
};
```

### 5.3 次のステップの決定

AIモデルは、ツールの実行結果に基づいて次のステップを決定します。

1. 結果が成功した場合、AIモデルはタスクを続行します
2. 結果がエラーを含む場合、AIモデルはエラーを処理し、別のアプローチを試みます
3. タスクが完了した場合、AIモデルは`attempt_completion`ツールを使用してタスクを完了します

## 6. 特殊なツールの実行フロー

一部のツールは、特殊な実行フローを持っています。

### 6.1 ファイル編集ツール

`write_to_file`や`apply_diff`などのファイル編集ツールは、以下の特殊なフローを持っています：

1. ファイルの内容が変更されます
2. 変更がDiffViewProviderに表示されます
3. ユーザーは変更を承認または拒否します
4. 承認された場合、変更が保存されます
5. ユーザーが編集を加えた場合、編集が保存され、AIモデルに通知されます

```typescript
// Cline.ts (write_to_fileの例)
case "write_to_file": {
    // パラメータの検証
    // ...
    
    // DiffViewProviderを使用してファイルの変更を表示
    if (!this.diffViewProvider.isEditing) {
        await this.diffViewProvider.open(relPath);
    }
    await this.diffViewProvider.update(
        everyLineHasLineNumbers(newContent) ? stripLineNumbers(newContent) : newContent,
        true,
    );
    
    // ユーザーの承認を待機
    const didApprove = await askApproval("tool", completeMessage);
    if (!didApprove) {
        await this.diffViewProvider.revertChanges();
        break;
    }
    
    // 変更を保存
    const { newProblemsMessage, userEdits, finalContent } = await this.diffViewProvider.saveChanges();
    this.didEditFile = true;
    
    // ユーザーが編集を加えた場合
    if (userEdits) {
        await this.say(
            "user_feedback_diff",
            JSON.stringify({
                tool: fileExists ? "editedExistingFile" : "newFileCreated",
                path: getReadablePath(cwd, relPath),
                diff: userEdits,
            } satisfies ClineSayTool),
        );
        pushToolResult(
            `The user made the following updates to your content:\n\n${userEdits}\n\n` +
                `The updated content, which includes both your original modifications and the user's edits, has been successfully saved to ${relPath.toPosix()}. Here is the full, updated content of the file, including line numbers:\n\n` +
                `<final_file_content path="${relPath.toPosix()}">\n${addLineNumbers(
                    finalContent || "",
                )}\n</final_file_content>\n\n` +
                `Please note:\n` +
                `1. You do not need to re-write the file with these changes, as they have already been applied.\n` +
                `2. Proceed with the task using this updated file content as the new baseline.\n` +
                `3. If the user's edits have addressed part of the task or changed the requirements, adjust your approach accordingly.` +
                `${newProblemsMessage}`,
        );
    } else {
        pushToolResult(
            `The content was successfully saved to ${relPath.toPosix()}.${newProblemsMessage}`,
        );
    }
    await this.diffViewProvider.reset();
    break;
}
```

### 6.2 ブラウザ操作ツール

`browser_action`ツールは、以下の特殊なフローを持っています：

1. ブラウザセッションが管理されます
2. ブラウザアクションが実行されます（起動、クリック、スクロールなど）
3. スクリーンショットとコンソールログが取得されます
4. 結果がAIモデルに返されます

```typescript
// Cline.ts (browser_actionの例)
case "browser_action": {
    const action: BrowserAction | undefined = block.params.action as BrowserAction;
    // パラメータの検証
    // ...
    
    try {
        let browserActionResult: BrowserActionResult;
        if (action === "launch") {
            // ブラウザの起動
            await this.browserSession.launchBrowser();
            browserActionResult = await this.browserSession.navigateToUrl(url);
        } else {
            // その他のブラウザアクション
            switch (action) {
                case "click":
                    browserActionResult = await this.browserSession.click(coordinate!);
                    break;
                case "type":
                    browserActionResult = await this.browserSession.type(text!);
                    break;
                case "scroll_down":
                    browserActionResult = await this.browserSession.scrollDown();
                    break;
                case "scroll_up":
                    browserActionResult = await this.browserSession.scrollUp();
                    break;
                case "close":
                    browserActionResult = await this.browserSession.closeBrowser();
                    break;
            }
        }
        
        // 結果の返却
        await this.say("browser_action_result", JSON.stringify(browserActionResult));
        pushToolResult(
            formatResponse.toolResult(
                `The browser action has been executed. The console logs and screenshot have been captured for your analysis.\n\nConsole logs:\n${
                    browserActionResult.logs || "(No new logs)"
                }\n\n(REMEMBER: if you need to proceed to using non-\`browser_action\` tools or launch a new browser, you MUST first close this browser. For example, if after analyzing the logs and screenshot you need to edit a file, you must first close the browser before you can use the write_to_file tool.)`,
                browserActionResult.screenshot ? [browserActionResult.screenshot] : [],
            ),
        );
        break;
    } catch (error) {
        await this.browserSession.closeBrowser();
        await handleError("executing browser action", error);
        break;
    }
}
```

### 6.3 コマンド実行ツール

`execute_command`ツールは、以下の特殊なフローを持っています：

1. ターミナルマネージャーがコマンドを実行します
2. コマンドの出力がストリーミングされます
3. ユーザーはコマンドの実行中に入力を提供できます
4. コマンドの完了後、結果がAIモデルに返されます

```typescript
// Cline.ts (execute_commandの例)
async executeCommandTool(command: string): Promise<[boolean, ToolResponse]> {
    const terminalInfo = await this.terminalManager.getOrCreateTerminal(cwd);
    terminalInfo.terminal.show();
    const process = this.terminalManager.runCommand(terminalInfo, command);

    let userFeedback: { text?: string; images?: string[] } | undefined;
    let didContinue = false;
    const sendCommandOutput = async (line: string): Promise<void> => {
        try {
            const { response, text, images } = await this.ask("command_output", line);
            if (response === "yesButtonClicked") {
                // 実行中に進行
            } else {
                userFeedback = { text, images };
            }
            didContinue = true;
            process.continue(); // awaitを過ぎて続行
        } catch {
            // このaskプロミスが無視された場合のみ発生するため、このエラーを無視
        }
    };

    let lines: string[] = [];
    process.on("line", (line) => {
        lines.push(line);
        if (!didContinue) {
            sendCommandOutput(line);
        } else {
            this.say("command_output", line);
        }
    });

    let completed = false;
    process.once("completed", () => {
        completed = true;
    });

    process.once("no_shell_integration", async () => {
        await this.say("shell_integration_warning");
    });

    await process;

    // 短い遅延を待機して、すべてのメッセージがウェブビューに送信されるようにする
    await delay(50);

    const { terminalOutputLineLimit } = (await this.providerRef.deref()?.getState()) ?? {};
    const output = truncateOutput(lines.join("\n"), terminalOutputLineLimit);
    const result = output.trim();

    if (userFeedback) {
        await this.say("user_feedback", userFeedback.text, userFeedback.images);
        return [
            true,
            formatResponse.toolResult(
                `Command is still running in the user's terminal.${
                    result.length > 0 ? `\nHere's the output so far:\n${result}` : ""
                }\n\nThe user provided the following feedback:\n<feedback>\n${userFeedback.text}\n</feedback>`,
                userFeedback.images,
            ),
        ];
    }

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
}
```

## 7. エラー処理

ツールの実行中にエラーが発生した場合、適切に処理されます。

### 7.1 パラメータ検証エラー

ツールの実行前に、必須パラメータが存在するかが検証されます。パラメータが不足している場合、エラーが返されます。

```typescript
// Cline.ts
if (!relPath) {
    this.consecutiveMistakeCount++;
    pushToolResult(await this.sayAndCreateMissingParamError("read_file", "path"));
    break;
}

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

### 7.2 実行エラー

ツールの実行中にエラーが発生した場合、エラーメッセージがユーザーに表示され、AIモデルに返されます。

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

### 7.3 連続エラーの処理

AIモデルが連続してエラーを発生させる場合、ユーザーにフィードバックを求めることができます。

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

## 8. まとめ

Roo Codeのツール実行フローは、AIモデルがタスクを完了するために使用するツールの登録、実行、結果の処理のプロセスを表します。このフローは、以下の主要なコンポーネントで構成されています：

1. **ツールの定義**: 各ツールの機能と使用方法が定義されます
2. **ツールの選択**: AIモデルがタスクに最適なツールを選択します
3. **ツールの実行**: 選択されたツールが実行され、結果が生成されます
4. **結果の処理**: ツールの実行結果がAIモデルに返され、次のステップが決定されます

このアーキテクチャにより、Roo Codeは柔軟で拡張可能なツール実行フローを実現しています。AIモデルは、様々なツールを使用してタスクを完了することができ、ユーザーはツールの使用を承認または拒否することができます。