---
title: webpack的插件体系结构
date: 2017-09-25 13:58:51
tags:
    - webpack
    - javascript
    - 源码分析
    - 插件体系
---

# 前置知识
1. javascript
1. webpack基本使用
2. node.js异步编程基础

# 环境信息
* webpack 1.13.2
* node.js 6.9.5

# 关键词
1. webpack
2. 插件体系
3. Tapable.js

----

# 正文内容

## 引入
webpack概括来讲是一个打包工具，但也不仅仅是一个打包工具。这个从它复杂繁多的配置项里就可见一斑。webpack不仅能完成前端代码的打包，配合其他工具，它能做的事情几乎没有边界。配合babel完成es6到es5代码的转码，配合webpack-dev-server完成代码serve及开发时的hot-reload，配合uglify.js完成代码混淆，配合vue-loader转换`.vue`文件实现vue的单文件组件特性。而webpack完成这一强大扩展性的核心在于其基于`Tapable.js`的插件体系结构。`Tapable.js`本质上是一个发布-订阅模式的具体实现。

下面将从发布-订阅模式开始简要介绍webpack的插件结构。

## 从发布-订阅模式谈起
一个简单的发布-订阅模式中包括发布者，订阅者；发布者提供订阅方法和发布方法；订阅者提供发布回调方法。

### 模拟一个简单的发布-订阅模式
下面code展示了一个最简单的发布-订阅模式（推的形式）的实现。
```javascript
// 发布者，的管理节点，维护订阅者列表，发布具体数据
function Publisher() {
  this._subs = [];
}

// 订阅方法，callback是订阅者
Publisher.prototype.subscribe = function(callback) {
  this._subs.push(callback);
}

// 发布方法, 发布者通过发布方法传递数据给订阅者
Publisher.prototype.publish = function(data) {
  this._subs.forEach(function(sub) {
    sub(data);
  });
}

// 取消订阅方法，发布者移除某个订阅者
Publisher.prototype.unsubscribe = function(callback) {
  this._subs = this._subs.filter(function(sub) {
    return sub !== callback;
  });
}
```
在这个简单模型里，因为javascript里函数也是对象，所以可以将订阅者和发布回调方法使用同一个callback函数实现。发布者还可以提供一个可选的取消订阅方法供客户代码取消特定订阅者，不再继续接受发布者发布的数据。

### 浏览器中的发布-订阅模式
真实环境中发布-订阅模式随处可见。

最常见的是浏览器里的事件机制。`EventTarget`是发布者，`EventTarget.prototype.addEventListener`是订阅方法，`EventTarget.prototype.dispatchEvent`是发布方法，`EventTarget.prototype.removeEventListener`是取消订阅方法。每一个继承了EventTarget的对象都是发布者。比较需要留意的是`Element`,`window`,`document`和`XMLHttpRequest`四个对象继承了`EventTarget`。

此外node.js的`EventEmitter`及继承了`EventEmitter`的对象也是发布者的实现。

## Tapable.js插件引擎
Tapable.js是webpack实现的一个插件引擎，本质上也是一个发布者的实现。每个Tapable类的实例都是一个发布者，内部维护了命名插件列表。同时Tapable类提供了一套管理订阅者的API，提供不同形式的调用发布回调方法。

作为webpack实现插件结构的基础类，我们有必要熟悉这个类提供的基础能力。下面将简要介绍Tapable.js的API的功能。

每个API通过一个代码块来描述，每个代码块包含四个部分：

- API的功能描述
- API定义的方法签名
- 一段测试代码
- 测试代码的示例输出

1. Tapable类定义

    ```javascript
      // 构造函数， 初始化命名插件列表
      function Tapable() {}

      // 注册插件
      Tapable.prototype.plugin = function(name, fn) {}

      // 应用插件，代理到插件实例的apply方法
      Tapable.prototype.apply = function() {}      

      // 重置同步插件列表开始下标
      Tapable.prototype.restartApplyPlugins                 
    ```

1. applyPlugins

    ```javascript  
    // 调用一个同步插件列表
    Tapable.prototype.applyPlugins = function(name) {}

    // 对于如下测试代码：
    var tapable = new Tapable();

    tapable.plugin('applyPlugins', function() {
      console.log('this is plugin 1');
    })

    tapable.plugin('applyPlugins', function() {
      console.log('this is plugin 2');
    })

    tapable.plugin('applyPlugins', function() {
      console.log('this is plugin 3');
    })

    tapable.applyPlugins('applyPlugins');

    // 输出为
    // this is plugin 1
    // this is plugin 2
    // this is plugin 3
    ```

2. applyPluginsWaterfall

    ```javascript
    // 链式调用一个同步插件列表, 以init为初始值链式调用一个同步插件列表，列表中每个插件的返回值成为下一个插件的第一个参数
    Tapable.prototype.applyPluginsWaterfall = function(name, init) {}

    // 对于如下测试代码：
    var tapable = new Tapable();

    tapable.plugin('applyPluginsWaterfall', function(value) {
      console.log('this is plugin 1', ' i get value: ' + value++);
      return value;
    })

    tapable.plugin('applyPluginsWaterfall', function(value) {
      console.log('this is plugin 2', ' i get value: ' + value++);
      return value;
    })

    tapable.plugin('applyPluginsWaterfall', function(value) {
      console.log('this is plugin 3', ' i get value: ' + value++);
      return value;
    })

    var last = tapable.applyPluginsWaterfall('applyPluginsWaterfall', 0);
    console.log('the last value: ' + last);

    // 输出为：
    // this is plugin 1  i get value: 0
    // this is plugin 2  i get value: 1
    // this is plugin 3  i get value: 2
    // the last value: 3
    ```

3. applyPluginsBailResult

    ```javascript
    // 调用一个同步插件列表，第一个返回非undefinded的插件可提前返回
    Tapable.prototype.applyPluginsBailResult = function() {}

    // 对于如下测试代码：
    var tapable = new Tapable();

    tapable.plugin('applyPluginsBailResult', function() {
      console.log('this is plugin 1');
    })

    tapable.plugin('applyPluginsBailResult', function() {
      console.log('this is plugin 2');
      return 2;
    })

    tapable.plugin('applyPluginsBailResult', function() {
      console.log('this is plugin 3');
    })

    var last = tapable.applyPluginsBailResult('applyPluginsBailResult');
    console.log('the last value: ' + last);

    // 输出为
    // this is plugin 1
    // this is plugin 2
    // the last value: 2
    ```

4. applyPluginsAsync

    ```javascript
    // 链式调用一个异步插件列表，上一个插件执行结束后执行下一个插件
    Tapable.prototype.applyPluginsAsync = functoin(name) {}
    Tapable.prototype.applyPluginsAsyncSeries = Tapable.prototype.applyPluginsAsync

    // 对于如下测试代码：
    var tapable = new Tapable();

    tapable.plugin('applyPluginsAsync', function(callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 1 with delay: ' + delay );
        callback();
      }, delay);
    })

    tapable.plugin('applyPluginsAsync', function(callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 2 with delay: ' + delay );
        callback();
      }, delay);
    })

    tapable.plugin('applyPluginsAsync', function(callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 3 with delay: ' + delay );
        callback();
      }, delay);
    })

    tapable.applyPluginsAsync('applyPluginsAsync', function() {
      console.log('done');
    });

    // 输出为：
    // this is plugin 1 with delay: 365.28033813757423
    // this is plugin 2 with delay: 615.922299406326
    // this is plugin 3 with delay: 327.0493787299531
    // done
    ```

5. applyPluginsAsyncWaterfall

    ```javascript  
    // 链式调用一个异步插件列表，上一个插件执行结束后执行下一个插件，上一插件的值通过第一个参数传递到下一个插件
    Tapable.prototype.applyPluginsAsyncWaterfall = function(name, init, callback) {}    

    // 对于如下测试代码:
    var tapable = new Tapable();

    tapable.plugin('applyPluginsAsyncWaterfall', function(value, callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 1 with delay: ' + delay );
        callback(null, value + '-1');
      }, delay);
    })

    tapable.plugin('applyPluginsAsyncWaterfall', function(value, callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 2 with delay: ' + delay );
        callback(null, value + '-2');
      }, delay);
    })

    tapable.plugin('applyPluginsAsyncWaterfall', function(value, callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 3 with delay: ' + delay );
        callback(null, value + '-3');
      }, delay);
    })

    tapable.applyPluginsAsyncWaterfall('applyPluginsAsyncWaterfall', '0', function(err, message) {
      console.log('the last message: ' + message);
      console.log('done');
    });

    // 输出为：
    // this is plugin 1 with delay: 293.49776146010197
    // this is plugin 2 with delay: 543.2495323181399
    // this is plugin 3 with delay: 932.6130373949233
    // the last message: 0-1-2-3
    // done
    ```

6. applyPluginsParallel

    ```javascript  
    // 并发调用一个异步插件列表，所有插件结束后返回
    Tapable.prototype.applyPluginsParallel = function(name) {}

    // 对于如下测试代码:
    var tapable = new Tapable();

    var totalDelay = 0;
    tapable.plugin('applyPluginsParallel', function(callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 1 with delay: ' + delay );
        totalDelay += delay;
        callback();
      }, delay);
    })

    tapable.plugin('applyPluginsParallel', function(callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 2 with delay: ' + delay );
        totalDelay += delay;
        callback();
      }, delay);
    })

    tapable.plugin('applyPluginsParallel', function(callback) {
      var delay = Math.random() * 1000;
      setTimeout(function() {
        console.log('this is plugin 3 with delay: ' + delay );
        totalDelay += delay;
        callback();
      }, delay);
    })

    tapable.applyPluginsParallel('applyPluginsParallel', function(err) {
      console.log('totalDelay: ' + totalDelay);
      console.log('done');
    });

    // 输出为：
    // this is plugin 2 with delay: 45.88772912860905
    // this is plugin 3 with delay: 353.1725171220388
    // this is plugin 1 with delay: 491.4572602446987
    // totalDelay: 890.5175064953467
    // done
    ```

7. applyPluginsParallelBailResult

    ```javascript  
    // 并发调用一个异步插件列表，所有插件结束后返回, 返回最后一个有返回值的插件的返回值
    Tapable.prototype.applyPluginsParallelBailResult = function(name, callback) {}  

    // 对于如下测试代码:
    var tapable = new Tapable();

    var totalDelay = 0;
    tapable.plugin('applyPluginsParallelBailResult', function(callback) {
      var delay = Math.random() * 1000;
      console.log('this is plugin 1 with delay: ' + delay );
      setTimeout(function() {
        totalDelay += delay;
        callback();
      }, delay);
    })

    tapable.plugin('applyPluginsParallelBailResult', function(callback) {
      var delay = Math.random() * 1000;
      console.log('this is plugin 2 with delay: ' + delay );
      setTimeout(function() {
        totalDelay += delay;
        var time = Date.now();
        console.log('plugin 2 returns at time: '+ time);
        callback(null, time);
      }, delay);
    })

    tapable.plugin('applyPluginsParallelBailResult', function(callback) {
      var delay = Math.random() * 1000;
      console.log('this is plugin 3 with delay: ' + delay );
      setTimeout(function() {
        totalDelay += delay;
        var time = Date.now();
        console.log('plugin 3 returns at time: '+ time);
        callback(null, time);
      }, delay);
    })

    tapable.applyPluginsParallelBailResult('applyPluginsParallelBailResult', function(err, value) {
      console.log('totalDelay: ' + totalDelay);
      console.log('the last value: ' + value);
      console.log('done');
    });

    // 输出为:
    // this is plugin 1 with delay: 635.2381208798292
    // this is plugin 2 with delay: 347.9609880914285
    // this is plugin 3 with delay: 90.2209742134965
    // plugin 3 returns at time: 1506257839951
    // plugin 2 returns at time: 1506257840208
    // totalDelay: 1073.4200831847543
    // the last value: 1506257840208
    // done
    ```

    **[获取Tapable.js测试代码]()**

## wepack的插件结构

### 通过插件的方式组织代码
webpack因为其宏大的愿景，为了满足各种情况下的使用需求，必须使用一个可扩展的方式来实现。现在我们知道webpack使用了插件的体系结构来实现的。实现其插件体系结构的核心是Tapable.js这个插件引擎。但是Tapable.js只提供了基础的插件插件核心，如何使用Tapable.js实现webpack繁多复杂的功能则是另一个内容庞大的话题。

webpack中每一个继承了`Tapable`的类都是一个可通过插件扩展的节点。

比较重要的类有：

- `Compiler`:    编译器抽象，顶层插件管理者
- `Compilation`：编译过程抽象
- `Resolver`：   路径解析过程抽象
- `Parser`：     ast解析过程抽象
- `NormalModuleFactory`： 模块构建过程抽象
- `Template`：   编译结果输出抽象

每个类所做的工作不是本文关注重点，按下不表。

webpack通过`webpack.config.js`的`plugins`配置项将插件注册入口暴露给开发者。配置在`plugins`里的插件实例通过`apply`方法获得`Compiler`实例引用。`Compiler`管理着webpack顶层的生命周期。一个生命周期对应着一个命名插件列表。获得`Compiler`实例引用的插件可以在其不同生命周期注册新的回调。回调通过参数获取其他插件节点（包括`Compilation`，`Resolver`，`NormalModuleFactory`等）的实例。回调执行时可以在新持有的插件节点实例上注册新的回调。

一个生命周期内注册的插件只能是还未执行的生命周期的插件，已执行过的生命周期上注册的新的插件不会被触发执行。

要在特定生命周期上注册插件，我们需要先获得对应节点的实例。而获得得对应节点的实例我们可能需要通过该节点实例的管理者的特定生命周期间接获取。

### 一个典型的webpack插件生命周期
下面是一个典型的webpack插件的定义（来自[webpack1.x documentation - how-to-write-a-plugin](http://webpack.github.io/docs/how-to-write-a-plugin.html)）。

```javascript
function HelloWorldPlugin(options) {
  // Setup the plugin instance with options...
}

HelloWorldPlugin.prototype.apply = function(compiler) {
  compiler.plugin('done', function() {
    console.log('Hello World!');
  });
};

module.exports = HelloWorldPlugin;
```

一个插件通常在`webpack.config.js`配置文件里创建实例，如下所示（改自[webpack1.x documentation - using-plugins](http://webpack.github.io/docs/using-plugins.html)）：

```
module.exports = {
    plugins: [
        new webpack.HelloWorldPlugin()
    ]
};
```

在执行构建命令：
```shell
$ webpack -c webpack.config.js
```
时会执行`compiler.apply`方法，这个方法遍历执行所有配置的插件实例上的`apply`方法。`HelloWorldPlugin`插件实例会在`compiler`的`done`这一生命周期注册回调。当`compiler`通过上面描述的`Tapable.js`的`applyXXX`方法发布`done`这个事件时，`HelloWorldPlugin`实例注册的`done`回调会被执行。

webpack本身就是由这种方式通过各种插件如同积木一样搭建起来，这些插件有决定`require`函数解析文件路径方式的，有决定全局变量查找方式的，有决定模块编译之后输出格式的，有决定抽取公共模块方式的...

## 写在最后

在以webpack为工具链核心的工作流里，了解webpack实现是很有必要的。webpack实现时是通过插件的架构组织的代码，其插件的核心则是`Tapable.js`这个模块。`Tapable.js`本质上又一个发布-订阅模式的变体。上文先解释了一个简单的发布订阅模式的实现，然后通过代码描述了`Tapable.js`的API功能，最后通过一个简单的插件的例子描述了一个webpack插件的生命周期——定义，实例化和执行。

一些微小的工作，希望对大家有所帮助。

----

# 相关资料
- [webpack源码](https://github.com/webpack/webpack/tree/webpack-1)
- [webpack 1.x Documentation](http://webpack.github.io/docs/)
- [Tapable.js源码](https://github.com/webpack/tapable/tree/v0.1.9)
