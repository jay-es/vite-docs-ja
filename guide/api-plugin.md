# プラグイン API

Vite プラグインは、Rollup の優れた設計のプラグインインターフェースを Vite 特有のオプションで拡張しています。その結果、Vite プラグインを一度作成すれば、開発とビルドの両方で動作させることができます。

**以下のセクションを読む前に、まず [Rollup のプラグインドキュメント](https://rollupjs.org/guide/en/#plugin-development)を読むことをお勧めします。**

## 規約

プラグインが Vite 特有のフックを使用せず、[Rollup 互換のプラグイン](#rollup-plugin-compatibility)として実装できる場合は、[Rollup プラグインの命名規則](https://rollupjs.org/guide/en/#conventions)を使用することをお勧めします。

- Rollup プラグインは、`rollup-plugin-` のプレフィックスが付いた明確な名前を持つ必要があります。
- package.json に `rollup-plugin` および `vite-plugin` キーワードを含めます。

これにより、プラグインが公開され、純粋な Rollup または WMR ベースのプロジェクトでも使用できるようになります。

Vite 専用プラグインの場合

- Vite プラグインは、`vite-plugin-` のプレフィックスが付いた明確な名前を持つ必要があります。
- package.json に `vite-plugin` キーワードを含めます。
- プラグインのドキュメントに、Vite 専用プラグインになっている理由を詳しく説明するセクションを含める（例えば、Vite 特有のプラグインフックを使用するなど）。

プラグインが特定のフレームワークでしか動作しない場合は、その名前をプレフィックスの一部として含めるべきです。

- Vue プラグインには `vite-plugin-vue-` のプレフィックス
- React プラグインには `vite-plugin-react-` のプレフィックス
- Svelte プラグインには `vite-plugin-svelte-` のプレフィックス

## プラグインの設定

ユーザーはプロジェクトの `devDependencies` にプラグインを追加し、 `plugins` 配列のオプションを使って設定します。

```js
// vite.config.js
import vitePlugin from 'vite-plugin-feature'
import rollupPlugin from 'rollup-plugin-feature'

export default {
  plugins: [vitePlugin(), rollupPlugin()]
}
```

falsy なプラグインは無視されます。これにより、プラグインを簡単に有効化・無効化できます。

`plugins` は複数のプラグインを含むプリセットも単一の要素として受け入れます。これは複数のプラグインを使って実装された複雑な機能（フレームワーク統合など）に便利です。配列は内部的にフラット化されます。

```js
// framework-plugin
import frameworkRefresh from 'vite-plugin-framework-refresh'
import frameworkDevtools from 'vite-plugin-framework-devtools'

export default function framework(config) {
  return [frameworkRefresh(config), frameworkDevTools(config)]
}
```

```js
// vite.config.js
import framework from 'vite-plugin-framework'

export default {
  plugins: [framework()]
}
```

## シンプルな例

:::tip
Vite/Rollup プラグインは、実際のプラグインオブジェクトを返すファクトリー関数として作成するのが一般的です。この関数はユーザーがプラグインの動作をカスタマイズするためのオプションを受け付けます。
:::

### 仮想ファイルのインポート

```js
export default function myPlugin() {
  const virtualFileId = '@my-virtual-file'

  return {
    name: 'my-plugin', // 必須、警告やエラーで表示されます
    resolveId(id) {
      if (id === virtualFileId) {
        return virtualFileId
      }
    },
    load(id) {
      if (id === virtualFileId) {
        return `export const msg = "from virtual file"`
      }
    }
  }
}
```

これにより、JavaScript でファイルをインポートできます:

```js
import { msg } from '@my-virtual-file'

console.log(msg)
```

### カスタムファイルタイプの変換

```js
const fileRegex = /\.(my-file-ext)$/

export default function myPlugin() {
  return {
    name: 'transform-file',

    transform(src, id) {
      if (fileRegex.test(id)) {
        return {
          code: compileFileToJS(src),
          map: null // ソースマップがあれば提供する
        }
      }
    }
  }
}
```

## 共通のフック

開発中、Vite 開発サーバーは、Rollup が行なうのと同じ方法で [Rollup ビルドフック](https://rollupjs.org/guide/en/#build-hooks)を呼び出すプラグインコンテナを作成します。

以下のフックはサーバー起動時に一度だけ呼び出されます:

- [`options`](https://rollupjs.org/guide/en/#options)
- [`buildStart`](https://rollupjs.org/guide/en/#buildstart)

以下のフックはモジュールのリクエストが来るたびに呼び出されます:

- [`resolveId`](https://rollupjs.org/guide/en/#resolveid)
- [`load`](https://rollupjs.org/guide/en/#load)
- [`transform`](https://rollupjs.org/guide/en/#transform)

以下のフックはサーバーが閉じられる時に呼び出されます:

- [`buildEnd`](https://rollupjs.org/guide/en/#buildend)
- [`closeBundle`](https://rollupjs.org/guide/en/#closebundle)

Vite はパフォーマンスを向上させるために完全な AST のパースを避けるので、[`moduleParsed`](https://rollupjs.org/guide/en/#moduleparsed) フックは開発中には**呼び出されない**ことに注意してください。

[出力生成フック](https://rollupjs.org/guide/en/#output-generation-hooks)（`closeBundle` を除く）は開発中には**呼び出されません**。Vite の開発サーバーは `bundle.generate()` を呼び出さず、`rollup.rollup()` だけを呼び出していると考えることができます。

## Vite 特有のフック

Vite プラグインは Vite 特有の目的を果たすフックを提供することもできます。これらのフックは Rollup には無視されます。

### `config`

- **型:** `(config: UserConfig, env: { mode: string, command: string }) => UserConfig | null | void`
- **種類:** `sync`, `sequential`

  Vite の設定を解決される前に変更します。このフックは生のユーザー設定（CLI オプションが設定ファイルにマージされたもの）と使用されている `mode` と `command` を公開する現在の設定環境を受け取ります。既存の設定に深くマージされる部分的な設定オブジェクトを返したり、設定を直接変更できます（デフォルトのマージで目的の結果が得られない場合）。

  **例:**

  ```js
  // 部分的な設定を返す（推奨）
  const partialConfigPlugin = () => ({
    name: 'return-partial',
    config: () => ({
      alias: {
        foo: 'bar'
      }
    })
  })

  // 設定を直接変更する（マージが動作しない場合のみ使用する）
  const mutateConfigPlugin = () => ({
    name: 'mutate-config',
    config(config, { command }) {
      if (command === 'build') {
        config.root = __dirname
      }
    }
  })
  ```

  ::: warning 注意
  ユーザープラグインはこのフックを実行する前に解決されるので、`config` フックの中に他のプラグインを注入しても効果はありません。
  :::

### `configResolved`

- **型:** `(config: ResolvedConfig) => void | Promise<void>`
- **種類:** `async`, `parallel`

  Vite プラグインが解決された後に呼び出されます。このフックを使って、最終的に解決された設定を読み取って保存します。このフックはプラグインがコマンドの実行に基づいて何か別のことをする必要がある場合にも便利です。

  **例:**

  ```js
  const examplePlugin = () => {
    let config

    return {
      name: 'read-config',

      configResolved(resolvedConfig) {
        // 解決された設定を保存
        config = resolvedConfig
      },

      // 保存された設定を他のフックで使用
      transform(code, id) {
        if (config.command === 'serve') {
          // serve: 開発サーバーから呼び出されるプラグイン
        } else {
          // build: Rollup から呼び出されるプラグイン
        }
      }
    }
  }
  ```

### `configureServer`

- **型:** `(server: ViteDevServer) => (() => void) | void | Promise<(() => void) | void>`
- **種類:** `async`, `sequential`
- **参考:** [ViteDevServer](./api-javascript#vitedevserver)

  開発サーバーを設定するためのフック。内部の [connect](https://github.com/senchalabs/connect) アプリにカスタムミドルウェアを追加するのが最も一般的な使用例です:

  ```js
  const myPlugin = () => ({
    name: 'configure-server',
    configureServer(server) {
      server.middlewares.use((req, res, next) => {
        // カスタムハンドルリクエスト...
      })
    }
  })
  ```

  **ポストミドルウェアの注入**

  `configureServer` フックは内部ミドルウェアがインストールされる前に呼び出されるため、カスタムミドルウェアはデフォルトで内部ミドルウェアより先に実行されます。内部ミドルウェアの**後に**ミドルウェアを注入したい場合は `configureServer` から関数を返すと、内部ミドルウェアのインストール後に呼び出されます:

  ```js
  const myPlugin = () => ({
    name: 'configure-server',
    configureServer(server) {
      // 内部ミドルウェアがインストールされた後に呼び出される
      // ポストフックを返す
      return () => {
        server.middlewares.use((req, res, next) => {
          // カスタムハンドルリクエスト...
        })
      }
    }
  })
  ```

  **サーバーアクセスの保存**

  場合によっては、他のプラグインフックが開発サーバーのインスタンスへのアクセスを必要とすることがあります（たとえば、Web ソケットサーバー、ファイルシステムウォッチャー、モジュールグラフへのアクセス）。このフックは他のフックでアクセスするためにサーバーインスタンスを保存するためにも使用できます:

  ```js
  const myPlugin = () => {
    let server
    return {
      name: 'configure-server',
      configureServer(_server) {
        server = _server
      },
      transform(code, id) {
        if (server) {
          // サーバーを使用...
        }
      }
    }
  }
  ```

  `configureServer` は本番ビルドの実行時には呼び出されないため、他のフックはこれがなくても動くようにしておく必要があります。

### `transformIndexHtml`

- **型:** `IndexHtmlTransformHook | { enforce?: 'pre' | 'post' transform: IndexHtmlTransformHook }`
- **種類:** `async`, `sequential`

  `index.html` を変換するための専用フック。このフックは現在の HTML 文字列と変換コンテキストを受け取ります。コンテキストは開発時には [`ViteDevServer`](./api-javascript#vitedevserver) を公開し、ビルド時には Rollup の出力バンドルを公開します。

  このフックは非同期にすることも可能で、次のいずれかを返すことができます:

  - 変換された HTML 文字列
  - 既存の HTML に注入するタグ記述子オブジェクト（`{ tag, attrs, children }`）の配列。各タグは注入箇所を指定できます（デフォルトでは `<head>` の前）
  - 両方を含むオブジェクト `{ html, tags }`

  **基本的な例:**

  ```js
  const htmlPlugin = () => {
    return {
      name: 'html-transform',
      transformIndexHtml(html) {
        return html.replace(
          /<title>(.*?)<\/title>/,
          `<title>Title replaced!</title>`
        )
      }
    }
  }
  ```

  **フックの完全なシグネチャ:**

  ```ts
  type IndexHtmlTransformHook = (
    html: string,
    ctx: {
      path: string
      filename: string
      server?: ViteDevServer
      bundle?: import('rollup').OutputBundle
      chunk?: import('rollup').OutputChunk
    }
  ) =>
    | IndexHtmlTransformResult
    | void
    | Promise<IndexHtmlTransformResult | void>

  type IndexHtmlTransformResult =
    | string
    | HtmlTagDescriptor[]
    | {
        html: string
        tags: HtmlTagDescriptor[]
      }

  interface HtmlTagDescriptor {
    tag: string
    attrs?: Record<string, string | boolean>
    children?: string | HtmlTagDescriptor[]
    /**
     * default: 'head-prepend'
     */
    injectTo?: 'head' | 'body' | 'head-prepend' | 'body-prepend'
  }
  ```

### `handleHotUpdate`

- **型:** `(ctx: HmrContext) => Array<ModuleNode> | void | Promise<Array<ModuleNode> | void>`

  カスタム HMR 更新処理を実行します。このフックは以下のシグネチャのコンテキストオブジェクトを受け取ります:

  ```ts
  interface HmrContext {
    file: string
    timestamp: number
    modules: Array<ModuleNode>
    read: () => string | Promise<string>
    server: ViteDevServer
  }
  ```

  - `modules` は変更されたファイルに影響を受けるモジュールの配列です。単一のファイルが複数の提供モジュールに対応している場合があるため（Vue の SFC など）、配列になっています。

  - `read` はファイルの内容を返す非同期の read 関数です。システムによってはファイル変更コールバックがエディタのファイル更新完了前に発生してしまい、`fs.readFile` が空の内容を返すため、この関数が提供されています。渡される read 関数は、この動作を正規化します。

  このフックは以下を選択できます:

  - 影響を受けるモジュールをフィルターして絞り込むことで、HMR がより正確になります。

  - 空の配列を返し、クライアントにカスタムイベントを送信して、完全なカスタム HMR 処理を実行します:

    ```js
    handleHotUpdate({ server }) {
      server.ws.send({
        type: 'custom',
        event: 'special-update',
        data: {}
      })
      return []
    }
    ```

    クライアントコードは [HMR API](./api-hmr) を使用して対応するハンドラを登録する必要があります（これは同じプラグインの `transform` フックによって注入される可能性があります）:

    ```js
    if (import.meta.hot) {
      import.meta.hot.on('special-update', (data) => {
        // カスタムアップデートの実行
      })
    }
    ```

## Plugin Ordering

A Vite plugin can additionally specify an `enforce` property (similar to webpack loaders) to adjust its application order. The value of `enforce` can be either `"pre"` or `"post"`. The resolved plugins will be in the following order:

- Alias
- User plugins with `enforce: 'pre'`
- Vite core plugins
- User plugins without enforce value
- Vite build plugins
- User plugins with `enforce: 'post'`
- Vite post build plugins (minify, manifest, reporting)

## Conditional Application

By default plugins are invoked for both serve and build. In cases where a plugin needs to be conditionally applied only during serve or build, use the `apply` property to only invoke them during `'build'` or `'serve'`:

```js
function myPlugin() {
  return {
    name: 'build-only',
    apply: 'build' // or 'serve'
  }
}
```

## Rollup Plugin Compatibility

A fair number of Rollup plugins will work directly as a Vite plugin (e.g. `@rollup/plugin-alias` or `@rollup/plugin-json`), but not all of them, since some plugin hooks do not make sense in an unbundled dev server context.

In general, as long as a Rollup plugin fits the following criterias then it should just work as a Vite plugin:

- It doesn't use the [`moduleParsed`](https://rollupjs.org/guide/en/#moduleparsed) hook.
- It doesn't have strong coupling between bundle-phase hooks and output-phase hooks.

If a Rollup plugin only makes sense for the build phase, then it can be specified under `build.rollupOptions.plugins` instead.

You can also augment an existing Rollup plugin with Vite-only properties:

```js
// vite.config.js
import example from 'rollup-plugin-example'

export default {
  plugins: [
    {
      ...example(),
      enforce: 'post',
      apply: 'build'
    }
  ]
}
```

Check out [Vite Rollup Plugins](https://vite-rollup-plugins.patak.dev) for a list of compatible official Rollup plugins with usage instructions.

## Path normalization

Vite normalizes paths while resolving ids to use POSIX separators ( / ) while preserving the volume in Windows. On the other hand, Rollup keeps resolved paths untouched by default, so resolved ids have win32 separators ( \\ ) in Windows. However, Rollup plugins use a [`normalizePath` utility function](https://github.com/rollup/plugins/tree/master/packages/pluginutils#normalizepath) from `@rollup/pluginutils` internally, which converts separators to POSIX before performing comparisons. This means that when these plugins are used in Vite, the `include` and `exclude` config pattern and other similar paths against resolved ids comparisons work correctly.

So, for Vite plugins, when comparing paths against resolved ids it is important to first normalize the paths to use POSIX separators. An equivalent `normalizePath` utility function is exported from the `vite` module.

```js
import { normalizePath } from 'vite'

normalizePath('foo\\bar') // 'foo/bar'
normalizePath('foo/bar') // 'foo/bar'
```
