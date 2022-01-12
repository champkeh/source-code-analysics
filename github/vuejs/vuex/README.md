# vuejs/vuex

> v3版本适用于Vue2，v4版本适用于Vue3

`Vuex`是Vue专属的状态管理模式和库(既是模式，也是库)，相比于每个组件维护自己的状态，`Vuex`则是将整个应用程序的状态集中到一个地方进行维护，解决了父子组件进行`event+props`通讯导致的复杂性。同时这个全局的数据也是响应式的

总结一下，`Vuex`有如下特点：
- State Management Pattern + Library
- Single Source of Truth
- State is Reactive

`Vuex`既是协议，又是实现。说它是协议，是因为它参考了`Flux`协议，规定了数据的访问和修改的约定，即数据访问通过在计算属性中返回`Store`的状态，修改数据通过在组件方法中提交`mutation`。说它是实现，是因为它实现了`mapGetters/mapState`等辅助工具。

`Vuex3`借助`Vue`的响应式功能，将`Store`中的状态变成响应式的，然后再通过代理`Store`中的`getters`属性，将最终的`getters`访问代理到对应的`handler`，同时这些`getters`又变成了`Vue`的计算属性，拥有了缓存功能。

所有的状态修改都放在了`commit`内部进行，非`commit`内部进行的状态修改都会报错，这样就保证了状态修改的可预测性，不会像全局变量那样被莫名其妙的修改而查不出原因。
