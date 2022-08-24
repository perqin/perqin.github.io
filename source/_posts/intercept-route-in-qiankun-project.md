---
title: qiankun微应用项目中的路由拦截
date: 2022-08-23 15:17:34
tags:
---

在应用了qiankun的微应用架构项目中，路由的处理有时候会出问题。我们部门的项目使用了qiankun框架，父应用使用的是Vue 2 + Vue Router，子应用使用的是React + React Router。

## 路由跳转拦截

在父应用中，一个非常常见的场景是路由拦截，也就是在用户有未保存的内容的时候跳转到其他路由，需要阻止用户并弹出提示。

在Vue项目中，我们直接使用Vue Router提供的`beforeRouteLeave`钩子，在其中拦截即可。然而如果是子应用需要拦截，那么就没那么容易了，毕竟子应用里无法干预父应用里的路由组件。因此，我们在子应用中通过props传入的回调通知父应用自己的拦截请求，在父应用中拦截。

```typescript
// Main app (Vue)
export default Vue.extend({
  data() {
    return {
      shouldConfirmBeforeNavigate: false,
    };
  },
  beforeRouteLeave(to, from, next) {
    if (this.shouldConfirmBeforeNavigate) {
      // Show confirm dialog...
      return;
    }
    next();
  },
  mounted() {
    registerMicroApps([
      {
        name: 'child-app',
        props: {
          onEditStateChange: (unsaved: boolean) => {
            this.shouldConfirmBeforeNavigate = unsaved;
          },
        },
        // Other config...
      },
    ]);
  },
  // ...
});
```

## 子应用白屏问题

考虑如下场景：应用使用hash模式路由，路由A对应子应用中的某个页面，路由B对应非子应用的父应用某个页面，用户当前进入A并进入了编辑态，然后点击父应用中的某个`<a>`标签跳转到了路由B。此时父应用的Vue Router会监听`hashchange`事件，阻止页面跳转，并弹出确认弹窗。然而此时我们会发现容器内的子应用区域变成了白屏。

出现这个问题的原因在于，子应用中的React Router也监听了`hashchange`事件，在用户点击`<a>`标签的时候，虽然父应用中的Vue Router进行了拦截，但浏览器的地址栏仍然已经变成了路由B的路径，对应的`hashchange`事件也已经被发出并被子应用监听了。子应用中没有路由B对应的页面，因此会渲染成白屏。

## 解决路由拦截(`<router-link>`)

要解决上述问题，就需要在拦截路由的时候不要发出`hashchange`事件，而最快的办法就是使用Vue Router提供的`<router-link>`组件。使用该组件后，用户点击时如果需要拦截路由，Vue Router会阻止URL中hash的变更和`hashchange`事件的触发。那么Vue Router是怎么实现的呢？我们来看看`<router-link>`组件的源码（以3.4.3版本为例，代码摘录有省略）：

```javascript
// src/components/link.js

export default {
  name: 'RouterLink',
  props: {
    to: {
      type: toTypes,
      required: true
    },
    tag: {
      type: String,
      default: 'a'
    },
    // Omitted...
  },
  render (h: Function) {
    const router = this.$router
    const current = this.$route
    const { location, route, href } = router.resolve(
      this.to,
      current,
      this.append
    )

    // Omitted

    const handler = e => {
      if (guardEvent(e)) {
        if (this.replace) {
          router.replace(location, noop)
        } else {
          router.push(location, noop)
        }
      }
    }

    const on = { click: guardEvent }
    if (Array.isArray(this.event)) {
      this.event.forEach(e => {
        on[e] = handler
      })
    } else {
      on[this.event] = handler
    }

    const data: any = { class: classes }

    // Omitted...

    if (this.tag === 'a') {
      data.on = on
      data.attrs = { href, 'aria-current': ariaCurrentValue }
    } else {
      // Omitted...
    }

    return h(this.tag, data, this.$slots.default)
  }
}

function guardEvent (e) {
  // don't redirect with control keys
  if (e.metaKey || e.altKey || e.ctrlKey || e.shiftKey) return
  // don't redirect when preventDefault called
  if (e.defaultPrevented) return
  // don't redirect on right click
  if (e.button !== undefined && e.button !== 0) return
  // don't redirect if `target="_blank"`
  if (e.currentTarget && e.currentTarget.getAttribute) {
    const target = e.currentTarget.getAttribute('target')
    if (/\b_blank\b/i.test(target)) return
  }
  // this may be a Weex event which doesn't have this method
  if (e.preventDefault) {
    e.preventDefault()
  }
  return true
}
```

可见默认情况下`<router-link>`渲染出来的也是一个`<a>`标签，但是这个`<a>`标签被点击时，触发的默认行为在`guardEvent`中被`preventDefault`了，正是这个调用阻止了URL的改变和`hashchange`事件的触发。后续的逻辑就是调用`VueRouter`实例的`replace`或`push`函数手动跳转了。

## 子应用内路由跳转问题

上述方法带来了一个新的问题：在子应用内无法跳转路由了。

考虑这样的场景：父应用上有两个`<router-link>`，分别对应子应用内的路由A和B。当前用户在路由A，用户点击`<router-link>`跳转到了路由B。此时我们会发现，子应用内仍然是路由A的页面！

这个问题在qiankun已经有对应的[issue](https://github.com/umijs/qiankun/issues/1199)，其原因为：当使用Vue Router提供的API来跳转路由时（这也是`<router-link>`内的实现方式），Vue Router会判断当前浏览器是否支持HTML5 Browser History API，如果支持，那么就不会通过修改Location href的方式修改hash，而是直接调用对应的`pushState`、`replaceState`函数来跳转路由，而这种跳转方式虽然会修改地址栏URL，但是不会触发`hashchange`事件！

原issue中也有人给出了[解决方案](https://github.com/umijs/qiankun/issues/1199#issuecomment-818740774)：

> 目前可用的解决方案：
>
> 1. 将 vue 主应用中的 Link 超链方式替换成原生的 a 标签，从而触发浏览器的 hashchange
> 2. 主应用手动监听路由变更，同时手动触发 hashchange 事件
> 3. 主应用跟子应用都改用 browser history 模式

显然，根据前面的描述，我们不可能再用回原生的`<a>`标签；而我们的项目目前也无法突然迁移到Browser History API，因此只有第二条路可以走，实现如下：

```javascript
// SiderMenu.vue
export default {
  watch: {
    $route(route, oldRoute) {
      if (route.fullPath !== oldRoute?.fullPath) {
        const { href } = window.location;
        const baseUrl = href.indexOf('#') !== -1 ? href.substring(0, href.indexOf('#')) : href;
        const oldURL = `${baseUrl}#${oldRoute?.fullPath || ''}`;
        const newURL = `${baseUrl}#${route.fullPath || ''}`;
        window.dispatchEvent(new HashChangeEvent('hashchange', { oldURL, newURL }));
      }
    },
  },
};
```

## EXTRA：使用slot的场景

到这里，qiankun场景下的路由拦截问题已经解决完毕，但对我手里这个项目来说，到上面这一步还没有结束。

我们有一个路由页面，`<router-link>`上使用的路由是这样的：

```html
<router-link
  :to="{
    name: 'route-books',
    params: { shelfId: '0' },
  }"
>Books</router-link>
```

可以看到点击时`shelfId`参数会设置为`0`，而实际上，该路由对应的页面组件在`created`的时候就会执行初始化逻辑，获取到当前用户的shelfId，然后修改当前路由的`shelfId`参数。如果用户停留在这个页面的时候再点击一次链接会怎样？会把URL里的`shelfId`参数重新修改为`0`！根据Vue Router的文档，[路由仅有参数变化的时候，组件是不会销毁重建的](https://v3.router.vuejs.org/guide/essentials/dynamic-matching.html#reacting-to-params-changes)。这就意味着此时`created`中的初始化逻辑不会被重新执行，而该组件中又有很多其它地方的逻辑会直接从`$route.params.shelfId`中获取shelfId，这就会带来问题。

其中一种解决方式当然是按照上述文档中推荐的，去监听参数变更、重新调用接口拉取用户的shelfId，但用户只是点击了一下当前路由就触发一次请求未免有些浪费，因此我们选择的是直接屏蔽重复的路由跳转。

如何屏蔽重复的路由跳转呢？

如果没有使用scoped slot，那么直接在`<router-link>`组件内部的`<a>`标签中增加`click`事件监听并`preventDefault`即可，`<router-link>`组件会自动追加他自带的`click`事件到后面。`<router-link>`的逻辑如下：

```javascript
// src/components/link.js
export default {
  render () {
    // Omitted...
    if (this.tag === 'a') {
      data.on = on
      data.attrs = { href, 'aria-current': ariaCurrentValue }
    } else {
      // find the first <a> child and apply listener and href
      const a = findAnchor(this.$slots.default)
      if (a) {
        // in case the <a> is a static node
        a.isStatic = false
        const aData = (a.data = extend({}, a.data))
        aData.on = aData.on || {}
        // transform existing events in both objects into arrays so we can push later
        for (const event in aData.on) {
          const handler = aData.on[event]
          if (event in on) {
            aData.on[event] = Array.isArray(handler) ? handler : [handler]
          }
        }
        // append new listeners for router-link
        for (const event in on) {
          if (event in aData.on) {
            // on[event] is always a function
            aData.on[event].push(on[event])
          } else {
            aData.on[event] = handler
          }
        }

        const aAttrs = (a.data.attrs = extend({}, a.data.attrs))
        aAttrs.href = href
        aAttrs['aria-current'] = ariaCurrentValue
      } else {
        // doesn't have <a> child, apply listener to self
        data.on = on
      }
    }

    return h(this.tag, data, this.$slots.default)
  }
}
```

因此我们这样写就可以：

```html
<router-link
  :to="{
    name: 'route-books',
    params: { shelfId: '0' },
  }"
>
  <a @click="(e) => $route.name === 'route-books' && e.preventDefault()">Books</a>
</router-link>
```

而如果使用了scoped slot，那么就需要手动处理点击逻辑了：

```html
<router-link
  :to="{
    name: 'route-books',
    params: { shelfId: '0' },
  }"
  v-slot="{ href, isActive, isExactActive, route, navigate }"
>
  <a
    :href="href"
    :class="{
      'router-link-active': isActive || $route.name == 'route-books',
      'router-link-exact-active': isExactActive,
    }"
    @click="(e) => {
      if ('route-books' !== this.$route.name) {
        navigate(e);
      }
      e.preventDefault();
    }"
  >Books</a>
</router-link>
```

这里需要注意，点击事件中：
- 必须调用`e.preventDefault()`，因为我们的`click`事件是放在`<a>`标签的，如前面解释的，需要用这种方式屏蔽`hashchange`事件；
- 调用`e.preventDefault()`必须在`navigate`调用之后，否则在`guardEvent`中会被拦截掉。
