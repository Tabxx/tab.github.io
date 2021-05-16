---
title: Vue3.0——Provide/Inject的妙用
date: 2021-05-15 23:07:43
cover: 'top.jpg'
---


# Vue3.0——Provide/Inject的妙用

说起Vue的状态管理工具，应该大部分人都会想到Vuex，毕竟是官方提供的工具，但是都进入Vue3时代了，不会还有人不知道依赖注入吧。不会吧，不会吧。
![不会吧不会吧](1556.jpg)
为什么会突然提到依赖注入呢？跟状态管理工具有什么关系呢？
当然有关系，关系还很特别呢，特别的绿茶。因为依赖注入在Vue3中可以替代Vuex。我们知道Vue3提供的ref/reactive API具有组件解耦的特性，也就是说我们可以在组件之外创建响应式数据，这么一来跨组件共享数据的需求就在Vue3新框架内部得到了解决。



## Provide / Inject

> 通常，当我们需要从父组件向子组件传递数据时，我们使用 props。如果组件层级比较深，那通过props传递下去会比较麻烦。针对这种情况我们可以使用一对 provide 和 inject。无论组件层次结构有多深，父组件都可以作为其所有子组件的依赖提供者。这个特性有两个部分：父组件有一个 provide 选项来提供数据，子组件有一个 inject 选项来开始使用这些数据。

Provide / Inject不是什么新的api，在Vue2的时候就已经存在了，只不过在Vue3得到新的应用。这里只做一个简单的介绍，需要详细的介绍见[官方文档](https://v3.cn.vuejs.org/guide/component-provide-inject.html#%E5%A4%84%E7%90%86%E5%93%8D%E5%BA%94%E6%80%A7)。

![components_provide](components_provide.png)

我们理解一些官方给出的示例图，粗暴点来说就是父组件注入一些数据，所有子组件不管嵌套有多深都能直接获取到这些数据。从乡下人的角度来说就是一人得道鸡犬升天，就好比爷爷有钱了，爸爸也跟富裕了，孙子呢撒个娇零花钱也到手了。

![](mq.jpg)


## Provide / Inject服用方式

首先呢，在父级组件`App.vue`注入数据
```javascript
<template>
  <Child msg="Hello Vue 3 + TypeScript + Vite" />
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



