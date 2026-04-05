### webpack构建的流程

Webpack 的运行流程是一个串行的过程，从启动到结束会依次执行以下流程：

- **初始化参数**：从配置文件和 Shell 语句中读取与合并参数，得出最终的参数
- **开始编译**：用上一步得到的参数初始 `Compiler` 对象，加载所有配置的插件，通 过执行对象的 `run` 方法开始执行编译
- **确定入口：**根据配置中的 `Entry` 找出所有入口文件
- **编译构建流程**：从 入口文件 出发，针对每个 模块 串行调用对应的 `Loader` 去翻译文件内容，，递归地进行编译处理。
- **完成模块编译**  翻译完所有模块后， 得到了每个模块被编译后的最终内容及它们之间的依赖关系 
- **输出流程**：根据入口和模块之间的依赖关系，对编译后的 模块 组合成 `Chunk`，把 `Chunk` 转换成文件，输出到文件系统。

在以上过程中，Webpack 会在特定的时间点广播出特定的事件，插件在监听到目标事件后会执行特定的逻辑，并且插件可以调用 Webpack 提供的 API 改变 Webpack 的运行结果。

简单说

- 初始化：启动构建，读取与合并配置参数，加载 Plugin，实例化 Compiler
- 编译：从 Entry 出发，针对每个 Module 串行调用对应的 Loader 去翻译文件的内容，再找到该 Module 依赖的 Module，递归地进行编译处理
- 输出：将编译后的 Module 组合成 Chunk，将 Chunk 转换成文件，输出到文件系统中



### loader和plugin的区别

1. **Loader**

- Loader 由于 webpack 本身只能打包 commonjs 规范的js文件，所以，针对 css，图片等格式的文件没法打包，就需要引入第三方的模块进行打包。
- 所以 Loader 就成了翻译官，对其他类型的资源进行转译的预处理工作。
- loader虽然是扩展了 webpack ，但是它只专注于转化文件（transform）这一个领域，完成压缩，打包，语言翻译。
- 所以Loader本质就是一个函数，在该函数中对接收到的内容进行转换，返回转换后的结果。

2. **Plugin**

- Plugin 也是为了扩展 webpack 的功能，但是 plugin 是作用于webpack本身上的。
- 而且 plugin 不仅只局限在打包，资源的加载上，它的功能要更加丰富。
- 从打包优化和压缩，到重新定义环境变量，功能强大到可以用来处理各种各样的任务。

1. **Loader和Plugin的区别**
2. **职责不同：**

3. Loader负责文件的转换，将不同类型的文件转换成Webpack可识别的模块； 
4. Plugin则更侧重于扩展Webpack的功能，可以对Webpack输出的结果进行优化、处理和修改。

5. **使用方式不同：**

6. Loader通过module.rules配置使用，可以用于文件类型转换，例如将LESS、SCSS、ES6、JSX等文件转换并打包成最终的静态资源文件；
7. Plugin则通过插件机制绑定到Webpack实例上，在Webpack构建生命周期的不同节点执行特定的操作。

8. **作用范围不同：**

9. Loader是针对每个模块进行处理，对于特定类型的文件，Webpack在打包的过程中，找到对应的Loader进行处理，其作用范围仅限于自己匹配的文件；
10. Plugin则作用于构建过程的整个生命周期中，Webpack 运行的生命周期中会广播出许多事件，Plugin 可以监听这些事件，在合适的时机通过 Webpack 提供的 API 改变输出结果。

11. **编写方式不同：**

12. 由于Loader和Plugin处理的方式不同，因此它们的编写方式也不同。
13. Loader一般是以纯函数的形式编写，接受模块源码作为输入，输出转换后的结果；
14. Plugin则是一个JavaScript类，包含一个apply方法，用于接收Webpack的Compiler对象，通过Compiler对象可以获取到Webpack在打包过程中产生的各种hook，然后在hook中完成特定操作。

总之，Loader和Plugin都是Webpack中非常重要的概念，分别承担着不同的职责，在Webpack的构建过程中发挥着各自独特的作用。



### 常见的loader和plugin

**loader**

- babel-loader：把 ES6 转换成 ES5
- less-loader：将 Less 代码转换成CSS
- sass-loader：将 SCSS/SASS 代码转换成CSS
- css-loader：加载 CSS，支持模块化、压缩、文件导入等特性
- style-loader：把 CSS 代码注入到 JavaScript 中，通过 DOM 操作去加载 CSS
- @svgr/webpack ：是一个 Webpack 插件，用于将 SVG 文件转换为 React 组件
- postcss-loader：扩展 CSS 语法，可以配合 autoprefixer 插件自动补齐 CSS3 前缀
- file-loader：把文件输出到文件夹中，在代码中通过相对 URL 去引用输出的文件 (处理图片和字体)
- url-loader：与 file-loader 类似，区别是用户可以设置一个阈值，大于阈值会交给 file-loader 处理，小于阈值时返回文件 base64 形式编码 (处理图片和字体)
- ts-loader: 将 TypeScript 转换成 JavaScript
- image-loader：加载并且压缩图片文件
- raw-loader：加载文件原始内容（utf-8）
- source-map-loader：加载额外的 Source Map 文件，以方便断点调试
- svg-inline-loader：将压缩后的 SVG 内容注入代码中
- json-loader 加载 JSON 文件（默认包含）
- handlebars-loader: 将 Handlebars 模版编译成函数并返回
- awesome-typescript-loader：将 TypeScript 转换成 JavaScript，性能优于 ts-loader
- eslint-loader：通过 ESLint 检查 JavaScript 代码
- tslint-loader：通过 TSLint检查 TypeScript 代码
- mocha-loader：加载 Mocha 测试用例的代码
- coverjs-loader：计算测试的覆盖率
- vue-loader：加载 Vue.js 单文件组件
- i18n-loader: 国际化
- cache-loader: 可以在一些性能开销较大的 Loader 之前添加，目的是将结果缓存到磁盘里

**plugin**

- clean-webpack-plugin: 用于在构建新版本的项目时自动清理输出目录。这样可以确保每次构建时，输出目录只包含最新的构建文件，而不会保留旧的、过时的文件。
- mini-css-extract-plugin: 分离样式文件，CSS 提取为独立文件，支持按需加载
- html-webpack-plugin：简化 HTML 文件创建 (依赖于 html-loader)
- webpack-bundle-analyzer: 可视化 Webpack 输出文件的体积 
- terser-webpack-plugin: 支持压缩 ES6 (Webpack4)
- define-plugin：定义环境变量 (Webpack4 之后指定 mode 会自动配置)
- ignore-plugin：忽略部分文件
- web-webpack-plugin：可方便地为单页应用输出 HTML，比 html-webpack-plugin 好用
- webpack-parallel-uglify-plugin: 多进程执行代码压缩，提升构建速度
- serviceworker-webpack-plugin：为网页应用增加离线缓存功能
- ModuleConcatenationPlugin: 开启 Scope Hoisting
- speed-measure-webpack-plugin: 可以看到每个 Loader 和 Plugin 执行耗时 (整个打包耗时、每个 Plugin 和 Loader 耗时)



## webpack打包优化方式

#### **提高webpack的打包速度**

1. 优化loader配置
2. 优化resolve.alias 起别名
3. 使用cache-loader 进行缓存
4. HappyPack 启动多线程
5. 合理使用 sourceMap

**优化loader配置，精准匹配文件，缩小文件范围**

在使用`loader`时，可以通过配置`include`、`exclude`、`test`属性来精准的匹配哪些文件应用`loader`。

使用include来指定编译文件夹，exclude排除指定文件夹。

```
module.exports = {
  module: {
    rules: [
      {
        // 如果项目源码中只有 js 文件就不要写成 /\.jsx?$/，提升正则表达式性能
        test: /\.js$/,
        // babel-loader 支持缓存转换出的结果，通过 cacheDirectory 选项开启
        use: ['babel-loader?cacheDirectory'],
        //使用include来指定编译文件夹只对项目根目录下的 src 目录中的文件采用 babel-loader
        include: path.resolve(__dirname, 'src'),
  			//使用exclude排除指定文件夹
  			exclude: /node_modules/,
      },
    ]
  },
};
```

**优化 resolve.alias 起别名**

alias给一些常用的路径起一个别名，特别当我们的项目目录结构比较深的时候，一个文件的路径可能是./../../的形式

通过配置alias以减少查找过程

```
module.exports = {
    ...
    resolve:{
        alias:{
            "@":path.resolve(__dirname,'./src')
        }
    }
}
```

**使用 cache-loader 缓存**

在一些性能开销较大的`loader`之前添加`cache-loader`，以将结果缓存到磁盘里，显著提升二次构建的速度。

注意：保存和读取这些缓存文件也会有一些时间开销，所以只对性能开销较大的`loader`使用此`loader`

```
module.exports = {
    module: {
        rules: [
            {
                test: /\.ext$/,
                use: ['cache-loader', ...loaders],
                include: path.resolve('src'),
            },
        ],
    },
};
```

**HappyPack 启动多线程**

HappyPack 可以将 Loader 的同步执行转换为并行的，这样就能充分利用系统资源来加快打包效率了。

这是一个Webpack插件，它可以将模块的加载和构建过程分解为多个子进程，从而实现多线程构建。

```
module: {
  rules: [
    {
      test: /.js$/,
      loader: 'happypack/loader?id=js',
      include: [resolve('src')],
      exclude: /node_modules/
    }
  ]
},
plugins: [
  new HappyPack({
    id: 'js',//与rule中指定的id一致
    // 开启四个线程
    threads: 4,//必须配置，用于指定开启几个 worker，建议设置成 os.cpus().length - 1
    verbose: true,//是否输出详细的日志信息
    loaders: ['babel-loader?cacheDirectory'],//如何处理 js 文件，如将其转成 es5 代码
  })
]
```

**合理使用Source map**

在开发环境下，因为需要调试源代码，一般会开启Source Map。

而在生产环境下，为了保护代码的安全且获得更好的性能，我们一般不开启Source Map。

Source map 的模式

1. 开发环境下开启：在开发环境下启用Source map可以帮助开发者快速定位错误和调试代码。

- 但是在构建生产代码时应该禁用，确保代码被混淆和压缩。

1. 使用`Cheap`模式：开启Cheap模式可以大幅度减少构建时间，它只会标记行而不会标记列，因此精度较低，但对于代码调试已经足够。
2. 使用`eval`模式：开启eval模式可以进一步缩短构建时间，因为它会将Source map嵌入到eval中，从而避免了Source map文件的生成和加载。
3. 使用`inline`模式：将Source map嵌入到打包文件中，可以避免Source map文件的下载，但会增加打包文件的大小。
4. 使用`external`模式：将Source map文件分离到单独的文件中，可以避免打包文件过大，同时也便于对Source map文件进行管理。

#### 减少webpack的打包体积

**按需加载**

在开发 SPA 项目的时候，项目中都会存在很多路由页面。

如果将这些页面全部打包进一个 JS 文件的话，虽然将多个请求合并了，但是同样也加载了很多并不需要的代码，耗费了更长的时间。

那么为了首页能更快地呈现给用户，希望首页能加载的文件体积越小越好，这时候就可以使用按需加载，将每个路由页面单独打包为一个文件。

在 Webpack 中，可以结合 React Router（或者其他前端路由库）实现路由级别的代码分割：

1. **动态** **import()**:
   使用动态 import() 语法来导入路由组件。这会指示 Webpack 将这些组件的代码分割出去并在需要时才加载。
2. **React Router** **React.lazy** **和** **Suspense**:
   React 提供了内建的懒加载工具 React.lazy 和 Suspense。React.lazy 允许你定义一个动态加载的组件，而 Suspense 组件可以指定加载指示器（如旋转的加载图标），在组件正在加载时显示。

```javascript
import React, { Suspense, lazy } from 'react';
import { BrowserRouter as Router, Route, Switch } from 'react-router-dom';

// 使用 React.lazy 和动态 import() 定义懒加载组件
const HomePage = lazy(() => import('./pages/HomePage'));
const AboutPage = lazy(() => import('./pages/AboutPage'));
const ContactPage = lazy(() => import('./pages/ContactPage'));

function App() {
  return (
    <Router>
      <Suspense fallback={<div>Loading...</div>}>
        <Switch>
          <Route exact path="/" component={HomePage} />
          <Route path="/about" component={AboutPage} />
          <Route path="/contact" component={ContactPage} />
          // ...其他路由
        </Switch>
      </Suspense>
    </Router>
  );
}
```

**MinCssExtractPlugin**

提取单独提取 CSS 文件

```
const MinCssExtractPlugin = require('mini-css-extract-plugin')
```

```
module.exports = {
  {
    test:/\.css$/,
    use:[
      // 'style-loader' 开发阶段使用
      MiniCssExtractPlugin.loader,// 生产阶段使用
      'css-loader'
    ]  
  }
    plugins:[
    new MinCssExtractPlugin({
      // 放到一个单独的css文件夹里面
      // 正常导入的包的名称
      filename:'css/[name].css',
      // 动态导入的包的名称
      chunkFilename:'css/[name]_chunk.ss'
    });
  ]
}
```

**代码拆包**

将相同的代码压缩到一个公共文件中，并将其分离出来，以提高页面加载速度。

通过`SplitChunkPlugin`可以将应用程序的公共库和应用程序代码分开，防止其中的一个库过大并阻止页面初始加载。

`Webpack` 提供了 `SplitChunksPlugin` 默认的配置，我们也可以手动来修改他的配置

- 比如默认配置中，chunks 仅仅针对异步async 请求，我们也可以设置为all
- 需要在 optimization 配置下进行。

```
module.export = {
  // 优化配置
  optimization:{
    SplitChunks:{
      // chunks 仅仅针对异步async 请求，我们也可以设置为all
     	chunks:"all",
      // 当一个包大于指定的大小时，继续进行拆包
      maxSize:20000
    }
  }
}
// 根据 Module 使用频率分包
module.exports = {
  //...
  optimization: {
    splitChunks: {
      // 设定引用次数超过 2 的模块才进行分包
      minChunks: 2
    },
  },
}
```

### ESmodule和CommonJS有什么区别

1. **用法不同：**

- ES module是原生支持的 Javascript 模块系统，使用import/export 关键字实现模块的导入和导出。
- 而 CommonJs 是 Node 最早引入的模块化方案，采用 require 和 module.exports 实现模块的导入和导出

2. **加载方式不同：**

- 编译时加载: ES module 模块不是对象，而是通过 export 命令显式指定输出的代码，import时采用静态命令的形式。即在import时可以指定加载某个输出值，而不是加载整个模块，这种加载称为“编译时加载”。
- 运行时加载: CommonJS 模块就是对象；即在输入时是先加载整个模块，生成一个对象，然后再从这个对象上面读取方法，这种加载称为“运行时加载”。

3. **ES6 模块输出的是值的引用，CommonJS 模块输出的是一个值的拷贝**

- ES6 模块的运行机制与 CommonJS 不一样。JS 引擎对脚本静态分析的时候，遇到模块加载命令import，就会生成一个只读引用。等到脚本真正执行时，再根据这个只读引用，到被加载的那个模块里面去取值。原始值变了，import加载的值也会跟着变。**因此，ES6 模块是动态引用，并且不会缓存值，模块里面的变量绑定其所在的模块。**
- CommonJS 模块输出的是值的**拷贝**(**浅拷贝**)，也就是说，输出一个值，模块内部的变化影响不到这个值。

```
// lib.js
let store = { version: '1.0' };
function update() { store.version = '2.0'; } // 修改属性
function reset() { store = { version: '0.0' }; } // 重新赋值

module.exports = { store, update, reset };

// main.js
const { store, update, reset } = require('./lib.js');

update(); 
console.log(store.version); // '2.0' (同步了，因为是浅拷贝，指向同一个对象)

reset();
console.log(store.version); // '2.0' ！！没变。因为 lib.js 里的 store 指向了新对象，但这里的 store 还指向旧对象
```

## 常见模块化打包方案有哪些？

### 1.CommonJS

- **环境**：服务器端（Node.js），同步加载。

- **特点**：模块在运行时加载，整个模块生成一个对象，然后从对象上读取方法。

- **语法**：

  ```
  // 模块输出 math.js
  module.exports = {
    printRandom,
    printIntRandom
  }
  
  // 加载模块
  var math = require('math')
  ```

### 2. AMD (Asynchronous Module Definition

- **环境**：浏览器端，异步加载模块，不阻塞后续语句。

- **特点**：模块加载完成后，通过回调执行依赖逻辑。

- **语法**：

  ```
  // 定义模块
  define(id?, dependencies?, factory);
  
  // 使用模块
  require([module], callback);
  ```

### 3. UMD (Universal Module Definition)

- **特点**：兼容 CommonJS 和 AMD，同时支持浏览器全局变量。
- **实现思路**：

1. 判断是否支持 Node 模块，使用 CommonJS。
2. 判断是否支持 AMD，使用 AMD。
3. 否则注册为浏览器全局变量。

```
(function (window, factory) {
  if (typeof exports === "object") {
    module.exports = factory(); // CommonJS
  } else if (typeof define === "function" && define.amd) {
    define(factory); // AMD
  } else {
    window.eventUtil = factory(); // 浏览器全局
  }
})(this, function () {
  // 模块内容
});
```

### 4.ES6 Module

- **特点**：静态化模块，编译时就能确定依赖关系，支持 Tree-shaking。
- **加载方式**：编译时加载，不是对象，而是通过 `export`/`import` 显式指定输出和输入。
- **语法**：

```jsx
// 导出模块
export { ul };

// 导入模块
import { ul } from "../test/test_ul.js";
```

- **优势**：

- 支持静态分析，便于优化和按需加载
- 与现代前端构建工具（Webpack、Vite）深度集成
- Tree-shaking 支持剔除未使用代码



### webpack热更新的原理

HMR全称 Hot Module Replacement，可以理解为模块热替换，指在应用程序运行过程中，替换、添加、删除模块，而无需重新刷新整个应用。

例如，我们在应用运行过程中修改了某个模块，通过自动刷新会导致整个应用的整体刷新，那页面中的状态信息都会丢失。

如果使用的是 HMR，就可以实现只将修改的模块实时替换至应用中，不必完全刷新整个应用

在webpack中配置开启热模块也非常的简单，如下代码：

```
const webpack = require('webpack')
module.exports = {
  // ...
  devServer: {
    // 开启 HMR 特性
    hot: true
    // hotOnly: true
  }
}
```

整个流程分为客户端和服务端，通过 `websocket` 建立起 浏览器端 和 服务器端 之间的通信

- `webpack` 会创建 `webserver` 静态服务器，让浏览器可以请求编译生成的静态资源
- 还会创建 `websocket` 服务，建立本地服务和浏览器的双向通信。
- 然后以`watch`模式启动编译，监听到文件变化后，根据配置文件对模块重新编译打包，通过`websocket` 告知浏览器执行热更新逻辑。
- 浏览器接收服务器端推送的消息，如果需要热更新，浏览器发起 `http`请求去服务器端获取新的模块资源解析并局部刷新页面。