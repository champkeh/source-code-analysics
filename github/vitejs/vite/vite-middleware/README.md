# vite 中间件组成的请求处理管道

在启动开发服务器的时候，我们会使用`connect`包来创建一个中间件应用，然后不管是插件还是 vite 本身，在后续过程中都会不断地给这个应用注册新的中间件，这些中间件共同组成了开发服务器处理请求的 pipeline。

这篇文章我们就来分析下这个中间件组成的处理管道。

## 前置知识

可以查看 [Connect](https://www.npmjs.com/package/connect) 的文档，了解这个中间件框架的特性。

主要有 3 点需要说明一下：

1. 中间件注册类似于栈结构，先注册的中间件在栈的底部，后注册的中间件的栈的顶部；
2. 当请求到达服务器时，会从栈的底部开始调用中间件，如果这个中间件调用了`next`，则会继续调用它后面的中间件，以此类推；
3. 当某个中间件在调用`next`时传递了`error`对象，则开始顺着管道寻找错误处理中间件，并调用找到的第一个错误处理中间件处理该错误，中间遇到的非错误处理中间件都会被忽略。

## 开发服务器的请求处理管道

通过 [vite 开发服务器启动流程梳理](learn/vite-dev-server/README.md) 和 [vite 插件是如何运作的](learn/vite-plugins/README.md)
我们知道，在服务器启动过程中依次注册了下面这些中间件：

- /\_\_inspect => (vite-plugin-inspect 插件注册)
- /\_\_inspect_api => (vite-plugin-inspect 插件注册)
- / => viteTimeMiddleware
- / => corsMiddleware
- / => viteProxyMiddleware
- / => viteBaseMiddleware
- /\_\_open-in-editor => launchEditorMiddleware
- /\_\_vite_ping => viteHMRPingMiddleware
- / => viteServePublicMiddleware
- / => viteTransformMiddleware
- / => viteServeRawFsMiddleware
- / => viteServeStaticMiddleware
- / => viteSpaFallbackMiddleware
- / => viteIndexHtmlMiddleware
- / => vite404Middleware
- / => viteErrorMiddleware

下面就对这些中间件的处理过程进行一个简单的分析。

### timeMiddleware

time 中间件通过给`res.end()`方法注入代码，用于记录每个请求的耗时，如下：

```ts
const start = performance.now()
const end = res.end
res.end = (...args: any[]) => {
  logTime(`${timeFrom(start)} ${prettifyUrl(req.url!, root)}`)
  // @ts-ignore
  return end.call(res, ...args)
}
next()
```

### corsMiddleware

开启 cors 支持，采用的是`cors`包。

### proxyMiddleware

根据配置的`server.proxy`启动对应的代理服务器，用于处理代理请求。

### baseMiddleware

这个中间件会把路径开头的`base`去掉，避免后续中间件处理 base 问题。
如果请求路径不是以`config.base`开头，对于`/`或者`/index.html`
，则重定向到`config.base`，然后回到刚开始的处理逻辑。如果请求的是非`base`目录下的文档资源，则返回 404 错误。如果是非 html 资源，则继续后续处理。

### launchEditorMiddleware

访问`/__open-in-editor`路径时，会调用该中间件在编辑器打开。

### viteHMRPingMiddleware

这个中间件用于处理客户端 websocket 发送的 ping 心跳请求。

### servePublicMiddleware

这个中间件采用`sirv`来处理 /public 目录下的静态资源，配置如下：

```ts
const sirvOptions: Options = {
  dev: true,
  etag: true,
  extensions: [],
  setHeaders(res, pathname) {
    // Matches js, jsx, ts, tsx.
    // The reason this is done, is that the .ts file extension is reserved
    // for the MIME type video/mp2t. In almost all cases, we can expect
    // these files to be TypeScript files, and for Vite to serve them with
    // this Content-Type.
    if (/\.[tj]sx?$/.test(pathname)) {
      res.setHeader('Content-Type', 'application/javascript')
    }
  }
}
const serve = sirv(publicDir, sirvOptions)
```

> 关于`sirv`包的用法，可以查看[文档](https://www.npmjs.com/package/sirv).

这个中间件会跳过一些 **重要请求** 和 **内部请求**，如下所示：

```ts
// skip import request and internal requests `/@fs/ /@vite-client` etc...
if (isImportRequest(req.url!) || isInternalRequest(req.url!)) {
  return next()
}
serve(req, res, next)
```

那么何为重要请求和内部请求呢？

满足下面的正则的请求即为**重要请求**：

```ts
const importQueryRE = /(\?|&)import=?(?:&|$)/
```

这段正则的意思是，请求路径中包含`import`查询参数，并且这个参数不能有值，`=`可出现可不出现。

满足下面的正则的请求即为**内部请求**：

```ts
const InternalPrefixRE = /^(?:\/@fs\/|\/@id\/|\/@vite\/client|\/@vite\/env)/
```

这段正则的意思是，请求路径以下面四个字符串开头：

- /@fs/
- /@id/
- /@vite/client
- /@vite/env

如果该中间件命中了静态资源，则请求不再经过后续中间件处理而开始返回；如果没有命中静态资源的话，会继续调用后续中间件进行处理。

### transformMiddleware

这个中间件是重点。后面会详细分析。

开头会有一个判断，如果是非 GET 请求，或者请求的 url 是 / 或者 /favicon.ico，则该中间件不进行处理。

### serveRawFsMiddleware

这个中间件用于处理文件系统资源请求，也就是以`/@fs/`开头的**重要请求**。非文件系统资源的请求则忽略该中间件。

这个中间件也是采用`sirv`包处理的，如下：

```ts
const sirvOptions: Options = {
  dev: true,
  etag: true,
  extensions: [],
  setHeaders(res, pathname) {
    // Matches js, jsx, ts, tsx.
    // The reason this is done, is that the .ts file extension is reserved
    // for the MIME type video/mp2t. In almost all cases, we can expect
    // these files to be TypeScript files, and for Vite to serve them with
    // this Content-Type.
    if (/\.[tj]sx?$/.test(pathname)) {
      res.setHeader('Content-Type', 'application/javascript')
    }
  }
}

const serveFromRoot = sirv('/', sirvOptions)
```

### serveStaticMiddleware

这个中间件也是采用`sirv`包处理项目根目录(root)下的静态资源。

```ts
const serve = sirv(root, sirvOptions)
```

这个中间件会忽略**内部请求**、html 文件请求和以`/`结尾的请求。

### spaFallbackMiddleware

这个中间件用于处理 SPA 应用通常会遇到的用户刷新问题，也就是将服务器无法处理的请求重定向到`/index.html`请求。 这个中间件的重定向逻辑如下：

```ts
const historySpaFallbackMiddleware = history({
  logger: createDebugger('vite:spa-fallback'),
  // support /dir/ without explicit index.html
  rewrites: [
    {
      from: /\/$/,
      to({ parsedUrl }: any) {
        const rewritten = decodeURIComponent(parsedUrl.pathname) + 'index.html'

        if (fs.existsSync(path.join(root, rewritten))) {
          return rewritten
        } else {
          return `/index.html`
        }
      }
    }
  ]
})
```

从上面的规则可知，这个中间件会把`**/`这样的请求重定向为`**/index.html`，如果这个`index.html`文件不存在的话，则重定向为`/index.html`。

> 内部采用的是`connect-history-api-fallback`包 [文档](https://www.npmjs.com/package/connect-history-api-fallback).

### indexHtmlMiddleware

这个中间件用于处理 html 文件内容。

根据请求路径，定位到文件系统中真实的 html 文件，读取这个文件内容，并调用`server.transformIndexHtml`方法对 html 内容进行一系列的转换，最终发送给浏览器。

我们在 [vite 插件是如何运作的](learn/vite-plugins/README.md) 这篇文章中知道，在服务器实例创建完成后会把所有插件的`transformIndexHtml`
钩子函数收集起来，并保存在`server.transformIndexHtml`这个方法中进行处理。

> todo: 这里我们需要分析下每个插件是如何处理这个 HTML 内容的。

### vite404Middleware

这个中间件就是给客户端返回一个 404 错误。

如果一个请求能到达这个中间件，说明前面的中间件都无法处理该请求，因此也只能返回 404 了。

### errorMiddleware

错误处理中间件，用于处理请求在管道处理过程中遇到的错误。

这个中间件会把错误打印在服务器端的控制台，也会通过`websocket`把错误发送给客户端，客户端通过`ErrorOverlay`组件进行显示。

不过，如果是在处理第一个请求时发生的错误，由于此时客户端还没有拿到`/@vite/client`代码，websocket 链接还没有建立，所以会通过该中间件直接把错误信息通过 html 返回给客户端，如下：

```ts
res.statusCode = 500
res.end(`
    <!DOCTYPE html>
    <html lang="en">
      <head>
        <meta charset="UTF-8" />
        <title>Error</title>
        <script type="module">
          import { ErrorOverlay } from '/@vite/client'
          document.body.appendChild(new ErrorOverlay(${JSON.stringify(
            prepareError(err)
          ).replace(/</g, '\\u003c')}))
        </script>
      </head>
      <body>
      </body>
    </html>
  `)
```

## 管道中的几个静态服务中间件

### 1. public 目录

```ts
const serve = sirv(config.publicDir, sirvOptions)
```

该中间件不处理 **重要请求** 和 **内部请求**，且该中间件服务的目录默认是`/{projectRoot}/public`，逻辑如下：

```ts
// skip import request and internal requests `/@fs/ /@vite-client` etc...
if (isImportRequest(req.url!) || isInternalRequest(req.url!)) {
  return next()
}
serve(req, res, next)
```

因为这个中间件比较靠前，请求的资源优先匹配`/public`目录下面的资源，如果能够匹配，则会被这个中间件拦截而提前返回，不会再被管道后面的中间件所处理。

> 注意，请求的资源路径中不能再包含`/public`，否则`sirv`内部会拼接成`/{projectRoot}/public/public/...`这样的路径，这样的路径在文件系统中是不存在的，所以该请求不会被该中间件命中，而是会继续被后面的中间件处理。

### 2. 系统 root 目录

```ts
const serveFromRoot = sirv('/', sirvOptions)
```

该中间件用于处理 **内部请求**，具体来说就是以`/@fs/`开头的内部请求。

如果请求路径以`/@fs/`开头，则会命中这个中间件，如下：

```ts
const FS_PREFIX = `@/fs/`

if (url.startsWith(FS_PREFIX)) {
  url = url.slice(FS_PREFIX.length)
  if (isWindows) url = url.replace(/^[A-Z]:/i, '')

  req.url = url
  serveFromRoot(req, res, next)
} else {
  next()
}
```

> 这个中间件内部会检查文件访问权限。

### 3. 项目 root 目录

```ts
const serve = sirv(config.root, sirvOptions)
```

该中间件用于处理项目根目录下面的静态资源，并且不处理 **目录**、**.html** 文件和 **内部请求**，相关逻辑如下：

```ts
const cleanedUrl = cleanUrl(req.url!)
if (
  cleanedUrl.endsWith('/') ||
  path.extname(cleanedUrl) === '.html' ||
  isInternalRequest(req.url!)
) {
  return next()
}
```

并且在处理之前，会替换配置的别名(alias)，如下：

```ts
const url = decodeURI(req.url!)

// apply aliases to static requests as well
let redirected: string | undefined
for (const { find, replacement } of server.config.resolve.alias) {
  const matches =
    typeof find === 'string' ? url.startsWith(find) : find.test(url)
  if (matches) {
    redirected = url.replace(find, replacement)
    break
  }
}
```

> 这里也可以看到，`config.resolve.alias`最终是一个数组，然后从数组的开头开始遍历，找到第一个匹配就会退出这个循环。

## transformIndexHtml： 对 html 内容的处理

上面我们知道，在拿到 html 内容后，会调用`server.transformIndexHtml`对它进行处理，而`server.transformIndexHtml`内部包含了所有插件的`transformIndexHtml`
钩子，如下：

```ts
server.transformIndexHtml = (url, html, originalUrl) => {
  return applyHtmlTransforms(html, [...preHooks, devHtmlHook, ...postHooks], {
    path: url,
    filename: getHtmlFilename(url, server),
    server,
    originalUrl
  })
}
```

在我们之前的例子中，没有一个插件有`transformIndexHtml`钩子的，所以`preHooks`和`postHooks`都是空，只有一个`devHtmlHook`，代码如下：

```ts
const s = new MagicString(html)

await traverseHtml(html, htmlPath, (node) => {
  if (node.type !== NodeTypes.ELEMENT) {
    return
  }

  // script tags
  if (node.tag === 'script') {
    const { src, isModule } = getScriptInfo(node)
    if (isModule) {
      scriptModuleIndex++
    }

    if (src) {
      processNodeUrl(src, s, config, htmlPath, originalUrl, moduleGraph)
    } else if (isModule) {
      const url = filePath.replace(normalizePath(config.root), '')

      const contents = node.children
        .map((child: any) => child.content || '')
        .join('')

      // add HTML Proxy to Map
      addToHTMLProxyCache(config, url, scriptModuleIndex, contents)

      // inline js module. convert to src="proxy"
      const modulePath = `${
        config.base + htmlPath.slice(1)
      }?html-proxy&index=${scriptModuleIndex}.js`

      // invalidate the module so the newly cached contents will be served
      const module = server?.moduleGraph.getModuleById(modulePath)
      if (module) {
        server?.moduleGraph.invalidateModule(module)
      }

      s.overwrite(
        node.loc.start.offset,
        node.loc.end.offset,
        `<script type="module" src="${modulePath}"></script>`
      )
    }
  }

  // elements with [href/src] attrs
  const assetAttrs = assetAttrsConfig[node.tag]
  if (assetAttrs) {
    for (const p of node.props) {
      if (
        p.type === NodeTypes.ATTRIBUTE &&
        p.value &&
        assetAttrs.includes(p.name)
      ) {
        processNodeUrl(p, s, config, htmlPath, originalUrl)
      }
    }
  }
})

html = s.toString()

return {
  html,
  tags: [
    {
      tag: 'script',
      attrs: {
        type: 'module',
        src: path.posix.join(base, '/@vite/client')
      },
      injectTo: 'head-prepend'
    }
  ]
}
```

可以看到，上半部分主要是处理 html 内容，最后返回的 tags 数组表明，在`head`的开头插入一个`script`元素，内容为：

```html
<script type="module" src="/base/@vite/client"></script>
```

这个脚本为浏览器注入 vite 的客户端代码。

> 我们后面会单独分析这个客户端代码是干嘛的。

## transformMiddleware：处理请求的核心
