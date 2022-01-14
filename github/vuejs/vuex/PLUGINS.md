# Vuex相关插件

`Vuex`插件是一个函数，该函数接收唯一的一个参数`store`。
你可以在创建`store`的时候将插件作为`plugins`选项传入，也可以在创建`store`之后，手动调用插件将`store`传入。

插件可以通过`store.commit`修改内部状态，也可以通过`store.subscribe`监听状态的变更。

## vuex-persistedstate
由于`Vuex`中的状态在页面被刷新之后会丢失，该插件用于持久化保存`Vuex`中的状态。

核心代码如下：
```js
function createPluginInstance(options) {
    
    function setState(key, state, storage) {
        return storage.setItem(key, JSON.stringify(state))
    }
    
    return (store) => {
        store.subscribe((mutation, state) => {
            setState('vuex', state, window.localStorage)
        })
    }
}
```

## vuex-router-sync
这个插件用`Vuex`的方式管理`Vue Router`的当前路由对象`$route`。

核心代码如下：
```js
function sync(store, router, options) {
    // 动态注册一个 route 模块用来保存 $route
    store.registerModule('route', {
        namespaced: true,
        state: cloneRoute(router.currentRoute),
        mutations: {
            ROUTE_CHANGED(_state, transition) {
                store.state.route = cloneRoute(transition.to, transition.from)
            }
        }
    })
    
    // 监听 store.route 状态变化
    store.watch(
        (state) => state.route, 
        (route) => {
            // 导航到新的路由
            router.push(route)
        },
        { sync: true })
    
    // 通过 router 的导航守卫同步 store 内部状态
    router.afterEach((to, from) => {
        store.commit('route/ROUTE_CHANGED', {to, from})
    })
}
```
