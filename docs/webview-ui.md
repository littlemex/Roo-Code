# Webview UI の実装

このドキュメントでは、Roo Code の Webview UI の実装について説明します。

## 概要

Roo Code の UI は React を使用して実装されており、VSCode の Webview API を通じて表示されます。UI のソースコードは `webview-ui/` ディレクトリに格納されています。

## プロジェクト構造

```
webview-ui/
├── .storybook/       # Storybook 設定
├── public/           # 静的ファイル
└── src/              # ソースコード
    ├── __mocks__/    # テスト用モック
    ├── __tests__/    # テスト
    ├── components/   # UI コンポーネント
    ├── context/      # React コンテキスト
    ├── lib/          # ライブラリ
    ├── stories/      # Storybook ストーリー
    ├── utils/        # ユーティリティ関数
    ├── App.tsx       # メインアプリケーション
    └── index.tsx     # エントリーポイント
```

## エントリーポイント

`src/index.tsx` はアプリケーションのエントリーポイントです。React アプリケーションを初期化し、DOM にレンダリングします。

```tsx
import React from "react"
import ReactDOM from "react-dom/client"
import "./index.css"
import "./preflight.css"
import App from "./App"

const root = ReactDOM.createRoot(document.getElementById("root") as HTMLElement)
root.render(
    <React.StrictMode>
        <App />
    </React.StrictMode>,
)
```

## メインアプリケーション

`src/App.tsx` はメインのアプリケーションコンポーネントです。VSCode との通信を設定し、UI のレイアウトを定義します。

```tsx
import { useEffect, useState } from "react"
import { vscode } from "./utils/vscode"
import { ChatView } from "./components/ChatView"
import { StateProvider } from "./context/StateContext"
import { ThemeProvider } from "./context/ThemeContext"

function App() {
    const [initialized, setInitialized] = useState(false)

    useEffect(() => {
        // VSCode との通信を設定
        const handleMessage = (event: MessageEvent) => {
            const message = event.data
            // メッセージを処理
        }

        window.addEventListener("message", handleMessage)

        // 初期状態を要求
        vscode.postMessage({ type: "getState" })

        return () => {
            window.removeEventListener("message", handleMessage)
        }
    }, [])

    return (
        <ThemeProvider>
            <StateProvider>
                <div className="app">
                    <ChatView />
                </div>
            </StateProvider>
        </ThemeProvider>
    )
}

export default App
```

## コンテキスト

### StateContext

`src/context/StateContext.tsx` では、アプリケーションの状態を管理する React コンテキストを定義しています。

```tsx
import { createContext, useContext, useState } from "react"
import { ClineMessage, State } from "../types"

interface StateContextType {
    state: State
    setState: (state: State) => void
    updateMessages: (messages: ClineMessage[]) => void
    // その他の状態更新関数
}

const StateContext = createContext<StateContextType | undefined>(undefined)

export function StateProvider({ children }: { children: React.ReactNode }) {
    const [state, setState] = useState<State>({
        messages: [],
        mode: "code",
        // その他の初期状態
    })

    // 状態更新関数
    const updateMessages = (messages: ClineMessage[]) => {
        setState((prevState) => ({ ...prevState, messages }))
    }

    return (
        <StateContext.Provider
            value={{
                state,
                setState,
                updateMessages,
                // その他の状態更新関数
            }}
        >
            {children}
        </StateContext.Provider>
    )
}

export function useStateContext() {
    const context = useContext(StateContext)
    if (context === undefined) {
        throw new Error("useStateContext must be used within a StateProvider")
    }
    return context
}
```

### ThemeContext

`src/context/ThemeContext.tsx` では、テーマを管理する React コンテキストを定義しています。

```tsx
import { createContext, useContext, useEffect, useState } from "react"

interface ThemeContextType {
    theme: string
    setTheme: (theme: string) => void
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined)

export function ThemeProvider({ children }: { children: React.ReactNode }) {
    const [theme, setTheme] = useState("dark")

    useEffect(() => {
        // VSCode のテーマを取得
        const handleMessage = (event: MessageEvent) => {
            const message = event.data
            if (message.type === "theme") {
                setTheme(message.theme)
            }
        }

        window.addEventListener("message", handleMessage)
        return () => {
            window.removeEventListener("message", handleMessage)
        }
    }, [])

    return (
        <ThemeContext.Provider value={{ theme, setTheme }}>
            {children}
        </ThemeContext.Provider>
    )
}

export function useThemeContext() {
    const context = useContext(ThemeContext)
    if (context === undefined) {
        throw new Error("useThemeContext must be used within a ThemeProvider")
    }
    return context
}
```

## コンポーネント

### ChatView

`src/components/ChatView.tsx` は、チャットビューを表示するコンポーネントです。

```tsx
import { useEffect, useRef } from "react"
import { useStateContext } from "../context/StateContext"
import { ChatRow } from "./ChatRow"
import { InputBox } from "./InputBox"

export function ChatView() {
    const { state } = useStateContext()
    const messagesEndRef = useRef<HTMLDivElement>(null)

    useEffect(() => {
        // 新しいメッセージが追加されたときに自動スクロール
        messagesEndRef.current?.scrollIntoView({ behavior: "smooth" })
    }, [state.messages])

    return (
        <div className="chat-view">
            <div className="messages">
                {state.messages.map((message) => (
                    <ChatRow key={message.ts} message={message} />
                ))}
                <div ref={messagesEndRef} />
            </div>
            <InputBox />
        </div>
    )
}
```

### ChatRow

`src/components/ChatRow.tsx` は、チャットの各行を表示するコンポーネントです。

```tsx
import { ClineMessage } from "../types"
import { AskRow } from "./AskRow"
import { SayRow } from "./SayRow"

interface ChatRowProps {
    message: ClineMessage
}

export function ChatRow({ message }: ChatRowProps) {
    if (message.type === "ask") {
        return <AskRow message={message} />
    } else if (message.type === "say") {
        return <SayRow message={message} />
    }
    return null
}
```

### AskRow

`src/components/AskRow.tsx` は、ユーザーのメッセージを表示するコンポーネントです。

```tsx
import { ClineAskMessage } from "../types"

interface AskRowProps {
    message: ClineAskMessage
}

export function AskRow({ message }: AskRowProps) {
    return (
        <div className="ask-row">
            <div className="avatar">👤</div>
            <div className="content">
                <div className="text">{message.text}</div>
                {message.images && message.images.length > 0 && (
                    <div className="images">
                        {message.images.map((image, index) => (
                            <img key={index} src={image} alt="User uploaded" />
                        ))}
                    </div>
                )}
            </div>
        </div>
    )
}
```

### SayRow

`src/components/SayRow.tsx` は、アシスタントのメッセージを表示するコンポーネントです。

```tsx
import { ClineSayMessage } from "../types"
import { Markdown } from "./Markdown"

interface SayRowProps {
    message: ClineSayMessage
}

export function SayRow({ message }: SayRowProps) {
    return (
        <div className="say-row">
            <div className="avatar">🤖</div>
            <div className="content">
                {message.text && <Markdown text={message.text} />}
                {message.images && message.images.length > 0 && (
                    <div className="images">
                        {message.images.map((image, index) => (
                            <img key={index} src={image} alt="Assistant response" />
                        ))}
                    </div>
                )}
            </div>
        </div>
    )
}
```

### InputBox

`src/components/InputBox.tsx` は、ユーザー入力を受け付けるコンポーネントです。

```tsx
import { useState } from "react"
import { useStateContext } from "../context/StateContext"
import { vscode } from "../utils/vscode"

export function InputBox() {
    const { state } = useStateContext()
    const [input, setInput] = useState("")

    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault()
        if (!input.trim()) return

        // VSCode に入力を送信
        vscode.postMessage({
            type: "askResponse",
            response: "messageResponse",
            text: input,
        })

        setInput("")
    }

    return (
        <form className="input-box" onSubmit={handleSubmit}>
            <textarea
                value={input}
                onChange={(e) => setInput(e.target.value)}
                placeholder="メッセージを入力..."
                disabled={state.inputDisabled}
            />
            <button type="submit" disabled={!input.trim() || state.inputDisabled}>
                送信
            </button>
        </form>
    )
}
```

### Markdown

`src/components/Markdown.tsx` は、マークダウンテキストをレンダリングするコンポーネントです。

```tsx
import ReactMarkdown from "react-markdown"
import { Prism as SyntaxHighlighter } from "react-syntax-highlighter"
import { vscDarkPlus } from "react-syntax-highlighter/dist/esm/styles/prism"

interface MarkdownProps {
    text: string
}

export function Markdown({ text }: MarkdownProps) {
    return (
        <ReactMarkdown
            components={{
                code({ node, inline, className, children, ...props }) {
                    const match = /language-(\w+)/.exec(className || "")
                    return !inline && match ? (
                        <SyntaxHighlighter
                            style={vscDarkPlus}
                            language={match[1]}
                            PreTag="div"
                            {...props}
                        >
                            {String(children).replace(/\n$/, "")}
                        </SyntaxHighlighter>
                    ) : (
                        <code className={className} {...props}>
                            {children}
                        </code>
                    )
                },
            }}
        >
            {text}
        </ReactMarkdown>
    )
}
```

## VSCode との通信

`src/utils/vscode.ts` では、VSCode との通信を抽象化するユーティリティを定義しています。

```typescript
// VSCode API へのアクセスを提供
export const vscode = {
    postMessage: (message: any) => {
        // @ts-ignore
        if (window.acquireVsCodeApi) {
            // @ts-ignore
            const vscodeApi = window.acquireVsCodeApi()
            vscodeApi.postMessage(message)
        } else {
            console.log("VSCode API not available", message)
        }
    },
}
```

## スタイリング

Roo Code の UI は CSS を使用してスタイリングされています。主要なスタイルファイルは以下の通りです：

- `src/index.css`: グローバルスタイル
- `src/preflight.css`: リセット CSS

## まとめ

Roo Code の Webview UI は React を使用して実装されており、VSCode の Webview API を通じて表示されます。UI は複数のコンポーネントに分割され、React コンテキストを使用して状態を管理しています。VSCode との通信は、`postMessage` API を使用して行われます。