# vuex源码学习

## Module

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
new Moudle 处理以后，我们的新moudle 变成了
```
{
  state : {
    count: 0,
  },
  runtime: false,
  _children: {}，
  _rawMoudule: options
}
```
经过我们的 moduleCollction 构造函数以后。

```
{
  root: Module
}
```

## installModule

```
installModule(this, state, [], this._modules.root)

实际上传入的参数就是 ,state = {count: 0},
root: {
  state : {
    count: 0,
  },
  runtime: false,
  _children: {}，
  _rawMoudule: options,
  // 各种 Module的方法
  ...
}

context = {
  dispatch: store.dispath,
  commit :  store.commit
  getters:  // 只读的store.getters
  state: // 制度的store.state
}

然后执行的是， 初始化 _mumations
对每一个 原始的 mumations 做了一个遍历， 同一类型的 [type], 也就是 opitons.mumation 的key ,新建了一个数组，然后，然后里面保存着
所有的我们之前的同名 mumations 这个对象里面的方法。

我们之前的commit 函数，其实就是提交的mumations 方法。 这个方法到底是如合做拿到 state，我们在这也能得出答案。 穿进去的就是local.state。

local.state，但是，这个值不应该是只读的么？ 因为我们只给了get属性。



