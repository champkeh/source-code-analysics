# vite 开发服务器启动流程梳理

据官方文档介绍，vite 主要由两部分组成：

- 一个开发服务器
- 一套构建指令

这里我们重点关注这个开发服务器。

## 背景介绍

在启动开发服务器的时候，config 有 2 个来源：

- 命令行选项，也被称为`InlineConfig`
- 配置文件，也被称为`UserConfig`，比如`vite.config.js`

它们的优先级依次降低。

## 开发服务器启动流程

下面分析一下在执行`vite serve root`启动开发服务器时的流程：

首先，`vite serve`命令会命中`cli.ts`中的`dev/serve`子命令，会执行对应的`action`，`action`回调函数收到`root`参数和解析后的`options`参数，执行以下代码创建并启动服务器：

```ts
// 创建服务器
const server = await createServer({
  root,
  base: options.base,
  mode: options.mode,
  configFile: options.config,
  logLevel: options.logLevel,
  clearScreen: options.clearScreen,
  server: cleanOptions(options)
})

// 启动服务器
await server.listen()

// 打印服务地址
server.printUrls()
```

## 创建服务器流程

```ts
// 传递给 createServer 的选项，这些选项都来自于命令行，并且命令行也只能传递这些选项
interface InlineConfig {
  root: string
  base: string
  mode: string
  configFile: string
  logLevel: LogLevel
  clearScreen: boolean
  server: {
    host: string
    port: number
    https: boolean
    open: boolean | string
    cors: boolean
    strictPort: boolean
    force: boolean
  }
}

function createServer(inlineConfig: InlineConfig = {}): Promise<ViteDevServer> {
  // 创建服务器实例
}
```

### 一、确定配置

创建服务器的第一步就是确定配置，这一步由`resolveConfig`函数负责，签名如下：

```ts
async function resolveConfig(
  inlineConfig: InlineConfig,
  command: 'build' | 'serve',
  defaultMode = 'development'
): Promise<ResolvedConfig> {
  //
}
```

这个函数分为下面 3 个主要步骤，分别从不同地方去加载配置：

#### 1. 加载配置文件中的配置

这个函数首先调用`loadConfigFromFile`从指定的配置文件加载配置(UserConfig)，如果没有明确指定配置文件的话，会依次检查根目录下面的`vite.config.js`/`vite.config.mjs`
/`vite.config.ts`/`vite.config.cjs`。如果这些文件都不存在，那这一步的结果就是`null`。

如果存在配置文件的话，会先调用`esbuild.build`将配置文件打包，然后使用打包后的文件作为最终的配置文件。内部`esbuild`配置如下：

> 关于 esbuild 的配置选项，可以查看[这里](https://esbuild.github.io/api/#simple-options).  
> 关于 esbuild 的插件，可以查看[这里](https://esbuild.github.io/plugins/#using-plugins).

```ts
const result = await build({
  absWorkingDir: process.cwd(),
  entryPoints: [fileName],
  outfile: 'out.js',
  write: false,
  platform: 'node',
  bundle: true,
  format: isESM ? 'esm' : 'cjs',
  sourcemap: 'inline',
  metafile: true,
  plugins: [
    {
      name: 'externalize-deps',
      setup(build) {
        build.onResolve({ filter: /.*/ }, (args) => {
          const id = args.path
          if (id[0] !== '.' && !path.isAbsolute(id)) {
            return {
              external: true
            }
          }
        })
      }
    },
    {
      name: 'replace-import-meta',
      setup(build) {
        build.onLoad({ filter: /\.[jt]s$/ }, async (args) => {
          const contents = await fs.promises.readFile(args.path, 'utf8')
          return {
            loader: args.path.endsWith('.ts') ? 'ts' : 'js',
            contents: contents
              .replace(
                /\bimport\.meta\.url\b/g,
                JSON.stringify(`file://${args.path}`)
              )
              .replace(
                /\b__dirname\b/g,
                JSON.stringify(path.dirname(args.path))
              )
              .replace(/\b__filename\b/g, JSON.stringify(args.path))
          }
        })
      }
    }
  ]
})
```

从这个配置我们可以知道，`vite.config.js`可以导入其它包(因为最终会通过 esbuild 打包)，可以采用 esm/cjs 格式，可以使用 js/ts 语言，内部可以使用`__dirname`/`__filename`
/`import.meta.url`这些变量。

由于配置文件导出的可能是一个函数，所以这一步会进行判断，如果是函数的话就先调用这个函数，从而拿到真正的 UserConfig，如下：

```ts
const config = await(
  typeof userConfig === 'function' ? userConfig(configEnv) : userConfig
)
```

> 由于函数调用前有`await`，所以配置文件中的函数可以是异步函数

拿到配置文件中的 UserConfig 以后，需要将这个配置与命令行中的 InlineConfig 进行合并，并且命令行配置的优先级更高：

```ts
config = mergeConfig(loadResult.config, inlineConfig)
```

到此，基础配置解析完成。

#### 2. 执行插件的`config`钩子注入插件的配置

配置文件中`plugins`列表配置的插件并不都对开发服务器(serve 命令)有效，因为插件分为`serve`插件和`build`插件，因此我们需要过滤出`serve`插件。

```ts
const rawUserPlugins = config.plugins.flat().filter((p) => {
  if (!p) {
    return false
  } else if (!p.apply) {
    // 如果插件没有`apply`字段的话，同时适用于`serve`和`build`命令
    return true
  } else if (typeof p.apply === 'function') {
    return p.apply({ ...config, mode }, configEnv)
  } else {
    return p.apply === command
  }
}) as Plugin[]
```

> 关于`apply`配置，可以查看文档 [Conditional Application](https://vitejs.dev/guide/api-plugin.html#conditional-application)

过滤出来要执行的插件之后，还需要对这些插件进行一个分类，分为 pre/normal/post 三类，分别表示前置插件、普通插件、后置插件。代码如下：

```ts
rawUserPlugins.forEach((p) => {
  if (p.enforce === 'pre') prePlugins.push(p)
  else if (p.enforce === 'post') postPlugins.push(p)
  else normalPlugins.push(p)
})

const userPlugins = [...prePlugins, ...normalPlugins, ...postPlugins]
```

> 关于`enforce`配置，可以查看文档 [Plugin Ordering](https://vitejs.dev/guide/api-plugin.html#plugin-ordering)
>
> 关于插件的执行顺序，可以查看 [vite 相关插件对 serve 及 build 命令的影响](../vite-plugins/README.md)

接下来就开始执行插件的`config`钩子：

```ts
for (const p of userPlugins) {
  if (p.config) {
    const res = await p.config(config, configEnv)
    if (res) {
      config = mergeConfig(config, res)
    }
  }
}
```

可以看到，这个钩子的返回值会合并到最终的配置中。

> 在插件里面可以直接修改传递进去的`config`对象，也可以把要修改的配置返回，vite 会自动合并到配置中。

#### 3. 加载 .env 文件配置

不过在加载 .env 文件之前，需要根据上面已经确定的配置决定项目的根，也就是`root`配置。

```ts
const resolvedRoot = normalizePath(
  config.root ? path.resolve(config.root) : process.cwd()
)
```

然后从项目的`root`去加载 .env 配置：

```ts
const envDir = config.envDir
  ? normalizePath(path.resolve(resolvedRoot, config.envDir))
  : resolvedRoot
const userEnv =
  inlineConfig.envFile !== false &&
  loadEnv(mode, envDir, resolveEnvPrefix(config))
```

> 目前的`loadEnv`实现无法跨文件展开变量，也就是说，`.env.development`和`.env.test`都引用`.env`中的变量。

#### 4. 对配置进行一些调整

最后，基于所有的外部配置做一些调整。

比如，将 vite 客户端的两个别名注入到`config.resolve.alias`中。

> 客户端的两个别名分别是: `@vite/env`和`@vite/client`，这两个别名会放在用户配置的别名的后面。
>
> todo: 验证这个别名的顺序是怎样的。

确定下面这些配置：

- BASE_URL
- cacheDir
- publicDir
- config.build
- config.server (主要是文件系统访问权限配置)
- config.preview
- 更多，见下面的配置对象 resolved

到此，最终的配置被确定了。创建`resolved`变量保存这个最终确定下来的配置：

```ts
const resolved: ResolvedConfig = {
  ...config,
  configFile: configFile ? normalizePath(configFile) : undefined,
  configFileDependencies,
  inlineConfig,
  root: resolvedRoot,
  base: BASE_URL,
  resolve: resolveOptions,
  publicDir: resolvedPublicDir,
  cacheDir,
  command,
  mode,
  isProduction,
  plugins: userPlugins,
  server,
  build: resolvedBuildOptions,
  preview: resolvePreviewOptions(config.preview, server),
  env: {
    ...userEnv,
    BASE_URL,
    MODE: mode,
    DEV: !isProduction,
    PROD: isProduction
  },
  assetsInclude(file: string) {
    return DEFAULT_ASSETS_RE.test(file) || assetsFilter(file)
  },
  logger,
  packageCache: new Map(),
  createResolver,
  optimizeDeps: {
    ...config.optimizeDeps,
    esbuildOptions: {
      keepNames: config.optimizeDeps?.keepNames,
      preserveSymlinks: config.resolve?.preserveSymlinks,
      ...config.optimizeDeps?.esbuildOptions
    }
  },
  worker: resolvedWorkerOptions
}
```

接下来，为了插件 hooks 的完整性，这里需要调用插件的`configResolved`钩子。

> 在此时，vite 会给`config.worker.plugins`和`config.plugins`这两个插件列表注入大量的内置插件，关于这些内置插件可以查看 [vite 内置插件一览](../vite-plugins/BUILTIN.md)

到此，`resolveConfig`函数的工作完成，最终的配置也确定下来了。

在这个过程中涉及到插件的 2 个钩子函数：

- config 这个钩子在解析完配置文件中的配置对象后会进行调用
- configResolved 这个钩子用于配置对象确定之后进行调用

---

### 二、创建 http 服务器和 ws 服务器

```ts
const middlewares = connect() as Connect.Server
const httpServer = middlewareMode
  ? null
  : await resolveHttpServer(serverConfig, middlewares, httpsOptions)
const ws = createWebSocketServer(httpServer, config, httpsOptions)
```

可以看到，内部服务器采用`connect`这个包来实现中间件架构，同时，如果我们在启动 vite 服务器的时候没有指定以中间件模式运行的话，vite 会创建一个 http 服务器，如下所示：

```ts
async function resolveHttpServer(
  { proxy }: CommonServerOptions,
  app: Connect.Server,
  httpsOptions?: HttpsServerOptions
): Promise<HttpServer> {
  if (!httpsOptions) {
    return require('http').createServer(app)
  }

  if (proxy) {
    // #484 fallback to http1 when proxy is needed.
    return require('https').createServer(httpsOptions, app)
  } else {
    return require('http2').createSecureServer(
      {
        ...httpsOptions,
        allowHTTP1: true
      },
      app
    )
  }
}
```

然后基于这个 httpServer 创建一个 wsServer，websocket 服务是使用的`ws`这个包，并且是采用的多个 ws 服务共享一个 http 服务的模式，如下：

```ts
wss = new WebSocket({ noServer: true })

wsServer.on('upgrade', (req, socket, head) => {
  if (req.headers['sec-websocket-protocol'] === HMR_HEADER) {
    wss.handleUpgrade(req, socket as Socket, head, (ws) => {
      wss.emit('connection', ws, req)
    })
  }
})
```

> 参考[Multiple servers sharing a single HTTP/S server](https://github.com/websockets/ws#multiple-servers-sharing-a-single-https-server)

### 三、创建 watcher

watcher 用于监听文件系统的变化，采用`chokidar`包实现：

```ts
const watcher = chokidar.watch(path.resolve(root), {
  ignored: [
    '**/node_modules/**',
    '**/.git/**',
    ...(Array.isArray(ignored) ? ignored : [ignored])
  ],
  ignoreInitial: true,
  ignorePermissionErrors: true,
  disableGlobbing: true,
  ...watchOptions
}) as FSWatcher
```

### 四、创建 ModuleGraph 和 PluginContainer

```ts
const moduleGraph: ModuleGraph = new ModuleGraph((url, ssr) =>
  container.resolveId(url, undefined, { ssr })
)

const container = await createPluginContainer(config, moduleGraph, watcher)
```

创建一个关闭服务器的辅助函数`closeHttpServer`：

```ts
const closeHttpServer = createServerCloseFn(httpServer)
```

到此，`viteDevServer`实例已经创建完成：

```ts
const server: ViteDevServer = {
  config,
  middlewares,
  get app() {
    config.logger.warn(
      `ViteDevServer.app is deprecated. Use ViteDevServer.middlewares instead.`
    )
    return middlewares
  },
  httpServer,
  watcher,
  pluginContainer: container,
  ws,
  moduleGraph,
  ssrTransform,
  transformWithEsbuild,
  transformRequest(url, options) {
    return transformRequest(url, server, options)
  },
  transformIndexHtml: null!, // to be immediately set
  ssrLoadModule(url) {
    server._ssrExternals ||= resolveSSRExternal(
      config,
      server._optimizeDepsMetadata
        ? Object.keys(server._optimizeDepsMetadata.optimized)
        : []
    )
    return ssrLoadModule(url, server)
  },
  ssrFixStacktrace(e) {
    if (e.stack) {
      const stacktrace = ssrRewriteStacktrace(e.stack, moduleGraph)
      rebindErrorStacktrace(e, stacktrace)
    }
  },
  listen(port?: number, isRestart?: boolean) {
    return startServer(server, port, isRestart)
  },
  async close() {
    process.off('SIGTERM', exitProcess)

    if (!middlewareMode && process.env.CI !== 'true') {
      process.stdin.off('end', exitProcess)
    }

    await Promise.all([
      watcher.close(),
      ws.close(),
      container.close(),
      closeHttpServer()
    ])
  },
  printUrls() {
    if (httpServer) {
      printCommonServerUrls(httpServer, config.server, config)
    } else {
      throw new Error('cannot print server URLs in middleware mode.')
    }
  },
  async restart(forceOptimize: boolean) {
    if (!server._restartPromise) {
      server._forceOptimizeOnRestart = !!forceOptimize
      server._restartPromise = restartServer(server).finally(() => {
        server._restartPromise = null
        server._forceOptimizeOnRestart = false
      })
    }
    return server._restartPromise
  },

  _optimizeDepsMetadata: null,
  _ssrExternals: null,
  _globImporters: Object.create(null),
  _restartPromise: null,
  _forceOptimizeOnRestart: false,
  _isRunningOptimizer: false,
  _registerMissingImport: null,
  _pendingReload: null,
  _pendingRequests: new Map()
}

// note: 注意这里，后面会解释
server.transformIndexHtml = createDevHtmlTransformFn(server)
```

### 五、配置 watcher

将 json 文件添加到 watch 列表中：

```ts
const { packageCache } = config
const setPackageData = packageCache.set.bind(packageCache)
packageCache.set = (id, pkg) => {
  if (id.endsWith('.json')) {
    watcher.add(id)
  }
  return setPackageData(id, pkg)
}
```

设置 watcher 的事件处理程序：

```ts
watcher.on('change', async (file) => {})
watcher.on('add', (file) => {})
watcher.on('unlink', (file) => {})
```

到这里，服务器实例已经创建成功，此时需要执行插件的钩子来注入自定义中间件。

### 六、安装中间件

在安装 vite 内部中间件之前，vite 提供了一种提前安装中间件的机制，那就是直接在`configureServer`钩子中使用传递进来的`server`实例注册中间件。

当然，这个钩子也可以返回一个函数，不过在返回的函数中安装的中间件都是位于内部中间件之后了。

> 关于中间件，可以阅读 [vite 内置中间件的作用](../vite-middleware/README.md)

#### 1. 执行插件的`configureServer`钩子

在这个钩子中可以给服务器添加自定义的中间件函数，默认这些中间件位于内部中间件之前执行，如果想把这些中间件安装在内部中间件之后，需要返回一个函数。

```ts
const postHooks: ((() => void) | void)[] = []
for (const plugin of config.plugins) {
  if (plugin.configureServer) {
    postHooks.push(await plugin.configureServer(server))
  }
}
```

#### 2. 安装内部中间件

包括下面这些内部中间件：

- timeMiddleware (调试模式下)用于打印请求的处理时长
- corsMiddleware (启用 cors 的情况下)
- proxyMiddleware
- baseMiddleware
- launchEditorMiddleware
- viteHMRPingMiddleware
- servePublicMiddleware
- transformMiddleware
- serveRawFsMiddleware
- serveStaticMiddleware
- spaFallbackMiddleware

#### 3. 安装第 1 步中 postHooks 中的中间件

#### 4. 安装其他中间件(兜底中间件)

其他中间件包括：

- indexHtmlMiddleware
- vite404Middleware
- errorMiddleware

### 七、修改`httpServer.listen`

主要目的是在启动服务器之前自动执行下面这两句代码：

```ts
await container.buildStart({})
await runOptimize()
```

> 这两句代码的作用会在后面分析。

## 启动服务器

```ts
await server.listen()
```

```ts
const server = {
  listen(port?: number, isRestart?: boolean) {
    return startServer(server, port, isRestart)
  }
}
```

```ts
async function startServer(
  server: ViteDevServer,
  inlinePort?: number,
  isRestart: boolean = false
): Promise<ViteDevServer> {
  const httpServer = server.httpServer
  if (!httpServer) {
    throw new Error('Cannot call server.listen in middleware mode.')
  }

  const options = server.config.server
  const port = inlinePort || options.port || 3000
  const hostname = resolveHostname(options.host)

  const protocol = options.https ? 'https' : 'http'
  const info = server.config.logger.info
  const base = server.config.base

  const serverPort = await httpServerStart(httpServer, {
    port,
    strictPort: options.strictPort,
    host: hostname.host,
    logger: server.config.logger
  })

  if (options.open && !isRestart) {
    const path = typeof options.open === 'string' ? options.open : base
    openBrowser(
      path.startsWith('http')
        ? path
        : `${protocol}://${hostname.name}:${serverPort}${path}`,
      true,
      server.config.logger
    )
  }

  return server
}
```

### 关于`httpServer.listen`的监听

通过以上启动代码可以知道，调用`server.listen()`启动服务器时，最终会调用`httpServerStart(server.httpServer)`，`httpServerStart`函数如下：

```ts
async function httpServerStart(
  httpServer: HttpServer,
  serverOptions: {
    port: number
    strictPort: boolean | undefined
    host: string | undefined
    logger: Logger
  }
): Promise<number> {
  return new Promise((resolve, reject) => {
    let { port, strictPort, host, logger } = serverOptions

    const onError = (e: Error & { code?: string }) => {
      if (e.code === 'EADDRINUSE') {
        if (strictPort) {
          httpServer.removeListener('error', onError)
          reject(new Error(`Port ${port} is already in use`))
        } else {
          logger.info(`Port ${port} is in use, trying another one...`)
          httpServer.listen(++port, host)
        }
      } else {
        httpServer.removeListener('error', onError)
        reject(e)
      }
    }

    httpServer.on('error', onError)

    httpServer.listen(port, host, () => {
      httpServer.removeListener('error', onError)
      resolve(port)
    })
  })
}
```

这里在会监听 httpServer 的 error 事件，如果这个错误是由于端口被占用导致的错误，并且`strictPort`配置不为`true`的情况下，会尝试更换端口号重试。

注意这里调用的`httpServer.listen`是经过重写的，我们在创建服务器的最后有这样一段代码：

```ts
if (!middlewareMode && httpServer) {
  let isOptimized = false
  // overwrite listen to run optimizer before server start
  const listen = httpServer.listen.bind(httpServer)
  httpServer.listen = (async (port: number, ...args: any[]) => {
    if (!isOptimized) {
      try {
        await container.buildStart({})
        await runOptimize()
        isOptimized = true
      } catch (e) {
        httpServer.emit('error', e)
        return
      }
    }
    return listen(port, ...args)
  }) as any
} else {
  await container.buildStart({})
  await runOptimize()
}
```

而这个`listen`函数有可能会执行多次(由于端口号被占用导致的失败)，所以内部有一个`isOptimized`变量用来控制这段代码只会被执行一次。

同时，服务器启动成功之后，会回调 httpServer 上绑定的`listening`事件回调。目前已知的注册这个事件的地方有：

1. 在 watcher 事件注册之后，会给 httpServer 注册一个`listening`回调，用于将真正的端口号更新到`config.server.port`上。
2. 在`createServerCloseFn`函数执行的时候，会给 httpServer 注册一个`listening`回调，用于更新闭包变量`hasListened`

## 打印服务器地址

```ts
server.printUrls()
```
