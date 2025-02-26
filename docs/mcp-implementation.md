# Model Context Protocol (MCP) の実装

このドキュメントでは、Roo Code における Model Context Protocol (MCP) の実装について説明します。

## 概要

Model Context Protocol (MCP) は、AI モデルが外部リソースやツールにアクセスするための標準化されたプロトコルです。Roo Code では、MCP を使用して AI モデルが様々なツールやリソースにアクセスできるようにしています。

MCP の実装は主に `src/services/mcp/` ディレクトリに格納されています。

## 主要なクラスとファイル

### McpHub

`src/services/mcp/McpHub.ts` に定義されています。

McpHub は MCP サーバーとの通信を管理するクラスです。サーバーの起動、停止、ツールの呼び出し、リソースへのアクセスなどを担当します。

```typescript
export class McpHub {
    private servers: Map<string, McpServerConnection> = new Map()
    private serverConfigs: Record<string, McpServerConfig> = {}
    isConnecting: boolean = false

    constructor(private context: vscode.ExtensionContext) {}

    async initialize(serverConfigs: Record<string, McpServerConfig>) {
        // サーバー設定を初期化
    }

    async startServer(name: string) {
        // サーバーを起動
    }

    async stopServer(name: string) {
        // サーバーを停止
    }

    async callTool(serverName: string, toolName: string, args?: Record<string, unknown>) {
        // ツールを呼び出す
    }

    async readResource(serverName: string, uri: string) {
        // リソースを読み取る
    }

    async listTools() {
        // 利用可能なツールの一覧を取得
    }

    async listResources() {
        // 利用可能なリソースの一覧を取得
    }

    async listResourceTemplates() {
        // 利用可能なリソーステンプレートの一覧を取得
    }
}
```

### McpServerManager

`src/services/mcp/McpServerManager.ts` に定義されています。

McpServerManager は MCP サーバーのプロセスを管理するクラスです。サーバーの起動、停止、環境変数の設定などを担当します。

```typescript
export class McpServerManager {
    private static servers: Map<string, McpServerProcess> = new Map()

    static async startServer(name: string, config: McpServerConfig): Promise<McpServerProcess> {
        // サーバーを起動
    }

    static async stopServer(name: string): Promise<void> {
        // サーバーを停止
    }

    static async cleanup(context: vscode.ExtensionContext): Promise<void> {
        // すべてのサーバーを停止
    }
}
```

### McpServerConnection

`src/services/mcp/McpHub.ts` に定義されています。

McpServerConnection は MCP サーバーとの接続を管理するクラスです。サーバーとの通信プロトコルを実装しています。

```typescript
class McpServerConnection {
    private client: McpClient
    private tools: Map<string, McpTool> = new Map()
    private resources: Map<string, McpResource> = new Map()
    private resourceTemplates: Map<string, McpResourceTemplate> = new Map()

    constructor(private serverProcess: McpServerProcess) {
        // クライアントを初期化
    }

    async connect() {
        // サーバーに接続
    }

    async disconnect() {
        // サーバーから切断
    }

    async callTool(toolName: string, args?: Record<string, unknown>) {
        // ツールを呼び出す
    }

    async readResource(uri: string) {
        // リソースを読み取る
    }

    async listTools() {
        // 利用可能なツールの一覧を取得
    }

    async listResources() {
        // 利用可能なリソースの一覧を取得
    }

    async listResourceTemplates() {
        // 利用可能なリソーステンプレートの一覧を取得
    }
}
```

### McpServerProcess

`src/services/mcp/McpServerManager.ts` に定義されています。

McpServerProcess は MCP サーバーのプロセスを表すクラスです。プロセスの起動、停止、標準入出力の管理などを担当します。

```typescript
class McpServerProcess {
    private process: ChildProcess | null = null
    private stdin: Writable | null = null
    private stdout: Readable | null = null
    private stderr: Readable | null = null
    private onDataCallbacks: ((data: Buffer) => void)[] = []
    private onErrorCallbacks: ((data: Buffer) => void)[] = []
    private onExitCallbacks: ((code: number | null, signal: NodeJS.Signals | null) => void)[] = []

    constructor(
        private command: string,
        private args: string[],
        private env: Record<string, string> = {},
    ) {}

    async start() {
        // プロセスを起動
    }

    async stop() {
        // プロセスを停止
    }

    onData(callback: (data: Buffer) => void) {
        // 標準出力のデータを受け取るコールバックを登録
    }

    onError(callback: (data: Buffer) => void) {
        // 標準エラー出力のデータを受け取るコールバックを登録
    }

    onExit(callback: (code: number | null, signal: NodeJS.Signals | null) => void) {
        // プロセスの終了を通知するコールバックを登録
    }

    write(data: string | Buffer) {
        // 標準入力にデータを書き込む
    }
}
```

## MCP の設定

MCP サーバーの設定は、`~/.local/share/code-server/User/globalStorage/rooveterinaryinc.roo-cline/settings/cline_mcp_settings.json` ファイルに保存されます。

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": ["/path/to/weather-server/build/index.js"],
      "env": {
        "OPENWEATHER_API_KEY": "user-provided-api-key"
      }
    }
  }
}
```

## MCP サーバーの実装

MCP サーバーは、MCP SDK を使用して実装されます。サーバーは、ツールとリソースを提供します。

### ツール

ツールは、AI モデルが実行できる関数です。例えば、天気情報を取得するツールや、ファイルを操作するツールなどがあります。

```typescript
server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
        {
            name: "get_forecast",
            description: "Get weather forecast for a city",
            inputSchema: {
                type: "object",
                properties: {
                    city: {
                        type: "string",
                        description: "City name",
                    },
                    days: {
                        type: "number",
                        description: "Number of days (1-5)",
                        minimum: 1,
                        maximum: 5,
                    },
                },
                required: ["city"],
            },
        },
    ],
}))

server.setRequestHandler(CallToolRequestSchema, async (request) => {
    if (request.params.name !== "get_forecast") {
        throw new McpError(
            ErrorCode.MethodNotFound,
            `Unknown tool: ${request.params.name}`
        )
    }

    // ツールの実装
    // ...

    return {
        content: [
            {
                type: "text",
                text: JSON.stringify(result, null, 2),
            },
        ],
    }
})
```

### リソース

リソースは、AI モデルがアクセスできるデータです。例えば、天気情報や、ファイルの内容などがあります。

```typescript
server.setRequestHandler(ListResourcesRequestSchema, async () => ({
    resources: [
        {
            uri: `weather://San Francisco/current`,
            name: `Current weather in San Francisco`,
            mimeType: "application/json",
            description: "Real-time weather data for San Francisco",
        },
    ],
}))

server.setRequestHandler(ListResourceTemplatesRequestSchema, async () => ({
    resourceTemplates: [
        {
            uriTemplate: "weather://{city}/current",
            name: "Current weather for a given city",
            mimeType: "application/json",
            description: "Real-time weather data for a specified city",
        },
    ],
}))

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
    const match = request.params.uri.match(/^weather:\/\/([^/]+)\/current$/)
    if (!match) {
        throw new McpError(
            ErrorCode.InvalidRequest,
            `Invalid URI format: ${request.params.uri}`
        )
    }
    const city = decodeURIComponent(match[1])

    // リソースの実装
    // ...

    return {
        contents: [
            {
                uri: request.params.uri,
                mimeType: "application/json",
                text: JSON.stringify(result, null, 2),
            },
        ],
    }
})
```

## Cline クラスでの MCP の使用

`src/core/Cline.ts` では、MCP を使用して AI モデルがツールやリソースにアクセスできるようにしています。

```typescript
case "use_mcp_tool": {
    const server_name: string | undefined = block.params.server_name
    const tool_name: string | undefined = block.params.tool_name
    const mcp_arguments: string | undefined = block.params.arguments
    try {
        // ...
        const toolResult = await this.providerRef
            .deref()
            ?.getMcpHub()
            ?.callTool(server_name, tool_name, parsedArguments)
        // ...
    } catch (error) {
        // ...
    }
    break
}

case "access_mcp_resource": {
    const server_name: string | undefined = block.params.server_name
    const uri: string | undefined = block.params.uri
    try {
        // ...
        const resourceResult = await this.providerRef
            .deref()
            ?.getMcpHub()
            ?.readResource(server_name, uri)
        // ...
    } catch (error) {
        // ...
    }
    break
}
```

## システムプロンプトでの MCP の使用

`src/core/prompts/system.ts` では、MCP サーバーの情報をシステムプロンプトに含めています。

```typescript
export const SYSTEM_PROMPT = async (
    context: vscode.ExtensionContext,
    cwd: string,
    supportsComputerUse: boolean,
    mcpHub?: McpHub,
    // ...
) => {
    // ...

    // MCP サーバーの情報を取得
    let mcpServersSection = ""
    if (mcpHub) {
        const tools = await mcpHub.listTools()
        const resources = await mcpHub.listResources()
        const resourceTemplates = await mcpHub.listResourceTemplates()

        if (tools.length > 0 || resources.length > 0 || resourceTemplates.length > 0) {
            mcpServersSection = `
# Connected MCP Servers

When a server is connected, you can use the server's tools via the \`use_mcp_tool\` tool, and access the server's resources via the \`access_mcp_resource\` tool.

${tools.length > 0 ? `## Available Tools\n\n${formatTools(tools)}` : ""}
${resources.length > 0 ? `## Available Resources\n\n${formatResources(resources)}` : ""}
${resourceTemplates.length > 0 ? `## Available Resource Templates\n\n${formatResourceTemplates(resourceTemplates)}` : ""}
`
        } else {
            mcpServersSection = `
# Connected MCP Servers

When a server is connected, you can use the server's tools via the \`use_mcp_tool\` tool, and access the server's resources via the \`access_mcp_resource\` tool.

(No MCP servers currently connected)
`
        }
    }

    // ...

    return `
${basePrompt}

${mcpServersSection}

${mcpServerCreationSection}
`
}
```

## まとめ

Roo Code における MCP の実装は、AI モデルが外部リソースやツールにアクセスするための標準化されたプロトコルを提供しています。McpHub クラスが MCP サーバーとの通信を管理し、McpServerManager クラスがサーバーのプロセスを管理しています。MCP サーバーは、ツールとリソースを提供し、AI モデルはこれらを使用して様々なタスクを実行できます。