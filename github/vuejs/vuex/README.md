# vuejs/vuex

> v3版本适用于Vue2，v4版本适用于Vue3

`Vuex`是Vue专属的状态管理模式和库(既是模式，也是库)，相比于每个组件维护自己的状态，`Vuex`则是将整个应用程序的状态集中到一个地方进行维护(全局)，解决了父子组件进行`event+props`通讯导致的复杂性。同时这个全局的数据也是响应式的。

用`Vuex`管理状态有如下特点：
1. 整个应用程序只有一个`Store`，保存了整个程序所有的**共享状态**。(组件可以有自己的私有状态)
2. `Store`中的`state`和`getters`都是响应式的。(具体原理参考下面的解释) 
3. 需要遵循特定的模式使用。(比如修改状态只能通过`commit mutation`)


`Vuex`既是`pattern`，又是`library`。说它是`pattern`，是因为它参考了[Flux][Flux]，规定了数据的访问和修改的模式，即数据访问通过在计算属性中返回`Store`的状态，修改数据通过在组件方法中提交`mutation`。说它是`library`，是因为它实现了`mapGetters/mapState`等辅助工具。

[Flux]: https://facebook.github.io/flux/


### `Vuex`状态的响应式原理
`Vuex3`借助`Vue`的数据响应式功能，将`Store`中的状态(state)保存到`Vue`实例的`data`内部，变成响应式的，然后再把`getters`代理到`Vue`实例内部的计算属性上，也实现了响应式功能，同时还拥有了缓存功能。

所有的状态修改都放在了`commit`内部进行，非`commit`内部进行的状态修改都会报错，这样就保证了状态修改的可预测性，不会像全局变量那样被莫名其妙的修改而查不出原因。

### `Vuex4`
`Vuex4`只是适配了`Vue3`新的响应式特性，保留了`Vuex3`的API不变。

### 源码实现的一些细节

#### 模块注册过程

模块中的`mutations`和`actions`会被像注册事件一样注册在`store`内部的`_mutations`和`_actions`容器中(这个容器目前是用对象实现的，对应的`handler`以`namespace/type`这样的键进行保存)。

与`addEventListener`注册事件一样，`mutations`和`actions`也是支持同一个类型对应多个`handler`的，因此内部使用一个数组保存的。(虽然支持这个特性，但目前还没有发现对应的场景)

模块的所有`state`会形成一颗树，子模块的`state`会以模块名内嵌在父模块中。
```js
const rootModule = {
    state: {
        a: 1,
        b: 2,
    },
    modules: {
        c: {
            state: {
                d: 3,
            }
        }
    }
}
```
对应的`state`树如下：
```js
const state = {
    a: 1,
    b: 2,
    c: {
        d: 3,
    }
}
```
> 通过这我们也知道，子模块的模块名不能与父模块`state`中的状态名重名，也就是这里的`c`不能出现在父模块的`state`中，否则会有冲突。

这样的状态树最终会通过`Vue#data`变成响应式的，而`store`上面又定义了一个`state` getter，通过这个 getter 可以访问内部`Vue`实例上面保存的响应式的状态树。

模块的`getters`类似于`mutations`和`actions`，会被注册到`store`内部的`_wrappedGetters`容器中，但与之不同的是，`getters`不支持同一个`type`注册多个`handler`，因此内部不是数组。(这个容器目前是用对象实现的，对应的`handler`以`namespace/type`这样的键进行保存)
这个`_wrappedGetters`会在后面调用`resetStoreVM`的时候，注册在内部`Vue`实例的计算属性上面，同时定义了一个`store.getters`来对它进行代理访问。

#### 模块的上下文对象

每个模块都有一个上下文对象，保存在这个模块的`context`属性上面，这个上下文对象封装了四个属性，分别是`dispatch`、`commit`、`getters`和`state`。`context`上面的这些属性都是针对当前模块的，与全局(根模块)的那些属性的区别就是会考虑当前模块的命名空间。比如，你用`context.dispatch`派发一个`action`时，`type`会自动加上该模块的命名空间路径，变成`namespace/type`，然后再去`store`的`_actions`内部去查找`handler`。
另外，`context.dispatch`相比`store.dispatch`多了一个参数`options`，通过这个参数的`root`选项可以控制是否要添加`namespace`到`type`上面，因此`context.dispatch`其实是包含了`store.dispatch`的功能的。
`commit`属性也是类似的。

`getters`和`state`属性则是通过`get`访问器定义的，也就是说在真正访问它的时候才会去求值。`state`属性相对简单，仅仅是访问`store.state`对应路径下面的属性。比如，当前模块的路径是`['foo', 'bar']`，你访问`context.state`时，其实是访问的`store.state.foo.bar`。

`getters`相对复杂，每次访问`context.getters`都会创建一个本地的`gettersProxy`对象，用来处理命名空间问题并最终代理到`store.getters`上面。具体逻辑如下：
```js
Object.defineProperty(context, 'getters', {
    get: () => makeLocalGetters(store, namespace)
})

function makeLocalGetters(store, namespace) {
    if (!store._makeLocalGettersCache[namespace]) {
        const gettersProxy = {}
        const splitPos = namespace.length
        Object.keys(store.getters).forEach(type => {
            // skip if the target getter is not match this namespace
            if (type.slice(0, splitPos) !== namespace) return

            // extract local getter type
            const localType = type.slice(splitPos)

            // Add a port to the getters proxy.
            // Define as getter property because
            // we do not want to evaluate the getters in this time.
            Object.defineProperty(gettersProxy, localType, {
                get: () => store.getters[type],
                enumerable: true
            })
        })
        store._makeLocalGettersCache[namespace] = gettersProxy
    }

    return store._makeLocalGettersCache[namespace]
}
```
比如，当你访问`context.getters.finishedTodos`时，假如这个模块的路径是`['foo', 'bar']`，那么会过滤出`store.getters`上所有以`foo/bar/`开头的`getter`，组成一个临时的代理对象，以这个命名空间为键保存在缓存对象`_makeLocalGettersCache`中。访问这个代理对象的`finishedTodos`属性时，实际访问的是`store.getters['foo/bar/finishedTodos']`。

> 由上可知，只有在第一次访问模块的`getters`属性时，才会去创建这个`getters`代理对象，后续的访问都是使用的缓存技术。只有当`store.getters`发生变化的时候，才会清除这个缓存。也就是`resetStoreVM`中所做的那样。

#### 关于`mapXXX`这些 helper

在使用`Vuex`的时候一直有一个困惑，使用`mapState`、`mapGetters`、`mapMutations`等这些 helper 的时候，应该把返回值放在`computed`里面，还是放在`methods`里面？

`mapState`函数如下：
```js
function mapState(namespace, states) {
    const res = {}
    normalizeMap(states).forEach(({ key, val }) => {
        res[key] = function mappedState () {
            let state = this.$store.state
            let getters = this.$store.getters
            if (namespace) {
                const module = getModuleByNamespace(this.$store, 'mapState', namespace)
                if (!module) {
                    return
                }
                state = module.context.state
                getters = module.context.getters
            }
            return typeof val === 'function'
                ? val.call(this, state, getters)
                : state[val]
        }
        // mark vuex getter for devtools
        res[key].vuex = true
    })
    return res
}
```
可以看到，`mapState`最终返回的是一个全新的对象，这个对象里面每一个属性的值都是一个函数(`mappedState`函数，**注意这个函数是没有参数的**)，比如，你像下面这样调用：
```js
mapState({a: state => state.a, b: state => state.b})
```
那么，它的返回值是这样的：
```js
const res = {
    a() {
        return this.$store.state.a
    },
    b() {
        return this.$store.state.b
    }
}
```
这样的一个对象，既可以放在`computed`里面，也可以放在`methods`里面，效果没有任何区别。
那为什么官方文档都是放在`computed`里面呢？因为访问`state`和`getters`其实跟访问`data`和`computed`没什么区别，并且底层也是用`data`和`computed`实现的，所以就放在了`computd`上面，使用的时候不需要带括号。
而`mapMutations`和`mapActions`分别是对`commit`和`dispatch`的封装，他们都需要参数，所以需要放在`methods`里面。

