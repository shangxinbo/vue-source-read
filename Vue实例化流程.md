### Vue 实例化流程

***

从入口文件分析流程可知instance/index是Vue的主程序入口，做了几件事情：

```javascript
initGlobalAPI(Vue)
Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})
Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})
Vue.version = '__VERSION__'
```

* initGlobalAPI包装Vue，给Vue添加全局的通用api，可对照官网的[全局 API](https://cn.vuejs.org/v2/api/#全局-API)
* 定义Vue.prototype.$isServer和Vue.prototype.$ssrContext
* 标记Vue版本号

之后我们得知真正的Vue构造函数就在/src/core/instance/index里

> entry-runtime-with-compiler  > runtime/index.js  >  core/index.js > instance/index.js

```javascript
// Vue 构造函数
// 做了一个判断，如果是生产环境，保证是Vue的实例而不是直接使用Vue
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
// 通过Mixin方式为Vue挂在方法
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

这里的Mixin是为了解决javascript的多继承的一种设计模式(混合模式/织入模式)，常用于写一些框架。在这里的实现方法就是将Vue对象传递到不同的模块中为Vue挂在新的方法和属性。

```javascript
// initMixin(Vue)    src/core/instance/init.js **************************************************
Vue.prototype._init = function (options?: Object) {}

// stateMixin(Vue)    src/core/instance/state.js **************************************************
Vue.prototype.$data
Vue.prototype.$props
Vue.prototype.$set = set
Vue.prototype.$delete = del
Vue.prototype.$watch = function(){}

// renderMixin(Vue)    src/core/instance/render.js **************************************************
Vue.prototype.$nextTick = function (fn: Function) {}
Vue.prototype._render = function (): VNode {}
Vue.prototype._o = markOnce
Vue.prototype._n = toNumber
Vue.prototype._s = toString
Vue.prototype._l = renderList
Vue.prototype._t = renderSlot
Vue.prototype._q = looseEqual
Vue.prototype._i = looseIndexOf
Vue.prototype._m = renderStatic
Vue.prototype._f = resolveFilter
Vue.prototype._k = checkKeyCodes
Vue.prototype._b = bindObjectProps
Vue.prototype._v = createTextVNode
Vue.prototype._e = createEmptyVNode
Vue.prototype._u = resolveScopedSlots
Vue.prototype._g = bindObjectListeners

// eventsMixin(Vue)    src/core/instance/events.js **************************************************
Vue.prototype.$on = function (event: string, fn: Function): Component {}
Vue.prototype.$once = function (event: string, fn: Function): Component {}
Vue.prototype.$off = function (event?: string, fn?: Function): Component {}
Vue.prototype.$emit = function (event: string): Component {}

// lifecycleMixin(Vue)    src/core/instance/lifecycle.js **************************************************
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {}
Vue.prototype.$forceUpdate = function () {}
Vue.prototype.$destroy = function () {}
```

以上是vue的样子。

之后我们看到Vue构造函数里有一个_init，这个是在initMixin的过程中给Vue.prototype加上的方法。这个方法就是Vue实例化的初始化函数。在instance/init.js

```javascript
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // 组件实例的唯一标识，这个标识在打包或使用时会从0自增，以保证唯一不重复，这个计算法则的前提是保证一个页面上有唯一的Vue依赖，见 https://github.com/vuejs/vue/issues/4958
    vm._uid = uid++

    // 添加性能标记的代码省略，可以了解window.perfermance

    // a flag to avoid this being observed
    vm._isVue = true
    
    // _isComponent 是vue内部组件的标识
    // 合并配置选项
    if (options && options._isComponent) {
      initInternalComponent(vm, options) // 内部组件不需要动态合并选项
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    // 这里如果是开发环境会使用Proxy对象来检测用户使用的属性是否在实例上，如果不在就给予警告
    // 如果是生产环境则代理直接等于实例，也就不做多余的操作，只做一个别名而已，类似_self
    // 这里是作者尝试的一个新的方法，保证vm选项合法
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    vm._self = vm
    
    // 各个模块初始化
    initLifecycle(vm)
    initEvents(vm)
    initRender(vm)
    callHook(vm, 'beforeCreate')
    initInjections(vm) // resolve injections before data/props
    initState(vm)
    initProvide(vm) // resolve provide after data/props
    callHook(vm, 'created')
 
    // 挂载元素，这个$mount 方法是platforms/web/runtime/index.js里
    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

这个初始化做了以下几个操作

* 定义Vue.uid用于唯一标示组件实例
* 定义Vue.isVue用于阻止数据监听
* 合并选项options
* 做_renderProxy代理实例，这个代理的目的是为了在开发环境中检测用户的选项是否合法，并非核心内容。
* 做_self 备用实例引用
* 各个模块初始化，并触发beforeCreate和created周期
* 若选项中有el字段则挂载实例

在合并选项时，因为动态枚举赋值要慢，对没有必要特殊处理的内部组件做了优化处理，在第一个分支用了`initInternalComponent(vm, options)`。看一下这个方法的处理

```javascript
function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  /*
  	vm.constuctor实例的构造函数就是Vue，vm.constructor.options == Vue.options
  	这时的Vue.options 里有directives和components，是在程序入口文件中添加的
  	Objedt.create 创建一个新的对象，将这个对象赋值给实例的$options参数
  	这里用连等的方式为vm.$options 做一个直接指针，
  	然后用这个指针进行内部属性赋值操作，这样可以比直接操作vm.$options更优雅
  */
  const opts = vm.$options = Object.create(vm.constructor.options)
  opts.parent = options.parent
  opts.propsData = options.propsData
  opts._parentVnode = options._parentVnode
  opts._parentListeners = options._parentListeners
  opts._renderChildren = options._renderChildren
  opts._componentTag = options._componentTag
  opts._parentElm = options._parentElm
  opts._refElm = options._refElm
  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
  // 这里的合并算法是直接将实例的选项覆盖掉父类(Vue)的选项
}
```

如果不是内部组件则进入第二个分支，这里用了公共函数mergeOptions,这个函数介绍请看下节。其中各个模块的初始化我们也会独立分出一个章节说。而挂载函数要涉及到渲染函数的逻辑，我们单独在渲染逻辑里讲。