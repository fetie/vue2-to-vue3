## 前言

因公司内有vue2的项目需升级到vue3，故记录下升级过程。

下面的参考代码是我拿实际项目删改后得来，仅供参考。

## 环境版本

需用vite重新建一个项目，这里用的是js，没有使用ts。

`vite:2.5.1+vue:3.2.6+vue-router:4.0.11+vuex:4.0.2+element-plus:1.1.0-beta.8`

## 项目升级

### vite

vite.config.js参考代码

```
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
// 如果编辑器提示 path 模块找不到，则可以安装一下 @types/node -> npm i @types/node -D
import { resolve } from 'path'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(__dirname, 'src') // 设置 `@` 指向 `src` 目录
    },
    //因import引入的vue文件都没有加.vue后缀导致报404，所以加了这个配置
    extensions: ['.mjs', '.js', '.ts', '.jsx', '.tsx', '.json', '.vue']
  },
  base: './', // 设置打包路径
  server: {
    port: 3000, // 设置服务启动端口号
    // 是否自动浏览器打开
    open: false,
    // 是否开启https
    https: false,
    cors: true, // 允许跨域

    // 设置代理，根据我们项目实际情况配置
    proxy: {
      '^/api/.*': {
        target: 'https://xxx.com/',
        changeOrigin: true,
        rewrite: path => path.replace(/^\/api/, '')
      },
      //图片的代理
      '^/img-test1|/img-test2': {
        target: 'https://xxx.com',
        changeOrigin: true,
      },
    }
  },
  build: {
    outDir:'dist',
    assetsDir: './assets'
  },
  publicDir:'static'
})

```

这里vite配置重点在`extensions`，他的默认值是`['.mjs', '.js', '.ts', '.jsx', '.tsx', '.json']`，加了`.vue`让他支持vue文件。

官方*不建议忽略自定义导入类型的扩展名（例如：`.vue`），因为它会影响 IDE 和类型支持。*但可能是因为用的js所以并没发现有什么影响。

### vue-router

参考代码

```
import {
  createRouter,
  createWebHashHistory
} from 'vue-router'
import Home from '@/views/home.vue'
import Vuex from '@/views/vuex.vue'

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home
  },
  {
    path: '/vuex',
    name: 'Vuex',
    component: Vuex
  },
  {
    path: '/axios',
    name: 'Axios',
    component: () => import('@/views/axios.vue') // 懒加载组件
    //component: resolve => require(['@/views/axios.vue'], resolve)	//使用require会报错
  }
]

const router = createRouter({
  history: createWebHashHistory(),
  routes
})

export default router

```

> 注意点：使用require的地方要换成import，项目中可能的地方有路由和导出

### vuex

参考代码，改动较少

```
import { createStore } from 'vuex'
import user from './modules/user'
import cameras from './modules/cameras'
import getters from './getters'
import screen from './modules/screen'

const store = createStore({
    getters,
    modules: {
        user,
        cameras,
        screen
    }
})
export default store
```

### element-plus

官方提供的不兼容列表：https://github.com/element-plus/element-plus/issues/162

element升到plus后发现有影响的组件：`menu`、`timePicker`、`tabs`、`tooltip`、`popover`、`dialog`、`table`、`pagination`

#### menu

主要是`el-submenu`更名为`el-sub-menu`，包括`css`

#### timePicker

删除了`picker-options`属性，转为使用`shortcuts`与`disabledDate`，其对应的属性方法也有相应改动；

```
{
  disabledDate(time) {
    return time.getTime() > Date.now()
  },
  shortcuts: [
    {
      text: '最近一周',
      value: () => {
        const end = new Date()
        const start = new Date()
        start.setTime(start.getTime() - 3600 * 1000 * 24 * 6)
        return [start, end]
      },
    },
    {
      text: '最近一个月',
      value: () => {
        const end = new Date()
        const start = new Date()
        start.setTime(start.getTime() - 3600 * 1000 * 24 * 30)
        return [start, end]
      },
    },
    {
      text: '最近三个月',
      value: () => {
        const end = new Date()
        const start = new Date()
        start.setTime(start.getTime() - 3600 * 1000 * 24 * 90)
        return [start, end]
      },
    },
  ],
}
```

日期格式使用`dayjs`的日期格式，如之前是`yyyy-MM-dd`，现在就应该是`YYYY-MM-DD`；

#### tabs

主要是`tab-click`事件返回的值有所变化。`element`要想获取`el-tab-pane`的`name`，直接`.name`即可获得。而`plus`需通过`.props.name`才可获取

#### tooltip

内部有多个`v-if`节点会报错，就如同下面的代码：

```
<el-tooltip
  effect="dark"
  placement="top"
  popper-class="short-popper"
>
  <template v-slot:content>
    <span>
      {{ scope.row.sex }}
    </span>
  </template>
  
<!--这里的i标签都有v-if判断会出现报错-->
  <i
    class="iconfont icon-female_list"
    v-if="scope.row.sex == '女'"
  ></i>
  <i
    class="iconfont icon-male_list"
    v-if="scope.row.sex == '男'"
  ></i>
  <i
    class="iconfont icon-unknown"
    v-if="scope.row.sex != '女' && scope.row.sex != '男'"
  ></i>
  
</el-tooltip>
```

如果在使用其它标签包裹这些含`v-if`的标签则正常

```
<el-tooltip
  effect="dark"
  placement="top"
  popper-class="short-popper"
  v-if="scope.row.sex"
>
  <template v-slot:content>
    <span>
      {{ scope.row.sex }}
    </span>
  </template>
  <!--将多个i标签使用一个标签包裹-->
  <span>
    <i
      class="iconfont icon-female_list"
      v-if="scope.row.sex == '女'"
    ></i>
    <i
      class="iconfont icon-male_list"
      v-if="scope.row.sex == '男'"
    ></i>
    <i
      class="iconfont icon-unknown"
      v-if="scope.row.sex != '女' && scope.row.sex != '男'"
    ></i>
  </span>
</el-tooltip>
```

#### popover

`visible-arrow`换成`show-arrow`

#### dialog

`:visible.sync`换成`v-model`

`class`换成`custom-class`

#### table

`<template slot-scope="{ row }">`换成`<template #default="{ row }">`

#### pagination

`:current-page.sync`换成`v-model:current-page`

### vue

本人升级实际解决的问题：过滤器转方法、slot换v-slot、事件总线换mitt、自定义指令、$set、input事件换为update:value、keep-alive

https://v3.cn.vuejs.org/guide/migration/introduction.html

#### main.js文件

```
//不支持import Vue from 'vue'
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

import ElementPlus from 'element-plus'
import zhCn from 'element-plus/es/locale/lang/zh-cn'

//element也升到了plus，如果重写了index.css覆盖则样式可能会有问题，最好还是使用原生样式
// import '../theme/index.css'

import 'element-plus/dist/index.css'

//如果是在其他js文件内注册组件的则直接在这里注册
// import './icons' // icon
import SvgIcon from '@/views/components/SvgIcon' // svg component


import store from '@/store/index.js'

import htmlToPdf from '@/util/htmlToPdf'

import domain from '@/util/domain'

//vue3中事件总线无法使用，这里使用mitt代替
import mitt from 'mitt';

const emitter = mitt();

const app=createApp(App)

//替代事件总线event bus
app.config.globalProperties.emitter = emitter

//prototype换成config.globalProperties
app.config.globalProperties.$isFollow = domain.isFollow

app.use(htmlToPdf)
app.use(ElementPlus, {
  locale: zhCn,
})
app.component('svg-icon', SvgIcon)


app.use(router)
app.use(store)
app.mount('#app')

```

#### gogocode-cli工具

https://gogocode.io/zh/docs/vue/vue2-to-vue3

`gogocode-cli`可以自动将`vue2`的代码转成`vue3`，但实际使用后还是有很多问题，建议只转`views`目录下的文件，其他目录自己转。

如果项目不大需改动的地方不多感觉就别用这个，自己手动改。但如果项目较大可以试试，他可以帮你做很多重复性的工作

```
gogocode -s ./views -t gogocode-plugin-vue -o ./views-out
```

转化的过程中有可能会失败，失败的原因一般都是在转slot时失败。这种情况可以先自行手动修改转化，之后在执行命令重新进行转化

转化完后除了原有的文件，还会新生成一个gogocodeTransfer.js文件，这个文件主要对$on、$once、$off、$emit、$children单独写了方法

转化出现的问题：

1. 部分`slot`转化出错，有一部分`slot`的值取了该标签内的其他属性如`class`
2. `filters`转化遗漏，在将过滤器转为方法后没有将`template`内的过滤器转化
3. 将`v-model`转为了`v-model:value`。实际上只有小部分需要转化，大部分都不需要加`:value`
4. 代码格式发生变化

可能会有的问题（实际在项目中还没发现有什么问题）：

1. $set直接移除改为使用变量直接赋值
2. 对$on、$once、$off、$emit、$children单独写了方法进行引用和使用

#### import Vue from 'vue'

`import Vue from 'vue'`理论上应该可以用`getCurrentInstance`去代替。但官方提醒*`getCurrentInstance` 只暴露给高阶使用场景，典型的比如在库中。强烈反对在应用的代码中使用 `getCurrentInstance`。请**不要**把它当作在组合式 API 中获取 `this` 的替代方案来使用。*

##### 解决方案

我使用了传入app的方式，如下面的注册全局指令：

main.js

```
import { createApp } from 'vue'
import App from './App.vue'

import { setupGlobDirectives } from '@/directives';

const app=createApp(App)

//注册全局指令
setupGlobDirectives(app)
```

directives/index.js

```
/**
 * Configure and register global directives
 */
import { setupDebounceDirective } from './debounce';

export function setupGlobDirectives(app) {
  setupDebounceDirective(app)
}
```

directives/debounce.js

```
import { debounce } from '@/util'

const debounceDirective={
    beforeMount (el, binding) {
        const executeFunction = debounce(binding.value, 500)
        el.addEventListener('click', executeFunction)
    }
}

export function setupDebounceDirective(app) {
  app.directive('debounce', debounceDirective);
}

export default debounceDirective;
```

但是有些时候不一定要用这种方法解决，在其他js文件内转化`import Vue from 'vue'`可以具体问题具体分析，使用其他方式解决。

#### input事件

`emit('input',val)`要替换为`emit('uplate:value',val)`，并且对应标签的`v-model`换为`v-model:value`

#### 多余slot

有些代码里写了多余的slot，在vue2里可能只是多一个属性不会有错，但在转了vue3的v-slot后代码会报错，需要去除多余的slot

#### keep-alive

改成了下面这样

```
<router-view v-slot="{ Component }">
  <transition>
    <keep-alive include="xxx">
      <component :is="Component" />
    </keep-alive>
  </transition>
</router-view>
```



### 其他

`package.json`里与`vue`无关的包可以不升级，有些包升级了反而可能出错，比如`echarts`。

名字里带有vue的基本都是基于vue开发的包，这部分可能就需要进行更换或升级。

打包过程中发现`<template v-else>`这种写法会报错，需改成其他写法

vue3开源项目参考：https://vue3js.cn/

vite搭建vue3：https://juejin.cn/post/6951649464637636622