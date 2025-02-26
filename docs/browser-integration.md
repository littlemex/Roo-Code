# ブラウザ統合の詳細分析

このドキュメントでは、Roo Code拡張機能のブラウザ統合機能について詳細に説明します。ブラウザ統合は、Puppeteerを使用してブラウザを制御し、Webコンテンツの取得や操作を可能にする重要なコンポーネントです。

## 概要

ブラウザ統合機能は、`src/services/browser/`ディレクトリに実装されています。この機能により、Roo Codeは以下のことが可能になります：

1. ヘッドレスChromiumブラウザを起動・制御する
2. Webページにアクセスし、スクリーンショットを取得する
3. マウスクリック、キーボード入力、スクロールなどのユーザーアクションをシミュレートする
4. コンソールログを収集する
5. URLからコンテンツを取得し、Markdownに変換する

## アーキテクチャ

ブラウザ統合機能は、以下の2つの主要なクラスで構成されています：

1. **BrowserSession**: ブラウザセッションを管理し、ブラウザアクションを実行するクラス
2. **UrlContentFetcher**: URLからコンテンツを取得し、Markdownに変換するクラス

これらのクラスの役割と関係について詳しく見ていきましょう。

## BrowserSession

`BrowserSession`クラスは、Puppeteerを使用してブラウザセッションを管理し、ブラウザアクションを実行します。

### 主要メソッド

```typescript
async launchBrowser(): Promise<void>
```
Chromiumブラウザを起動します。このメソッドは、Chromiumが存在するかどうかを確認し、存在しない場合はダウンロードします。

```typescript
async closeBrowser(): Promise<BrowserActionResult>
```
ブラウザを閉じます。

```typescript
async doAction(action: (page: Page) => Promise<void>): Promise<BrowserActionResult>
```
指定されたアクションをブラウザページで実行します。このメソッドは、アクションの実行後にスクリーンショットとコンソールログを含む結果を返します。

```typescript
async navigateToUrl(url: string): Promise<BrowserActionResult>
```
指定されたURLにナビゲートします。

```typescript
async click(coordinate: string): Promise<BrowserActionResult>
```
指定された座標でクリックします。

```typescript
async type(text: string): Promise<BrowserActionResult>
```
テキストを入力します。

```typescript
async scrollDown(): Promise<BrowserActionResult>
```
ページを下にスクロールします。

```typescript
async scrollUp(): Promise<BrowserActionResult>
```
ページを上にスクロールします。

### Chromiumの管理

`BrowserSession`は、Chromiumブラウザの管理を担当します。Chromiumが存在しない場合は、`puppeteer-chromium-resolver`（PCR）を使用してダウンロードします。

```typescript
private async ensureChromiumExists(): Promise<PCRStats>
```
このメソッドは、Chromiumが存在するかどうかを確認し、存在しない場合はダウンロードします。

### ブラウザアクションの実行

`BrowserSession`は、`doAction`メソッドを使用してブラウザアクションを実行します。このメソッドは、以下の手順で動作します：

1. コンソールログとエラーのリスナーを設定
2. 指定されたアクションを実行
3. コンソールの非アクティブ状態を待機
4. スクリーンショットを取得
5. リスナーを削除
6. スクリーンショット、ログ、現在のURL、マウス位置を含む結果を返す

```typescript
async doAction(action: (page: Page) => Promise<void>): Promise<BrowserActionResult> {
  // リスナーの設定
  const logs: string[] = []
  let lastLogTs = Date.now()
  const consoleListener = (msg: any) => { /* ... */ }
  const errorListener = (err: Error) => { /* ... */ }
  this.page.on("console", consoleListener)
  this.page.on("pageerror", errorListener)

  try {
    // アクションの実行
    await action(this.page)
  } catch (err) {
    // エラー処理
  }

  // コンソールの非アクティブ状態を待機
  await pWaitFor(() => Date.now() - lastLogTs >= 500, {
    timeout: 3_000,
    interval: 100,
  }).catch(() => {})

  // スクリーンショットの取得
  let screenshotBase64 = await this.page.screenshot({ /* ... */ })
  let screenshot = `data:image/webp;base64,${screenshotBase64}`

  // リスナーの削除
  this.page.off("console", consoleListener)
  this.page.off("pageerror", errorListener)

  // 結果の返却
  return {
    screenshot,
    logs: logs.join("\n"),
    currentUrl: this.page.url(),
    currentMousePosition: this.currentMousePosition,
  }
}
```

### ページの安定性の確認

`BrowserSession`は、ページが完全に読み込まれたことを確認するために、`waitTillHTMLStable`メソッドを使用します。このメソッドは、HTMLのサイズが一定期間変化しなくなるまで待機します。

```typescript
private async waitTillHTMLStable(page: Page, timeout = 5_000) {
  const checkDurationMsecs = 500
  const maxChecks = timeout / checkDurationMsecs
  let lastHTMLSize = 0
  let checkCounts = 1
  let countStableSizeIterations = 0
  const minStableSizeIterations = 3

  while (checkCounts++ <= maxChecks) {
    let html = await page.content()
    let currentHTMLSize = html.length

    if (lastHTMLSize !== 0 && currentHTMLSize === lastHTMLSize) {
      countStableSizeIterations++
    } else {
      countStableSizeIterations = 0
    }

    if (countStableSizeIterations >= minStableSizeIterations) {
      break
    }

    lastHTMLSize = currentHTMLSize
    await delay(checkDurationMsecs)
  }
}
```

## UrlContentFetcher

`UrlContentFetcher`クラスは、URLからコンテンツを取得し、Markdownに変換します。

### 主要メソッド

```typescript
async launchBrowser(): Promise<void>
```
Chromiumブラウザを起動します。

```typescript
async closeBrowser(): Promise<void>
```
ブラウザを閉じます。

```typescript
async urlToMarkdown(url: string): Promise<string>
```
指定されたURLのコンテンツを取得し、Markdownに変換します。

### Markdownへの変換

`UrlContentFetcher`は、cheerioとturndownを使用してHTMLをMarkdownに変換します。

```typescript
async urlToMarkdown(url: string): Promise<string> {
  // ページにアクセス
  await this.page.goto(url, { timeout: 10_000, waitUntil: ["domcontentloaded", "networkidle2"] })
  const content = await this.page.content()

  // cheerioを使用してHTMLをパース・クリーンアップ
  const $ = cheerio.load(content)
  $("script, style, nav, footer, header").remove()

  // turndownを使用してHTMLをMarkdownに変換
  const turndownService = new TurndownService()
  const markdown = turndownService.turndown($.html())

  return markdown
}
```

## ブラウザセッションとUrlContentFetcherの関係

`BrowserSession`と`UrlContentFetcher`は、どちらもPuppeteerを使用してChromiumブラウザを制御しますが、目的が異なります：

- `BrowserSession`は、ブラウザアクション（ナビゲーション、クリック、タイプなど）を実行し、スクリーンショットとコンソールログを取得することに焦点を当てています。
- `UrlContentFetcher`は、URLからコンテンツを取得し、Markdownに変換することに焦点を当てています。

両方のクラスは、`ensureChromiumExists`メソッドを使用してChromiumの存在を確認し、必要に応じてダウンロードします。

## ブラウザ操作のフロー

### ブラウザセッションの起動フロー

1. `BrowserSession.launchBrowser`メソッドが呼び出される
2. `ensureChromiumExists`メソッドがChromiumの存在を確認
3. Chromiumが存在しない場合は、ダウンロード
4. Puppeteerを使用してブラウザを起動
5. 新しいページを開く

### ブラウザアクションの実行フロー

1. `BrowserSession`の各アクションメソッド（`navigateToUrl`、`click`など）が呼び出される
2. アクションメソッドは、`doAction`メソッドを使用して実際のアクションを実行
3. `doAction`メソッドは、コンソールログとエラーのリスナーを設定
4. 指定されたアクションが実行される
5. コンソールの非アクティブ状態を待機
6. スクリーンショットが取得される
7. リスナーが削除される
8. 結果（スクリーンショット、ログ、現在のURL、マウス位置）が返される

### URLからMarkdownへの変換フロー

1. `UrlContentFetcher.urlToMarkdown`メソッドが呼び出される
2. 指定されたURLにアクセス
3. ページのコンテンツを取得
4. cheerioを使用してHTMLをパース・クリーンアップ
5. turndownを使用してHTMLをMarkdownに変換
6. Markdownが返される

## エラー処理

ブラウザ統合機能には、堅牢なエラー処理メカニズムが組み込まれています：

1. **タイムアウト処理**: ページの読み込みやナビゲーションにタイムアウトを設定し、無限に待機することを防止しています。
2. **エラーキャッチ**: アクションの実行中に発生したエラーをキャッチし、ログに記録します。
3. **フォールバックメカニズム**: WebPスクリーンショットが失敗した場合は、PNGにフォールバックします。

```typescript
// WebPスクリーンショットが失敗した場合のフォールバック
if (!screenshotBase64) {
  console.log("webp screenshot failed, trying png")
  screenshotBase64 = await this.page.screenshot({
    ...options,
    type: "png",
  })
  screenshot = `data:image/png;base64,${screenshotBase64}`
}
```

## 実装の詳細

### ビューポートサイズの設定

`BrowserSession`は、VSCodeの設定からビューポートサイズを取得します。デフォルトのサイズは900x600ピクセルです。

```typescript
defaultViewport: (() => {
  const size = (this.context.globalState.get("browserViewportSize") as string | undefined) || "900x600"
  const [width, height] = size.split("x").map(Number)
  return { width, height }
})(),
```

### ユーザーエージェントの設定

`BrowserSession`と`UrlContentFetcher`は、ブラウザのユーザーエージェントを設定して、一般的なブラウザとして認識されるようにします。

```typescript
args: [
  "--user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/128.0.0.0 Safari/537.36",
],
```

### ネットワークアクティビティの監視

`BrowserSession.click`メソッドは、クリックがネットワークアクティビティをトリガーしたかどうかを監視し、トリガーした場合はナビゲーションの完了を待機します。

```typescript
// ネットワークリクエストの監視を設定
let hasNetworkActivity = false
const requestListener = () => {
  hasNetworkActivity = true
}
page.on("request", requestListener)

// クリックを実行
await page.mouse.click(x, y)
this.currentMousePosition = coordinate

// 小さな遅延を入れてネットワークアクティビティをチェック
await delay(100)

if (hasNetworkActivity) {
  // ネットワークアクティビティが検出された場合、ナビゲーション/読み込みを待機
  await page
    .waitForNavigation({
      waitUntil: ["domcontentloaded", "networkidle2"],
      timeout: 7000,
    })
    .catch(() => {})
  await this.waitTillHTMLStable(page)
}
```

## まとめ

ブラウザ統合機能は、Roo Code拡張機能の重要なコンポーネントであり、Puppeteerを使用してブラウザを制御し、Webコンテンツの取得や操作を可能にします。`BrowserSession`と`UrlContentFetcher`の2つの主要なクラスが、それぞれ異なる目的でブラウザを制御します。

`BrowserSession`は、ブラウザアクション（ナビゲーション、クリック、タイプなど）を実行し、スクリーンショットとコンソールログを取得することに焦点を当てています。一方、`UrlContentFetcher`は、URLからコンテンツを取得し、Markdownに変換することに焦点を当てています。

両方のクラスは、Chromiumの存在を確認し、必要に応じてダウンロードする機能を共有しています。また、堅牢なエラー処理メカニズムが組み込まれており、タイムアウト処理、エラーキャッチ、フォールバックメカニズムなどが実装されています。

ブラウザ統合機能は、Roo Codeの他の機能（特にAIによるコード生成や編集）と連携して、ユーザーがWebコンテンツを操作する際の支援を提供します。