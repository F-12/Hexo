title: 知其所以然-Binding_System_of_Vue
toc: true
thumbnail: /images/banner.jpg
author: F-12
tags:
  - Vue
  - Javascript
  - Web
nanoid: FQcqDZNuTrOT1XWQkDnPk
date: 2016-09-19 22:42:00
---


# 目标
- [x] 理解Vue如何根据data生成DOM及挂载到Document对象上
- [x] 理解Vue如何实现更新data属性值时自动更新DOM
- [x] 理解Vue如何实现v-*指令

# 引入

## DOM更新

浏览器构建DOM Tree，渲染视图，并提供DOM API更新视图。浏览器保证视图的渲染结果和DOM Tree中的数据同步。开发者经常会从服务器加载数据更新到视图上，或根据用户的交互改变视图。这些改变视图的方式可以抽象成改变视图状态数据（即数据驱动视图）。

我们一般都会使用data object保存从服务器加载到的数据或记录用户交互的状态数据，然后把这些data object里的数据同步到DOM对象的各个属性值上，触发视图的更新。

一般的做法是这样的：

```
<div>
  Hello,<span id="name"></span>
</div>

<script>
  var name = 'f-12';
  $('#name').text(name);
</script>
```
name代表data object（既可能是从服务器上加载的数据，也可能是用户交互产生的状态数据），当我们把值赋给DOM的属性（使用原生DOM API或jQuery)，修改视图后，data，object就和DOM的属性值没有关系了，假如我们再次更新name，DOM属性值并不会变化，因而视图也不会变化。我们必须再次修改DOM属性才可以更新视图，即每次状态数据发生变动时我们必须手动更新视图。

手动更新DOM显得很繁琐。前端工程更加复杂，用户交互更加复杂，单页应用更加流行，产生了更多的状态数据，这些因素也凸显了手动更新DOM的缺点。

## Data Binding
Data Binding的思想就是建立data object和DOM对象之前的关系，使得我们修改data object上的属性值时自动完成DOM对象的更新。要完成这个特性，我们很自然想到了观察者模式（发布/订阅模式）

## 观察者模式
- **定义**

> The Observer Pattern defines a one-to-many dependency between objects so that when one object changes state, all of its dependents are notified and updated automatically.

+ **原理**
{% asset_img js_observer_pattern.png %}

+ **Pull vs Push**
javascript有两种observer pattern的实现方式：pull和push。

  - Push： observable状态更新时使用变更的数据作为参数调用observer注册的回调
  - Pull： observable状态变更时简单通知observer更新，observer使用自身持有的observable引用获取感兴趣的状态。

  ( Vue中使用的Pull方式实现的observer pattern )

+ **用到Observer的地方**
  - jQuery对象的on/trigger方法
  - Node里EventEmitter的on/emmit方法
  - DOM里EventTarget的onXXX

# Vue的Data Binding
{% asset_img vue_binding_model.png %}
> Reference: vuejs.org


Web开发中比较频繁出现的场景是：
- 有很多状态数据需要同步到DOM上（DOM操作，jQuery流行的主要因素）
- 多个DOM对象用到同一个状态数据，状态发生变动时需要更新所有DOM(Data Binding，MVVM流行的推动力)

本次我们主要关注Vue的Data Binding实现的方式。

继续进行前，我们需要引入一个最简单的🌰。
```html
<div id="app">
  This is a {{ message }}.
</div>
```
```javascript
import Vue from 'vue';
const vm = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue.js!'
  }
})
```
> Reference: [vuejs.org](http://vuejs.org/guide/)

这个代码段的效果是页面渲染时，message的值会自动同步到DOM中，当修改message时，DOM会同步更新。
接下来，我们来探究一下这一切是怎么发生的。

## import
`new Vue()`之前，我们首先得引入Vue的库，我们使用ES6的`import`方式。
`import Vue from 'vue'`执行时，会首先找到`vue`包，然后执行里面的`index.js`代码，将其中导出的对象绑定到`Vue`这个标识符上。这一行代码完成Vue类的定义，导出一个构造函数来供使用者实例化。

`import`过程体现了`Vue`的扩展能力，提现了`Vue`支持自定义`directive`，`filter`，`watch`等特性的方式。但是这个不是我们这次的关注重点，略过不讲。
有了Vue的构造函数，就可以用它进行实例化了。Vue实例化时做了很多事情，支撑起了Vue的`运行时`。

**debate**  `Vue`是库(`lib`)还是框架(`framework`) ?

## 实例化过程
### `el`选项
以下段落来自Vuejs官方文档对`el`选项的描述。
- **Type:** `string | HTMLElement`

- **Restriction:** only respected in instance creation via `new`.

- **Details:**

  Provide the Vue instance an existing DOM element to mount on. It can be a CSS selector string or an actual HTMLElement.

  After the instance is mounted, the resolved element will be accessible as `vm.$el`.

  If this option is available at instantiation, the instance will immediately enter compilation; otherwise, the user will have to explicitly call `vm.$mount()` to manually start the compilation.

  <p class="tip">The provided element merely serves as a mounting point. Unlike in Vue 1.x, the mounted element will be replaced with Vue-generated DOM in all cases. It is therefore not recommended to mount the root instance to `<html>` or `<body>`.</p>

- **See also:** [Lifecycle Diagram](/guide/instance.html#Lifecycle-Diagram)

`el`选项提供`Vue`实例对象的挂载点，最终`Vue`实例将编译出一个`DOM`对象替换`el`对象的`DOM`。

`el`的值可以是一个选择器`String`也可以是一个DOM `HTMLElement`元素，这样提供我们动态挂载Vue实例对象的能力，可以先`new Vue()`时不提供`el`值，得到`vm`对象后动态决定挂载点，调用`vm.$mount()`进行手动挂载。

{% asset_img vue_el_option_init.png %}

### `data`选项
以下段落来自Vuejs官方文档对`data`选项的描述。
- **Type:** `Object | Function`

- **Restriction:** Only accepts `Function` when used in a component definition.

- **Details:**

  The data object for the Vue instance. Vue will recursively convert its properties into getter/setters to make it "reactive". **The object must be plain**: native objects such as browser API objects and prototype properties are ignored. A rule of thumb is that data should just be data - it is not recommended to observe objects with its own stateful behavior.

`data`选项的值可以是`Object`，一般在根实例中这样定义；可以是`Function`，组件中必须是`Function`。原因是javascript的对象是可变对象，这样做可以避免多个组件实例公用同样的对象引用。

实例化过程中会执行`_initData()`，在这个过程的最后一步是`observe`过程，将`data`选项转化为响应式数据。转化结果是`data`对象的`__ob__`保存新建的`Observer`对象，且此`__ob__.value`属性保存`data`选项对象。同时会遍历`data`选项对象的每个键值，使用ES6的`Object.defineProperty`定义其Getter和Setter。如果键值也是对象，会递归调用`observe`函数，即`observe(key)`。
完成`observe`调用后的`this.$data`结构
```javascript
{
  __ob__: {
    dep: {
      subs: []
    }
    value: {} //即this.$data自身
    vms: [] //所有持有此data对象的vm实例
  },
  get message() {}
  set message(newVal) {}
}
```
因为`observe`过程是在`_initData()`过程中发生，而这个过程又在`new Vue(options)`构造函数的执行中进行，所以一旦实例创建结束后，再往`data`上增加的新键值并不是响应式，因为没有经过`observe`过程转化为响应式属性。Vue解决这种场景的方式是提供了`Vue.$set`方法。

### Text Node中的`mustache`编译
#### 提取表达式
- **NodeType识别Text Node**
- **正则表达式提取expression**
  safe: `\{\{((?:.|\\n)+?)\}\}`
  unsafe: `\{\{\{((?:.|\\n)+?)\}\}\}`
- **创建包装getter**
  创建包装的getter函数时使用`new Function(string)`完成，string是上一步提取的`表达式`的值。包装过程中一个`scope`的概念，指的是当前组件上下文，对于root组件来说是`undefined`。这也是访问没有定义的变量时console里经常报错`scope.xxx`找不到等错误的原因。

  上面例子中的 `This is a {{ message }}.`提取后形成下面的`tokens`列表：
```javascript
[
  {value: 'this is a '},
  {
    html: false,        // 是否由unsafe模式匹配到的
    oneTime: false,     // 稍后介绍
    tag: true,          // 是否由safe或unsafe模式匹配到的
    value: 'message'    // 匹配到的文本值
  },
  {value: '.'}
]
```

  Text中的每一个`mustache`匹配到的文本将生成类似列表中第二个对象的一个token对象。其他token是由其余简单文本生成的。

#### 创建Text Node
  根据每个`tag`属性为true的token使用`function processTextToken (token, options) {}`处理，创建对应的Text Node对象并加入到一个新创建的`DocumentFragment`对象中。此过程中将进一步对token标记`descriptor`。`mustache`内部可能包含的过滤器调用也是在`descriptor`中标记。`token.tag`为`false`直接生成简单的Text Node。
  此时的token结构如下：

```javascript
{
  html: false,
  oneTime: false,
  tag: true,
  value: 'message',
  descriptor: {
    name: 'text',           // 表示当前token代表一个text node
    // directives/public/text预先定义的directive
    def: {
      bind() {},
      update() {}
    },
    expression: 'message',  // 由token中的值生成的表达式
    filters: undefined      // 可能包含的过滤器调用
  }
}
```

#### Link
递归编译每个DOM节点时，会对每个node生成对应的linker函数，这些linker会最终组合成一个linker。编译过程结束后，调用这个函数会开始link过程。代码概括为`compile(el, options)(this, el)`。link过程会递归调用所有的linker。

每个linker调用会创建一个`Directive`对象并添加到Vue实例的`_directives`数组中。
```
this._directives.push(
  new Directive(
    descriptor, // 编译过程中token中标记的discriptor
    this,       // 当前Vue实例
    node,       // 创建的对应DOM node对象
    host,       // 宿主DOM node
    scope,
    frag
  )
);
```

当所有的`Directive`对象添加到Vue实例的`_directives`数组中后，会遍历新增加的所有`Directive`，调用其上的`_bind`方法。（由于Vue提供了通过自定义`Directive`复用代码的方式，这里为了防止在实例化过程中调用到这些`Directive`的`_bind`方法，Vue通个`Directive.discriptor.priority`中保存自身优先级， link过程中创建的`Directive`对象都具有默认优先级。然后遍历前会现根据优先级排序。）

如下代码所示：
```javascript
  var originalDirCount = vm._directives.length
  linker()
  var dirs = vm._directives.slice(originalDirCount)
  sortDirectives(dirs)
  for (var i = 0, l = dirs.length; i < l; i++) {
    dirs[i]._bind()
  }
```

至此link过程完成了一半，继续进行下一半前，我们需要引入`Directive`类和`Watcher`类。

一个directive对象将DOM Element和data中的属性链接起来，并在内部注册一个Watchre对象，在data属性更新时调用update函数更新DOM对象。
```javascript
// 内部结构如下：
{
  vm: Object,           // Vue实例
  el: Object,           // 根DOM
  descriptor: Object,   // discriptor
  name: String,         // discriptor.name
  expression: Object,   // discriptor.expression
  arg: Object,          // discriptor.arg
  modifiers: [],    // discriptor.modifiers
  filters: [],      // discriptor.filters
  literal: ,
  _locked: Boolean, //
  _bound: Boolean,   // intial bind后为true
  _listeners: [], //
  _host: Object,        // context信息
  _scope: Object,     // context信息
  _frag: Object     // context信息
}

/**
 * @constructor
 */
export default function Directive (descriptor, vm, el, host, scope, frag) {}

Directive.prototype._bind = function () {}

/** 其他一些方法 */

```

一个Watcher对象封装一个表达式，收集该表达式的依赖项，在表达式的值发生变化时调用回调函数。
```
// 内部数据结构如下
{
  vm： Vue, // Vue实例
  expression: String | Function,   // 被封装的表达式，来自discriptor.expression
  cb: Function,
  id: Number,                      // 内部自增全局变量uid，批处理使用
  active: Boolean,
  dirty: Boolean,                  // lazy watchers标志量
  deps: [],                        //
  newDeps: [],
  depIds: Set,
  newDepIds: Set,
  prevError: null, // for async error stacks
  getter: Function,
  setter: Function,
  queued: Boolean, // for avoiding false triggers for deep and Array
  shallow: Boolean // for avoiding false triggers for deep and Array
}
/**
 * @constructor
 */
export default function Watcher (vm, expOrFn, cb, options) {}

Watcher.prototype.get = function() {}           // 计算expression的值并重新计算依赖
Watcher.prototype.set = function(value) {}      // 先应用可能的filter然后设置对应属性值
Watcher.prototype.beforeGet = function() {}     // 设置全局Dep.target为当前Watcher实例，准备收集依赖
Watcher.prototype.afterGet = function() {}      //
Watcher.prototype.update = function(shallow) {} // Subscriber接口，被Publisher调用，Vue的Publisher由Dep实现
Watcher.prototype.run = function() {}           // Batch job接口，被batcher调用
Watcher.prototype.evalute = function() {}
Watcher.prototype.addDep = function() {}        //
Watcher.prototype.depend = function() {}        // 遍历调用deps上的depend
Watcher.prototype.teardown = function() {}      // 清楚一个watcher，遍历调用deps里的removeSub()移除对其的订阅
```

在directive对象bind的过程中， 会删除`el` DOM上声明的v-*指令，然后将discriptor上定义的属性mixin到自身，并调用`discriptor.bind()`完成initial bind。
然后会new一个Watcher对象，并初始化更新DOM，将data上的初始化值更新到DOM上（这里需要例外处理v-model，将input的inline value回写到data中）。

此时我们生成的`Directive`和`Watcher`对象如下:
```
{
    _bound: true,
    _frag: undefined,
    _host: undefined,
    _listeners: null,
    _locked: false,
    _scope: undefined,
    _update() {},

    _watcher: {                   // 每个directive对象都会持有一个watcher对象
        id: 1,
        active: true,
        cb(val, oldVal) {},
        deep: undefined,
        depIds: [1],
        deps: [Watcher],
        dirty: undefined,
        expression: "message",
        filters: undefined,
        getter() {}
        newDepIds: [],
        newDeps: [],
        postProcess: null,
        preProcess: null,
        prevError: null,
        queued: false,
        scope: undefined,
        setter: undefined,
        shallow: false,
        twoWay: undefined,
        value: "hello world"
    },
    arg: undefined,
    attr: "data",               // 因为是data选项中定义的属性，所以值为data，如果只是简单text则为textContent
    descriptor: Object,         // token.descriptor
    bind: bind(),               // token.descriptor.bind
    update(value) {},           // token.descriptor.update
    el: Text,                   // DOM中的Text类实例
    expression: "message",
    filters: undefined,
    literal: undefined,
    modifiers: undefined,
    name: "text",               // 标明此directive对象注册时key为text，自定义directive时将由Vue.directive(name, Object)提供name
    vm : Vue
}
```
#### 依赖系统
前面讲解link过程时，提到了收集依赖。所以一并讲解下Vue的依赖系统。依赖系统本质上是一个发布-订阅模式的实现。`Watcher`实现订阅者职责，`Dep`实现发布者职责，发布的内容是自身值的改变，`Watcher`对应的响应是使用依赖重新计算自己的值并调用回调函数。

为此引入`Dep`类。
一个Dep对象是一个发布者，可以被多个directive（实际上是directive里面的watcher）订阅。
```javascript
// 内部结构
{
  id: Number,      // 自增的uid
  subs: []         // 实现了update方法的对象
}

export default function Dep () {}

Dep.target        // 全局变量，标志当前收集依赖的Watcher对象

Dep.prototyp.addSub = function() {}        // 增加订阅者
Dep.prototyp.removeSub = function() {}        // 移除订阅者
Dep.prototyp.depend = function() {}        // 将自身增加当Dep.target的依赖中
Dep.prototyp.notify   = function() {}        // 遍历调用subs里对象的update方法，即发布更新
```
#### Binding流程
下面使用流程图总结一下这个过程中的行为。
{% asset_img vue_binding_sequence.png %}

**Workshop**  debug `vm.message = 'Goodbye World'`时的执行时序

# 指令处理
`Vue`会根据写在`template`里的模板编译出`DOM`对象。`template`模板本身是`html`代码，`Vue`的功能通过写在html代码里的指令，`mustache`代码以及自定义`tag`实现。
模板的编译本质上是手动创建`DOM`树的过程。Vue中指令主要有`v-*`指令以及`{{ mustache }}`指令，还有`number`, `transition`, `keep-alive`等属性指令，还有`<component></component>`等元素指令，还有自定义组件标签指令。

## Terminal指令
### `v-if`指令
#### `v-if` **compile**
开始前，我们需要小小修改下实例代码，如下。
```html
<div id="app">
  This is a <span v-if="show">{{ message }}</span>.
</div>
```
```javascript
import Vue from 'vue';
const vm = new Vue({
  el: '#app',
  data: {
    show: true,
    message: 'Hello Vue.js!'
  }
})
```

结合之前讲的过程，直接从compile阶段开始。
```javascript
function compileElement(el, options) {
  // preprocess textareas.
  // textarea treats its text content as the initial value.
  // just bind it as an attr directive for value.
  // ...

  var linkFn;
  var hasAttrs = el.hasAttributes();
  var attrs = hasAttrs && toArray(el.attributes);
  // check terminal directives (for & if)
  if (hasAttrs) {
    linkFn = checkTerminalDirectives(el, attrs, options);
  }
  // check element directives
  if (!linkFn) {
    linkFn = checkElementDirectives(el, options);
  }
  // check component
  if (!linkFn) {
    linkFn = checkComponent(el, options);
  }
  // normal directives
  if (!linkFn && hasAttrs) {
    linkFn = compileDirectives(attrs, options);
  }
  return linkFn;
}
```
上面代码会从`el`选项指示的挂载点`div#app`开始，可以断定，在实例代码中，会执行两次，第一次针对`div#app`, 第二次针对`span`。跳过对`div#app`的编译过程，直接看对`span`执行的过程。
代码标明对指令的处理主要经过了两个过程：check和compile。其中check会分别检查**terminal directive**，**element directive**， **component**。其中**terminal directive**指的是`v-*`指令，**element directive**指类似`<component></component>`这样的自定义标签指令，**component**指令指自Vue组件。
实例代码比较简单所以只会执行`checkTerminalDirectives`。

```javascript
function checkTerminalDirectives(el, attrs, options) {
  // skip v-pre
  // ...
  // skip v-else block, but only if following v-if
  // ...
  var attr, name, value, modifiers, matched, dirName, rawName, arg, def, termDef;
  for (var i = 0, j = attrs.length; i < j; i++) {
    attr = attrs[i];
    name = attr.name.replace(modifierRE, '');
    if (matched = name.match(dirAttrRE)) {
      def = resolveAsset(options, 'directives', matched[1]);
      if (def && def.terminal) {
        if (!termDef || (def.priority || DEFAULT_TERMINAL_PRIORITY) > termDef.priority) {
          termDef = def;
          rawName = attr.name;
          modifiers = parseModifiers(attr.name);
          value = attr.value;
          dirName = matched[1];
          arg = matched[2];
        }
      }
    }
  }

  if (termDef) {
    return makeTerminalNodeLinkFn(el, dirName, value, options, termDef, rawName, arg, modifiers);
  }
}
```
这个过程会遍历element attribute node列表，`name.match(dirAttrRE)`来匹配到需要处理的`v-*`指令。这里`dirAttrRE`是正则表达式：`/^v-([^:]+)(?:$|:(.*)$)/`。
对每个特定的terminal directive，`def = resolveAsset(options, 'directives', matched[1])`获取内置的指令`def`对象, 实例中即为`def = resolveAsset(options, 'directives', 'if')`。Terminal directive 的`def`对象在'src/directives/public/'目录下定义，每个`def`对象定义了`bind`过程中调用的接口方法, 如下所示：
```javascript
{
  priority: DEFAULT_TERMINAL_PRIORITY,
  terminal: true,
  bind() {},
  update(value) {},
  insert() {},
  remove() {},
  updateRef() {},
  unbind() {}
}
```
这个`def`对象，讲解`Text Node`编译过程中也涉及到了，当时只有`bind`和`update`方法。

#### `v-if` link
编译完成后，将进行`link`过程。此过程和Text Node的link过程基本一致，最终会执行如下代码：
```javascript
this._directives.push(
  new Directive(
    descriptor, // 编译过程中token中标记的discriptor
    this,       // 当前Vue实例
    node,       // 创建的对应DOM node对象
    host,       // 宿主DOM node
    scope,
    frag
  )
);

// descriptor:
{
  arg: undefined,
  attr: 'v-if',
  def: {},              // 上文有详述，略去细节
  expression: 'show',
  filters: undefined,
  modifiers: {}         // 空对象
  name: 'if',
  raw: 'show'
}
// node: <span v-if="show"> {{ show }}</span>
```
#### `v-if` **bind**
上面**link**过程结束后，会开始**bind**过程，具体请见上一节。`v-if`的**bind**过程中多态部分是执行`v-if def`对象上的`bind() {}`的过程。如下代码所示：
```javascript
function bind() {
    var el = this.el;
    if (!el.__vue__) {
      // check else block
      // ...
      // check main block
      this.anchor = createAnchor('v-if');
      replace(el, this.anchor);
    } else {
      // debug info
    }
}
```
这个过程其实就是简单的创建了一个空白的Text Node替换了`<span></span>`DOM元素，所以页面中开始出现`{{ message }}`会出现短暂地变成空白。但是`createAnchor`的设计意图可以深究一下，目前所知就是为了和后面的**初始化update**结合使用，`update`环节会获取`show`的值，根据值计算节点, 调用`def.insert`方法，将正式DOM插入到`createAnchor`所在位置。

**bind**的其他过程同上节所示。

### `v-for`指令
`v-for`指令的**compile**和**link**过程和`v-if`指令大致相同，区别仅在**bind**过程调用`def.bind`和`def.update`上。
但是`v--for`指令有性能优化的问题要解决，比如列表更新时如何最少化DOM操作，如何进行实例缓存等。
Vue使用`diff`算法解决列表渲染中列表动态变化的问题。

### 其他`v-*`指令
类似的处理，但是涉及不同情况的细节处理。可以细看`src/directive/public/`目录下的代码。

## `Element`指令
### `<component></component>`指令

### `<slot></slot>`指令
这个也很简单，处理node attribute时检查slot属性，然后替换父节点内的name匹配的slot节
点即可。

## `Custom Component`指令

## 推荐阅读
- [banama/abountVue](https://github.com/banama/aboutVue)
- [jinks](http://jiongks.name/blog/vue-code-review/)
