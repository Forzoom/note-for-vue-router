### vue-router工作原理

`VueRouter`是工作中一个常用组件，用于渲染匹配到的路由组件。在复杂项目中，有时路由结构会变得相当复杂，了解`vue-router`可以让我们在开发时更加得心应手。

### 如果要自己实现一个路由框架

如果要让你自己实现一个小路由框架或者路由组件，你该怎么做呢？

```html
<div>
    <div v-if="type == 'home'">
        home content
    </div>
    <div v-else-if="type == 'user'">
        user content
    </div>
</div>

<script>
export default {
    data() {
        return {
            type: 'home',
        };
    },
    watch: {
        // 假设这里能够实现对于location的监听
        location() {
            if (location.pathname === '/home') {
                this.type = 'home';
            } else if (location.pathname === '/user') {
                this.type = 'user';
            }
        },
    },
};
</script>
```

上面的例子能够“监听”url的变化，并且根据变化来渲染对应的组件，是否可以说是实现了“路由”功能？可能这和`vue-router`有亿点点差距，让我们来慢慢完善。

### 加一点点细节

开发者在`VueRouter`的配置和使用中，大致上按照下面三个步骤操作：

1. 定义`RouteConfig`：例如路由的`path`和`component`，使用者需要提供自己定义的`RouteConfig`。
2. 匹配`RouteConfig`：即用什么来触发不同`RouteConfig`中的`path`，例如`VueRouter`中的`hash`模式使用`URL`中的`hash`部分进行匹配。
3. 渲染组件：渲染匹配到的`RouteConfig`中的组件。

当然这是`VueRouter`逻辑的简化版本，实际情况要复杂的多。

##### 定义`RouteConfig`

先来看一段示例，下面是一段简单的`VueRouter`构造过程，在其中我们向`routes`字段传递的就是`RouteConfig`列表。

```typescript
const routes: RouteConfig[] = [
    {
        path: '/home',
        component: Home,
    },
    {
        path: '/user',
        component: User,
    },
];
const router = new VueRouter({
    routes,
});
```

好了，现在我们可以稍微写的优雅一些，不再是无穷无尽的if/else。

```html
<script>
export default {
    data() {
        return {
            type: 'home',
        };
    },
    render(h) {
        // 假设这里能够实现对于location的监听
        const route = routes.find((route) => route.path === location.path);
        return h(route.component, ...args);
    },
};
</script>
```

##### 匹配`RouteConfig`

`VueRouter`有两个比较重要的概念，`Matcher`和`History`，其中`Matcher`提供了路由匹配功能。

```javascript
export function createMatcher() {
    const { nameMap, pathMap, pathList } = createRouteMap(routes);

    function addRoutes(routes) {
        createRouteMap(routes, nameMap, pathMap, pathList);
    }
}
```

三个容器就是在创建`Matcher`时构造出来的，并且未来调用`VueRouter.addRoutes`时，所添加的新的`RouteConfig`也会添加到三个容器中。

一般来说，切换路由都使用如下的函数：

```typescript
router.push({
    name: 'home',
});
```

匹配过程有较多的细节，这里使用较为简单的`name`方式作为示例，`Matcher`需要匹配`name="home"`的路由时，由于之前将所有路由存入`nameMap`之中，此时便可直接在`nameMap`中找到对应的`RouteConfig`。

```typescript
const record = nameMap[location.name];
```

##### 渲染组件

当获取到正确的`RouteConfig`之后，直接取出其中的`components`字段中包含的组件，在`render`函数中渲染就完成了。

```typescript
export default {
    name: 'router-view',
    render(h) {
        // 此处省略了其他代码，record由匹配过程中获取
        return h(routeConfig.component, ...args);
    },
};
```

`<router-view>`所提供主要就是`渲染组件`这部分功能，当然不像上面示例中这么简单，其中仍然有许多细节。

至此，我们在三个步骤中都加入了一点细节，在这之后会将更多的细节串联一起。

### `RouteConfig`还不够强大

`RouteConfig`数据一般由使用者提供，数据结构不统一，例如有些有`name`字段，有些则没有。在`VueRouter`中会将`RouteConfig`转换成`RouteRecord`，其中多了一些`RouteConfig`所没有的字段，也对一些字段进行了整理，例如：

1. path: `RouteConfig`所提供的`path`字段五花八门，并且可能出错，所以这里会进行统一。
1. regex: 由`RouteConfig`中的`path`所转换而来的一个正则表达式，`path`应该遵守`express`式的路由，这样才能够正确生成`regex`字段，`regex`在第二步匹配中使用。
1. components: 统一了`RouteConfig`中的`component`和`components`字段，用来支持“命名视图”。
1. parent: 用来记录`路由`和`子路由`之间的父子关系。

其中`components`字段这样设计

```typescript
const record: RouteRecord = {
    // ...
    components: route.components || { default: route.component },
    // ...
}
```

现在`<router-view>`可能看起来像这样

```typescript
export default {
    name: 'router-view',
    props: {
        name: { type: String, default: 'default' },
    },
    render(h) {
        // 此处省略了其他代码
        return h(record.components[this.name], ...args);
    },
};
```

创建`RouteRecord`前，会定义`nameMap`、`pathMap`、`pathList`三个容器，创建后`RouteRecord`将被存储于其中，用于之后的匹配过程。

```typescript
nameMap[record.name] = record;
pathMap[record.path] = record;
pathList.push(record.path);
```

在这一步，我们放弃了`RouteConfig`，整理并使用更加强大的`RouteRecord`，并且实现了“命名视图”的功能：`<router-view>`能够根据`name`来选择需要渲染的组件。

### `RouteRecord`还不够强大

单层路由有时候并不能很好地组织代码，为了实现嵌套路由，需要对匹配过程做一些修改。当匹配到一个`RouteRecord`时，不再直接使用该`RouteRecord`，而是使用`Route`，其和`RouteRecord`的不同主要在于其`matched`字段。

```typescript
// vue-router/src/util/route.js

export function createRoute (
  record: ?RouteRecord,
  location: Location,
  redirectedFrom?: ?Location,
  router?: VueRouter
): Route {
    const route: Route = {
        // ...
        matched: record ? formatMatch(record) : []
    };
}

function formatMatch(record?: RouteRecord): Array<RouteRecord> {
    const res = []
    while (record) {
        res.unshift(record);
        record = record.parent;
    }
    return res
}
```

其实从上面的代码可以看出，`matched`是所匹配到的`RouteRecord`及其所有祖先`RouteRecord`。

至此我们大致上了解到`vue-router`中会使用的一些数据格式，对`vue-router`的逻辑也有了初步的印象。（当然`vue-router`中还有许多细节，后续继续补充。
