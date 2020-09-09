---
title: computed 实现原理
date: 2019-11-18 10:35:42
cover: 'topbg.jpg'
---

# computed 实现原理

> 模板内的表达式非常便利，但是设计它们的初衷是用于简单运算的。在模板中放入太多的逻辑会让模板过重且难以维护。
> 计算属性是基于它们的响应式依赖进行**缓存**的。只在相关响应式依赖发生改变时它们才会重新求值

那么计算属性为何可以做到当它的依赖项发生改变时才会进行重新的计算，否则当前数据是被缓存的？

## 使用

```javascript
export default {
  computed: {
    // function
    status() {
      return this.$store.state.status
    },
    // getter
    fullName: {
      // getter
      get: function() {
        return this.firstName + ' ' + this.lastName
      },
      // setter
      set: function(newValue) {
        var names = newValue.split(' ')
        this.firstName = names[0]
        this.lastName = names[names.length - 1]
      }
    }
  }
}
```

## 实现原理

1. 初始化``computed``遍历当前实例下computed的所有属性，源码`src/core/instance/state.js`

```javascript
const computedWatcherOptions = { lazy: true }
function initComputed(vm: Component, computed: Object) {
  // 遍历computed，因为computed本身就是一个对象
  for (const key in computed) {
    const userDef = computed[key]
    // getter记录当前计算属性的回调函数，计算属性有两种写法，function or gettter见上文使用
    const getter = typeof userDef === 'function' ? userDef : userDef.get

    if (!isSSR) {
      // 实例化一个watcher对象
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }
   // computed不能与data、props重名
    if (!(key in vm)) {
      // 不在vue实例中就定义
      defineComputed(vm, key, userDef)
    }
  }
}
```

2. 将定义的computed属性的每一项使用Watcher类进行实例化，不过这里是按照computed-watcher的形式。源码在`src/core/observer/watcher.js`

```typescript
/**
 * 传入的参数
 * vm           实例化一个vue
 * expOrFn   getter 回调函数
 * cb           noop
 * options    { lazy: true }
 * */
constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    vm._watchers.push(this)
    // options
    if (options) {
      // 传入参数中lazy为true，用于标识是计算属性的watcher
      this.lazy = !!options.lazy
    }
    // lazy 赋值给dirty    旧版没有lazy只要dirty
    this.dirty = this.lazy
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      // watcher 有一种$route.path写法
      this.getter = parsePath(expOrFn)
    }
    // computed不会立马取值
    this.value = this.lazy
      ? undefined
      : this.get()
  }
```

3. vue 实例中就定义计算属性，让computed成为一个响应式数据，并定义它的get属性。

```javascript
/**
 * 传入的参数
 * vm
 * key     当前计算属性key
 * userDef 当前计算属性对象
 * */

// sharedPropertyDefinition用于Object.defineProperty()调用
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}
export function defineComputed(
  target: any,
  key: string,
  userDef: Object | Function
) {
  // 是否为服务器渲染
  const shouldCache = !isServerRendering()
  // function和getter两种写法分别处理
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
// 创建计算属性get方法
function createComputedGetter(key) {
  return function computedGetter() {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      // 前面初始化watcher时将lazy赋值给dirty
      // computed缓存
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```

4. 订阅者触发更新

```javascript
// dep调度中心收集watcher和通知消息
export default class Dep {
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
  // 通知观察者
  notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      // 调用watcher 的update方法
      subs[i].update()
    }
  }
}

// 重新计算computed的值，watcher.js
evaluate () {
  this.value = this.get()
  this.dirty = false
}
// 获取当前computed对应的值
get () {
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      value = this.getter.call(vm, vm)
    }
    return value
}
```

## 分析完源码总结一下流程
1. 初始化``computed`` 调用 ``initComputed`` 函数
2. 注册``watcher``实例，让计算属性成为其他``  watcher ``的消息订阅器的订阅者
3. 定义计算属性``Object.defineProperty``的``get``访问器函数，进而调用``watcher``的``evaluate``来获取computed的值
4. 当某个属性发生变化，触发 ``set`` 拦截函数，然后调用自身消息订阅器`` dep`` 的`` notify`` 方法，遍历当前`` dep ``中保存着所有订阅者并逐个调用`` watcher`` 的`` update`` 方法，完成响应更新。

