### 编译函数compiler

***

vue 的模板编译从html字符串到render函数需要3个步骤，

```Javascript
const ast = parse(template.trim(), options)  // htmlstring to ast
optimize(ast, options)                       // 处理ast静态部分
const code = generate(ast, options)          // 由ast生成渲染函数
return {
	ast,
	render: code.render,
	staticRenderFns: code.staticRenderFns
}
```

### AST 数据结构

[AST](https://zh.wikipedia.org/wiki/%E6%8A%BD%E8%B1%A1%E8%AA%9E%E6%B3%95%E6%A8%B9) 的全称是 Abstract Syntax Tree（抽象语法树），是源代码的抽象语法结构的树状表现形式，计算机学科中编译原理的概念。Vue 源码中借鉴 jQuery 作者 [John Resig](https://zh.wikipedia.org/wiki/%E7%B4%84%E7%BF%B0%C2%B7%E9%9B%B7%E8%A5%BF%E6%A0%BC) 的 [HTML Parser](http://ejohn.org/blog/pure-javascript-html-parser/) 对模板进行解析，得到的就是 AST 代码。

我们看一下 Vue 2.0 源码中 [AST 数据结构](https://github.com/vuejs/vue/blob/v2.4.2/flow/compiler.js#L67-L161) 的定义：

```javascript
declare type ASTNode = ASTElement | ASTText | ASTExpression
declare type ASTElement = {
  type: 1;
  tag: string;
  attrsList: Array<{ name: string; value: string }>;
  attrsMap: { [key: string]: string | null };
  parent: ASTElement | void;
  children: Array<ASTNode>;
  //......
}
declare type ASTExpression = {
  type: 2;
  expression: string;
  text: string;
  static?: boolean;
}
declare type ASTText = {
  type: 3;
  text: string;
  static?: boolean;
}
```

我们看到 ASTNode 有三种形式：ASTElement，ASTText，ASTExpression。用属性 `type` 区分。

### render function

render function 顾名思义就是渲染函数，这个函数是通过编译模板文件得到的，其运行结果是 VNode。render function 与 JSX 类似，Vue 2.0 中除了 Template 也支持 JSX 的写法。大家可以使用 [Vue.compile(template)](https://vuejs.org/v2/api/?#Vue-compile) 方法编译下面这段模板：

```Html
<div id="app">
  <header>
    <h1>I'm a template!</h1>
  </header>
  <p v-if="message">
    {{ message }}
  </p>
  <p v-else>
    No message.
  </p>
</div>
```

方法会返回一个对象，对象中有 `render` 和 `staticRenderFns` 两个值。看一下生成的 render function ：

```javascript
(function() {
  with(this){
    return _c('div',{   //创建一个 div 元素
      attrs:{"id":"app"}  //div 添加属性 id
      },[
        _m(0),  //静态节点 header，此处对应 staticRenderFns 数组索引为 0 的 render function
        _v(" "), //空的文本节点
        (message) //三元表达式，判断 message 是否存在
         //如果存在，创建 p 元素，元素里面有文本，值为 toString(message)
        ?_c('p',[_v("\n    "+_s(message)+"\n  ")])
        //如果不存在，创建 p 元素，元素里面有文本，值为 No message. 
        :_c('p',[_v("\n    No message.\n  ")])
      ]
    )
  }
})
```

除了 render function，还有一个 `staticRenderFns` 数组，这个数组中的函数与 VDOM 中的 diff 算法优化相关，我们会在编译阶段给后面不会发生变化的 VNode 节点打上 static 为 true 的标签，那些被标记为静态节点的 VNode 就会单独生成 staticRenderFns 函数：

```javascript
(function() { //上面 render function 中的 _m(0) 会调用这个方法
  with(this){
    return _c('header',[_c('h1',[_v("I'm a template!")])])
  }
})
```

要看懂上面的 render function，只需要了解 _c，_m，_v，_s 这几个函数的定义，其中 _c 是 `createElement`，_m 是 `renderStatic`，_v 是 `createTextVNode`，_s 是 `toString`。除了这个 4 个函数，还有另外 10 个函数，我们可以在源码 [src/core/instance/render.js](https://github.com/vuejs/vue/blob/v2.4.2/src/core/instance/render.js#L71-L145) 中可以查看这些函数的定义。

## 模板渲染的过程

在前面讲初始化流程时看到在_init函数的最后一步就是$mount方法，这个方法就是模板渲染的入口。流程如下

![22F3F2A7-B180-4804-B101-47393FB0947E](/Users/xinboshang/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/314911714/QQ/Temp.db/22F3F2A7-B180-4804-B101-47393FB0947E.png)

### createCompiler

上面说的createCompiler（src/compiler/index.js）就是将 template 编译成 render function 的字符串形式。这一小节我们就详细讲解这个函数：

```javascript
export const createCompiler = createCompilerCreator(function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  optimize(ast, options)
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
})
```

这里是一个嵌套函数，使用[createCompilerCreator](https://github.com/vuejs/vue/blob/v2.4.2/src/compiler/create-compiler.js#L7-L55)的返回值{compile,compieToFunctions},参数是基础的编译函数，这个编译函数接受两个参数template和options,这个函数主要有三个步骤组成：`parse`，`optimize` 和 `generate`，最终输出一个包含 ast，render 和 staticRenderFns 的对象。

[**parse**](https://github.com/vuejs/vue/blob/v2.4.2/src/compiler/parser/index.js#L46-L288) 函数（src/compiler/parser/index.js）采用了 jQuery 作者 [John Resig](https://zh.wikipedia.org/wiki/%E7%B4%84%E7%BF%B0%C2%B7%E9%9B%B7%E8%A5%BF%E6%A0%BC) 的 [HTML Parser](http://ejohn.org/blog/pure-javascript-html-parser/) ，将 template字符串解析成 AST。感兴趣的同学可以深入代码去了解原理。

[**optimize**](https://github.com/vuejs/vue/blob/v2.4.2/src/compiler/optimizer.js#L21-L29) 函数（src/compiler/optimizer.js）主要功能就是标记静态节点，为后面 patch 过程中对比新旧 VNode 树形结构做优化。被标记为 static 的节点在后面的 diff 算法中会被直接忽略，不做详细的比较。

```javascript
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  // staticKeys 是那些认为不会被更改的ast的属性
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no
  // first pass: mark all non-static nodes.
  markStatic(root)
  // second pass: mark static roots.
  markStaticRoots(root, false)
}
```

[**generate**](https://github.com/vuejs/vue/blob/v2.4.2/src/compiler/codegen/index.js#L40-L50) 函数（src/compiler/codegen/index.js）主要功能就是根据 AST 结构拼接生成 render function 的字符串。

```Javascript
const code = ast ? genElement(ast) : '_c("div")' 
staticRenderFns = prevStaticRenderFns
onceCount = prevOnceCount
return {
    render: `with(this){return ${code}}`, //最外层包一个 with(this) 之后返回
    staticRenderFns: currentStaticRenderFns
}
```

其中 [**genElement**](https://github.com/vuejs/vue/blob/v2.4.2/src/compiler/codegen/index.js#L52-L86) 函数（src/compiler/codegen/index.js）是会根据 AST 的属性调用不同的方法生成字符串返回。

以上就是 compile 函数中三个核心步骤的介绍，compile 之后我们得到了 render function 的字符串形式，后面通过 new Function 得到真正的渲染函数。数据发现变化后，会执行 Watcher 中的 **_update**函数（src/core/instance/lifecycle.js），_update 函数会执行这个渲染函数，输出一个新的 VNode 树形结构的数据。然后在调用 \_\_patch__ 函数，拿这个新的 VNode 与旧的 VNode 进行对比，只有发生了变化的节点才会被更新到真实 DOM 树上。
