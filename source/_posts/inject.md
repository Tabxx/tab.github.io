---
title: Vue3.0——Provide/Inject的妙用
date: 2021-05-15 23:07:43
cover: 'top.jpg'
---


# Vue3.0——Provide/Inject的妙用

说起Vue的状态管理工具，应该大部分人都会想到Vuex，毕竟是官方提供的工具，但是都进入Vue3时代了，不会还有人不知道依赖注入吧。
为什么会突然提到依赖注入呢？跟状态管理工具有什么关系呢？
当然有关系，因为依赖注入在Vue3中可以替代Vuex。我们知道Vue3提供的ref/reactive API具有组件解耦的特性，也就是说我们可以在组件之外创建响应式数据，这么一来跨组件共享数据的需求就在Vue3新框架内部得到了解决。



## Provide / Inject

> 通常，当我们需要从父组件向子组件传递数据时，我们使用 props。如果组件层级比较深，那通过props传递下去会比较麻烦。针对这种情况我们可以使用一对 provide 和 inject。无论组件层次结构有多深，父组件都可以作为其所有子组件的依赖提供者。这个特性有两个部分：父组件有一个 provide 选项来提供数据，子组件有一个 inject 选项来开始使用这些数据。

Provide / Inject不是什么新的api，在Vue2的时候就已经存在了，只不过在Vue3得到新的应用。这里只做一个简单的介绍，需要详细的介绍见[官方文档](https://v3.cn.vuejs.org/guide/component-provide-inject.html#%E5%A4%84%E7%90%86%E5%93%8D%E5%BA%94%E6%80%A7)。

![components_provide](components_provide.png)

我们理解一些官方给出的示例图，粗暴点来说就是父组件注入一些数据，所有子组件不管嵌套有多深都能直接获取到这些数据。



## Provide / Inject 使用方式

首先在父级组件`App.vue`注入数据
```javascript
<template>
  <button @click="editUserInfo">修改用户信息</button>
</template>

<script lang="ts">
import { defineComponent, provide, reactive } from 'vue'
import Child from './components/Child.vue'

export default defineComponent({
  name: 'App',
  components: {
    Child
  },
  setup() {
    const userInfo = reactive({
      username: '',
      age: 25
    })
    // 依赖注入
    provide('userInfo', userInfo)

    // 修改依赖数据
    const editUserInfo = () => {
      userInfo.age = 18
      userInfo.username = 'Tab'
    }

    return { userInfo, editUserInfo }
  }
})
</script>
```

子组件获取注入的数据项

```javascript
<template>
  <div class="inject-box">
    username: {{ userInfo.username }}, age: {{ userInfo.age }}
  </div>
</template>

<script lang="ts">
import { reactive, defineComponent, inject } from 'vue'
export default defineComponent({
  name: 'Child',
  setup: () => {
    // 获取注入数据
    const userInfo = inject('userInfo');
    return { userInfo }
  }
})
</script>
```

## Provide / Inject 原理

provide/inject原理并不复杂，源码的实现也很简单。在看源代码之前，我们可以先看一眼`Vue`实例上有没有`provide`和`inject`的线索。如果`Vue`实例上就存在相关属性，那么我们就需要从`Vue`实例的声明开始看源码，否则就可以猜测`provide`和`inject`是工具函数，直接搜对应的代码实现即可。

1. 万能的`console.log`查看`Vue`实例属性。(`Vue3`不能直接打印`this`,需要导入`getCurrentInstance`来获取当前实例)

```typescript
<script lang="ts">
import { defineComponent, getCurrentInstance } from 'vue'
export default defineComponent({
  name: 'HelloWorld',
  setup: () => {
    // 获取当前组件实例并在控制台输出
    const instance = getCurrentInstance()
    console.log(instance)
  }
})
</script>
```

查看控制台日志发现，`Vue`实例中存在一个`provides`属性，那就说明`Vue`实例声明的时候肯定做了一些什么才会有了`provide`和`inject`的存在。

![](vue_instance.png)

2. 然后我们就可以转到源码声明`Vue`实例的地方，源码位置`./packages/runtime-core/src/component.ts`

```typescript
export function createComponentInstance(
  vnode: VNode,
  parent: ComponentInternalInstance | null,
  suspense: SuspenseBoundary | null
) {
  const type = vnode.type as ConcreteComponent
  // appContext继承父组件的上下文，如果是根节点采用从根vnode
  const appContext =
    (parent ? parent.appContext : vnode.appContext) || emptyAppContext

  const instance: ComponentInternalInstance = {
    uid: uid++,
    vnode,
    type,
    parent,
    appContext,
    // provides 继承父组件的provides
    provides: parent ? parent.provides : Object.create(appContext.provides),
    // ................. 
  }
  if (__DEV__) {
    instance.ctx = createRenderContext(instance)
  } else {
    instance.ctx = { _: instance }
  }
  instance.root = parent ? parent.root : instance
  instance.emit = emit.bind(null, instance)
  return instance
}

// ./packages/runtime-core/src/apiCreateApp.ts
export function createAppContext(): AppContext {
  return {
    // ..... 忽略其他属性
    // emptyAppContext中provides默认是个空对象
    provides: Object.create(null)
  }
}
```

上述代码可以看出，每个组件的`provides`都是继承自父组件的`provides`，根组件的`provides`其实是个空**对象**。

看到这里大致可以猜出来`provide/inject`是什么了，剩下我们只要了解下`provide`和`inject`两个工具函数的原理就能明白其中的缘由了。

3. `provide`方法原理

> provide 方法的作用是往当前实例provides写入新的key-value,如果key已存在则覆盖

```typescript
export function provide<T>(key: InjectionKey<T> | string | number, value: T) {
  if (!currentInstance) {
    if (__DEV__) {
      warn(`provide() can only be used inside setup().`)
    }
  } else {
    let provides = currentInstance.provides
    const parentProvides =
      currentInstance.parent && currentInstance.parent.provides
    if (parentProvides === provides) {
      provides = currentInstance.provides = Object.create(parentProvides)
    }
    // provides 中注入新值, 如果根组件与父组件有相同的key则会被父组件注入的key-value覆盖
    provides[key as string] = value
  }
}
```

4. `inject`方法原理

> inject 方法的作用是从当前实例的provides获取key对应的value值。

```typescript
export function inject(
  key: InjectionKey<any> | string,
  defaultValue?: unknown,
  treatDefaultAsFactory = false
) {
  // 当前组件的实例
  const instance = currentInstance || currentRenderingInstance
  if (instance) {
    // 实例父级的provides；如果组件为根组件，就是自身appContext的provides
    const provides =
      instance.parent == null
        ? instance.vnode.appContext && instance.vnode.appContext.provides
        : instance.parent.provides
	// 判断provides对象中是否存在key属性, 有就返回对应的value
    if (provides && (key as string | symbol) in provides) {
      return provides[key as string]
    } else if (arguments.length > 1) {	// 判断当前函数是否存在第二参数
      // 如果treatDefaultAsFactory为true且第二个参数是个函数, 直接返回这个函数的默认结果, 否则会直接返回第二个参数。
      // 作用：provides中不存在当前key时起到默认值的作用
      return treatDefaultAsFactory && isFunction(defaultValue)
        ? defaultValue()
        : defaultValue
    } else if (__DEV__) {
      warn(`injection "${String(key)}" not found.`)
    }
  } else if (__DEV__) {
    warn(`inject() can only be used inside setup() or functional components.`)
  }
}
```

源码总结：子组件继承父组件的`provides`属性，从而达到深层的组件也能访问到注入组件的数据。


##  简易版Provide / Inject

通过上面源码的学习，我们可以自己写一个简易版的`Provide / Inject`，原理就是es6的class继承，直接附上代码。

```javascript
class Root {
	static provides = {}
	constructor() {}
 	provide(key, value) {
		Root.provides[key] = value
	}
	inject(key, defaultValue, treatDefaultAsFactory) {
		if (key && key in Root.provides) {
			return Root.provides[key]
		} else if (arguments.length > 1) {
			return treatDefaultAsFactory && typeof defaultValue === 'function' ? defaultValue() : defaultValue
		}
	}
}
//  继承root
class Parent extends Root {
	constructor() {
		super();
	}
	provide(key, value) {
         // 重写父类provide，并调用父类provide方法
		super.provide(key, value)
	}
}

class Child extends Parent {
	constructor() {
		super();
	}
}

let rootInstance = new Root();
rootInstance.provide('username', 'Yxx')
let parentInstance = new Parent();

let instance = new Child();
parentInstance.provide('age', 18);
console.log(instance.inject('age'))
console.log(instance.inject('username'))
```

## 应用场景（妙用）

重点来了，了解完原理之后我们来看下`Provide / Inject`的应用场景，如何来代替现在的状态管理工具`Vuex`呢？

1. 首先创建一个`store`文件夹（文件夹名称可随意修改）用于存放共享数据相关的代码，并且创建一个入口文件`index.ts`

```typescript
import { reactive, readonly } from "vue";

// interface--导出接口用于变量类型声明(ts专用)
export interface Store {
  state: {
    count: number;
  },
  increment(): void,
}

// state--存放需要全局共享的数据。通过reactive创建对象，为了实现响应式，如果直接赋值普通对象或基本类型，则不会有响应式
const state = reactive({
  count: 0,
});
// 修改state的方法，类似于vuex的mutations
const increment = (): void => {
  state.count++;
}

// 导出数据和方法。readonly为防止调用方直接修改state
export const global: Store = {
  state: readonly(state),
  increment,
}
```

2. 然后在根组件(`App.vue`)注入依赖数据

```javascript
<script lang="ts">
import { defineComponent, getCurrentInstance } from 'vue'
// 导入store
import { global } from './store/index'

export default defineComponent({
  name: 'App',
  // 注入依赖数据
  provide: {
    global
  },
})
</script>
```

3. 最后子组件通过`inject`获取注入的数据，以及调用相关的方法来修改注入的数据

```typescript
<template>
  <div class="inject-box">
    <p>{{ global.state.count }}</p>
    <button @click="global.increment">++</button>
  </div>
</template>

<script lang="ts">
import { ref, defineComponent} from 'vue'
export default defineComponent({
  name: 'HelloWorld',
  // 获取注入的数据，模板内可直接通过global.xxxxx调用
  inject: ['global'],
  setup: () => {}
})
</script>
```

4. 通过前3步，已经完成了全局状态管理。但是有一个问题，当项目特别庞大，全部的状态数据存在一份代码里面肯定是不合理的。所以接下去再实现下模板化，使状态数据能模块化管理。
5. 创建`modules`文件夹存放模块数据，并创建`user.ts`

```typescript
import { reactive, readonly } from "vue";
// interface
export interface UserStore {
  state: {
    username: string;
  },
  editUserName(username: string): void
}
// state
const state = reactive({
  username: ""
})
// 修改state的方法
const editUserName = (username: string) => {
  state.username = username;
}

export const User = reactive({
  state: readonly(state),
  editUserName
});
```

6. 在`store/index.ts`导入`user`模块

```typescript
// 省略部分代码
import { reactive, readonly, toRefs } from "vue";
// 导入模块
import { User } from './modules/user';
// state
const state = reactive({
  count: 0,
  // 安装模块数据到state
  ...toRefs(User.state)
});

export const global: Store = {
  state: readonly(state),
  // 导出模块暴露的方法
  editUserName: User.editUserName
}

```

7. 最后我们在子组件调用即可，方法同第3步类型

```typescript
<template>
  <div class="inject-box">
    <p>{{ global.state.count }}</p>
    <button @click="global.increment">++</button>
  </div>
  <div class="inject-box">
    <p>{{ global.state.username }}</p>
    <input type="text" v-model="uname" />
  </div>
</template>

<script lang="ts">
import { ref, defineComponent, inject, watch } from 'vue'
// 导入变量类型
import { Store } from '../store/index'
export default defineComponent({
  name: 'HelloWorld',
  inject: ['global'],
  setup: () => {
    const uname = ref('')
    // setup 内使用inject方法获取到注入的数据
    const global: Store | undefined = inject('global')

    watch(uname, (newValue: string): void => {
      // 调用 editUserName 方法修改state里面的username
      global && global.editUserName(newValue)
    })

    return { uname }
  }
})
</script>
```



<div class="aplayer no-destroy" data-id="热门" data-server="tencent" data-type="search" data-fixed="true" data-mini="true" data-listFolded="false" data-order="random" data-preload="none" data-autoplay="true" muted></div>















































































































