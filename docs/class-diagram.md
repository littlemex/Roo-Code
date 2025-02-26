# Roo Code クラス図

このドキュメントでは、Roo Code の主要なクラスとその関係を mermaid 記法で図示します。

## 全体のクラス図

```mermaid
classDiagram
    %% メインクラス
    ClineProvider "1" --> "*" Cline : creates & manages
    Cline --> "1" ApiHandler : uses
    Cline --> "1" TerminalManager : uses
    Cline --> "1" BrowserSession : uses
    Cline --> "1" DiffViewProvider : uses
    Cline --> "0..1" CheckpointService : uses
    Cline --> "1" McpHub : uses
    
    %% API 関連
    ApiHandler <|-- AnthropicProvider : implements
    ApiHandler <|-- OpenAIProvider : implements
    ApiHandler <|-- GeminiProvider : implements
    ApiHandler <|-- BedrockProvider : implements
    ApiHandler <|-- MistralProvider : implements
    ApiHandler <|-- OllamaProvider : implements
    ApiHandler <|-- OpenRouterProvider : implements
    
    %% MCP 関連
    McpHub --> "*" McpServerConnection : manages
    McpServerConnection --> "1" McpServerProcess : uses
    McpServerManager --> "*" McpServerProcess : manages
    
    %% エディタ関連
    DiffViewProvider --> "1" DiffStrategy : uses
    
    %% チェックポイント関連
    CheckpointServiceFactory --> CheckpointService : creates
    CheckpointService <|-- LocalCheckpointService : implements
    CheckpointService <|-- ShadowCheckpointService : implements
    
    %% 拡張機能のアクティベーション
    VSCodeExtension --> "1" ClineProvider : creates
    VSCodeExtension --> "1" CodeActionProvider : registers
    VSCodeExtension --> "1" DiffContentProvider : registers
    
    %% ウェブビュー関連
    ClineProvider --> "1" WebviewPanel : creates
    
    class VSCodeExtension {
        +activate(context: ExtensionContext)
        +deactivate()
    }
    
    class ClineProvider {
        +context: vscode.ExtensionContext
        +outputChannel: vscode.OutputChannel
        +cline: Cline
        +initClineWithTask(task: string)
        +postStateToWebview()
        +handleModeSwitch(mode: string)
        +getMcpHub(): McpHub
    }
    
    class Cline {
        +taskId: string
        +api: ApiHandler
        +terminalManager: TerminalManager
        +browserSession: BrowserSession
        +diffViewProvider: DiffViewProvider
        +startTask(task?: string, images?: string[])
        +recursivelyMakeClineRequests(userContent: UserContent)
        +executeCommandTool(command: string)
        +presentAssistantMessage()
    }
    
    class ApiHandler {
        <<interface>>
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class AnthropicProvider {
        -client: Anthropic
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class OpenAIProvider {
        -client: OpenAI
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class GeminiProvider {
        -client: GoogleGenerativeAI
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class TerminalManager {
        -terminals: Map<number, TerminalInfo>
        +getOrCreateTerminal(cwd: string)
        +runCommand(terminalInfo: TerminalInfo, command: string)
        +getUnretrievedOutput(terminalId: number)
        +disposeAll()
    }
    
    class BrowserSession {
        -browser: Browser
        -page: Page
        +launchBrowser()
        +navigateToUrl(url: string)
        +click(coordinate: string)
        +type(text: string)
        +closeBrowser()
    }
    
    class DiffViewProvider {
        -cwd: string
        -editor: TextEditor
        -originalContent: string
        -editType: "create" | "modify"
        +open(relPath: string)
        +update(content: string, isComplete: boolean)
        +saveChanges()
        +revertChanges()
        +reset()
    }
    
    class DiffStrategy {
        <<interface>>
        +applyDiff(originalContent: string, diffContent: string, startLine: number, endLine: number)
    }
    
    class CheckpointService {
        <<interface>>
        +saveCheckpoint(message: string)
        +restoreCheckpoint(commitHash: string)
        +getDiff(options: DiffOptions)
    }
    
    class LocalCheckpointService {
        -workspaceDir: string
        +saveCheckpoint(message: string)
        +restoreCheckpoint(commitHash: string)
        +getDiff(options: DiffOptions)
    }
    
    class ShadowCheckpointService {
        -workspaceDir: string
        -shadowDir: string
        +saveCheckpoint(message: string)
        +restoreCheckpoint(commitHash: string)
        +getDiff(options: DiffOptions)
    }
    
    class McpHub {
        -servers: Map<string, McpServerConnection>
        -serverConfigs: Record<string, McpServerConfig>
        +initialize(serverConfigs: Record<string, McpServerConfig>)
        +startServer(name: string)
        +stopServer(name: string)
        +callTool(serverName: string, toolName: string, args?: Record<string, unknown>)
        +readResource(serverName: string, uri: string)
    }
    
    class McpServerConnection {
        -client: McpClient
        -tools: Map<string, McpTool>
        -resources: Map<string, McpResource>
        -resourceTemplates: Map<string, McpResourceTemplate>
        +connect()
        +disconnect()
        +callTool(toolName: string, args?: Record<string, unknown>)
        +readResource(uri: string)
    }
    
    class McpServerProcess {
        -process: ChildProcess
        -stdin: Writable
        -stdout: Readable
        -stderr: Readable
        +start()
        +stop()
        +write(data: string | Buffer)
    }
    
    class McpServerManager {
        -static servers: Map<string, McpServerProcess>
        +static startServer(name: string, config: McpServerConfig)
        +static stopServer(name: string)
        +static cleanup(context: vscode.ExtensionContext)
    }
    
    class CodeActionProvider {
        +provideCodeActions(document: TextDocument, range: Range, context: CodeActionContext)
    }
    
    class DiffContentProvider {
        +provideTextDocumentContent(uri: Uri): string
    }
    
    class WebviewPanel {
        -panel: vscode.WebviewPanel
        +postMessage(message: any)
        +dispose()
    }
```

## コアモジュールのクラス図

```mermaid
classDiagram
    Cline --> ApiHandler : uses
    Cline --> TerminalManager : uses
    Cline --> BrowserSession : uses
    Cline --> DiffViewProvider : uses
    Cline --> CheckpointService : uses
    Cline --> McpHub : uses
    
    class Cline {
        +taskId: string
        +api: ApiHandler
        +terminalManager: TerminalManager
        +browserSession: BrowserSession
        +diffViewProvider: DiffViewProvider
        +apiConversationHistory: MessageParam[]
        +clineMessages: ClineMessage[]
        +checkpointsEnabled: boolean
        +diffEnabled: boolean
        +startTask(task?: string, images?: string[])
        +resumeTaskFromHistory()
        +initiateTaskLoop(userContent: UserContent)
        +recursivelyMakeClineRequests(userContent: UserContent)
        +abortTask(isAbandoned: boolean)
        +ask(type: ClineAsk, text?: string, partial?: boolean)
        +say(type: ClineSay, text?: string, images?: string[], partial?: boolean)
        +presentAssistantMessage()
        +executeCommandTool(command: string)
        +attemptApiRequest(previousApiReqIndex: number, retryAttempt: number)
        +checkpointSave(options: { isFirst: boolean })
        +checkpointRestore(options: { ts: number, commitHash: string, mode: string })
        +checkpointDiff(options: { ts: number, commitHash: string, mode: string })
    }
    
    class ApiHandler {
        <<interface>>
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class TerminalManager {
        -terminals: Map<number, TerminalInfo>
        +getOrCreateTerminal(cwd: string)
        +runCommand(terminalInfo: TerminalInfo, command: string)
        +getUnretrievedOutput(terminalId: number)
        +disposeAll()
    }
    
    class BrowserSession {
        -browser: Browser
        -page: Page
        +launchBrowser()
        +navigateToUrl(url: string)
        +click(coordinate: string)
        +type(text: string)
        +closeBrowser()
    }
    
    class DiffViewProvider {
        -cwd: string
        -editor: TextEditor
        -originalContent: string
        -editType: "create" | "modify"
        +open(relPath: string)
        +update(content: string, isComplete: boolean)
        +saveChanges()
        +revertChanges()
        +reset()
    }
    
    class CheckpointService {
        <<interface>>
        +saveCheckpoint(message: string)
        +restoreCheckpoint(commitHash: string)
        +getDiff(options: DiffOptions)
    }
    
    class McpHub {
        -servers: Map<string, McpServerConnection>
        -serverConfigs: Record<string, McpServerConfig>
        +initialize(serverConfigs: Record<string, McpServerConfig>)
        +startServer(name: string)
        +stopServer(name: string)
        +callTool(serverName: string, toolName: string, args?: Record<string, unknown>)
        +readResource(serverName: string, uri: string)
    }
```

## API プロバイダーのクラス図

```mermaid
classDiagram
    ApiHandler <|-- AnthropicProvider : implements
    ApiHandler <|-- OpenAIProvider : implements
    ApiHandler <|-- GeminiProvider : implements
    ApiHandler <|-- BedrockProvider : implements
    ApiHandler <|-- MistralProvider : implements
    ApiHandler <|-- OllamaProvider : implements
    ApiHandler <|-- OpenRouterProvider : implements
    
    class ApiHandler {
        <<interface>>
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class AnthropicProvider {
        -client: Anthropic
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class OpenAIProvider {
        -client: OpenAI
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class GeminiProvider {
        -client: GoogleGenerativeAI
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class BedrockProvider {
        -client: BedrockRuntimeClient
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class MistralProvider {
        -client: MistralClient
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class OllamaProvider {
        -baseURL: string
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
    
    class OpenRouterProvider {
        -client: OpenAI
        -model: Model
        +getModel(): Model
        +createMessage(systemPrompt: string, history: MessageParam[])
    }
```

## MCP 関連のクラス図

```mermaid
classDiagram
    McpHub --> "*" McpServerConnection : manages
    McpServerConnection --> "1" McpServerProcess : uses
    McpServerManager --> "*" McpServerProcess : manages
    
    class McpHub {
        -servers: Map<string, McpServerConnection>
        -serverConfigs: Record<string, McpServerConfig>
        +initialize(serverConfigs: Record<string, McpServerConfig>)
        +startServer(name: string)
        +stopServer(name: string)
        +callTool(serverName: string, toolName: string, args?: Record<string, unknown>)
        +readResource(serverName: string, uri: string)
        +listTools()
        +listResources()
        +listResourceTemplates()
    }
    
    class McpServerConnection {
        -client: McpClient
        -tools: Map<string, McpTool>
        -resources: Map<string, McpResource>
        -resourceTemplates: Map<string, McpResourceTemplate>
        +connect()
        +disconnect()
        +callTool(toolName: string, args?: Record<string, unknown>)
        +readResource(uri: string)
        +listTools()
        +listResources()
        +listResourceTemplates()
    }
    
    class McpServerProcess {
        -process: ChildProcess
        -stdin: Writable
        -stdout: Readable
        -stderr: Readable
        +start()
        +stop()
        +write(data: string | Buffer)
        +onData(callback: (data: Buffer) => void)
        +onError(callback: (data: Buffer) => void)
        +onExit(callback: (code: number | null, signal: NodeJS.Signals | null) => void)
    }
    
    class McpServerManager {
        -static servers: Map<string, McpServerProcess>
        +static startServer(name: string, config: McpServerConfig)
        +static stopServer(name: string)
        +static cleanup(context: vscode.ExtensionContext)
    }
```

## チェックポイント関連のクラス図

```mermaid
classDiagram
    CheckpointServiceFactory --> CheckpointService : creates
    CheckpointService <|-- LocalCheckpointService : implements
    CheckpointService <|-- ShadowCheckpointService : implements
    
    class CheckpointServiceFactory {
        +static create(options: CheckpointServiceOptions): CheckpointService
    }
    
    class CheckpointService {
        <<interface>>
        +saveCheckpoint(message: string)
        +restoreCheckpoint(commitHash: string)
        +getDiff(options: DiffOptions)
    }
    
    class LocalCheckpointService {
        -workspaceDir: string
        -git: SimpleGit
        -baseHash: string
        +saveCheckpoint(message: string)
        +restoreCheckpoint(commitHash: string)
        +getDiff(options: DiffOptions)
    }
    
    class ShadowCheckpointService {
        -workspaceDir: string
        -shadowDir: string
        -git: SimpleGit
        -baseHash: string
        +saveCheckpoint(message: string)
        +restoreCheckpoint(commitHash: string)
        +getDiff(options: DiffOptions)
    }
```

## ウェブビュー関連のクラス図

```mermaid
classDiagram
    App --> StateProvider : uses
    App --> ThemeProvider : uses
    App --> ChatView : contains
    StateProvider --> State : manages
    ChatView --> ChatRow : contains
    ChatRow --> AskRow : contains
    ChatRow --> SayRow : contains
    SayRow --> Markdown : uses
    
    class App {
        -initialized: boolean
        +render()
    }
    
    class StateProvider {
        -state: State
        +setState(state: State)
        +updateMessages(messages: ClineMessage[])
    }
    
    class ThemeProvider {
        -theme: string
        +setTheme(theme: string)
    }
    
    class State {
        +messages: ClineMessage[]
        +mode: string
        +inputDisabled: boolean
    }
    
    class ChatView {
        +render()
    }
    
    class ChatRow {
        +message: ClineMessage
        +render()
    }
    
    class AskRow {
        +message: ClineAskMessage
        +render()
    }
    
    class SayRow {
        +message: ClineSayMessage
        +render()
    }
    
    class Markdown {
        +text: string
        +render()
    }
```

これらのクラス図は、Roo Code の主要なクラスとその関係を示しています。実際のコードベースはさらに複雑ですが、これらの図は全体的な構造を理解するのに役立ちます。