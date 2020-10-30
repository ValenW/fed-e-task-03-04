# SSR

使用`vue-server-renderer`模块进行服务端渲染, 最好**在header和HTML meta中都设置编码**

```js
const renderer = require('vue-server-renderer').createRenderer()
const express = require('express')

const server = express()

server.get('/', (req, res) => {
  const app = new Vue({ ... })

  renderer.renderToString(app, (err, html) => {
    if (err) { return res.status(500).end('Interal Server Error') }
    res.setHeader('Context-Type', 'text/html; charset=utf8')
    res.end(`
			<!DOCTYPE html>
      <html lang="en">
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>My vue</title>
      </head>
			<body>${html}</body>
			</html>
		`)
  })
})
```

## 编译模板

创建serverRenderer实例时可以传入template配置:

```js
const renderer = require('vue-server-renderer').createRenderer({
  template: fs.readFileSync('./index.template.html', 'utf-8')
})
```

可以在`renderer.renderToString`第二个参数中为template传入外部数据:

- 双大括号渲染为html
- 三大括号直接渲染, 内容为html时使用

## 构建配置

服务端渲染只会回传HTML字符串给客户端, 对于那些需要交互的功能, 则必须要由客户端js进行处理, 如`v-model`, `click`

所以除了服务端入口, 还需要一个客户端入口, 来在获取到服务端HTML之后进行静态标签激活和接管, 形成动态页面

![架构](08vue ssr ssg.assets/786a415a-5fee-11e6-9c11-45a2cfdf085c.png)

### webpack[源码结构](https://ssr.vuejs.org/zh/guide/structure.html#%E4%BD%BF%E7%94%A8-webpack-%E7%9A%84%E6%BA%90%E7%A0%81%E7%BB%93%E6%9E%84)

App.vue, app.js, entry-client.js, entry-server.js

```js
// app.js
import Vue from 'vue'
import App from './App.vue'

// factory function to create new app, router, store instance
export function createApp() {
  const app = new Vue({
    render: h => h(App)
  })
  return { app }
}

// entry-client.js
import { createApp } from './app'
const { app } = createApp()
app.$mount('#app')

// entry-server.js
import { createApp } from './app'
export default context => {
  const { app } = createApp()
  // router config, data pre-fetch, ect in server side
  return app
}
```

### 构建配置

#### 安装依赖

```shell
npm i vue vue-server-renderer express cross-env
npm i -D webpack webpack-cli webpack-merge webpack-node-externals @babel/core @babel/plugin-transform-runtime @babel/preset-env babel-loader css-loader urlloader file-loader rimraf vue-loader vue-template-compiler friendly-errorswebpack-plugin
```

- `cross-env`用于通过npm scripts设置跨平台环境变量
- `webpack-node-externals` 排除 webpack 中的 Node 模块
- `rimraf`基于 Node 封装的一个跨平台 rm -rf 工具
- `friendly-errors-webpack-plugin`友好的webpack错误提示

#### 配置文件以及打包命令

```js
// ./build/webpack.base.config.js
/**
 * 公共配置
 */
const VueLoaderPlugin = require("vue-loader/lib/plugin");
const path = require("path");
const FriendlyErrorsWebpackPlugin = require("friendly-errors-webpack-plugin");
const resolve = (file) => path.resolve(__dirname, file);

const isProd = process.env.NODE_ENV === "production";

module.exports = {
  mode: isProd ? "production" : "development",
  output: {
    path: resolve("../dist/"),
    publicPath: "/dist/",
    filename: "[name].[chunkhash].js",
  },
  resolve: {
    alias: {
      // 路径别名，@ 指向 src
      "@": resolve("../src/"),
    },
    // 可以省略的扩展名
    // 当省略扩展名的时候，按照从前往后的顺序依次解析
    extensions: [".js", ".vue", ".json"],
  },
  devtool: isProd ? "source-map" : "cheap-module-eval-source-map",
  module: {
    rules: [
      // 处理图片资源
      {
        test: /\.(png|jpg|gif)$/i,
        use: [
          {
            loader: "url-loader",
            options: {
              limit: 8192,
            },
          },
        ],
      },
      // 处理字体资源
      {
        test: /\.(woff|woff2|eot|ttf|otf)$/,
        use: ["file-loader"],
      },
      // 处理 .vue 资源
      {
        test: /\.vue$/,
        loader: "vue-loader",
      },
      // 处理 CSS 资源
      // 它会应用到普通的 `.css` 文件
      // 以及 `.vue` 文件中的 `<style>` 块
      {
        test: /\.css$/,
        use: ["vue-style-loader", "css-loader"],
      },
      // CSS 预处理器，参考：https://vue-loader.vuejs.org/zh/guide/preprocessors.html
      // 例如处理 Less 资源
      // {
      // test: /\.less$/,
      // use: [
      // 'vue-style-loader',
      // 'css-loader',
      // 'less-loader'
      // ]
      // },
    ],
  },
  plugins: [new VueLoaderPlugin(), new FriendlyErrorsWebpackPlugin()],
};
```

```js
// ./build/webpack.client.config.js
/**
 * 客户端打包配置
 */
const { merge } = require("webpack-merge");
const baseConfig = require("./webpack.base.config.js");
const VueSSRClientPlugin = require("vue-server-renderer/client-plugin");

module.exports = merge(baseConfig, {
  entry: {
    // 相对执行打包命令时的路径
    app: "./src/entry-client.js",
  },

  module: {
    rules: [
      // ES6 转 ES5
      {
        test: /\.m?js$/,
        exclude: /(node_modules|bower_components)/,
        use: {
          loader: "babel-loader",
          options: {
            presets: ["@babel/preset-env"],
            cacheDirectory: true,
            plugins: ["@babel/plugin-transform-runtime"],
          },
        },
      },
    ],
  },
  // 重要信息：这将 webpack 运行时分离到一个引导 chunk 中，
  // 以便可以在之后正确注入异步 chunk。
  optimization: {
    splitChunks: {
      name: "manifest",
      minChunks: Infinity,
    },
  },
  plugins: [
    // 此插件在输出目录中生成 `vue-ssr-client-manifest.json`
    // 描述客户端打包结果中的依赖, 需要加载的模块信息
    new VueSSRClientPlugin(),
  ],
});
```

```js
// ./build/webpack.server.config.js
/**
 * 服务端打包配置
 */
const { merge } = require("webpack-merge");
const nodeExternals = require("webpack-node-externals");
const baseConfig = require("./webpack.base.config.js");
const VueSSRServerPlugin = require("vue-server-renderer/server-plugin");

module.exports = merge(baseConfig, {
  // 将 entry 指向应用程序的 server entry 文件
  entry: "./src/entry-server.js",

  // 这允许 webpack 以 Node 适用方式处理模块加载
  // 并且还会在编译 Vue 组件时，
  // 告知 `vue-loader` 输送面向服务器代码(server-oriented code)。
  target: "node",

  output: {
    filename: "server-bundle.js",
    // 此处告知 server bundle 使用 Node 风格导出模块(Node-style exports)
    libraryTarget: "commonjs2",
  },

  // 不打包 node_modules 第三方包，而是保留 require 方式直接加载
  externals: [
    nodeExternals({
      // 白名单中的资源依然正常打包
      allowlist: [/\.css$/],
    }),
  ],

  plugins: [
    // 这是将服务器的整个输出构建为单个 JSON 文件的插件。
    // 默认文件名为 `vue-ssr-server-bundle.json`
    new VueSSRServerPlugin(),
  ],
});
```

```shell
打包命令
"scripts": {
  "build:client": "cross-env NODE_ENV=production webpack --config build/webpack.client.config.js",
  "build:server": "cross-env NODE_ENV=production webpack --config build/webpack.server.config.js",
	"build": "rimraf dist && yarn build:client && yarn build:server"
},
```

其中HTML会引用client script, 所以需要使用`express.static`插件将dist目录映射为静态目录, 这样才可以被前端访问

## 渲染流程

### 服务端

#### server-bundle.json

```json
{
  entry: "server-bundle.js",
  files: {
    // 将bundle文件代码直接以字符串形式导出到json文件中
    // 内容是entry-server.js代码, 得到vue实例
    server-bundle.js: "module.exports=..."
  },
  // source map信息, 用于debug
  maps: {
    server-bundle.js: {
    	version: 3,
    	sources: [...],
  		...
}}}
```

server-bundle.js内容就是`entry-server.js`导出的代码, 得到vue实例

在`createBundleRenderer`函数接收serverBundle, 加载并执行`entry-server.js`

得到vue实例, 渲染成HTML, 然后注入到template当中, 发送到客户端

### 客户端

#### clientManifest

是客户端打包资源的清单

```json
{
  // 客户端资源入口
  publicPath: "/dist/",
  // 客户端所有用到的资源文件
  all: [
    "app.hash.js",
    "app.hash.js.map"
  ],
  initial: [
    // 客户端渲染时会把initial中的文件注入到HTML中
    "app.hash.js"
  ],
  // 异步资源, 如异步组件, 异步js模块
  async: [],
  // 针对每个原始模块, 如App.vue, app.js 做的依赖信息说明
  modules: {
    // 每个模块有对应的hash值, 数组内容是其依赖的文件, 用其在all数组中的索引表示
    module_hash: [0, 1],
    // 当vue加载一个模块时, 通过这些信息知道其依赖于哪些资源, 之后vue就会加载这些资源
  }
}
```

在`createBundleRenderer`中传入clientManifest, 服务端渲染时就会在HTML中注入initial中的文件

#### [客户端激活](https://ssr.vuejs.org/zh/guide/hydration.html)

客户端渲染的HTML根节点上有个特殊属性`data-server-rendered="true"`, 客户端vue通过这个属性得知HTML已经被渲染好, 只需要激活就好了

可以通过`hydrating`参数强制使用激活模式:`app.$mount('#app', true)`

- 开发模式下vue会推断客户端生成的dom tree和服务端的是否匹配, 如果不匹配会丢弃并重新渲染
- 生产模式下会跳过检测

##### 一些坑

- 浏览器可能会更改一些HTML结构, 比如不标准的table结构, 没有`thead`, `tbody`, 则浏览器会自动加上, 这会导致vue丢弃服务端渲染结果

## 开发模式构建

### 自动化构建

关键在于`createBundleRenderer`创建renderer过程

- 依赖于多个文件, 变动后需要重新创建renderer
- 重新创建后重启服务, 刷新页面

#### 区分生产模式和开发模式

```
"start": "cross-env NODE_ENV=production node server.js",
"dev": "node server.js"
```

在`server.js`中根据`process.env.NODE_RNV === 'production'`判断是否是生产模式

生产模式直接引用文件创建, 开发模式需要监视打包构建, 然后重新生成renderer

开发模式需要有个promise确定renderer的生成状态

```js
const isProd = process.env.NODE_RNV === 'production';

let renderer;
let rendererReady;
if (isProd) {
  const serverBundle = require("./dist/vue-ssr-server-bundle.json");
  const clientManifest = require("./dist/vue-ssr-client-manifest.json");
  const template = fs.readFileSync("./index.template.html", "utf-8");
  // 生成一个渲染器
  renderer = createBundleRenderer(serverBundle, {
    template,
    clientManifest,
  });
  rendererReady = Promise.resolve();
} else {
  // 开发模式
  // 监视打包构建, 完成后生成Renderer渲染器
}

// 路由匹配时也要考虑模式, 开发模式下renderer不一定生成
server.get("/", isProd ? render : async (req, res) => {
  // 等待有了renderer之后调用render进行渲染
  await rendererReady;
  render(req, res)
});
```

#### 开发模式renderer

```js
setupDeverver(serverInstance, (...resources) => { updateRenderer }): Promise<rendererIsReady>
```

```js
// build/setup-server.js
module.exports = (server, callback) => {
  let ready
  const onReady = new Promise(r => ready = r)
  
  let templdate
  let serverBundle
  let clientManifest
  
  const update = () => {
    if (template && serverBuindle && clientManifest) {
      ready()
      callback(templdate, serverBundle, clientManifest)
    }
  }
  
  // 监视文件, 任意文件更新调用update更新renderer
  
  return onReady
}
```

##### 监视template

```js
// 使用chokidar监听template文件变化
const chokidar = require("chokidar");
const templatePath = resolve("../index.template.html");
template = fs.readFileSync(templatePath, "utf-8");
chokidar.watch(templatePath).on("change", () => {
  template = fs.readFileSync(templatePath, "utf-8");
  console.log("template update", template);
  update();
});
```

##### 监视serverConfig

```js
const webpack = require("webpack");
const serverConfig = require("./webpack.server.config");
const serverCompiler = webpack(serverConfig);

serverCompiler.watch({}, (err, stats) => {
  if (err) throw err; // webpack err, like config err
  if (stats.hasErrors()) return; // our code err
  serverBundle = JSON.parse(
    fs.readFileSync(resolve("../dist/vue-ssr-server-bundle.json"), "utf-8")
  ); // not use require, for it has cache for old content

  console.log("serverBundle update", serverBundle);

  update();
});
```

### 注意

**html template中不能出现以双大括号开头的注释**, 如 

```html
<!-- {{ }}会将内容HTML转码后替换 -->
<!-- {{{ }}}不转码为HTML, 直接字符串替换 -->
```

这会**导致模板渲染报错, 然后`createBundleRenderer`阻塞, 一直不能返回**

###### 使用内存加速构建更新

使用webpack官方推荐的内存文件系统`memfs`, 但需要比较复杂的[配置](https://webpack.js.org/api/node/#custom-file-systems)

使用中间件`yarn add webpack-dev-middleware -D`

```js
const serverMiddleware = devMiddleware(serverCompiler, {
  logLevel: "silent",
});
// 使用serverCompiler自带事件钩子监听
serverCompiler.hooks.done.tap("update server", () => {
  serverBundle = JSON.parse(
    serverMiddleware.fileSystem.readFileSync(
      resolve("../dist/vue-ssr-server-bundle.json", "utf-8")
    )
  );
  
  console.log("serverBundle update", serverBundle);

  update();
});
```

##### 监视客户端更新

因为客户端编译到内存, 没有dist目录, 所以需要在server中使用client middleware

```js
const clientConfig = require("./webpack.client.config");
const clientCompiler = webpack(clientConfig);
const clientMiddleware = devMiddleware(clientCompiler, {
  // 客户端和服务端多出一个publicPath的配置
  publicPath: clientConfig.output.publicPath,
  logLevel: "silent",
});
clientCompiler.hooks.done.tap("update client", () => {
  clientManifest = JSON.parse(
    clientMiddleware.fileSystem.readFileSync(
      resolve("../dist/vue-ssr-client-manifest.json", "utf-8")
    )
  );

  console.log("client update", clientManifest);
  update();
});

// 将clientMiddleware挂载到express服务中, 提供对其托管的内存数据访问
server.use(clientMiddleware)
```

### 热更新

使用[webpack-hot-middleware](https://github.com/webpack-contrib/webpack-hot-middleware#installation--usage)

```js
// yarn add -D webpack-hot-middleware
clientConfig.plugins.push(new webpack.HotModuleReplacementPlugin());
clientConfig.entry.app = [
  // 和服务端交互处理热更新的客户端脚本
  "webpack-hot-middleware/client",
  clientConfig.entry.app,
];
// 在热更新模式下确保每次编译生成的文件名字一致
clientConfig.output.filename = "[name].js";

server.use(hotMiddleware(clientCompiler, {
  noInfo: true,
  publicPath: clientConfig.output.publicPath,
}));
```

## 代码编写

[官方文档](https://ssr.vuejs.org/zh/guide/universal.html)

### 服务端的数据响应

- 每个请求都应该对应一个新的应用实例, 避免交叉请求造成状态污染
- 服务端渲染出来的页面是最终状态, 不需要进行响应式, 可以避免性能开销

### 组件生命周期钩子

- 只有`beforeCreate, created`会被调用, 其他钩子只会在客户端中执行
- 避免在create钩子函数中执行有全局副作用的代码
  - 比如设置一个timer, 然后在destroyed中销毁. 这样服务端代码的timer就永久保留了

### 访问特定平台api

- `window, document`对象在服务端不可访问
- 使用或构建library来兼容不同平台, 如axios
  - 同时要考虑library的兼容性, 否则需要mock一些全局变量来保证其运行
- 对于仅客户端可用的api, 在纯客户端的生命周期钩子中惰性访问

### 自定义命令

自定义指令会直接操作DOM, 会在ssr阶段导致错误

- 抽象为组件, 在vDOM层级进行操作
- 使用[directives](https://ssr.vuejs.org/zh/api/#directives)选项提供服务端版本

## [路由处理](https://ssr.vuejs.org/zh/guide/routing.html#%E4%BD%BF%E7%94%A8-vue-router-%E7%9A%84%E8%B7%AF%E7%94%B1)

### 创建路由

```js
// router/index.js
export const createRouter = () => {
  return new VueRouter({
    // 必须使用history模式以兼容前后端处理
    mode: 'history',
    routes: [...]
})}

// app.js
export function createApp() {
  const router = createRouter();
  const app = new Vue({
    // 根实例简单的渲染应用程序组件。
    render: (h) => h(App),
    router,
  });
  return { app, router };
}
```

### 服务端配置

```js
// entry-server.js
export default context => {
  const { app, router } = createApp()
  // 根据服务端req路径设置路由, context由renderer.renderToString函数提供
  router.push(context.url)
  // 等待router将可能的异步组件和钩子函数解析完毕
  await new Promise(router.onReady.bind(router))
  return app
}

// server.js
const render = async (req, res) => {
  try {
    const html = await renderer.renderToString({
      url: req.url,
      ...
    })
    ...
}}
// 设置服务端路由接受所有路径, 交由router判断
server.get('*', ...)
```

### 客户端配置

```js
// app.vue
<ul>
  <li>
    <router-link to="/">Home</router-link>
    <router-link to="/about">About</router-link>
  </li>
</ul>
<router-view />

// entry-client.js
const { app, router } = createApp()
// 必须提前解析路由配置中的异步组件, 然后才能调用组件中可能存在的路由钩子
router.onReady(() => {
  app.$mount('#app')
})
```

### 动态加载

在配置路由的时候支持动态懒加载, 即使用`component: () => import('@/pages.About')`的形式, 在使用到的时候才进行加载解析

但这并不意味着访问某一页面时一定不会访问其他动态组件, 因为ssr会**在head为下一页可能使用到的组件添加 prefetch link 标签**, 预加载动态组件的js. 如 `<link prefetch href="....js"/>`

### [head处理](https://ssr.vuejs.org/zh/guide/head.html)

支持不同页面不同head. 官方提供方式是使用`mixin`来控制

也可以使用[vue-meta](https://github.com/nuxt/vue-meta#examples), 支持[多种meta属性](https://vue-meta.nuxtjs.org/api/#metainfo-properties)的配置

```js
// app.js
Vue.use(VueMeta);
Vue.mixin({
  metaInfo: {
    titleTemplate: "%s - Common Title",
  },
});

// entry-server.js
async context => {
  const { app, router } = createApp();
  const meta = app.$meta();
  router.push(context.url);
  
  // 将meta信息暴露到context上下文中
  context.meta = meta;
}

// index.template.html
<head>
  {{{ meta.inject().title.text() }}}
  {{{ meta.inject().meta.text() }}}
</head>

// home.vue
export default {
  data: () => { ... },
  metaInfo: {
    title: 'Home'
  }
}
```

## [数据预取和状态管理](https://ssr.vuejs.org/zh/guide/data.html)

客户端的状态管理不适用于服务端

- 服务端只支持beforeCreate/created周期函数
- 不会等待钩子函数中的异步操作
- 不支持响应式数据

同时, 客户端在挂载之前, 还必须要获取到与服务端完全一致的数据, 否则会混合失败

### 思路

- 将数据获取放在视图渲染之前, 并放置在一个容器中, 比如vuex
- 将容器数据同步到客户端, 以保持数据状态同步

### 配置vuex

```js
// store/index.js
export const createStore = () => {
  return new Vuex.Store({
    state: ...,
    mutations: ...,
    actions: {
    	// 服务端渲染期间务必要返回promise
      async getPosts({ commit }) {
        const data = await axios.get(...)
        commit(...)
}}})}

// app.js
createApp() {
  const store = createStore()
  const app = new Vue({
    ...,
    store,
  })
  return { ..., store }
}
```

### 页面渲染

```js
// Posts.vue
export default {
  ...,
  computed: {
    ...mapState(["posts"]),
  },
  methods: {
    ...mapActions(["getPosts"]),
  },

  // vue ssr 为客户端渲染提供的生命周期钩子, 会在渲染前进行调用, 需要返回Promise
  serverPrefetch() {
    // return this.getPosts();
    return this.$store.dispatch("getPosts");
  },
}
```

### 数据状态传递

通过`window.__INITIAL_STATE__`来在前后端之间传递state状态

```js
// entry-server.js
async (context) => {
  ...;
  context.rendered = () => {
    // 将context.state数据对象内联到页面模板中
    // 最终生成页面上会包含一段脚本: `window.__INITIAL_STATE_ = ...`
    // 客户端就会将window.__INITIAL_STATE_填充到客户端store容器中
    context.state = store.state;
  };
}

// entry-client.js
...;
if (window.__INITIAL_STATE__) {
  store.replaceState(window.__INITIAL_STATE__);
}
```

# 静态站点生成

## 静态网站生成器

- 省钱, 不需要服务器, 只需要能托管静态文件的空间即可
- 快速, 不经过服务器处理, 只传输内容
- 安全, 不经过后端

常见的静态网站生成器

- Jekyll, Ruby
- Hexo, Node
- Hugo, Golang
- Gatsby, Node/React
- Gridsome, Node/Vue
- Next.js, Nuxt.js也可以生成静态网站, 但更多被认为是ssr框架

### JAMStack

JavaScript, API, Markup, 本质是一个胖前端, 通过调用各种API来实现更多功能

也是前后端模式, 只不过分的比较开, 前后端来自多个不同厂商

比如博客内容来自静态文件, 评论功能使用api厂商

### 使用场景

- 不适合有大量路由页面的应用
- 不适合有大量动态内容的应用
- 适合纯内容展示的应用, 如博客, 文档

## [Gridsome](https://www.gridsome.cn/)基础

### 安装

`yarn global add @gridsome/cli`

#### sharp

一个处理图片的库, 需要C++编译环境, 并且依赖libvips, 一个很大的库

- [按官方文档配置镜像](https://sharp.pixelplumbing.com/install#chinese-mirror)

  ```js
  npm config set sharp_binary_host "https://npm.taobao.org/mirrors/sharp"
  npm config set sharp_libvips_binary_host "https://npm.taobao.org/mirrors/sharp-libvips"
  ```

- 使用[node-gyp](https://github.com/nodejs/node-gyp)编译C++

  1. 安装Python环境

  2. 安装编译环境

     ```js
     // administrator 模式
     npm install -g windows-build-tools
     npm install -g node-gyp
     ```

### 创建项目

`gridsome create my-gridsome-site`

会创建package.json并自动安装, 但看不到进度. 建议创建后打断, 自己cd并安装

查看package.json发现有三个命令:

- build: 预渲染, 在dist目录生成静态页面
  - 预渲染后可以使用`serve`进行静态服务
  - `serve dist`
- develop 启动服务
- explore

### [目录结构](https://www.gridsome.cn/docs/directory-structure/)

#### src/

- main.js
  - `layout/Default.vue`获取并注册全局组件*Layout*
- pages
  - 使用*Layout*组件
  - 根据目录结构自动生成路由配置
- layouts
  - 放置布局组件
  - layout组件中使用`static-query`标签获取数据用于渲染
- components
  - 放置公用组件

#### .temp/

- 打包过程中自动生成的文件, 包括`routes.js`等

#### gridsome.server.js

- 打包过程中使用的server的配置文件
- 比如自定义配置路由等

### [项目配置](https://www.gridsome.cn/docs/config/)

### [pages](https://www.gridsome.cn/docs/pages/)

直接在pages目录下生成vue文件, 自动生成路由

- 文件名使用头字母大写驼峰, 路由是短线连接格式
- index是默认路由

api编程创建page

```js
// gridsome.server.js
module.exports = function (api) => {
  api.createPages( ({ createPage }) => {
    createPage({
      page: '/path-to-page',
      component: './src/templates/MyPage.vue'
    })
  })
}
```

#### [动态路由](https://www.gridsome.cn/docs/dynamic-routing/)

- 文件名中使用`[]`将参数名包裹起来

  - `src/pages/user/[id]/settings.vue` => `/user/:id/settings`
  - 文件中使用`const { id } = this.$route.params`获取参数
    - 必须在mounted中获取参数, `created`会在构建时被触发

- 直接配置路由

  ```js
  createPage({
    path: "/d-user/:id(\\d+)",
    component: "./src/templates/User.vue",
  });
  ```

#### [metaInfo](https://www.gridsome.cn/docs/pages/#page-meta-info)

使用`vue-meta`

#### 404页面

`src/pages/404.vue`

### [集合](https://www.gridsome.cn/docs/collections/)

从各种形式的数据源获取数据, 并形成集合, 每个集合可以有多个节点, 对应一个页面

比如博客, 标签各是一个集合, 博客中每篇文章是一个节点, 对应于一个页面

不同数据源可能会有对应[插件](https://www.gridsome.cn/docs/collections/#add-collections-with-source-plugins), 也可以使用api添加

#### GraphQL

每个集合都存储在gridsome自带的graphQL仓库中, 对应于`[name], all[Name]`两个库, 可以在开发模式下访问[graphQL](http://localhost:8081/___explore)查询

```js
query {
  allPost {
    edges {
      node {
        id
        title
        content
}}}}
```

- 在page和template中使用`<page-query>`顶部标签
- 在Component中使用`<static-query>`顶部标签
- 标签内直接写需要用到的query
- 查询到的数据默认绑定到`this.$page/this.$staitic`下

```vue
<template>
  <div>
    <div v-for="edge in $page.posts.edges" :key="edge.node.id">
      <h2>{{ edge.node.title }}</h2>
    </div>
  </div>
</template>

<page-query>
query {
  posts: allPost(sort: [{ by: "featured" }, { by: "date" }]) {
    edges {
      node {
        id
        title
      }
    }
  }
  post(id: '1') {
    content
  }
}
</page-query>
```

[集合查询](https://www.gridsome.cn/docs/querying-data/#querying-collections), [节点查询](https://www.gridsome.cn/docs/querying-data/#querying-single-nodes)

#### [节点模板](https://www.gridsome.cn/docs/templates/)

对于集合节点的渲染, 默认模板路径是`src/templates/{CollectionName}.vue`

```vue
// 依旧需要在template中写query, 只不过可以传参, 参数要和路由参数一致
<page-query>
query($id: ID!) {
  post(id: $id) {
    id
    title
    content
  }
}
</page-query>

<script>
export default {
  // 使用函数形式, 在this初始化好之后才能访问$page
  metaInfo() {
    return {
      title: this.$page.post.title
    }
  }
}
</script>
```

然后还需要在config.js中配置, 

```js
// gridsome.config.js
templates: {
  Post: [
    {
      path: '/posts/:id',
      component: './src/templates/Post.vue'
    },
  ]
}
```

## Gridsome案例

将[blog](https://github.com/ValenW/startbootstrap-clean-blog)迁移到gridsome实现

### index.html

- link中有*bootstrap*和*fontawesome-free*, 使用npm安装并在main.vue中引入

  ```js
  yarn add bootstrap @fortawesome/fontawesome-free
  // /src/main.js
  import "bootstrap/dist/css/bootstrap.min.css";
  import "@fortawesome/fontawesome-free/css/all.min.css";
  ```

- 两个google css和一个自定义css, 可在`/src/assets/css/index.css`中使用url引入

  ```css
  @import url("https://fonts.googleapis.com/css?family=Lora:400,700,400italic,700italic");
  @import url("https://fonts.googleapis.com/css?family=Open+Sans:300italic,400italic,600italic,700italic,800italic,400,300,600,700,800");
  
  /* custom css */
  ...
  ```

  ```js
  // main.js
  import "./assets/css/index.css";
  ```

其他文件也可以如此炮制

### 文章内容

使用本地markdowm文件作为数据源, 使用[plugin](https://www.gridsome.cn/plugins/@gridsome/source-filesystem)进行构建

markdown插件会自动加载使用, 可以在config中进行配置

```js
yarn add @gridsome/source-filesystem @gridsome/transformer-remark

// config.js
plugins: [{
  use: "@gridsome/source-filesystem",
  options: {
    typeName: "BlogPost",
    path: "./content/blog/**/*.md",
  },
}, ... ],
transformers: {
  remark: {
    // global remark options
}}
```

配置好之后可以在GraphQL playground中测试

```js
query {
  allBlogPost {
    edges {
      node {
        id
        content
        title
}}}}
```

纯文件管理会比较麻烦, 可以使用一个后台系统进行管理

### [strapi](https://strapi.io/)

`npx create-strapi-app my-project --quickstart`

图形化界面操作, 也可以使用[API](https://strapi.io/documentation/v3.x/content-api/api-endpoints.html#endpoints)

- 默认使用public角色, 需要配置查找权限
- 使用登录用户则需要登陆获取jwt, 然后在需要权限的请求中[设置header](https://strapi.io/documentation/v3.x/plugins/users-permissions.html#token-usage) `Authorization: 'Bearer ${token}`
- [GraphQL](https://strapi.io/documentation/v3.x/plugins/graphql.html#graphql)需要安装插件, 更新数据也需要提供jwt

gridsome提供了strapi插件以获取数据: `@gridsome/source-strapi`

```js
{
  use: '@gridsome/source-strapi',
  options: {
    apiURL: 'http://localhost:1337',
    queryLimit: 1000, // Defaults to 100
    contentTypes: ['post'],
    // singleTypes: ['impressum'],
    // Possibility to login with a Strapi user,
    // when content types are not publicly available (optional).
    // loginData: {
    //   identifier: '',
    //   password: ''
    // }
  }
}
```

更新配置重启后插件会自动导入strapi中的数据, 并创建`strapi{Name}`schema, 可以在gridsome的GraphQL playground中查询

### 数据结构设计

#### Post

- title
- content
- cover
- tags

#### tag

- title
- posts

```gql
query {
  allStrapiPost {
    edges {
      node {
        id
        title
        content
        cover {
          url
          height
          width
        }
        is_publish
        tags {
          title
        }
      }
    }
  }
}
```

### [分页](https://gridsome.org/docs/pagination/#paginate-data)

使用gridsome提供的`@paginate`指令和`perPage, page`参数

```js
query ($page: Int) {
  allBlogPost(perPage: 10, page: $page) @paginate {
    pageInfo {
      totalPages
      currentPage
    }
    edges {
      node {
        id
        title
        path
      }
    }
  }
}

<!-- Pager -->
<Pager :info="$page.posts.pageInfo" />

import { Pager } from "gridsome";
```

### 单个数据

snapi支持单个数据, 非集合. 但gridsome还是使用集合的形式处理, 生成`{name}, all{name}`两个, 并且查询单个的时候需要提供id, 即使有且仅有一个数据

所以推荐一直使用`all{name}`来查询, 避免id变化等情况

## 部署远端

### strapi

#### [切换数据库](https://strapi.io/documentation/v3.x/concepts/configurations.html#example)

```js
module.exports = ({ env }) => ({
  defaultConnection: 'default',
  connections: {
    default: {
      connector: 'bookshelf',
      settings: {
        client: 'mysql',
        host: env('DATABASE_HOST', 'localhost'), // 根据数据和前端是否部署到同一台机器
        port: env.int('DATABASE_PORT', 3306),
        database: env('DATABASE_NAME', 'strapi'), // 数据库名称
        username: env('DATABASE_USERNAME', 'strapi'), // 数据库用户名
        password: env('DATABASE_PASSWORD', 'strapi'), // 数据库密码
      },
      options: {},
    },
  },
});
```

#### 更新依赖

比如mysql就需要`yarn add mysql`

#### 远端安装运行

`pm2 start npm -- run start --name blog-backend`

### gridsome

#### [配置环境变量](https://gridsome.org/docs/environment-variables/#environment-variables)

```js
// .env.production, .env.development
// GRIDSOME_开头的可以在前后端访问, 否则只能在后端访问
GRIDSOME_API_URL=https://api.example.com
DB_USER=root
DB_PASS=s1mpl3
```

#### 数据更新自动重新部署

使用[vercel](https://vercel.com/)部署静态网站, 获取重新部署的trigger API

然后在strapi的webhook中添加新的hook, 使得每次更新都重新部署静态网站

































  





























































