# 组件开发

CDD, Component-Driven-Development

- 组件最大程度被重用
- 并行开发
- 可视化测试

## [边界情况](https://cn.vuejs.org/v2/guide/components-edge-cases.html)

`$root`访问根组件(APP), `$parent, $children`访问父子组件, children根据html结构排序

### `provide/inject`

在祖先组件provide方法或数据, 并命名, 在子组件inject获取父组件提供的方法/数据

需要注意的是这**不是响应式的**, 不应该更改祖先组件provide的数据

### `$attrs, $listerners`

- 父组件传入的属性默认绑定到子组件的根节点上
  - 都存储在子节点的`$attrs`上, 除了`class`和`style`
  - `class, style`不能成为子组件的props, 会合并到根节点的dom属性上
  - 子组件设置`inheritAttrs: false`可以禁用默认的继承属性行为
  - 通过`v-bind="$attrs"`为期望的标签设置上父组件传入的属性
- 子组件通过`@focus="$emit('focus', $event)"`, `@event/$emit`触发并传递事件
  - 可以通过`v-on="$listener"`将父组件所有监听函数绑定到子组件上

## 快速原型开发

vue提供插件, 使得单文件直接运行, 方便快速原型开发

```js
npm install -g @vue/cli-service-global
vue serve ./src/login.vue
// 默认当前目录的main.js, index.js, App.vue, app.vue为入口
```

### Element-UI

```js
vue add element

// main.js
import Vue from "vue";
import ElementUI from "element-ui";
import "element-ui/lib/theme-chalk/index.css";

Vue.use(ElementUI);

new Vue({
  el: "#app"
});
```

组件分类:

1. 第三方组件
2. 基础组件
3. 业务组件

### 表单组件

主要有form, form-item, input, (button)组件

在form中provide`form`, 值为自身, 在form-item中inject, 获取form组件

#### 验证

验证表单可以使用`async-validator`库

```js
import AsyncValidator = import 'async-validator'
validate() {
  const value = this.form.model[this.prop]
  const rules = this.form.rules[this.prop]
  const discriptor = { [this.prop]: rules }
  const validator = new AsyncValidator(descriptor)
  
  return validator.validate({ [this.prop]: value }, errors => {
    this.errorMsg = errors && errors[0].message || ''
  })
}
```

#### 触发表格验证

input组件中每次input都需要触发form的validate事件, 但`$emit`只能触发父组件的事件

一个解决方案是遍历$parent, 直到找到可以触发事件的组件, 如`form-item`和`form`

```js
const findParent = (comp, name) {
  while(comp) {
    if (comp.$options.name === name) return comp
    comp = comp.$parent
  }
}
```

另外form还需要向外提供·validate`函数方便进行全部验证. 这里需要获取form下的所有form-item子组件, 包括深层嵌套

```js
validate(cb) {
  // 假设只出现在直接子节点, 获取所有验证结果
  const tasks = this.$children
  	.filter(c => c.prop)
  	.map(c => c.validate())
  Promise.all(tasks)
    .then(() => cb(true))
    .catch(() => cb(false))
}
```

## Monorepo

库整体是很大的, 但用户可能只需要一部分功能,这就要求库分为多个模块, 分别进行开发, 管理, 测试, 发布

- multirepo,每个包对应一个项目
- Monorepo, (Monolithic Respository), 一个项目/仓库中管理多个模块/包

参考vue3和react的项目结构, 都是将每个功能拆分到不同的包中, 放置在`/packages`下, 并且每个包内的结构一样

```json
"private": true, // 不发布根目录包
"workspaces": [
  "packages/*"
],
```

```
/src
  |-- package.json
  |-- /packages
        |-- /button
        	|-- /__test__
        	|-- /dist
        	|-- /src
        	|-- index.js
        	|-- LICENSE
        	|-- package.json
        	|-- README.md
        |-- /form
        	...
```

```js
// /button/index.js
import Button from './src/button.vue'

Button.install = Vue => {
  Vue.component(Button.name, Button)
}

export default Button
```

## [Storybook](https://storybook.js.org/)

可视化组件展示平台, 在隔离环境中, 以交互式展示组件, 可以独立开发组件, 支持多种框架

### 安装

```js
npx -p @storybook/cli sb init --type vue
yarn add vue
yarn add vue-loader vue-template-compiler --dev
```

### 结构

#### /.storybook

放置配置文件

其中main.js中配置了插件和story目录, 默认在`/stories`目录下

如果在每个组件目录下放置各自的story方便管理, 就需要修改这个配置, 如`"../packages/**/*.stories.js",`

### 写法

```js
// /packages/input/stories/input.stories.js
import Input from "../"; // 直接引入index

export default {
  title: "input",
  component: Input,
};

export const Text = () => ({
  components: { MyInput: Input },
  template: '<my-input v-model="value" />',
  data() { return { value: "input" }; },
});

export const Password = () => ({
  components: { MyInput: Input },
  template: '<my-input type="password" v-model="value" />',
  data() { return { value: "input" }; },
});
```

## yarn workspaces

不同模块可能依赖相同包, 每个模块单独安装十分麻烦. yarn支持的workspaces特性可以解决这个问题:

```json
"private": true, // 不发布根目录包
"workspaces": [
  "packages/*"
],
```

- 只在根目录安装开发依赖
  - `yarn add jest -D -W`
- 给指定模块安装依赖
  - `yarn workspace my-input add lodash@4`
- 给所有模块安装依赖
  - `yarn install`
  - 会将相同依赖提升到根工作区, 不会重复下载依赖
  - 没有重复的包会下载到对应的模块中, 而不是根目录下的`node_modules`

## Lerna

优化使用git和npm**管理多包仓库**的工具流工具, 主要用于管理多包js项目, 可以一键提交代码到git和npm仓库

```js
yarn global add lerna
lerna init
lerna publish

// 发布前测试检查
- git有远程仓库
npm whoami
npm config get registry
```

## Vue组件单元测试

### 好处

- 提供描述组件行为的文档
- 节省手动测试时间
- 减少研发型特性时产生的bug
- 改进设计
- 促进重构

### 依赖

- [Vue test Utils](https://vue-test-utils.vuejs.org/zh/), 官方组件单元测试库, 需要一个测试库, 比如jest
- [Jest](https://github.com/facebook/jest), 不支持单元组件
- [vue-jest](https://github.com/vuejs/vue-jest), 提供vue使用jest进行组件的单元测试
- [babel-jest](https://github.com/facebook/jest/tree/master/packages/babel-jest), 测试中可能使用到js新特性

```js
// package.json
scripts: {
  test: "jest"
}
// jest.config.js
module.exports = {
  testMatch: ["**/__tests__/**/*.[jt]s?(x)"],
  moduleFileExtensions: [
    "js",
    "json",
    // Jest `*.vue`
    "vue",
  ],
  transform: {
    // use `vue-jest` to deal with `*.vue`
    ".*\\.(vue)$": "vue-jest",
    // use `babel-jest` to deal with js file
    ".*\\.(js)$": "babel-jest",
  },
};
// babel.config.js
module.exports = {
  presets: [
    [
      '@babel/preset-env'
    ]
  ]
}

// vue-test依赖的babel版本不一致, 需要使用babel桥接来解决
yarn add babel-core@bridge -D -W

```

### [Jest](https://jestjs.io/docs/zh-Hans/api)

- 全局函数
  - describe(name, fn), 组合相关的测试
  - test(name, fn), 测试方法
  - expect(value), 断言
- 匹配器
  - toBe, 值相等
  - toEqual(obj), 对象相等
  - toContain(value), 字符串/数组中包含
- 快照
  - toMatchSnapshot

### [Vue test](https://vue-test-utils.vuejs.org/zh/api/)

- mount
  - 创建一个包含目标组件的wrapper
- wrapper
  - vm, 包裹的组件实例
  - props, 组件属性
  - html, 组件生成的html标签
  - find, 匹配组件中的DOM元素
  - trigger, 触发DOM原生事件
  - vm.$emit, 触发自定义事件

### 测试代码

```js
// /packages/input/__tests__/input.test.js
describe('lg-input', () => {
  test('input-password', () => {
    const wrapper = mount(input, {
      propsData: {
        type: 'password',
        value: 'admin'
      }
    })
    expect(wrapper.html()).toContain('input type="password"')
    expect(wrapper.props('value')).toBe('admin')
  })

  test('input-snapshot', () => {
    const wrapper = mount(input, {
      propsData: {
        type: 'text',
        value: 'admin'
      }
    })
    // 第一次调用toMatchSnapshot会在当前目录下生成snapshot文件
    // 之后每次调用会对比已有的snapshot, 不匹配则报错
    // jest -u 删除已有snapshot
    expect(wrapper.vm.$el).toMatchSnapshot()
  })
}
```

## Rollup

模块打包器, **支持tree-shaking**, **打包结果小**, 适合开发框架或组件库

### 安装

- rollup
- rollup-plugin-terser, 压缩代码
- rollup-plugin-vue@5.1.9, 为rollup提供 单文件vue组件 的支持, 必须指定版本, 最新版本支持vue3
- vue-template-compiler

### 配置

```js
// */input/rollup.config.js
import { terser } from "rollup-plugin-terser";
import vue from "rollup-plugin-vue";

module.exports = [
  {
    input: "index.js",
    output: [
      {
        file: "dist/index.js",
        format: "es",
      },
    ],
    plugins: [
      vue({
        css: true,
        compileTemplate: true,
      }),
      terser(),
    ],
  },
];
// */input/package.json
scripts: {
  build: "rollup -c"
}
```

### 多模块打包

- @rollup/plugin-json
  - 支持json文件的打包
- rollup-plugin-postcss
- @rollup.plugin-node-resolve
  - 打包第三方依赖

需要为每个模块设置`main`和`module`字段, 用于生成不同js版本的代码

- main使用cjs, 用作执行代码时使用common js
- module使用es, 用作模块代码时使用更通用的es

```js
// /package.json
scripts: {
  build: "rollup -c"
}
// */input/package.json
main: "dist/cjs/index.js",
module: "dist/es/index.js"

// ./rollup.config.js
import fs from 'fs'
import path from 'path'
import json from '@rollup/plugin-json'
import vue from 'rollup-plugin-vue'
import postcss from 'rollup-plugin-postcss'
import { terser } from 'rollup-plugin-terser'
import { nodeResolve } from '@rollup/plugin-node-resolve'

const isDev = process.env.NODE_ENV !== 'production'

// 公共插件配置
const plugins = [
  vue({
    // Dynamically inject css as a <style> tag
    css: true,
    // Explicitly convert template to render function
    compileTemplate: true
  }),
  json(),
  nodeResolve(),
  postcss({
    // 把 css 插入到 style 中
    // inject: true,
    // 把 css 放到和js同一目录
    extract: true
  })
]

// 如果不是开发环境，开启压缩
isDev || plugins.push(terser())

// packages 文件夹路径
const root = path.resolve(__dirname, 'packages')

module.exports = fs.readdirSync(root)
  // 过滤，只保留文件夹
  .filter(item => fs.statSync(path.resolve(root, item)).isDirectory())
  // 为每一个文件夹创建对应的配置
  .map(item => {
    const pkg = require(path.resolve(root, item, 'package.json'))
    return {
      input: path.resolve(root, item, 'index.js'),
      output: [
        {
          exports: 'auto',
          file: path.resolve(root, item, pkg.main),
          format: 'cjs'
        },
        {
          exports: 'auto',
          file: path.join(root, item, pkg.module),
          format: 'es'
        },
      ],
      plugins: plugins
    }
  })
```

## 环境变量

使用cross-env包, 开发依赖, 在script中加入`cross-env NODE_ENV=development`即可

## 清理

- `lerna clean`清理所有模块的node_modules
- 使用rimraf清理dist目录
  - 每个模块添加命令`del: "rimraf dist"`
  - `yarn workspaces run del`对每个模块执行del命令

## 基于模板生成组件基本结构

使用plop, 并创建`plop-template/component`文件夹, 以及`./plopfile.js`配置文件

```js
// ./plopfile.js
module.exports = plop => {
  plop.setGenerator('component', {
    description: 'create a custom component',
    prompts: [
      {
        type: 'input',
        name: 'name',
        message: 'component name',
        default: 'MyComponent'
      }
    ],
    actions: [
      {
        type: 'add',
        path: 'packages/{{name}}/src/{{name}}.vue',
        templateFile: 'plop-template/component/src/component.hbs'
      },
      {
        type: 'add',
        path: 'packages/{{name}}/__tests__/{{name}}.test.js',
        templateFile: 'plop-template/component/__tests__/component.test.hbs'
      },
      {
        type: 'add',
        path: 'packages/{{name}}/stories/{{name}}.stories.js',
        templateFile: 'plop-template/component/stories/component.stories.hbs'
      },
      {
        type: 'add',
        path: 'packages/{{name}}/index.js',
        templateFile: 'plop-template/component/index.hbs'
      },
      {
        type: 'add',
        path: 'packages/{{name}}/LICENSE',
        templateFile: 'plop-template/component/LICENSE'
      },
      {
        type: 'add',
        path: 'packages/{{name}}/package.json',
        templateFile: 'plop-template/component/package.hbs'
      },
      {
        type: 'add',
        path: 'packages/{{name}}/README.md',
        templateFile: 'plop-template/component/README.hbs'
      }
    ]
  })
}
```

























