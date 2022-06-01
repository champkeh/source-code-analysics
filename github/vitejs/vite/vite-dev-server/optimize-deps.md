# 服务器在启动时执行的依赖优化具体是如何做的

> 关于启动时依赖优化，可以看官方的这篇文档：[Dependency Pre-Bundling](https://vitejs.dev/guide/dep-pre-bundling.html)

服务器在启动时会执行下面这个优化函数：

```ts
const runOptimize = async () => {
  server._isRunningOptimizer = true
  try {
    server._optimizeDepsMetadata = await optimizeDeps(
      config,
      config.server.force || server._forceOptimizeOnRestart
    )
  } finally {
    server._isRunningOptimizer = false
  }
  server._registerMissingImport = createMissingImporterRegisterFn(server)
}
```

优化逻辑在`optimizeDeps`函数中。
