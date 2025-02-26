# API プロバイダーの実装

このドキュメントでは、Roo Code が使用する様々な AI プロバイダーの実装について説明します。

## 概要

Roo Code は複数の AI プロバイダー（Anthropic, OpenAI, Google など）をサポートしています。これらのプロバイダーは `src/api/providers/` ディレクトリに実装されています。

各プロバイダーは共通のインターフェースを実装しており、Roo Code はこのインターフェースを通じてプロバイダーと通信します。

## API ハンドラー

`src/api/index.ts` では、`ApiHandler` インターフェースと `buildApiHandler` 関数が定義されています。

```typescript
export interface ApiHandler {
    getModel(): Model
    createMessage(systemPrompt: string, history: Anthropic.Messages.MessageParam[]): ApiStream
}

export function buildApiHandler(apiConfiguration: ApiConfiguration): ApiHandler {
    // プロバイダーに応じたハンドラーを作成
}
```

`buildApiHandler` 関数は、設定に応じて適切なプロバイダーのハンドラーを作成します。

## サポートされているプロバイダー

### Anthropic

`src/api/providers/anthropic.ts` に実装されています。

Anthropic の Claude モデルを使用するためのプロバイダーです。

```typescript
export class AnthropicProvider implements SingleCompletionHandler {
    private client: Anthropic
    private model: Model

    constructor(apiKey: string, model: string) {
        this.client = new Anthropic({ apiKey })
        this.model = {
            id: model,
            info: getModelInfo(model),
        }
    }

    getModel(): Model {
        return this.model
    }

    async *createMessage(
        systemPrompt: string,
        history: Anthropic.Messages.MessageParam[],
    ): AsyncGenerator<ApiChunk> {
        // メッセージを作成し、レスポンスをストリーミング
    }
}
```

### OpenAI

`src/api/providers/openai.ts` に実装されています。

OpenAI の GPT モデルを使用するためのプロバイダーです。

```typescript
export class OpenAIProvider implements SingleCompletionHandler {
    private client: OpenAI
    private model: Model

    constructor(apiKey: string, model: string, baseURL?: string) {
        this.client = new OpenAI({ apiKey, baseURL })
        this.model = {
            id: model,
            info: getModelInfo(model),
        }
    }

    getModel(): Model {
        return this.model
    }

    async *createMessage(
        systemPrompt: string,
        history: Anthropic.Messages.MessageParam[],
    ): AsyncGenerator<ApiChunk> {
        // メッセージを作成し、レスポンスをストリーミング
    }
}
```

### Google (Gemini)

`src/api/providers/gemini.ts` に実装されています。

Google の Gemini モデルを使用するためのプロバイダーです。

```typescript
export class GeminiProvider implements SingleCompletionHandler {
    private client: GoogleGenerativeAI
    private model: Model

    constructor(apiKey: string, model: string) {
        this.client = new GoogleGenerativeAI(apiKey)
        this.model = {
            id: model,
            info: getModelInfo(model),
        }
    }

    getModel(): Model {
        return this.model
    }

    async *createMessage(
        systemPrompt: string,
        history: Anthropic.Messages.MessageParam[],
    ): AsyncGenerator<ApiChunk> {
        // メッセージを作成し、レスポンスをストリーミング
    }
}
```

### AWS Bedrock

`src/api/providers/bedrock.ts` に実装されています。

AWS Bedrock を通じて様々なモデルを使用するためのプロバイダーです。

```typescript
export class BedrockProvider implements SingleCompletionHandler {
    private client: BedrockRuntimeClient
    private model: Model

    constructor(credentials: AwsCredentialIdentity, region: string, model: string) {
        this.client = new BedrockRuntimeClient({ credentials, region })
        this.model = {
            id: model,
            info: getModelInfo(model),
        }
    }

    getModel(): Model {
        return this.model
    }

    async *createMessage(
        systemPrompt: string,
        history: Anthropic.Messages.MessageParam[],
    ): AsyncGenerator<ApiChunk> {
        // メッセージを作成し、レスポンスをストリーミング
    }
}
```

### その他のプロバイダー

Roo Code は以下のプロバイダーもサポートしています：

- `src/api/providers/mistral.ts`: Mistral AI
- `src/api/providers/ollama.ts`: Ollama（ローカルモデル）
- `src/api/providers/lmstudio.ts`: LM Studio（ローカルモデル）
- `src/api/providers/openrouter.ts`: OpenRouter（複数のモデルへのアクセスを提供）
- `src/api/providers/vertex.ts`: Google Vertex AI
- `src/api/providers/vscode-lm.ts`: VSCode 言語モデル API
- `src/api/providers/deepseek.ts`: DeepSeek
- `src/api/providers/glama.ts`: Glama
- `src/api/providers/unbound.ts`: Unbound

## フォーマット変換

`src/api/transform/` ディレクトリには、各プロバイダーのレスポンスフォーマットを Roo Code の内部フォーマットに変換するためのコードが含まれています。

### ストリーミング

`src/api/transform/stream.ts` では、`ApiStream` 型と関連する変換関数が定義されています。

```typescript
export type ApiChunk =
    | {
          type: "text"
          text: string
      }
    | {
          type: "reasoning"
          text: string
      }
    | {
          type: "usage"
          inputTokens: number
          outputTokens: number
          cacheWriteTokens?: number
          cacheReadTokens?: number
          totalCost?: number
      }

export type ApiStream = AsyncGenerator<ApiChunk, void, unknown>
```

### モデル固有のフォーマット

各モデルのレスポンスフォーマットを変換するためのファイルが用意されています：

- `src/api/transform/bedrock-converse-format.ts`: AWS Bedrock
- `src/api/transform/gemini-format.ts`: Google Gemini
- `src/api/transform/mistral-format.ts`: Mistral AI
- `src/api/transform/openai-format.ts`: OpenAI
- `src/api/transform/r1-format.ts`: Anthropic R1 フォーマット
- `src/api/transform/simple-format.ts`: シンプルなフォーマット
- `src/api/transform/vscode-lm-format.ts`: VSCode 言語モデル

## モデル情報

`src/shared/api.ts` では、サポートされているモデルの情報が定義されています。

```typescript
export interface ModelInfo {
    contextWindow: number
    inputTokenCostPer1M: number
    outputTokenCostPer1M: number
    supportsImages?: boolean
    supportsComputerUse?: boolean
    maxImageSize?: number
    maxImageCount?: number
}

export const MODEL_INFO: Record<string, ModelInfo> = {
    "claude-3-opus-20240229": {
        contextWindow: 200000,
        inputTokenCostPer1M: 15000,
        outputTokenCostPer1M: 75000,
        supportsImages: true,
        supportsComputerUse: true,
        maxImageSize: 20971520, // 20MB
        maxImageCount: 20,
    },
    "claude-3-sonnet-20240229": {
        contextWindow: 200000,
        inputTokenCostPer1M: 3000,
        outputTokenCostPer1M: 15000,
        supportsImages: true,
        supportsComputerUse: true,
        maxImageSize: 20971520, // 20MB
        maxImageCount: 20,
    },
    // その他のモデル情報
}
```

## API 設定

`src/shared/api.ts` では、API 設定のインターフェースも定義されています。

```typescript
export interface ApiConfiguration {
    provider: string
    model: string
    apiKey?: string
    baseURL?: string
    region?: string
    accessKeyId?: string
    secretAccessKey?: string
    sessionToken?: string
}
```

## API メトリクス

`src/shared/getApiMetrics.ts` では、API 使用量のメトリクスを計算する関数が定義されています。

```typescript
export function getApiMetrics(messages: ClineMessage[]) {
    // API 使用量のメトリクスを計算
}
```

## コスト計算

`src/utils/cost.ts` では、API 使用量のコストを計算する関数が定義されています。

```typescript
export function calculateApiCost(
    modelInfo: ModelInfo,
    inputTokens: number,
    outputTokens: number,
    cacheWriteTokens: number = 0,
    cacheReadTokens: number = 0,
): number {
    // コストを計算
}
```

## まとめ

Roo Code は複数の AI プロバイダーをサポートしており、共通のインターフェースを通じてこれらのプロバイダーと通信します。各プロバイダーは、メッセージの作成とレスポンスのストリーミングを担当します。また、各プロバイダーのレスポンスフォーマットを Roo Code の内部フォーマットに変換するためのコードも用意されています。