
前篇中介绍了`vue-router`中出现的一些数据结构。本篇介绍`vue-router`的`install`过程和`<router-view>`的部分逻辑。

### 注册`vue-router`

以下是`vue-router`注册，即`Vue.use(VueRouter)`时执行的主要代码。

```typescript
// vue-router/src/install.js 省略了部分代码

export function install (Vue) {
    Vue.mixin({
        beforeCreate () {
            if (isDef(this.$options.router)) {
                this._routerRoot = this
                this._router = this.$options.router
                this._router.init(this)
                Vue.util.defineReactive(this, '_route', this._router.history.current)
            } else {
                this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
            }
        },
    })

    Object.defineProperty(Vue.prototype, '$router', {
        get () { return this._routerRoot._router }
    })

    Object.defineProperty(Vue.prototype, '$route', {
        get () { return this._routerRoot._route }
    })

    Vue.component('RouterView', View)
    Vue.component('RouterLink', Link)
}
```

可以看到其中使用`Vue.mixin`来让所有的组件在创建前都调用这个生命周期函数。另外我们需要知道的是`this.$options`是`Vue组件`构造时所传递的options信息。例如下面代码中所示：

```typescript
const $options = {
    el: '#app',
    router,
};
new Vue($options);
```

可以看出，如果是根组件
1. 对于我们一般`mount`在`#app`上的根组件，`this._routeRoot`会指向该组件本身。
1. 将`_router`也写入到根组件，值是`VueRouter`对象本身。
1. 将`_route`指向`_router.history.current`，值得一提的是，`_route`中的数据就是上篇中提到的`Route`。
其他情况下，如果不是根组件
1. 将`this._routeRoot`指向父组件中的`_routeRoot`

在全局混入之后，调用`Object.defineProperty`定义了两个开发中常用的变量`$router`和`$route`，以及注册了`<router-view>`和`<router-link>`两个组件。

其中还有一部分内容即`this._router.init(this)`暂时略过。

### <router-view>出场

`<router-view>`主要负责渲染匹配到的路由组件，在上篇中，为了实现嵌套路由，我们将

```typescript
// vue-router/src/components/view.js 其中省略部分代码

export default {
    // ...
    render(...args) {
        data.routerView = true;

        // ...
        const route = parent.$route;
        const cache = parent._routerViewCache || (parent._routerViewCache = {});

        let depth = 0;
    },
}
```
