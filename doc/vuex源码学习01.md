# vuex源码学习

vuex目前在我写的项目中使用的较少，写的时候，也基本照着官方文档上，一看，就开始写起来了。但是对于这个插件是怎么工作的，是一点也不了解。
今天，我们就打开vuex的源码稍微看一看vuex是如合工作的。去官方仓库，clone代码的过程，我们就直接略过了。

## 从pack.json开始看。

当我们下载完一个源码的时候，肯定是从```package.json```看这个库的构建过程。

```
"scripts": {
  "dev": "node examples/server.js",
}
```

和```vue-router```一样，是用express 和 webpackDevMiddleware 启用的一个服务，并且支持热更新。我们目前只需要知道，就是把webpack构建好的工程（但是其实并没有真正的有打包后的文件），设为一个静态的目录。并且支持热更新。

然后同样，我们主要看的项目中的```vuex```来自一个别命，指向的就是我们当前的 ```src/index.esm.js``` 然后。我们就可以看```index.esm.js```了

```

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}

```
从这里，我们知道， Vuex 就是这个对象，我们也知道，项目中写的 Vue.use(Vuex), 其实，就是调用了导出的 install 方法。 ```new Vuex.Store()```, 也就是Store构造函数了。

## 从 install 方法开始。

找到```install``` 方法。 在```src/store.js```中, 代码如下

```
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  applyMixin(Vue)
}
```

我们先看看代码中 Vue 是个什么，
```
let Vue // bind on install
```
在 ```store.js```中的第6行。
为什么要声明这样一个变量呢。其实，最主要是因为，别的地方要用到Vue 对象，但是又不好直接安装一个Vue,再来依赖Vue，所以，这样，Vuex代码中就不需要再额外引入Vue了。

接下来，找到 applyMinxin(Vue)

```
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    Vue.mixin({ beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```
根据Vue的版本做了一个适配，因为工作中使用的版本大于2， 直接走if分支
```
Vue.mixin({ beforeCreate: vuexInit })
```
调用了Vue.mixin方法，在所有的Vue 的beforeCreate的钩子中增加了一个方法, vuexInit。

然后，这段代码的意义主要是将所有组件的 this.$store 指向 父组件的 options.$store， 又因为，父组件的beforeCreate 先于子组件运行，所以，最终的话，所有组件的this.$store 都指向的是根组件的this.$store。 这里和```vue-rotuer```的处理方法一致。


然后我们 的 $store 又是指向的什么呢，其实就是我们的传入
```
new Vue({
  ...
  store
})
```
也就是我们 new Vuex.Store({...}),构造出的来的Store实例。想到我们平常用到的
```
this.$store.commit()
this.$store.dispath()
this.$store.getters
```
这些常用的API 都是出自 Store 构造出来的实例。

## 从一个简单的例子开始

因为我们这次看源码的目的只是为了稍微了解一下vuex背后工作的原理，所以，我们只需要找一个简单的例子从头到尾的过一遍，大概就知道，这里面的核心原理是什么了。其实，就算是我们要研究透每一行代码，我们也可以从简单的一个例子开始，因为，就算是最简单的例子，肯定也用到了核心。

简单的例子，vuex 的 example 已经给我们写好了，就是example下的 counter 这个文件夹下的代码。
我们发现，有getters ,mumations, actions, 还有一个简单的state 。


好现在，从 new Vuex.Store()看起，我们传入的参数是这样。
```
options: {
  state: {
    count: 0,
  },
  mutations: {
    increment (state) {
      state.count++
    },
    decrement (state) {
      state.count--
    }
  },
  actions: {
    increment: ({ commit }) => commit('increment'),
    decrement: ({ commit }) => commit('decrement'),
    incrementIfOdd ({ commit, state }) {
      if ((state.count + 1) % 2 === 0) {
        commit('increment')
      }
    },
    incrementAsync ({ commit }) {
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          commit('increment')
          resolve()
        }, 1000)
      })
    }
  },
  getters: {
    evenOrOdd: state => state.count % 2 === 0 ? 'even' : 'odd'
  }
}
```
然后， 我们看 Store 这个构造函数。

* constructor 构造函数

首先，做了一个判断，看是不是用了Vue.use(Vuex)。

```
const {
  plugins = [],
  strict = false
} = options

// store internal state
this._committing = false
this._actions = Object.create(null)
this._actionSubscribers = []
this._mutations = Object.create(null)
this._wrappedGetters = Object.create(null)
this._modules = new ModuleCollection(options)
this._modulesNamespaceMap = Object.create(null)
this._subscribers = []
console.log(this._subscribers)
this._watcherVM = new Vue()
```

然后， 从我们的options 中获取插件 和 是否启用了严格模式，这边我们都不传，所以就是都忽略他们，
然后，初始化了一大堆属性。其中
```
this._moudles = new ModuleCollection(options)
```
这个一看就是处理我传入的options的。

```
// bind commit and dispatch to self
    const store = this
    const { dispatch, commit } = this
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }
```
这段代码，上面也有注释，是为了绑定 commit 和 dispatch 方法的作用域的。

```
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)

// initialize the store vm, which is responsible for the reactivity
// (also registers _wrappedGetters as computed properties)
resetStoreVM(this, state)
```

然后，这段代码，我们可以看上面的注释，

```installMoule``` 主要是初始化 根module的， 包括 所有的sub-modules 还有收集所有的 getters的。

```resetStoreVM``` 让store 实例变成响应式的， 同事，将收集好的getters 变成 computed 属性。

然后，后面的代码是和插件有关的，我们暂时不用管他们。

那么至此，其实我们所有的 new Vuex.Store() 也就执行完了。那么至此，我们整个例子，在用户没有操作之前应该就算是完了。那么，今天我们要探寻的vuex背后的工作原理，我们应该就能从上面的代码中找到答案了。。 我们接下来就开始一步步从头走到尾，过一遍。

首先，我们去看 ```this._modules = new ModuleCollection(options)```

打开 ```module-collection.js```, 看constructor部分，我们看到只调用了 ```register```方法。

```
register (path, rawModule, runtime = true) {
    if (process.env.NODE_ENV !== 'production') {
      assertRawModule(path, rawModule)
    }
    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }
```

我们知道了```this.root = new Moudule(rawModule, false)```, 因为我们这个简单的例子，没有modules 所以， 就不走modules 这个分支了，原型上其他的方法，我们也不用管了。 直接去看 Module 这个构造函数吧。

打开```module.js```, 看constructor，
```
this.runtime = runtime
// Store some children item
this._children = Object.create(null)
// Store the origin module object which passed by programmer
this._rawModule = rawModule
const rawState = rawModule.state

// Store the origin module's state
this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
```
初始化了一些属性， 其中 this._rawModule = 我们最开始传递给Store的options , 然后 this.state = {count: 1}
其他的好像也没有做什么了，原型上的方法，我们也先不管了。

然后直接回到了 我们的 Store 构造函数中。我们现在的 this._modules 大概是这个样子

```
_modules: ModuleCollection {
  root: Module{
    state: {
      count:1,
    },
    _children: [],
    runtime: false,
    _rawModule: options
  }
}
```
基本上就玩了， 当然，我们还有一些原型 上的方法。

然后我们看```installModule(this, state, [], this._modules.root)```

```
function installModule (store, rootState, path, module, hot) {
  ...
  const local = module.context = makeLocalContext(store, namespace, path)
  console.log(local)
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```
不用考虑命名空间的事情，我们的代码现在， 只需要执行的部分大概就是上面。 从这，我们可以看出来，给我们的
this._modules.root 添加了一个 context 的属性。 我们再看，这个```makeLocalContext```

```
const noNamespace = namespace === ''

const local = {
  dispatch: store.dispatch
  commit: store.commit
}
Object.defineProperties(local, {
    getters: {
      get: () => store.getters
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })
return local
```
去掉命名空间的相关代码，我们看到， 这个函数的代码只剩下这么一小部分， 其中 Object.defineProperties(), 给 local 对象增加了两个只读的属性，一个是 getters ,返回的就是 store.getters, 也就是 this.getters。 再看 ```getNestedState``` 其实返回的就是 store.state, 于是上面的代码我们 就知道了 local 是个什么鬼了

```
local = {
  dispatch: store.dispatch,
  commit : store.commit,
  state: store.state, // 只读, 构造器中没有声明， 但是有 get 和 set 函数
  getters: store.getters // 只读， 构造器中没有声明， 在 resetStoreVM 中声明了。
}
```

然后，我们再来看
```
module.forEachMutation((mutation, key) => {
  const namespacedType = namespace + key
  registerMutation(store, namespacedType, mutation, local)
})
在Module中，
forEachMutation (fn) {
  if (this._rawModule.mutations) {
    forEachValue(this._rawModule.mutations, fn)
  }
}
在util.js 中
export function forEachValue (obj, fn) {
  Object.keys(obj).forEach(key => fn(obj[key], key))
}
```
所以，我们知道这些代码的意思就是遍历的options.mmutations, 对这个对象执行 ```registerMutation```
我们再看 ```regiesterMutation```


```
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler.call(store, local.state, payload)
  })
}
```
这段代码主要是是收集我们传入Store 中的 mutations。 而且，对于我们一个 mutations 中的函数的作用域，总是在Store中， 第一个参数永远是, local.state， 也就是只读的store.state。

```
get state () {
  return this._vm._data.$$state
}
```
this._vm 到底是什么呢，构造器里没有声明呀，我们待会再看就好了。

到这里，其实，我们就能知道，我们在mutations 中间写函数，为什么总是能拿到 state 这个对象了。

```
module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })
```
这些都和之前的类似，我们就不在继续看了。

总之，就是收集 mutations, getters, actions 然后，给他们传入可以访问的store.state。 因为我们不涉及module 和 命名空间部分，这点我们暂时就不看了。


接着我们顺着Store的构造器中看这个 ```resetStoreVM```。
```
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm
  // bind store public getters
  store.getters = {}
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // use computed to leverage its lazy-caching mechanism
    computed[key] = () => fn(store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // use a Vue instance to store the state tree
  // suppress warnings just in case the user has added
  // some funky global mixins
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent
  }
}
```
这段代码。遍历我们的```store._wrappedGetters```, 我们知道，这里面，反正所有相关的getters代码。 拷贝到computed 中去，并且，我们的fn 传入的第一个对象就是当前的实例，给store.getters添加了一个get, 返回的是 ```store._vm[key],```

然后我们再看
```
store._vm = new Vue({
  data: {
    $$state: state
  },
  computed
})
```

到这里，我们基本上就知道我们的 store._vm 原来就是一个Vue实例。 访问我们的this.$store.getters['xxx'], 其实也就是访问 我们的 store._vm['xxx']。

然后基本上到这里，整个初始化动作就算完成了。但是，从我们发起commit/dispatch 到视图更新这整个过程，我们还是不是很清楚。看commit 函数好了。

仍然以 这个例子为例，我们点击页面上
```
<button @click="increment">+</button>
```
我们知道触发了 一次commit, 我们看commit 方法。


commit (_type, _payload, _options) {
    // check object-style commit
    const {
      type,
      payload,
      options
    } = unifyObjectStyle(_type, _payload, _options)

    const mutation = { type, payload }
    const entry = this._mutations[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown mutation type: ${type}`)
      }
      return
    }
    this._withCommit(() => {
      entry.forEach(function commitIterator (handler) {
        handler(payload)
      })
    })
    this._subscribers.forEach(sub => sub(mutation, this.state))

    if (
      process.env.NODE_ENV !== 'production' &&
      options && options.silent
    ) {
      console.warn(
        `[vuex] mutation type: ${type}. Silent option has been removed. ` +
        'Use the filter functionality in the vue-devtools'
      )
    }
  }
```

```unifyObjectStyle``` 这个就是统一我们的提交格式。 其实我们传递的参数就是 'increment',
然后。在当前的Vuex.Store 实例中找到 _mutations 名为 'increment'的数组， 这个数组是一个，应该只有一个函数， [wrappedMutationHandler(payload){}], 然后遍历执行。 我们这个wrappedMutationHandler里面包含着的是 mutations.increment

其实就相当于执行 mutations.increment.call(store, store.state), 然后，当前的store.state 就是 store._vm 的 data。也就是
```
state: {
  count : 0
}
```
然后，执行了 state.count ++ , 也就是 Store._vm.$$state.count ++, 然后触发了Vue的响应式。
实际上，我们的computed 中也订阅了 state的变化，所以，我们的getters 就是响应式的。

然后我们再看一下```dispatch```

```
 dispatch (_type, _payload) {
    // check object-style dispatch
    const {
      type,
      payload
    } = unifyObjectStyle(_type, _payload)

    const action = { type, payload }
    const entry = this._actions[type]
    if (!entry) {
      if (process.env.NODE_ENV !== 'production') {
        console.error(`[vuex] unknown action type: ${type}`)
      }
      return
    }

    this._actionSubscribers.forEach(sub => sub(action, this.state))

    return entry.length > 1
      ? Promise.all(entry.map(handler => handler(payload)))
      : entry[0](payload)
  }
```

中间过程几乎都差不多，最后只是return 我们在 action 中的写的 函数。