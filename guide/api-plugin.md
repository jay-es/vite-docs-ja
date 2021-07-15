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

## Vite Specific Hooks

Vite plugins can also provide hooks that serve Vite-specific purposes. These hooks are ignored by Rollup.

### `config`

- **Type:** `(config: UserConfig, env: { mode: string, command: string }) => UserConfig | null | void`
- **Kind:** `sync`, `sequential`

  Modify Vite config before it's resolved. The hook receives the raw user config (CLI options merged with config file) and the current config env which exposes the `mode` and `command` being used. It can return a partial config object that will be deeply merged into existing config, or directly mutate the config (if the default merging cannot achieve the desired result).

  **Example**

  ```js
  // return partial config (recommended)
  const partialConfigPlugin = () => ({
    name: 'return-partial',
    config: () => ({
      alias: {
        foo: 'bar'
      }
    })
  })

  // mutate the config directly (use only when merging doesn't work)
  const mutateConfigPlugin = () => ({
    name: 'mutate-config',
    config(config, { command }) {
      if (command === 'build') {
        config.root = __dirname
      }
    }
  })
  ```

  ::: warning Note
  User plugins are resolved before running this hook so injecting other plugins inside the `config` hook will have no effect.
  :::

### `configResolved`

- **Type:** `(config: ResolvedConfig) => void | Promise<void>`
- **Kind:** `async`, `parallel`

  Called after the Vite config is resolved. Use this hook to read and store the final resolved config. It is also useful when the plugin needs to do something different based the command is being run.

  **Example:**

  ```js
  const examplePlugin = () => {
    let config

    return {
      name: 'read-config',

      configResolved(resolvedConfig) {
        // store the resolved config
        config = resolvedConfig
      },

      // use stored config in other hooks
      transform(code, id) {
        if (config.command === 'serve') {
          // serve: plugin invoked by dev server
        } else {
          // build: plugin invoked by Rollup
        }
      }
    }
  }
  ```

### `configureServer`

- **Type:** `(server: ViteDevServer) => (() => void) | void | Promise<(() => void) | void>`
- **Kind:** `async`, `sequential`
- **See also:** [ViteDevServer](./api-javascript#vitedevserver)

  Hook for configuring the dev server. The most common use case is adding custom middlewares to the internal [connect](https://github.com/senchalabs/connect) app:

  ```js
  const myPlugin = () => ({
    name: 'configure-server',
    configureServer(server) {
      server.middlewares.use((req, res, next) => {
        // custom handle request...
      })
    }
  })
  ```

  **Injecting Post Middleware**

  The `configureServer` hook is called before internal middlewares are installed, so the custom middlewares will run before internal middlewares by default. If you want to inject a middleware **after** internal middlewares, you can return a function from `configureServer`, which will be called after internal middlewares are installed:

  ```js
  const myPlugin = () => ({
    name: 'configure-server',
    configureServer(server) {
      // return a post hook that is called after internal middlewares are
      // installed
      return () => {
        server.middlewares.use((req, res, next) => {
          // custom handle request...
        })
      }
    }
  })
  ```

  **Storing Server Access**

  In some cases, other plugin hooks may need access to the dev server instance (e.g. accessing the web socket server, the file system watcher, or the module graph). This hook can also be used to store the server instance for access in other hooks:

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
          // use server...
        }
      }
    }
  }
  ```

  Note `configureServer` is not called when running the production build so your other hooks need to guard against its absence.

### `transformIndexHtml`

- **Type:** `IndexHtmlTransformHook | { enforce?: 'pre' | 'post' transform: IndexHtmlTransformHook }`
- **Kind:** `async`, `sequential`

  Dedicated hook for transforming `index.html`. The hook receives the current HTML string and a transform context. The context exposes the [`ViteDevServer`](./api-javascript#vitedevserver) instance during dev, and exposes the Rollup output bundle during build.

  The hook can be async and can return one of the following:

  - Transformed HTML string
  - An array of tag descriptor objects (`{ tag, attrs, children }`) to inject to the existing HTML. Each tag can also specify where it should be injected to (default is prepending to `<head>`)
  - An object containing both as `{ html, tags }`

  **Basic Example**

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

  **Full Hook Signature:**

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

- **Type:** `(ctx: HmrContext) => Array<ModuleNode> | void | Promise<Array<ModuleNode> | void>`

  Perform custom HMR update handling. The hook receives a context object with the following signature:

  ```ts
  interface HmrContext {
    file: string
    timestamp: number
    modules: Array<ModuleNode>
    read: () => string | Promise<string>
    server: ViteDevServer
  }
  ```

  - `modules` is an array of modules that are affected by the changed file. It's an array because a single file may map to multiple served modules (e.g. Vue SFCs).

  - `read` is an async read function that returns the content of the file. This is provided because on some systems, the file change callback may fire too fast before the editor finishes updating the file and direct `fs.readFile` will return empty content. The read function passed in normalizes this behavior.

  The hook can choose to:

  - Filter and narrow down the affected module list so that the HMR is more accurate.

  - Return an empty array and perform complete custom HMR handling by sending custom events to the client:

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

    Client code should register corresponding handler using the [HMR API](./api-hmr) (this could be injected by the same plugin's `transform` hook):

    ```js
    if (import.meta.hot) {
      import.meta.hot.on('special-update', (data) => {
        // perform custom update
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
