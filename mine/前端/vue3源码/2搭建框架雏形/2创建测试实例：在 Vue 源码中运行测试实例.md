那么现在我们已经大致的了解了 `vue` 源代码的基本结构，那么接下来我们来看一下，如何在 `Vue3` 的代码中运行测试实例，并进行 `debugger`



### 运行 `vue3` 源代码

1. 因为 `vue3` 是通过 [pnpm](https://pnpm.io/zh/) 作为包管理工具的，所以想要运行 `vue3` 那么首先需要 **安装 `pnpm`**

2. 我们可以直接通过如下指令安装 `pnpm`

   ```shell
   npm install -g pnpm
   
   ```

3. `pnpm` 会通过一个 **集中管理** 的方式来管理 **电脑中所有项目** 的依赖包，以达到 **节约电脑磁盘** 的目的，具体可点击 [这里](https://pnpm.io/zh/motivation) 查看

4. 安装完 `pnpm` 之后，接下来就可以在 **项目根目录** 下通过：

   ```shell
   pnpm i
   
   ```

   的形式安装依赖

5. 等待所有的依赖包安装完成之后，执行：

   ```shell
   npm run build
   
   ```

   进行项目打包

6. 执行完成之后，可以发现在 `packages/vue/dist` 下，将生成如下文件：

   ```shell
   dist
   ├── vue.cjs.js
   ├── vue.cjs.prod.js
   ├── vue.esm-browser.js
   ├── vue.esm-browser.prod.js
   ├── vue.esm-bundler.js
   ├── vue.global.js
   ├── vue.global.prod.js
   ├── vue.runtime.esm-browser.js
   ├── vue.runtime.esm-browser.prod.js
   ├── vue.runtime.esm-bundler.js
   ├── vue.runtime.global.js
   └── vue.runtime.global.prod.js
   
   ```

7. 那么至此，项目打包完成



### 运行测试实例

之前的时候说过 `packages/vue/examples` 下放的是测试的实例，所以我们可以在这里新建一个文件夹，用来表示代码的测试实例。

1. 创建 `packages/vue/examples/imooc` 文件夹

2. 在该文件夹中创建第一个测试实例：`reactive.html`，并写入如下代码（这些代码可能大家会不太理解，没有关系！后面会进行详细讲解，这里只是为了测试 `vue` 代码的 `debugger`）：

   ```html
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

3. 注意：**该代码不可以直接放入到浏览器中运行**。

4. 如果想要运行改代码，那么我们需要启动一个服务

5. 大家可以在 `VSCode` 中安装 `Live Server` 的插件，该插件可以帮助我们直接启动一个 **本地服务**

6. 安装完成之后，直接在 `html` 文件上：右键 -> `open with Live Server` 即可：

7. 此时，可以发现 **等待两秒钟之后，** 展示的文本由张三变为李四

那么至此我们已经成功了运行了一个测试的实例