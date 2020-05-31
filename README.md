# webpack 自定义loader

## 什么是loader

由于webpack 只能解析 JavaScript 和 JSON 文件，当引入非 JavaScript 文件时，要使用 Webpack的loader机制， loader让 webpack 能够去处理其他类型的文件，并将它们转换为有效模块，以供应用程序使用，以及被添加到依赖图中。

loader 模块需要导出一个函数。 这个导出的函数的工作就是获得处理前的原内容，对原内容执行处理后，返回处理后的内容。

## loader基础

### loader配置

Webpack配置里的 `module.rules` 数组配置了一组规则，告诉 webpack在遇到哪些文件时使用哪些 Loader 去加载和转换。

如 webpack里的配置文件webpack.config.js

```js
module: {
    rules: [
      {
        test: /\.js$/,
        // 把 ES6 转换成 ES5
        use: ['babel-loader'],
        exclude: /node_modules/
      },
      {
        test: /\.css$/,
        // postcss-loader：扩展 CSS 语法，使用下一代 CSS，在3-5使用 PostCSS中有介绍。
		// css-loader：加载 CSS，支持模块化、压缩、文件导入等特性。
		// style-loader：把 CSS 代码注入到 JavaScript 中，通过 DOM 操作去加载 CSS。
        use: ['style-loader', 'css-loader', 'postcss-loader']
      },
      {
        test: /\.(png|jpg|jpeg|gif|eot|ttf|woff|woff2|svg|svgz)(\?.+)?$/,
        use: [{
          // url-loader：能在文件很小的情况下以 base64 的方式把文件内容注入到代码中去
          loader: 'url-loader',
          options: {
            limit: 10000
          }
        }]
      }
    ]
  },
```

加载es6、样式、图片文件，都有其对应的loader

rules的选项配置方式如下：

1. `test` 属性，用于标识出应该被对应的 loader 进行转换的某个或某些文件。
2. `use` 属性，表示进行转换时，应该使用哪个 loader，可以指定一个或多个loader，当多个loader时，其执行顺序是从右到左。

### 获取loader的options

当使用loader时，Webpack是如何获取到用户传入的options

使用 loader-utils 中提供的 getOptions 方法 来提取给定 loader 的 option。

```js
const loaderUtils = require('loader-utils')
module.exports = function(source) {
  // 获取到用户给当前 Loader 传入的 options
  const options = loaderUtils.getOptions(this)
  return source
}
```

获取到options就是webpack配置 `module.rules` 里的options属性

```js
{
  test: /\.(png|jpg|jpeg|gif|eot|ttf|woff|woff2|svg|svgz)(\?.+)?$/,
  use: [{
    // url-loader：能在文件很小的情况下以 base64 的方式把文件内容注入到代码中去
    loader: 'url-loader',
    options: {
      limit: 10000
    }
  }]
}
```

### 返回结果

```
module.exports = function(source, map, meta) {
  // 获取到用户给当前 Loader 传入的 options
  const options = loaderUtils.getOptions(this)
  return source
}
```

```source```，上一个 loader 产生的结果或者资源文件

`map` 和 `meta` 参数是可选的

map，传递一个可选的 SourceMap 结果

### this.cacheable

设置是否可缓存标志的函数：

```typescript
cacheable(flag = true: boolean)
```

默认情况下，loader 的处理结果会被标记为可缓存。调用这个方法然后传入 `false`，可以关闭 loader 的缓存。

一个可缓存的 loader 在输入和相关依赖没有变化时，必须返回相同的结果。这意味着 loader 除了 `this.addDependency` 里指定的以外，不应该有其它任何外部依赖。

### this.async 

loader 将会异步地回调。返回 `this.callback`。

loader 有同步和异步之分，同步会从头到尾执行，但是一些操作如果是同步，就会阻塞构建进程，如读取本地文件，网络请求，所以可以采取异步操作。

## 编写自定义loader

下面编写一个自定义loader ```replace-html-loader```，该loader的作用是，解析引入的html模板。

```vue
@include(../templates/list.html)
```

转换成 ```list.html``` 里面的内容

该loader的使用场景是，在多个文件中，当使用同一段html代码时，可以通过引入模板路径，来渲染模板

replace-html-loader.js

```js
const fs = require('fs')
const path = require('path')
const loaderUtils = require('loader-utils')
module.exports = function(source) {
  // 关闭该 Loader 的缓存功能
  this.cacheable(false)
  // 获取到用户给当前 Loader 传入的 options
  const options = loaderUtils.getOptions(this)
  const { key } = options
  const reg = new RegExp(`(@${key}\\()(.*?)\\)`)
  const regResult = reg.exec(source)
  // 告诉 webpack本次转换是异步的，Loader 会在 callback 中回调结果
  var callback = this.async()
  if (regResult) {
    const request = loaderUtils.urlToRequest(regResult[2])
    // 像 require 语句一样获得指定文件的完整路径
    this.resolve('/src', request, (err, rs) => {
      if (err) {
        rs = path.resolve(this.resourcePath, '../', request)
      }
      try {
        var layoutHtml = fs.readFileSync(rs, 'utf-8')
      } catch (error) {
        throw error
      }
      this.addDependency(rs)
      // 用读取的文件内容 替换 @include(../templates/list.html)
      res = source.replace(regResult[0], layoutHtml)
      callback(null, res)
    })
  }  else {
    callback(null, source)
  }
}
```

webpack的config增加loader配置：

```js
{
  test: /\.vue$/,
  use: ['vue-loader',
    {
      // 因为该loader还没有上传的npm，所以要指定该loader的路径，才能使用
      loader: path.resolve(__dirname, './replace-html-loader.js'),
      options: {
        key: 'include'
      },
    }
  ]
}
```

因为是在 Vue 的项目中使用这个loader的，所以写在Vue 的 ```vue-loader``` 后面

然后在Vue 组件中使用

```vue
<template>
  <div id="user-list">
    <div>@include(../templates/list.html)</div>
  </div>
</template>

<script>
export default {
  name: 'user-list',
  data() {
    return {
      tableData: [{
        date: '2016-05-02',
        name: '王小虎',
        address: '上海市普陀区金沙江路 1518 弄'
      }, {
        date: '2016-05-04',
        name: '王小虎',
        address: '上海市普陀区金沙江路 1517 弄'
      }, {
        date: '2016-05-01',
        name: '王小虎',
        address: '上海市普陀区金沙江路 1519 弄'
      }, {
        date: '2016-05-03',
        name: '王小虎',
        address: '上海市普陀区金沙江路 1516 弄'
      }]
    }
  },
  methods: {
    
  }
}
</script>

<style>
</style>
```

list.html里面的内容可以使用当前Vue组件中的变量

```html
<el-table
  :data="tableData"
  style="width: 100%">
  <el-table-column
    prop="date"
    label="日期"
    width="180">
  </el-table-column>
  <el-table-column
    prop="name"
    label="姓名"
    width="180">
  </el-table-column>
  <el-table-column
    prop="address"
    label="地址">
  </el-table-column>
</el-table>
```

效果图：

![](https://user-gold-cdn.xitu.io/2020/6/1/1726bc6cb68e9412?w=1510&h=575&f=png&s=39647)
