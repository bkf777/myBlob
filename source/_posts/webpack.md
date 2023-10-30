---
title: 前端工程化之Webpack
subtitle: what is Webpack
tag: webpack
---
:::tip What is Webpack

- Webpack是前端最常用的构建工具，重要程度无需多言

- 之前看过很多关于 Webpack 的文章，总是感觉云里雾里，现在换一种方式，我们一起来解密它，尝试打开这个盲盒

:::

## 什么是 Webpack
:::tip 

本质上，webpack 是一个用于现代 JavaScript 应用程序的 静态模块打包工具。当 webpack 处理应用程序时，它会在内部从一个或多个入口点构建一个 依赖图(dependency graph)，然后将你项目中所需的每一个模块组合成一个或多个 bundles，它们均为静态资源，用于展示你的内容。

:::

### 以下是webpack的核心概念

:::details 入口(entry)
**入口起点(entry point)** 指示 webpack 应该使用哪个模块，来作为构建其内部 依赖图(dependency graph) 的开始。进入入口起点后，webpack 会找出有哪些模块和库是入口起点（直接和间接）依赖的。

默认值是 `./src/index.js`，但你可以通过在 webpack configuration 中配置 entry 属性，来指定一个（或多个）不同的入口起点。例如：

`webpack.config.js`

``` javascript
//此处将使用 `./path/to/my/entry/file.js` 作为入口起点
module.exports = {
  entry: './path/to/my/entry/file.js',
};
```
:::

:::details 输出(output)
**output** 属性告诉 webpack 在哪里输出它所创建的`bundle`，以及如何命名这些文件。主要输出文件的默认值是 `./dist/main.js`，其他生成文件默认放置在 `./dist` 文件夹中。

`webpack.config.js`

你可以通过在配置中指定一个 output 字段，来配置这些处理过程：
``` javascript
//引入nodejs的path 模块
const path = require('path');

module.exports = {
  entry: './path/to/my/entry/file.js',//入口已经在上面配置
  output: {
    path: path.resolve(__dirname, 'dist'),//输出目录
    filename: 'my-first-webpack.bundle.js',//打包后的文件
  },
};
```

:::

:::details loader

**webpack 只能理解 JavaScript 和 JSON 文件**，这是 webpack 开箱可用的自带能力。loader 让 webpack 能够去处理其他类型的文件，并将它们转换为有效 模块，以供应用程序使用，以及被添加到依赖图中。



在更高层面，在 webpack 的配置中，loader 有两个属性：

`test` 属性，识别出哪些文件会被转换。
`use` 属性，定义出在进行转换时，应该使用哪个 loader。  例如

仍然是`webpack.config.js`

``` javascript
const path = require('path');

module.exports = {
  entry: './path/to/my/entry/file.js',//入口已经在上面配置
  output: {
    path: path.resolve(__dirname, 'dist'),//输出目录
    filename: 'my-first-webpack.bundle.js',//打包后的文件
  },
  module: {
    rules: [{ test: /\.txt$/, use: 'raw-loader' }],//test:使用正则匹配文件 use:使用哪个loader
  },
};
```

:::

:::details 插件(plugin)

loader 用于转换某些类型的模块，而插件则可以用于执行范围更广的任务。包括：打包优化，资源管理，注入环境变量。


想要使用一个插件，你只需要 require() 它，然后把它添加到 plugins 数组中。多数插件可以通过选项(option)自定义。你也可以在一个配置文件中因为不同目的而多次使用同一个插件，这时需要通过使用 new 操作符来创建一个插件实例。

`webpack.config.js`

``` javascript
const HtmlWebpackPlugin = require('html-webpack-plugin');
const webpack = require('webpack'); // 用于访问内置插件


module.exports = {
  module: {
    rules: [{ test: /\.txt$/, use: 'raw-loader' }],
  },
  plugins: [new HtmlWebpackPlugin({ template: './src/index.html' })],
};
```
在上面的示例中，html-webpack-plugin 为应用程序生成一个 HTML 文件，并自动将生成的所有 bundle 注入到此文件中。
:::

:::details 模式(mode)
通过选择 `development`, `production` 或 `none` 之中的一个，来设置 mode 参数，你可以启用 webpack 内置在相应环境下的优化。其默认值为 production。
    
``` javascript

    module.exports = {
        mode: 'production',
    };

```
:::
## mini-webpack
我们不需要去掌握具体的实现细节，而是通过这个案例，了解 webpack 的整体打包流程，明白这个过程中做了哪些事情，最终输出了什么结果即可

```javascript
const fs = require('fs');
const path = require('path');
// babylon解析js语法，生产AST 语法树
// ast将js代码转化为一种JSON数据结构
const babylon = require('babylon');
// babel-traverse是一个对ast进行遍历的工具, 对ast进行替换
const traverse = require('babel-traverse').default;
// 将es6 es7 等高级的语法转化为es5的语法
const { transformFromAst } = require('babel-core');

// 每一个js文件，对应一个id
let ID = 0;

// filename参数为文件路径, 读取内容并提取它的依赖关系
function createAsset(filename) {
  const content = fs.readFileSync(filename, 'utf-8');

  // 获取该文件对应的ast 抽象语法树
  const ast = babylon.parse(content, {
    sourceType: 'module'
  });

  // dependencies保存所依赖的模块的相对路径
  const dependencies = [];

  // 通过查找import节点，找到该文件的依赖关系
  // 因为项目中我们都是通过 import 引入其他文件的，找到了import节点，就找到这个文件引用了哪些文件
  traverse(ast, {
    ImportDeclaration: ({ node }) => {
      // 查找import节点
      dependencies.push(node.source.value);
    }
  });

  // 通过递增计数器，为此模块分配唯一标识符, 用于缓存已解析过的文件
  const id = ID++;
  // 该`presets`选项是一组规则，告诉`babel`如何传输我们的代码.
  // 用`babel-preset-env`将代码转换为浏览器可以运行的东西.
  const { code } = transformFromAst(ast, null, {
    presets: ['env']
  });

  // 返回此模块的相关信息
  return {
    id, // 文件id（唯一）
    filename, // 文件路径
    dependencies, // 文件的依赖关系
    code // 文件的代码
  };
}

// 我们将提取它的每一个依赖文件的依赖关系，循环下去：找到对应这个项目的`依赖图`
function createGraph(entry) {
  // 得到入口文件的依赖关系
  const mainAsset = createAsset(entry);
  const queue = [mainAsset];
  for (const asset of queue) {
    asset.mapping = {};
    // 获取这个模块所在的目录
    const dirname = path.dirname(asset.filename);
    asset.dependencies.forEach((relativePath) => {
      // 通过将相对路径与父资源目录的路径连接,将相对路径转变为绝对路径
      // 每个文件的绝对路径是固定、唯一的
      const absolutePath = path.join(dirname, relativePath);
      // 递归解析其中所引入的其他资源
      const child = createAsset(absolutePath);
      asset.mapping[relativePath] = child.id;
      // 将`child`推入队列, 通过递归实现了这样它的依赖关系解析
      queue.push(child);
    });
  }

  // queue这就是最终的依赖关系图谱
  return queue;
}

// 自定义实现了require 方法，找到导出变量的引用逻辑
function bundle(graph) {
  let modules = '';
  graph.forEach((mod) => {
    modules += `${mod.id}: [
      function (require, module, exports) { ${mod.code} },
      ${JSON.stringify(mod.mapping)},
    ],`;
  });
  const result = `
    (function(modules) {
      function require(id) {
        const [fn, mapping] = modules[id];
        function localRequire(name) {
          return require(mapping[name]);
        }
        const module = { exports : {} };
        fn(localRequire, module, module.exports);
        return module.exports;
      }
      require(0);
    })({${modules}})
  `;
  return result;
}

// ❤️ 项目的入口文件
const graph = createGraph('./example/entry.js');
const result = bundle(graph);

// ⬅️ 创建dist目录，将打包的内容写入main.js中
fs.mkdir('dist', (err) => {
  if (!err)
    fs.writeFile('dist/main.js', result, (err1) => {
      if (!err1) console.log('打包成功');
    });
});


```
### 分析mini-webpack

:::tip 分析一下文件的执行流程

1. 整体大致分为 10 步，第一步从require(0)开始执行，调用内置的自定义 require 函数，跳转到第二步，执行fn函数 
2. 执行第三步require('./message.js')，继续跳转到第四步 require(mapping['./message.js']), 最终转化为require(1)
3. 继续执行require(1)，获取modules[1]，也就是执行message.js的内容
4. 第五步require('./name.js')，最终转化为require(2)，执行name.js的内容
5. 通过递归调用，将代码中导出的属性，放到exports对象中，一层层导出到最外层
6. 最终通过_message2.default获取导出的值，页面显示hello Webpack!

:::