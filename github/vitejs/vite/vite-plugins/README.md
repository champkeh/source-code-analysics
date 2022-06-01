# vite 插件是如何运作的

vite 有一个插件系统，这个插件系统的接口继承自 rollup 的插件接口，能够兼容一部分 rollup 插件的 hooks，同时也提供了 vite 独有的 hooks。 这篇文章主要是分析一下 vite 插件是如何影响 vite
工具链的。

## vite 插件支持的 Hooks

### 1. Universal Hooks (兼容 rollup)

> 这些 hooks 是通过 PluginContainer 来模拟执行的。具体内容后面会进行分析。

在开发模式(serve)下，会执行下面这些 hooks：

- options
- buildStart
- resolveId
- load
- transform
- buildEnd
- closeBundle

在构建模式(build)下，除了上面的 hooks 外，还会额外执行下面这些 hooks：

- moduleParsed
- [output generation hooks](https://rollupjs.org/guide/en/#output-generation-hooks)

> 毕竟目前阶段，vite 内部的构建(build)功能仍然是采用 rollup 实现的，因此在构建时支持完整的 rollup hooks。

### 2. vite 独有的 Hooks

- config
- configResolved
- configureServer
- transformIndexHtml
- handleHotUpdate

## 插件的 Hooks 执行流程解析

下面以一个假想的项目配置来分析插件是如何运作的：

```ts
// vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import legacy from '@vitejs/plugin-legacy'
import styleImport, { VantResolve } from 'vite-plugin-style-import'
import Inspect from 'vite-plugin-inspect'

export default defineConfig({
  plugins: [
    vue(),
    legacy({
      targets: ['defaults', 'not IE 11']
    }),
    styleImport({
      resolves: [VantResolve()]
    }),
    Inspect()
  ]
})
```

> 注意：
> vite.config.js 中配置的`plugins`插件并不是都会影响到`serve`或者`build`命令，而是要看这个插件的`apply`字段。如果未指定`apply`的话，默认会同时对`serve`和`build`命令生效，也可以单独配置成`serve`或者`build`，表示只对对应的命令生效。

下面就基于这个配置，看一下这 4 个插件对开发服务器(vite serve 命令)都有哪些影响。

### 1. 解析配置阶段执行`config`钩子注入配置

从 [vite 开发服务器启动流程梳理](learn/vite-dev-server/README.md) 这篇文章中我们知道，从配置文件中加载完 UserConfig 之后，会执行插件的`config`钩子来注入插件的配置。

首先，会根据当前执行的命令对插件进行过滤：

针对上面的插件配置，过滤结果如下：

- vite:vue (没有`apply`字段)
- vite:legacy-env (没有`apply`字段)
- vite:style-import (没有`apply`字段)
- vite-plugin-inspect (`apply`字段为`serve`)

再按照`enforce`字段排序为：

- vite:vue (normal)
- vite:legacy-env (normal)
- vite-plugin-inspect (normal)
- vite:style-import (post)

插件列表及顺序确定之后，就开始执行插件的`config`钩子：

#### 1) vite:vue 插件

`vite:vue`插件的`config`代码如下：

```ts
export default function vuePlugin(rawOptions: Options = {}): Plugin {
  return {
    name: 'vite:vue',

    config(config) {
      return {
        define: {
          __VUE_OPTIONS_API__: config.define?.__VUE_OPTIONS_API__ ?? true,
          __VUE_PROD_DEVTOOLS__: config.define?.__VUE_PROD_DEVTOOLS__ ?? false
        },
        ssr: {
          external: ['vue', '@vue/server-renderer']
        }
      }
    }
  }
}
```

`vite:vue`插件的`config`钩子返回如下：

```ts
const config = {
  define: {
    __VUE_OPTIONS_API__: true,
    __VUE_PROD_DEVTOOLS__: false
  },
  ssr: {
    external: ['vue', '@vue/server-renderer']
  }
}
```

#### 2) vite:legacy-env 插件

`vite:legacy-env`插件的`config`代码如下：

```ts
export default function viteLegacyEnvPlugin(options) {
  return {
    name: 'vite:legacy-env',

    config(config, env) {
      if (env) {
        return {
          define: {
            'import.meta.env.LEGACY':
              env.command === 'serve' || config.build.ssr
                ? false
                : legacyEnvVarMarker
          }
        }
      } else {
        envInjectionFailed = true
      }
    }
  }
}
```

`vite:legacy-env`插件的`config`钩子返回如下：

```ts
const config = {
  define: {
    'import.meta.env.LEGACY': false
  }
}
```

`vite-plugin-inspect`插件和`vite:style-import`插件没有`config`钩子。

这些插件的`config`钩子执行完以后，继续解析配置。比如注入客户端别名、加载环境文件、确定`build`配置、确定`cacheDir/publicDir`、确定`server`配置等等。

### 2. 解析配置阶段执行`configResolved`钩子保存配置

等所有的配置都解析完成之后，会执行插件的`configResolved`钩子。

> 注意，这时候的插件列表不仅仅是用户配置的插件，vite 会给`config.worker.plugins`和`config.plugins`这两个列表注入大量内置的插件。

此时会先执行`config.worker.plugins`插件列表，然后再执行普通`config.plugins`插件列表。

serve 命令下注入的内置插件列表如下(16 个)：

- vite:pre-alias
- alias
- (此处会插入用户的 **pre** 插件)
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
- (此处会插入用户的 **normal** 插件)
- vite:define
- vite:css-post
- (此处会插入用户的 **post** 插件)
- vite:client-inject
- vite:import-analysis

不过这些内置插件都没有`configResolved`钩子。

用户配置的 4 个插件中，有`configResolved`钩子的有：

#### 1) vite:vue 插件

代码如下：

```ts
export default function vuePlugin(rawOptions: Options = {}): Plugin {
  return {
    name: 'vite:vue',

    configResolved(config) {
      options = {
        ...options,
        root: config.root,
        sourceMap: config.command === 'build' ? !!config.build.sourcemap : true,
        isProduction: config.isProduction
      }
    }
  }
}
```

#### 2) vite:legacy-env 插件

代码如下：

```ts
export default function viteLegacyEnvPlugin(options) {
  return {
    name: 'vite:legacy-env',

    configResolved(config) {
      if (envInjectionFailed) {
        config.logger.warn(
          `[@vitejs/plugin-legacy] import.meta.env.LEGACY was not injected due ` +
            `to incompatible vite version (requires vite@^2.0.0-beta.69).`
        )
      }
    }
  }
}
```

#### 3) vite:style-import 插件

代码如下：

```ts
export default function (options: VitePluginOptions): Plugin {
  return {
    name: 'vite:style-import',
    enforce: 'post',

    configResolved(resolvedConfig) {
      needSourcemap = !!resolvedConfig.build.sourcemap
      isBuild =
        resolvedConfig.isProduction || resolvedConfig.command === 'build'
      external = resolvedConfig?.build?.rollupOptions?.external ?? undefined
      debug('plugin config:', resolvedConfig)
    }
  }
}
```

#### 4) vite-plugin-inspect 插件

代码如下：

```ts
export default function (options: Options = {}): Plugin {
  return {
    name: 'vite-plugin-inspect',
    apply: 'serve',

    configResolved(_config) {
      config = _config
      config.plugins.forEach(hijackPlugin)
    }
  }
}
```

可以看到，这个钩子的目的是为了保存这个最终的`resolvedConfig`对象，方便插件的其他钩子使用。

### 3. 创建 PluginContainer 时执行`options`钩子

> 关于这个钩子的文档，可以查看[这里](https://rollupjs.org/guide/en/#options).

代码如下：

```ts
const container: PluginContainer = {
  options: await(async () => {
    let options = rollupOptions
    for (const plugin of plugins) {
      if (!plugin.options) continue
      options = (await plugin.options.call(minimalContext, options)) || options
    }
    if (options.acornInjectPlugins) {
      parser = acorn.Parser.extend(options.acornInjectPlugins as any)
    }
    return {
      acorn,
      acornInjectPlugins: [],
      ...options
    }
  })()
}
```

可以看到，options 是一个 IIFE，所以在创建 PluginContainer 实例时会立即执行。

目前插件列表里面没有一个插件有`options`钩子。

### 4. 服务器实例创建完成后收集`transformIndexHtml`钩子(不在此处执行)

在服务器实例创建完成之后有这样一段代码：

```ts
server.transformIndexHtml = createDevHtmlTransformFn(server)
```

`createDevHtmlTransformFn`函数如下：

```ts
export function createDevHtmlTransformFn(
  server: ViteDevServer
): (url: string, html: string, originalUrl: string) => Promise<string> {
  const [preHooks, postHooks] = resolveHtmlTransforms(server.config.plugins)

  return (url: string, html: string, originalUrl: string): Promise<string> => {
    return applyHtmlTransforms(html, [...preHooks, devHtmlHook, ...postHooks], {
      path: url,
      filename: getHtmlFilename(url, server),
      server,
      originalUrl
    })
  }
}
```

这个函数会把`config.plugins`列表中的所有插件的`transformIndexHtml`钩子函数收集起来，并分类存放在`preHooks`和`postHooks`
两个闭包容器中，然后将返回的函数保存在服务器实例的`transformIndexHtml`字段上。

> 这个函数在后面进行分析。

### 5. 注册内部中间件之前执行`configureServer`钩子

有这个钩子的插件列表如下：

- vite:pre-alias
- vite:resolve
- vite:css
- vite:esbuild
- vite:vue
- vite-plugin-inspect
- vite:import-analysis

#### 1) vite:pre-alias 内置插件

代码如下：

```ts
export function preAliasPlugin(): Plugin {
  let server: ViteDevServer

  return {
    name: 'vite:pre-alias',

    configureServer(_server) {
      server = _server
    }
  }
}
```

#### 2) vite:resolve 内置插件

代码如下：

```ts
export default function resolvePlugin(
  baseOptions: InternalResolveOptions
): Plugin {
  let server: ViteDevServer | undefined

  return {
    name: 'vite:resolve',

    configureServer(_server) {
      server = _server
    }
  }
}
```

#### 3) vite:css 内置插件

代码如下：

```ts
export function cssPlugin(config: ResolvedConfig): Plugin {
  let server: ViteDevServer

  return {
    name: 'vite:css',

    configureServer(_server) {
      server = _server
    }
  }
}
```

#### 4) vite:esbuild 内置插件

代码如下：

```ts
export function esbuildPlugin(options: ESBuildOptions = {}): Plugin {
  return {
    name: 'vite:esbuild',

    configureServer(_server) {
      server = _server
      server.watcher
        .on('add', reloadOnTsconfigChange)
        .on('change', reloadOnTsconfigChange)
        .on('unlink', reloadOnTsconfigChange)
    }
  }
}
```

#### 5) vite:vue 用户插件

代码如下：

```ts
export default function vuePlugin(): Plugin {
  return {
    name: 'vite:vue',

    configureServer(server) {
      options.devServer = server
    }
  }
}
```

#### 6) vite-plugin-inspect 用户插件

代码如下：

```ts
export default function (): Plugin {
  return {
    name: 'vite-plugin-inspect',

    configureServer(server: ViteDevServer) {
      server.middlewares.use(
        '/__inspect',
        sirv(resolve(_dirname, '../dist/client'), {
          single: true,
          dev: true
        })
      )

      server.middlewares.use('/__inspect_api', (req, res) => {
        // 代码省略
      })
    }
  }
}
```

#### 7) vite:import-analysis 内置插件

代码如下：

```ts
export function importAnalysisPlugin(config: ResolvedConfig): Plugin {
  let server: ViteDevServer

  return {
    name: 'vite:import-analysis',

    configureServer(_server) {
      server = _server
    }
  }
}
```

可以看到，通过`configureServer`钩子我们可以拿到服务器实例，有了服务器实例，我们就可以做很多事情，比如，给 watcher 注册事件，给内部中间件添加新的中间件。

接下来是注册内部中间件，跟插件关系不大。

### 6. 启动服务器时执行`buildStart`钩子

还记得我们重写了`httpServer.listen`方法吗？

```ts
let isOptimized = false
// overwrite listen to run optimizer before server start
const listen = httpServer.listen.bind(httpServer)
httpServer.listen = async (port: number, ...args: any[]) => {
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
}
```

因此，在启动服务器的时候，会执行下面的代码：

```ts
await container.buildStart({})
await runOptimize()
```

`container`就是我们最开始说的 PluginContainer，用来模拟执行 rollup 的钩子函数的。

其中，PluginContainer 的 buildStart 方法如下：

```ts
const container: PluginContainer = {
  async buildStart() {
    await Promise.all(
      plugins.map((plugin) => {
        if (plugin.buildStart) {
          return plugin.buildStart.call(
            new Context(plugin) as any,
            container.options as NormalizedInputOptions
          )
        }
      })
    )
  }
}
```

其中，插件列表里面有`buildStart`钩子的插件如下：

- alias
- vite:css
- vite:asset
- vite:css-post

> todo: 这里可以检查一下每个插件的 buildStart 钩子都做了什么。

## 总结

以上，就是插件在整个开发服务器启动过程中所做的事情。

其中，在服务器启动阶段，hooks 的执行顺序如下：

1. config
2. configResolved
3. options
4. configureServer
5. buildStart
