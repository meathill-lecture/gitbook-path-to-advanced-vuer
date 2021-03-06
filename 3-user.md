用全家桶认证用户
==============

对 Vue 来说，全家桶就像天赋系统——刚开始的时候，你注意不到它的存在；但是当项目规模大到一定程度，合理使用全家桶会让你事半功倍。

平时浏览开发论坛的时候，经常看到有同学提问：如何做用户身份校验？其实这个时候，最合适的方案就是组合使用 Vuex + Vue-router。

Vuex
--------

首先，推荐大家读一下官方文档：[状态管理](https://cn.vuejs.org/v2/guide/state-management.html)。

好，现在假设你已经读完了。简单来说，Vuex 负责保存数据，并且把数据输出到其它 Vue 组件中，供它们使用。使用 Vuex 有以下优势：

1. 数据统一，可以避免在每个组件里都维护一份数据
2. Vuex 支持响应式
3. 组件直接从 Vuex 获取数据，而不是从父组件接收，组件之间解耦，也方便我们对组件进行独立测试

使用 Vuex 的方式很简单，大家直接看 [Vuex 官方文档](https://vuex.vuejs.org/) 即可。

Vue-Router
----------

还请先读官方文档：[路由](https://cn.vuejs.org/v2/guide/routing.html)。

Vue-Router 的作用在于帮我们组织单页应用。这个应该不需要太多解释。

用户认证系统的设计
---------------

这里我们拿后台产品做例子。一般来说，后台产品对用户的要求有：

1. 用户必须登录
2. 用户操作时先鉴权
3. 鉴权失败跳转到登录页

首先，用户登录保存 session 的动作，直接用 HTTP header 的 `set-cookie` 就行。很多同学这个地方还费劲吧啦的用 localStorage，然后用 axios 拦截器给每个请求加 token，其实完全没有必要，安全性也没有任何优势。

我们可以把登录过程直接放到 Vuex.Store 里，接着定义路由：

```js
// store.js
export default new Vuex.Store({
  state: {
    user: null,
  },
  mutations: {
    setUser(state, user) {
      state.user = user;
    },
  },
  actions: {
    login({commit}, data) {
      service.login(data)
        .then(user => {
          commit('setUser', user);
          return user;
        });
    },
  },
});

// router.js
export default new VueRouter({
  routes: [
    {
      path: '/',
      name: 'home',
    },
    {
      path: '/login',
      name: 'login',
    },
  ],
});
```

应用初始化的时候，先检查当前用户的登录状态——其实这个检查最好做成心跳脚本，不断检查，方便在用户 session 失效的时候跳出，避免错误。

```js
// app.vue
export default {
  beforeMount() {
    checkUser()
      .then(() => {
        // 已登录
      })
      .catch(() => {
        // 未登录，跳到登录页
        this.$router.replace({
          name: 'login',
        });
      });
  },
}
```

然后在登录页面，登录成功之后跳转到默认页面：

```js
// login.vue
export default {
  methods: {
    doLogin() {
      this.$store.dispatch('login', this.formData)
        .then(() => {
          // 登录成功，跳转到首页。其实这里可以记录下跳转前的 pathname，登录成功跳过去，比始终跳首页要好
          this.$router.replace({
            name: 'home',
          });
        })
        .catch(error => {
          // 登录失败，处理一下
        });
    },
  },
};
```

其它页面，需要用到用户信息，直接用 `Vuex.mapState` 就好：

```vue
<template lang="pug">
Welcome, {{user.name}}
</template>

<script>
import {mapState} from 'vuex';

export default {
  computed: {
    ...mapState(['user']),
  },
}
</script>
```

至于鉴权，因为 sessionId 写在 cookie 里，本身就是 HTTP 请求的一部分，不需要特别处理。不过我们可以拦截请求，如果触发 401，就跳到登录页。这里以 axios 为例：

```js
const http = axios.create();
http.interceptors.response.use(response => {
  return response;
}, error => {
  const {status} = error;
  if (statue === 401) {
    // 因为 router 路由绑在 vue 根组件上，axios 不好触及，这里就可以用到后面的“全局事件总线”
    bus.emit('401', error);
  }
  return Promise.reject(error);
});
```
