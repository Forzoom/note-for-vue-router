
前篇中介绍了`vue-router`中出现的一些数据结构。本篇介绍`vue-router`的`install`过程和`<router-view>`的部分逻辑。

### 注册`vue-router`

以下是`vue-router`注册，即`Vue.use(VueRouter)`时执行的主要代码。其中还有一部分内容与本章无关，略过不讲。

```typescript
// vue-router/src/install.js 省略了部分代码

export function install (Vue) {
    // ...

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

其中使用`Vue.mixin`混入了`beforeCreate`钩子函数，混入的钩子将在组件的同名钩子之前调用。`beforeCreate`钩子中让所有组件在都在`this`上定义好`_router`和`_routerRoot`。另外我们需要知道的是`this.$options`是`Vue组件`构造时所传递的options信息。例如下面代码中所示：

```typescript
const $options = {
    el: '#app',
    router,
};
new Vue($options);
```

可以看出，如果当前执行`beforeCreate`是根组件的话:
1. 对于我们一般`mount`在`#app`上的根组件，`this._routeRoot`会指向该组件本身。
1. 将`_router`也写入到根组件，值是`VueRouter`对象本身。
1. 将`_route`指向`_router.history.current`，值得一提的是，`_route`中的数据就是上篇中提到的`Route`。

其他情况下，不是根组件的话:
1. 将`this._routeRoot`指向父组件中的`_routeRoot`

上面的代码会让所有组件上的`_routerRoot`变量都代表的是`根组件`，这将在下一章节中使用到。

在全局混入之后，调用`Object.defineProperty`定义了两个开发中常用的变量`$router`和`$route`，以及注册了`<router-view>`和`<router-link>`两个组件。

### <router-view>出场

`<router-view>`主要负责渲染匹配到的路由组件，在上篇中，为了实现嵌套路由，在匹配`RouteRecord`后不再直接使用，而是使用`Route`，其中matched字段包含了匹配到的`RouteRecord`和其所有祖先`RouteRecord`。

此时，如果页面当中有多个层级的`<router-view>`，那么每个`<router-view>`必须匹配自己层级所对应的组件。

那么首先，每个`<router-view>`应该知道自己所在的层级。看下面的源码内容：

```typescript
// vue-router/src/components/view.js 其中省略部分代码

export default {
    // ...
    render(h, { parent }) {
        // 标记自己是一个`router-view`
        data.routerView = true;

        // ...

        // parent: 对于父组件的引用
        let depth = 0;
        let inactive = false;
        // 当还没有寻找到根组件时，可以不断循环
        while (parent && parent._routerRoot !== parent) {
            // 确定层级
            if (parent.$vnode && parent.$vnode.data.routerView) {
                depth++;
            }
            // 在一个vue组件中，$parent是对于父组件的引用
            parent = parent.$parent;
        }
        data.routerViewDepth = depth;
    },
}
```

每个`router-view`都会顺着DOM树结构向上寻找，另外每个`router-view`都会对自己进行标记，这样做，在寻找过程中，就能够确定自己所在的层级。

在`router-view`确定自己的层级之后，就可以进行渲染了。可以看下面的源码

```typescript
// vue-router/src/components/view.js 其中省略部分代码

export default {
    // ...
    render(h, { props, parent, children }) {
        // 其中props是包含所有的prop的对象
        const name = props.name
        // ...

        // 会获得RouteRecord
        const matched = route.matched[depth];
        // 渲染空节点
        if (!matched) {
            return h();
        }

        // 这里是组件选项对象
        const component = matched.components[name];

        // 最终的渲染
        return h(component, data, children);
    },
}
```

至此便是渲染过程的简化流程，当然`vue-router`中还有许多内容，之后再慢慢更新。
