- 去[vue3](https://github.com/vuejs/core)fork源码，到自己的仓库后，clone代码
- 下载pnpm@6.29.1 node@14.21.3
- 在本地vue3源码 执`pnpm i`
- 执行`npm run build`会在 `packages/vue/dist` 下生成一些vue.*.js文件
- `"build": "node scripts/build.js -s"`开启sourceMap，来跟踪 `vue` 代码的执行
- 在`packages/vue/examples/imooc`创建测试文件`reactive.html`，通过live-serve，本地运行测试代码
- 在控制台，打开sources面板看到源码，可以在`reactive.ts`打个断点，来跟踪整个 `reactive` 的代码执行逻辑
- 创建自己的`vue-next-mini`项目
  1. npm init -y
  2. 创建 `packages` 文件夹，作为：**核心代码** 区域
  3. 创建 `packages/vue` 文件夹：打包、测试实例、项目整体入口模块
  4. 创建 `packages/shared` 文件夹：共享公共方法模块
  5. 创建 `packages/compiler-core` 文件夹：编辑器核心模块
  6. 创建 `packages/compiler-dom` 文件夹：浏览器部分编辑器模块
  7. 创建 `packages/reactivity` 文件夹：响应性模块
  8. 创建 `packages/runtime-core` 文件夹：运行时核心模块
  9. 创建 `packages/runtime-dom` 文件夹：浏览器部分运行时模块
- 导入ts， 安装`npm install -g typescript@4.7.4`,执行`tsc -init`
- rollup小而美的模块打包器，创建rollup.config.js
- 配置@依赖，这表示，我们可以通过 `@vue/*` 代替 `packages/*/src/index` 的路径

```text
tsconfig.js
// 设置快捷导入
"baseUrl": ".",
"paths": {
"@vue/*": ["packages/*/src"]
```

- example代码

```JavaScript
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <script src="../../dist/vue.global.js"></script>
</head>

<body>
  <div id="app"></div>
</body>
<script>
  // 从 Vue 中结构出 reactie、effect 方法
  const { reactive, effect } = Vue

  // 声明响应式数据 obj
  const obj = reactive({
    name: '张三'
  })

  // 调用 effect 方法
  effect(() => {
    document.querySelector('#app').innerText = obj.name
  })

  // 定时修改数据，视图发生变化
  setTimeout(() => {
    obj.name = '李四'
  }, 2000);

</script>

</html>
```

- 本地git相关配置

```text
Your identification has been saved in /Users/liuruyue/.ssh/id_rsa
Your public key has been saved in /Users/liuruyue/.ssh/id_rsa.pub

+---[RSA 3072]----+
|   .oo...o...  oo|
|    ..  .  o. =..|
|    .     + .=oo |
|   o     o *o+=. |
|  . .   S B.=.o+ |
| .       + B..o .|
|E       . . oo ..|
|            o o..|
|           .o==*o|
+----[SHA256]-----+

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDjD40Ftije0U5ENzuDTjolx8vQZ267bTRxygaXyfauGm4pDXOPd7IZf02x5HqCWP7Ho7lsniVIaZqbuAAgCKF8Pg9H+mYXwTncoL/rhhRaW8iVFQMK6zVFrnL1zmT+oJGHGhpNcTOwwtEdT2j3uM3X6+nbN85NPskiwZzN1BSPguAxWbV8W7qMnjyMr757tRfRAA7JSX17Xxbx06w9FAqbcx1PAYrMlKbt2AO+lFNNp0YodMIdfvVoMl8lSfG/XbL75tqFZpxNpas8x/b9B3orYZKx40AsEFjbF/kqYfIv6NJpmciq+XD/Z0CEHH/wC+W2s45u7G+teN0YjRkMv2Pkz+N1IJLvAONqtUlTirWU1mIWTy84k8wHqRBsnl15oDzEInxKrBoxOS/48yppydkQahNVahmmE7o/MaDjqJNzbNDWmkTzZwk4PGYmjp3RoDzygqixAp/e09fnJBP4ichiSPw2OacX1DP+MVhK6lVZa4IoStrf1QyDyGpRfFDhV9U= 992819526@qq.com
```

### `vue` 的源码结构

下载好 `Vue` 原代码之后，我们来看下它的基本结构：

```
vue-next-3.2.37-master
├── tsconfig.json // TypeScript 配置文件
├── rollup.config.js // rollup 的配置文件
├── packages // 核心代码区
│   ├── vue-compat // 用于兼容 vue2 的代码
│   ├── vue // 重要：测试实例、打包之后的 dist 都会放在这里
│   ├── template-explorer // 提供了一个线上的测试（https://template-explorer.vuejs.org），用于把 tempalte 转化为 render
│   ├── size-check // 测试运行时包大小
│   ├── shared // 重要：共享的工具类
│   ├── sfc-playground // sfc 工具，比如：https://sfc.vuejs.org/
│   ├── server-renderer // 服务器渲染
│   ├── runtime-test // runtime 测试相关
│   ├── runtime-dom // 重要：基于浏览器平台的运行时
│   ├── runtime-core // 重要：运行时的核心代码，内部针对不同平台进行了实现
│   ├── reactivity-transform // 已过期，无需关注
│   ├── reactivity // 重要：响应性的核心模块
│   ├── global.d.ts // 全局的 ts 声明
│   ├── compiler-ssr // 服务端渲染的编译模块
│   ├── compiler-sfc // 单文件组件（.vue）的编译模块
│   ├── compiler-dom // 重要：浏览器相关的编译模块
│   └── compiler-core // 重要：编译器核心代码
├── package.json // npm 包管理工具
├── netlify.toml // 自动化部署相关
├── jest.config.js // 测试相关
├── api-extractor.json // TypeScript 的 API 分析工具
├── SECURITY.md // 报告漏洞，维护安全的声明文件
├── README.md // 项目声明文件
├── LICENSE // 开源协议
├── CHANGELOG.md // 更新日志
├── BACKERS.md // 赞助声明
├── test-dts // 测试相关，不需要关注
├── scripts // 配置文件相关，不需要关注
├── pnpm-workspace.yaml // pnpm 相关配置
└── pnpm-lock.yaml // 使用 pnpm 下载的依赖包版本

```

