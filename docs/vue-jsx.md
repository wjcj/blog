Vue JSX 深入解析
====

*本文也发布在[这里](https://zhuanlan.zhihu.com/p/59434351)。*

当引入 `babel-plugin-transform-vue-jsx` 这个 Babel 插件后，就可以在 Vue 使用 JSX 语法。本文主要通过介绍 `babel-plugin-transform-vue-jsx` 转换 JSX 的过程，让你在Vue 中得心应手的使用 JSX 语法。

在此之前先做一个小测试，Vue render 选项下面两种写法 change 事件触发后分别输出什么？

``` js
render(h) {
  function handler1() { console.log(1) }
  function handler2() { console.log(2) }
  function handler3() { console.log(3) }
  function handler4() { console.log(4) }
  // 1
  return <MyCpt
    {...{ on: { change: handler1 } }}
    onChange={ handler2 }
    onChange={ handler3 }
    {...{ on: { change: handler4 } }}/>
  // 2
  return <MyCpt
    onChange={ handler1 }
    {...{ on: { change: handler2 } }}
    onChange={ handler3 }
    {...{ on: { change: handler4 } }}/>
}
```

结果是分别为 `1 3 4`、`1 2 3 4`，如果这个结果出乎你的预期之外，你继续阅读可以解答你的疑惑，但之前需要先了解一下 `createElement`。

## `createElement`

在没有使用 `babel-plugin-transform-vue-jsx` 之前，可以通过定义 `render` 选项作为字符串模板的代替方案，`render` 函数接收一个 `createElement` 方法作为第一个参数用来创建 `VNode` 供 `render` 返回：

``` js
render: function (createElement) {
  var vNode = createElement(
    // {String | Object | Function}
    // HTML 标签名称，或组件选项对象，或一个最终解析(resolve)为 HTML 标签名称或组件选项对象的异步函数。必选参数。
    'div',

    // {Object} 一个数据对象，和模板中用到的属性相对应。可选参数。
    {
      class: { foo: true },
      props: { myProp: 'bar' },
      on: { click: this.clickHandler },
      // ...
    },

    // {String | Array}
    // 子虚拟 DOM 节点(children VNode)组成，或使用 `createElement()` 生成，
    // 或使用字符串的“文本虚拟 DOM 节点(text VNode)”。可选参数。
    [
      'Some text comes first.',
      createElement('h1', 'A headline'),
      createElement(MyComponent, {
        props: {
          someProp: 'foobar'
        }
      })
    ]
  )
  return vNode
}
```

`createElement` 唯一难点就是要了解数据对象(The Data Object In-Depth)，而定义数据对象的过程就是一个对属性进行分类的过程。

``` js
/* 数据对象(The Data Object In-Depth) */
{
  // 和 `v-bind:class` 的 API 相同，
  // 也接收一个字符串、对象，或字符串对象构成的数组
  class: {
    foo: true,
    bar: false
  },
  // 和 `v-bind:style` 的 API 相同，
  // 也接收一个字符串、对象，或字符串对象构成的数组
  style: {
    color: 'red',
    fontSize: '14px'
  },
  // 普通的 HTML 属性
  attrs: {
    id: 'foo'
  },
  // 组件 props
  props: {
    myProp: 'bar'
  },
  // DOM 属性
  domProps: {
    innerHTML: 'baz'
  },
  // 事件处理程序嵌套在 `on` 字段下，
  // 然而不支持在 `v-on:keyup.enter` 中的修饰符。
  // 因此，你必须手动检查
  // 处理函数中的 keyCode 值是否为 enter 键值。
  on: {
    click: this.clickHandler
  },
  // 仅对于组件，
  // 用于监听原生事件，而不是组件内部
  // 使用 `vm.$emit` 触发的事件。
  nativeOn: {
    click: this.nativeClickHandler
  },
  // 自定义指令。
  // 注意，由于 Vue 会追踪旧值，
  // 所以不能对`绑定`的`旧值`设值
  directives: [
    {
      name: 'my-custom-directive',
      value: '2',
      expression: '1 + 1',
      arg: 'foo',
      modifiers: {
        bar: true
      }
    }
  ],
  // 作用域插槽(scoped slot)的格式如下
  // { name: props => VNode | Array<VNode> }
  scopedSlots: {
    default: props => createElement('span', props.text)
  },
  // 如果此组件是另一个组件的子组件，
  // 需要为插槽(slot)指定名称
  slot: 'name-of-slot',
  // 其他特殊顶层(top-level)属性
  key: 'myKey',
  ref: 'myRef'
}
```

React 中组件上传递的所有属性都会均挂载在 `props` 上，所以不需要做属性的分类。而 Vue 中不是这样，原因是由于 Vue 2.0 的 `vnode` 格式与 `React` 有所不同，所以 Vue 中需要在传递属性对象供 `createElement` 生成 `VNode` 之前需要对属性进行分类，这个分类后的属性对象在 Vue 中称为数据对象，数据对象保证将预期的属性添加到正确的 `vnode` 属性上。比如一个组件中 `click: handler` 即可以定义在 `on` 或 `nativeOn` 中会有完全不同的表现，这完全取决于你的期望。

`createElement` 虽然使得 `render` 选项更具有 JavaScript 的编程能力，但写起来过于复杂，主要体现在写数据对象的过程，但这是 Vue 灵活使用 JSX 的基础，后面会频繁提到数据对象，如果对 `createElement` 的使用还不了解，那需要回炉一下。

## `babel-plugin-transform-vue-jsx`

在 Vue 使用 `[babel-plugin-transform-vue-jsx](https://link.zhihu.com/?target=https%3A//github.com/vuejs/babel-plugin-transform-vue-jsx%23difference-from-react-jsx)` 后可以让我们回归到更接近模板的语法上。下面是 `babel-plugin-transform-vue-jsx` 文档中两段示例代码：

``` js
render (h) {
  return h('div', {
    // Component props
    props: {
      msg: 'hi'
    },
    // normal HTML attributes
    attrs: {
      id: 'foo'
    },
    // DOM props
    domProps: {
      innerHTML: 'bar'
    },
    // Event handlers are nested under "on", though
    // modifiers such as in v-on:keyup.enter are not
    // supported. You'll have to manually check the
    // keyCode in the handler instead.
    on: {
      click: this.clickHandler
    },
    // For components only. Allows you to listen to
    // native events, rather than events emitted from
    // the component using vm.$emit.
    nativeOn: {
      click: this.nativeClickHandler
    },
    // class is a special module, same API as `v-bind:class`
    class: {
      foo: true,
      bar: false
    },
    // style is also same as `v-bind:style`
    style: {
      color: 'red',
      fontSize: '14px'
    },
    // other special top-level properties
    key: 'key',
    ref: 'ref',
    // assign the `ref` is used on elements/components with v-for
    refInFor: true,
    slot: 'slot'
  })
}
render (h) {
  return (
    <div
      // normal attributes or component props.
      id="foo"
      // DOM properties are prefixed with `domProps`
      domPropsInnerHTML="bar"
      // event listeners are prefixed with `on` or `nativeOn`
      onClick={this.clickHandler}
      nativeOnClick={this.nativeClickHandler}
      // other special top-level properties
      class={{ foo: true, bar: false }}
      style={{ color: 'red', fontSize: '14px' }}
      key="key"
      ref="ref"
      // assign the `ref` is used on elements/components with v-for
      refInFor
      slot="slot">
    </div>
  )
}
```

## 普通的JSX属性 & 扩展属性

上面示例是普通的JSX属性传递方式，使用引号来定义以字符串为值的属性，使用大括号来定义以 JavaScript 表达式为值的属性。 唯一需要注意的就是部分属性需要根据数据对象的对属性的分类要求增加额外的前缀（如 `on`、`nativeOn`、`domProps`）。其中传递普通 html 属性（`normal html attributes`）和组件属性（`Component props`）时可以看出两者 JSX 属性写法上是相同的，只是组件属性（`Component props`）还需要在子组件的 `props` 选项中声明才能生效。

同时插件支持以 `JSX Spread` 的方式传递属性，后文统一将这种方式称为**扩展属性**方式。即使用 `...` 作为扩展操作符来传递整个属性对象。这种方式需要注意的就是属性对象的定义需要遵守数据对象的约定格式，原因在后面会提到。

``` jsx
// 正确方式
const data = {
  on: { click: handler }
}
const vnode = <Cpt {...data}/>
// 或
const vnode = <Cpt {...{ on: { click: handler } }}/>

// 错误方式
const data = {
  onClick: handler
}
const vnode = <Cpt {...data}/>
// 或
const vnode = <Cpt {...{ onClick: handler }}/>
```

普通的JSX属性与扩展属性也可以混合使用，插件会智能地将其进行合并和嵌套，如：

``` jsx
const data = {
  class: ['b', 'c']
}
const vnode = <div class="a" {...data}/>
```
合并后相当于：

``` js
{ class: [{ a: true}, 'b', 'c'] }
```

大部分情况下属性合并后表现的结果都是在预期内的，但也可能例外，比如文章开头那个示例。只有熟悉黑盒中的合并规则，才能预期经插件转换JSX后的结果。 下面将开始深入探究`babel-plugin-transform-vue-jsx` 中细节，只是在此之前需要了解 Babel 一些基础知识。

## 关于 `Babel`

以 `babel-plugin-transform-vue-jsx` 部分源码为例，如果你能看明白下面的结构和内容可以直接跳过这部分内容。

```js
module.exports = function (babel) {
  var t = babel.types
  return {
    visitor: {
      JSXElement: {
        exit (path, file) {
          // 省略大量代码
        }
      },
      // 省略大量代码
    }
  }
}
```

Babel处理步骤分三步：解析（parse），转换（transform），生成（generate）。

**解析步骤**

Babel 接收代码，然后解析后输出抽象语法树（AST）。比如 `function square(n) { return n * n; }` 生成的 AST 用 JavaScript Object 可以表示未下面的结构：

``` js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

上面的结构出于简化的目的移除了某些属性，想查看更多属性可以[看这里](https://link.zhihu.com/?target=https%3A//astexplorer.net/%23/Z1exs6BWMq)。
上面的每一层结构也被叫做 节点（Node），每个节点都有个 `type` 属性。

**转换步骤**

此步骤接收 AST 并对其进行遍历，在此过程中可以对节点进行添加、更新及移除等操作, 这也是 Babel 插件（转换插件）将要介入工作的步骤。

**生成步骤**

此步骤把最终经过一系列转换之后的 AST 转换成字符串形式的代码。由于本文只关注 Babel 插件可以作用的环节，生成步骤这里就不过多赘述。

转换步骤是在对 AST 进行深度优先的递归的树形遍历，并且是通过 `visitor`（访问者） 使得在遍历过程中获取并访问具体的节点。

**`visitor`（访问者）**

回顾上面 `babel-plugin-transform-vue-jsx` 的代码，`visitor` 就是访问者对象，而 `JSXElement` 表示的是一种节点类型，由于在深度优先的遍历过程中，同一个节点将在 `enter` 和 `exit` 两个时机各被访问一次，并且调用对应的方法，方法中即可添加添加、移除、转换等节点操作。

``` jsx
module.exports = function (babel) {
  var t = babel.types
  return {
    visitor: {
      JSXElement: {
        // enter() {},
        exit (path) {
          // ...
        }
      },
      // ...
    }
  }
}
```

*提示：如果见到类似 `JSXElement() { ... }` 的写法，表示的是 `JSXElement: { enter() { ... } }` 的简写形式。*

**`Paths`（路径）**

AST 通常会有许多节点，Path 是表示两个节点之间连接的对象。比如：

``` jsx
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  // ...
}
```

将上面的子节点 `Identifier` 表示为一个路径（`Path`）的话，看起来是这样的：

``` js
{
  "parent": {
    "type": "FunctionDeclaration",
    "id": {...},
    ....
  },
  "node": {
    "type": "Identifier",
    "name": "square"
  }
}
```
*出于简化的目的，上面对象中移除了其他元数据。*

路径对象还包含添加、更新、移动和删除节点有关的其他很多方法。比如删除一个节点：

``` js
FunctionDeclaration(path) {
  path.remove()
}
```

当你调用一个修改树的方法后，路径信息也会被更新。

当你定义了一个 `JSXElement()` 成员方法的访问者时，你实际上是在访问 `JSXElement` 的路径而非 `JSXElement` 节点，所以你操作的是节点的路径而非节点本身。

**Babel Types**

它包含了构造、验证以及变换 AST 节点的方法。比如判断一个节点是否 JSXElement 类型：`t.isJSXElement(node, opts)`，创建一个 `JSXElement` 类型的节点：`t.jsxElement(openingElement, closingElement, children, selfClosing)`。

那我要怎么确定要调 API 的名称呢？只要你确定节点的类型后，根据类型名称可以到[这里](https://link.zhihu.com/?target=https%3A//babeljs.io/docs/en/babel-types)去查找，而查看节点的类型可以在之前提到的[网址](https://link.zhihu.com/?target=https%3A//astexplorer.net/)选择 JavaScript 模式贴入代码，然后可以在右边的 AST Tree 中去查看节点的 `type` 属性，这里不再过多赘述。

## `babel-plugin-transform-vue-jsx` 转换 JSX 的思路

了解了 Babel 一些基础知识后，我们可以预想到通过定义访问者，让其访问到 JSX 元素的时候将对应的 JSX 表达式转换为常规的 Javascript 代码，并回归到 `createElement` 创建 `vnode` 的方式，即：

``` js
render(h) {
  // JSX
  const vnode =  <Cpt {...data}>children</Cpt>
  // createElement
  const vnode = h('Cpt', data, children)
  return vnode
}
```

实际上插件的确也是这样做的。那具体的过程怎样的呢？问题的关键在于对比两种表达式解析成 AST 的差异，你同样可以在[这里](https://link.zhihu.com/?target=https%3A//astexplorer.net/)进行比较。

![img1](https://pic1.zhimg.com/80/v2-ff3f9d8a1a4ab04fe011dbee7c5f3508_1440w.jpg)

整个转换的过程包含了如处理 JSX标签、children，自动注入 `h` 等过程，感兴趣的话同样可以结合上面对比 AST 的方法去阅读和理解[源码](https://link.zhihu.com/?target=https%3A//github.com/vuejs/babel-plugin-transform-vue-jsx/blob/master/index.js)，但这里不讨论，本文只重点关注处理 JSX 属性的过程。

### `JSXIdentifier` vs `JSXSpreadAttribute`

``` jsx
const data = { propE: 'e' }
<Cpt propA propB="b" propC={c} {...{ propD: 'd' }} {...data}/>
```

上面 JSX 表达式将产生下面 AST 结构，标记处是五种不同的 JSX 属性对应节点：

![img2](https://pic1.zhimg.com/80/v2-099dcdb9026d1e92b209f16f403f1b6c_1440w.jpg)

可以看出所有 JSX 属性解析后的节点都存在于 `JSXOpeningElement` 的 `attributes` 属性中，且能看出扩展属性节点类型都为 `JSXSpreadAttribute`， 其他均为 `JSXAttribute` 类型。

在此之前先了解一个普通的对象表达式的 AST 结构：

``` js
const obj = {
  propA: true,
  propB: 'b',
  propC: c
}
```

`obj` 解析成的 AST 如下：

![img3](https://pic4.zhimg.com/80/v2-1a4c6a06228a20930129e6dec6c66d7f_1440w.jpg)

可以看出一个对象表达式节点为 `ObjectExpression`, 并可以包含多个 `ObjectProperty` 节点。

![img4](https://pic4.zhimg.com/80/v2-1a4c6a06228a20930129e6dec6c66d7f_1440w.jpg)

这样思路就很清晰了：

![img5](https://pic1.zhimg.com/80/v2-dfa4ceb5e9fdd733b18f24eda34b3538_1440w.jpg)

我们可以将 `JSXAttribute` 节点处理成 `ObjectProperty` 节点，然后可以组合转换成一个 `ObjectExpression` 节点。`JSXSpreadAttribute` 的处理思路基本类似，只是更为简单，`JSXSpreadAttribute` 节点中的 `argument` 属性获取到的节点就是我们需要的。 最后将两个 `ObjectExpression` 进行合并就是最后所需的数据对象。

下面是源码中处理 `JSXOpeningElement` 的 `attributes` 的函数代码。

``` js
function buildOpeningElementAttributes (attribs, path) {
  // 1. 收集 props
  var _props = []
  var objs = []
  function pushProps () {
    if (!_props.length) return
    // 多个objectProperty组装成一个objectExpression
    objs.push(t.objectExpression(_props))
    _props = []
  }
  while (attribs.length) {
    var prop = attribs.shift()
    if (t.isJSXSpreadAttribute(prop)) {
      // 1-1 处
      pushProps()
      // 1-2 处
      prop.argument._isSpread = true
      objs.push(prop.argument)
    } else {
      // JSXAttribute => objectProperty
      _props.push(convertAttribute(prop))
    }
  }
  pushProps()

  // 2. 对从 JSXAttribute 节点收集的 props 进行分组
  objs = objs.map(function (o) {
    // 2-1 处
    return o._isSpread ? o : groupProps(o.properties, t)
  })
  // 3. 合成成一个数据对象
  if (objs.length === 1) {
    attribs = objs[0]
  } else if (objs.length) {
    // 3-1 处
    var helper = addDefault(path, 'babel-helper-vue-jsx-merge-props', { nameHint: '_mergeJSXProps' })
    attribs = t.callExpression(
      helper,
      [t.arrayExpression(objs)]
    )
  }
  return attribs
}
```

省略一些细节讨论，整个过程基本可以分为三步：

第一步是收集 `props`。

收集的过程中，从对 `JSXSpreadAttribute` 节点中搜集 `props` 相对简单，取 `JSXSpreadAttribute` 的 `argument` 暂存即可，额外增加了 `_isSpread` 标记 （1-2 处注释）; 从 `JSXAttribute` 节点中搜集的过程就是先将 `JSXAttribute` 节点转换为一个 `ObjectProperty` `节点后，然后多个ObjectProperty` 组装可以生成一个 `ObjectExpression` 节点。

值得注意的是这个生成 `ObjectExpression` 节点的过程是在遇到 `JSXSpreadAttribute` 节点之前才会进行的（1-1 处注释）。由于 `JSXAttribute` 和 `JSXSpreadAttribute` 可能存在交替出现，所以这个过程结束后 `objs` 可能组装了多个 `ObjectExpression` 节点。

第二步是对从 JSXAttribute 节点提取出的 props 进行分组。

根据 `_isSpread` 区分出由 `JSXAttribute` 生成的 `ObjectExpression`，其 `properties` 属性包含了 `ObjectProperty` 节点。

``` js
return o._isSpread ? o : groupProps(o.properties, t)
```
这个过程是将 `ObjectProperty` 进行分组与合并的过程，分组依据数据对象作为标准。

分组主要是使用正则依据属性名称进行分类(顶层属性全等匹配)，可嵌套的属性转换为相应的嵌套对象的结构，看下面的部分源码和示例更易理解。

``` js
// groupProps 函数中的正则声明
var isTopLevel = makeMap('class,staticClass,style,key,ref,refInFor,slot,scopedSlots')
var nestableRE = /^(props|domProps|on|nativeOn|hook)([\-_A-Z])/
var dirRE = /^v-/
var xlinkRE = /^xlink([A-Z])/
// 省略很多代码
```
*想看完整源码移步[这里](https://link.zhihu.com/?target=https%3A//github.com/vuejs/babel-plugin-transform-vue-jsx/blob/master/lib/group-props.js)。*

分组示例：

``` js
// 分组前
{
  id: 'foo',
  domPropsInnerHTML: 'bar',
  onClick: clickHandler,
  nativeOnClick: nativeClickHandler,
  class: { foo: true, bar: false },
  // ...
}
// 分组后
{
  attrs: {
    id: 'foo'
  },
  domProps: {
    innerHTML: 'bar'
  },
  on: {
    click: clickHandler
  },
  nativeOn: {
    click: nativeClickHandler
  },
  class: { foo: true, bar: false },
  // ...
}
```

合并的过程是处理那些可嵌套的属性。比如：

``` js
// 合并前
{
  onClick: clickHandler,
  onChange: changeHandler
}
// 合并后
{
  on: {
    click: clickHandler,
    change: changeHandler
  }
}
```

`groupProps` 函数最终返回的是完全符合数据对象格式的 `objectExpression` 节点，只要你足够了解数据对象的格式，`groupProps` 中并不存在超出我们预期的特殊之处。

稍微说明一下的就是可以看到 `nestableRE` 的表达式中并不包含 `attrs`，原因如果一个属性名称既不是顶层属性(`class`、`staticClass`、`style`、`key`、`ref`、`refInFor`、`slot`、`scopedSlots`)，又不被上面的所有正则规则所匹配的话，都将作为普通的 `html` 属性添加在 `attrs` 下面。

第三步合成数据对象。

经过前面收集和分组的过程，所有的 `JSXAttribute` 处理成符合数据对象的 `ObjectExpression` 节点（可能存在多个）。对 `JSXSpreadAttribute` 的收集过程就是收集 `argument` 属性中节点的过程。 现在到了第三个步骤就是将它们进行合并组装一个数据对象。

``` js
if (objs.length === 1) {
  attribs = objs[0]
} else if (objs.length) {
  var helper = addDefault(path, 'babel-helper-vue-jsx-merge-props', { nameHint: '_mergeJSXProps' })
  attribs = t.callExpression(
    helper,
    [t.arrayExpression(objs)]
  )
}
return attribs
```

合并的过程使用 [mergeJSXProps的函数](https://zhuanlan.zhihu.com/p/59434351/http%3C/code%3Es://github.com/vuejs/babel-helper-vue-jsx-merge-props/blob/master/index.js) 返回一个新的对象：

``` js
module.exports = function mergeJSXProps (objs) {
  return objs.reduce(function (a, b) {
    // 省略大量代码
  }, {})
}
```

`mergeJSXProps` 函数主要的对 `attrs`、`props`、`on`、`nativeOn`、`class`、`style`、`hook` 的值进行合并，其中 `attrs`、`props` 的属性的值是对象类型，合并过程中类似于 `Object.assign`，比如：

``` js
var data1 = {
  props: {
    a: 1,
    b: { b1: 1 }
  }
}
var data2 = {
  props: {
    b: { b2: 2 },
    c: 3
  }
}
// 合并后
var data3 = {
  props: {
    a: 1,
    b: { b2: 1 },
    c: 3
  }
}
```

而 `class` 的值可以为 `String`、`Array` 类型，当值为 `String` 类型（如 `class: 'wrap'`）时将先处理为 `class: { wrap: true }`，当值存在 `Array` 类型时将通过 `concat` 进行拼接，比如 `{ class: 'a' }` 与 `{ class: ['b', 'c']}` 拼接的结果为 `{ class: [{ a: true }, 'b', 'c']}`。

其中更特殊一点的在于对 `on`、`nativeOn`、`hook` 的处理，因为这几个可嵌套属性中的键值对的值都是 `Function` 类型，同键名时将会将两个函数合并成一个。

``` js
var nestRE = /^(attrs|props|on|nativeOn|class|style|hook)$/

module.exports = function mergeJSXProps (objs) {
  // 省略大量代码
  if (key === 'on' || key === 'nativeOn' || key === 'hook') {
    // merge functions
    for (nestedKey in bb) {
      aa[nestedKey] = mergeFn(aa[nestedKey], bb[nestedKey])
    }
  }
  // 省略大量代码
}

function mergeFn (a, b) {
  return function () {
    a && a.apply(this, arguments)
    b && b.apply(this, arguments)
  }
}
```

在结果上表现为所有处理函数都会执行，比如下面 `change` 事件触发后三个处理函数都会执行，执行顺序跟 JSX 属性的注册顺序一致：

``` js
const data = { on: { change: onChange2 } }
<Cpt
  {...{ on: { change: onChange1 } }}
  {...data}
  onChange={onChange3}/>
```

除 `attrs`、`props`、`on`、`nativeOn`、`class`、`style`、`hook` 此之外的属性在合并过程中如果含有同名属性会对先添加的值进行覆盖。

## 思考

你可以使用扩展属性去传递 `props`、也可以使用普通的JSX属性去传递 `props`, 也可以两种方式混合使用。

``` js
// 普通JSX属性
render(h) {
  return <MyCpt id="foo" domPropsInnerHTML="bar" onClick={this.clickHandler}></MyCpt>
}
// 扩展属性
render(h) {
  const data = {
    attrs: {
      id: 'foo'
    },
    domProps: {
      innerHTML: 'bar'
    },
    on: {
      click: this.clickHandler
    }
  }
  return <MyCpt {...data}></MyCpt>
  // 或者
  return <MyCpt {...{ attrs: { id: 'foo' }, domProps: { innerHTML: 'bar' }, on: { click: clickHandler }}}></MyCpt>
}
// 混合使用
render(h) {
  const data = {
    on: {
      click: this.clickHandler
    }
  }
  return <MyCpt id="foo" domPropsInnerHTML="bar" {...data}></MyCpt>
}
```

`render` 中的 `h` 可以省略，插件在识别到 `ObjectMethod|ClassMethod` 类型的节点后，如果内容存在 JXS 表达式，将会自动注入 `h`。

当你不了解 `babel-plugin-transform-vue-jsx` 处理 JSX 属性的过程时，通常建议两种方式选其一，并且在选择使用普通 JSX 属性时需要根据属性的类型添加对应的前缀；选择使用扩展属性时需要遵守数据对象的约定去定义属性对象，避免超出预期的结果出现。当两种方式混合在一起使用时智能合并的结果可能在你预期之外。当然也有很多的场景混合使用是更方便的。只要你足够`babel-plugin-transform-vue-jsx`，一切都在掌控之中。

现在回答之前文章开头提出的问题：下面的两种写法 change 事件触发后分别输出什么？
 
``` js
function handler1() { console.log(1) }
function handler2() { console.log(2) }
function handler3() { console.log(3) }
function handler4() { console.log(4) }
// 1
<MyCpt
  {...{ on: { change: handler1 } }}
  onChange={ handler2 }
  onChange={ handler3 }
  {...{ on: { change: handler4 } }}/>
// 2
<MyCpt
  onChange={ handler1 }
  {...{ on: { change: handler2 } }}
  onChange={ handler3 }
  {...{ on: { change: handler4 } }}/>
```

结果是分别为 `1 3 4`、`1 2 3 4`。

之前讲到过在第一步收集 `props` 的过程中，`JSXAttribute` 会转换为 `ObjectProperty` 节点暂存起来，当遇到 `JSXSpreadAttribute` 时才会批量将之前暂存的一个或多个 `ObjectProperty` 合并生成一个 `ObjectExpression` 节点。所以 1 例中可以预期到只会出现一次组装 `ObjectExpression` 的过程。

由于 `onChange={ handler2 }` `onChange={ handler3 }` 生成 `ObjectProperty` 的 `key` 都是 `onChange`, 生成的 `ObjectExpression` 相当于 `{ onChange: handler3 }`。

在第三步 `mergeJSXProps` 的过程，相当于下面三个对象进行合并：

``` js
mergeJSXProps([
  { on: { change: handler1 } }
  { on: { change: handler3 } }
  { on: { change: handler4 } }
])
```

且由于 `mergeJSXProps` 过程中对 `on`、`nativeOn`、`hook` 这三个可嵌套对象将进行 `merge functions` 的操作，可以预期到 `handler1`， `handler3`， `handler4` 将合并一个函数，执行的顺序跟 JSX 属性注册顺序一致。所以输出 `1 3 4`。

而 2 例中的写法由于 两个 `JSXAttribute` 和 两个 `JSXSpreadAttribute` 节点间隔出现。第一步收集 `props` 会出现两次组装 `ObjectExpression` 的过程。在第三步 `mergeJSXProps` 的过程将存在四个对象进行合并，`merge functions` 后四个 `handler` 函数都能得到执行。

``` js
mergeJSXProps([
  { on: { change: handler1 } }
  { on: { change: handler2 } }
  { on: { change: handler3 } }
  { on: { change: handler4 } }
])
```
