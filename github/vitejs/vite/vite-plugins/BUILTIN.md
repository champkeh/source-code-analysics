# vite 内置插件

vite 在启动服务器的时候，会自动注入大量内置插件。这篇文章就来分析下这些内置插件都是干嘛的。

> 这些内置插件是通过`resolvePlugins`函数注入的。位于`src/node/plugins/index.ts`文件中

## 插件清单

- vite:pre-alias
- alias
- vite:modulepreload-polyfill
- vite:resolve
- vite:html-inline-proxy
- vite:css
- vite:esbuild
- vite:json
- vite:wasm
- vite:worker
- vite:worker-import-meta-url
- vite:asset
- (normal 用户插件)
- vite:define
- vite:css-post
- (post 用户插件)
- vite:client-inject
- vite:import-analysis

## 前置知识

在分析具体的插件代码之前，我们需要先了解一下插件是如何运作的。

我们在[开发服务器启动流程梳理](../vite-dev-server/README.md)和[vite 相关插件对 serve 及 build 命令的影响](../vite-plugins/README.md)这两篇文章中已经知道了
vite 独有的插件钩子的执行机制，然而 vite 还支持一部分 rollup 插件，所以我们还需要知道 rollup 插件的钩子是如何被执行的。

下面分别分析每个插件都做了什么。

## vite:pre-alias

> 这个插件只会注入到`serve`命令中

该插件代码如下：

```ts
function preAliasPlugin(): Plugin {
  let server: ViteDevServer
  return {
    name: 'vite:pre-alias',
    configureServer(_server) {
      server = _server
    },
    resolveId(id, importer, options) {
      if (!options?.ssr && bareImportRE.test(id)) {
        return tryOptimizedResolve(id, server, importer)
      }
    }
  }
}
```

可以看到，这个插件通过`configureServer`来保存 server 实例，通过`resolveId`尝试加载优化版本的依赖。

## alias

这个插件是[@rollup/plugin-alias](https://www.npmjs.com/package/@rollup/plugin-alias) ，用来定义别名。

## vite:modulepreload-polyfill

插件代码如下：

```ts
function modulePreloadPolyfillPlugin(config: ResolvedConfig): Plugin {
  const skip = config.build.ssr
  let polyfillString: string | undefined

  return {
    name: 'vite:modulepreload-polyfill',
    resolveId(id) {
      if (id === modulePreloadPolyfillId) {
        return id
      }
    },
    load(id) {
      if (id === modulePreloadPolyfillId) {
        if (skip) {
          return ''
        }
        if (!polyfillString) {
          polyfillString =
            `const p = ${polyfill.toString()};` + `${isModernFlag}&&p();`
        }
        return polyfillString
      }
    }
  }
}
```

这个插件会在`config.build.polyfillModulePreload`为`true`时开启，作用是提供`vite/modulepreload-polyfill`虚拟模块来支持现代模块的预加载功能。
如果构建目标不是现代浏览器，则会关闭这个插件。

## vite:resolve

插件代码如下：

```ts
function resolvePlugin(baseOptions: InternalResolveOptions): Plugin {
  let server: ViteDevServer | undefined

  return {
    name: 'vite:resolve',

    configureServer(_server) {
      server = _server
    },
    resolveId(id, importer, resolveOpts) {
      const ssr = resolveOpts?.ssr === true
      if (id.startsWith(browserExternalId)) {
        return id
      }

      // fast path for commonjs proxy modules
      if (/\?commonjs/.test(id) || id === 'commonjsHelpers.js') {
        return
      }

      // explicit fs paths that starts with /@fs/*
      if (asSrc && id.startsWith(FS_PREFIX)) {
        const fsPath = fsPathFromId(id)
        res = tryFsResolve(fsPath, options)
        isDebug && debug(`[@fs] ${colors.cyan(id)} -> ${colors.dim(res)}`)
        // always return here even if res doesn't exist since /@fs/ is explicit
        // if the file doesn't exist it should be a 404
        return res || fsPath
      }

      // URL
      // /foo -> /fs-root/foo
      if (asSrc && id.startsWith('/')) {
        const fsPath = path.resolve(root, id.slice(1))
        if ((res = tryFsResolve(fsPath, options))) {
          isDebug && debug(`[url] ${colors.cyan(id)} -> ${colors.dim(res)}`)
          return res
        }
      }

      // relative
      if (id.startsWith('.') || (preferRelative && /^\w/.test(id))) {
      }

      // absolute fs paths
      if (path.isAbsolute(id) && (res = tryFsResolve(id, options))) {
        isDebug && debug(`[fs] ${colors.cyan(id)} -> ${colors.dim(res)}`)
        return res
      }

      // external
      if (isExternalUrl(id)) {
        return {
          id,
          external: true
        }
      }

      // data uri: pass through (this only happens during build and will be
      // handled by dedicated plugin)
      if (isDataUrl(id)) {
        return null
      }

      // bare package imports, perform node resolve
      if (bareImportRE.test(id)) {
      }

      isDebug && debug(`[fallthrough] ${colors.dim(id)}`)
    },

    load(id) {
      if (id.startsWith(browserExternalId)) {
        return isProduction
          ? `export default {}`
          : `export default new Proxy({}, {
  get() {
    throw new Error('Module "${id.slice(
      browserExternalId.length + 1
    )}" has been externalized for browser compatibility and cannot be accessed in client code.')
  }
})`
      }
    }
  }
}
```

这个插件通过`resolveId`钩子处理一系列模块标识符。

## vite:html-inline-proxy

插件代码如下：

```ts
function htmlInlineProxyPlugin(config: ResolvedConfig): Plugin {
  htmlProxyMap.set(config, new Map())
  return {
    name: 'vite:html-inline-proxy',

    resolveId(id) {
      if (htmlProxyRE.test(id)) {
        return id
      }
    },

    load(id) {
      const proxyMatch = id.match(htmlProxyRE)
      if (proxyMatch) {
        const index = Number(proxyMatch[1])
        const file = cleanUrl(id)
        const url = file.replace(normalizePath(config.root), '')
        const result = htmlProxyMap.get(config)!.get(url)![index]
        if (typeof result === 'string') {
          return result
        } else {
          throw new Error(`No matching HTML proxy module found from ${id}`)
        }
      }
    }
  }
}
```

这个插件用于加载类如`/xyz?html-proxy&index=1.css`这样的引用。内部有一套缓存机制。

## vite:css

插件代码如下：

```ts
function cssPlugin(config: ResolvedConfig): Plugin {
  let server: ViteDevServer
  let moduleCache: Map<string, Record<string, string>>

  const resolveUrl = config.createResolver({
    preferRelative: true,
    tryIndex: false,
    extensions: []
  })
  const atImportResolvers = createCSSResolvers(config)

  return {
    name: 'vite:css',

    configureServer(_server) {
      server = _server
    },
    buildStart() {
      moduleCache = new Map<string, Record<string, string>>()
      cssModulesCache.set(config, moduleCache)

      removedPureCssFilesCache.set(config, new Map<string, RenderedChunk>())
    },
    async transform(raw, id, options) {
      // ...
    }
  }
}
```

这个插件主要用于处理 css 内容(通过`transform`钩子)，内部会创建两个`resolver`:

- resolveUrl
- atImportResolvers

## vite:esbuild

插件代码如下：

```ts
function esbuildPlugin(options: ESBuildOptions = {}): Plugin {
  const filter = createFilter(
    options.include || /\.(tsx?|jsx)$/,
    options.exclude || /\.js$/
  )

  return {
    name: 'vite:esbuild',
    configureServer(_server) {
      server = _server
      server.watcher
        .on('add', reloadOnTsconfigChange)
        .on('change', reloadOnTsconfigChange)
        .on('unlink', reloadOnTsconfigChange)
    },
    async transform(code, id) {
      if (filter(id) || filter(cleanUrl(id))) {
        const result = await transformWithEsbuild(code, id, options)
        return {
          code: result.code,
          map: result.map
        }
      }
    }
  }
}
```

这个插件通过 watcher 实现了修改`tsconfig.json`文件或任何被缓存在`tsconfigCache`缓存中的 json 文件时自动刷新浏览器。

同时通过`transform`钩子用`esbuild`去转换 jsx/tsx/ts 文件

## vite:json

插件代码如下：

```ts
function jsonPlugin(options: JsonOptions = {}, isBuild: boolean): Plugin {
  return {
    name: 'vite:json',
    transform(json, id) {
      if (!jsonExtRE.test(id)) return null
    }
  }
}
```

这个插件用于处理 json 内容

## vite:wasm

插件代码如下：

```ts
function wasmPlugin(config: ResolvedConfig): Plugin {
  return {
    name: 'vite:wasm',

    resolveId(id) {
      if (id === wasmHelperId) {
        return id
      }
    },
    async load(id) {
      if (id === wasmHelperId) {
        return `export default ${wasmHelperCode}`
      }

      if (!id.endsWith('.wasm')) {
        return
      }

      const url = await fileToUrl(id, config, this)

      return `
import initWasm from "${wasmHelperId}"
export default opts => initWasm(opts, ${JSON.stringify(url)})
`
    }
  }
}
```

通过`resolveId/load`加载 wasm 文件

## vite:worker

插件代码如下：

```ts
function webWorkerPlugin(config: ResolvedConfig): Plugin {
  return {
    name: 'vite:worker',

    load(id) {
      if (isBuild) {
        const parsedQuery = parseRequest(id)
        if (
          parsedQuery &&
          (parsedQuery.worker ?? parsedQuery.sharedworker) != null
        ) {
          return ''
        }
      }
    },
    async transform(_, id) {}
  }
}
```

处理 worker

## vite:worker-import-meta-url

插件代码如下：

```ts
function workerImportMetaUrlPlugin(config: ResolvedConfig): Plugin {
  return {
    name: 'vite:worker-import-meta-url',

    async transform(code, id, options) {}
  }
}
```

也是处理 worker

## vite:asset

插件代码如下：

```ts
function assetPlugin(config: ResolvedConfig): Plugin {
  assetHashToFilenameMap.set(config, new Map())

  return {
    name: 'vite:asset',

    buildStart() {
      assetCache.set(config, new Map())
      emittedHashMap.set(config, new Set())
    },
    resolveId(id) {
      if (!config.assetsInclude(cleanUrl(id))) {
        return
      }
      const publicFile = checkPublicFile(id, config)
      if (publicFile) {
        return id
      }
    },
    async load(id) {},
    renderChunk(code, chunk) {},
    generateBundle(_, bundle) {}
  }
}
```

处理静态资源

## vite:define

插件代码如下：

```ts
function definePlugin(config: ResolvedConfig): Plugin {
  return {
    name: 'vite:define',

    transform(code, id, options) {}
  }
}
```

替换`config.define`全局变量，开发期间会采用`vite:client-inject`插件处理 define。

## vite:css-post

插件代码如下：

```ts
function cssPostPlugin(config: ResolvedConfig): Plugin {
  return {
    name: 'vite:css-post',

    buildStart() {
      pureCssChunks = new Set<string>()
      outputToExtractedCSSMap = new Map<NormalizedOutputOptions, string>()
      hasEmitted = false
    },
    async transform(css, id, options) {},
    async renderChunk(code, chunk, options) {},
    async generateBundle(opts, bundle) {}
  }
}
```

css 的后处理

## vite:client-inject

> 只在开发期间有效，`serve`命令

插件代码如下：

```ts
function clientInjectionsPlugin(config: ResolvedConfig): Plugin {
  return {
    name: 'vite:client-inject',

    transform(code, id) {}
  }
}
```

替换客户端代码中的全局常量，主要是下面这两个文件：

- vite/dist/client/client.mjs
- vite/dist/client/env.mjs

## vite:import-analysis

> 只在开发期间有效，`serve`命令

插件代码如下：

```ts
function importAnalysisPlugin(config: ResolvedConfig): Plugin {
  let server: ViteDevServer

  return {
    name: 'vite:import-analysis',

    configureServer(_server) {
      server = _server
    },
    async transform(source, importer, options) {}
  }
}
```

重写`import/require`语句
