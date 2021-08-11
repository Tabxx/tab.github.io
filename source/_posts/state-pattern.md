---
title: 状态模式
date: 2019-09-26 14:57:23
cover: 'rollingLoad.png'
tags: 设计模式
---

## 状态模式

> 状态模式允许一个对象在其内部状态改变的时候改变它的行为，对象看起来似乎修改了它的类

```javascript
const obj = {
  state: ''
  action(){}
}
```

对象的行为取决于它的状态，并且它必须在运行时刻根据状态改变它的行为



#### 简单实现

王者荣耀就可能同时有好几个状态（攻击，移动，回城，死亡）等

```javascript
// if-else款
const change = state => {
  if (state == '攻击') {
    console.log('攻击')
  } else if (state == '移动') {
    console.log('移动')
  } else if (state == '回城') {
    console.log('回城')
  } else if (state == '死亡') {
    console.log('泉水看戏')
  }
}

change('攻击')
```

如果对这些动作进行处理判断，需要一堆 if-else 或者 switch，当遇到有组合动作的时候（边走边 A），实现起来就会变的更为复杂。

```javascript
// 状态模式款
class kingGlory {
  constructor() {
    this._currentState = []
    this.states = {
      attack() {
        console.log('攻击')
      },
      move() {
        console.log('移动')
      },
      back() {
        console.log('回城')
      },
      death() {
        console.log('泉水看戏')
      }
    }
  }

  change(arr) {
    // 更改当前动作
    this._currentState = arr
    return this
  }

  go() {
    this._currentState.forEach(T => this.states[T] && this.states[T]())
    return this
  }
}

new kingGlory().change(['move']).go()
// var king = new kingGlory()
// king.change(['move','attack']).go()
```

状态模式的思路是：首先创建一个状态对象保存状态变量，然后封装好每种动作对应的状态，然后状态对象返回一个接口对象，它可以对内部的状态修改或者调用。

### 使用场景

滚动加载，包含了初始化加载、加载成功、加载失败、滚动加载等状态，任意时间它只会处于一种状态

![loading](rollingLoad.png)

```javascript
class rollingLoad {
  constructor() {
    this._currentState = 'init'
    this.states = {
        init: { failed: 'error' },
      	init: { complete: 'normal' },
      	normal: { rolling: 'loading' },
      	loading: { complete: 'normal' },
      	loading: { failed: 'error' },
    }
    //[
    //	{ name: 'init', from: 'failed', to: 'error' },
    //	{ name: 'init', from: 'complete', to: 'normal' },
   // 	{ name: 'normal', from: 'rolling', to: 'loading' },
   // 	{ name: 'loading', from: 'complete', to: 'normal' },
   // 	{ name: 'loading', from: 'failed', to: 'error' },
   // ],
    this.actions = {
        init() {
        	console.log('初始化加载，大loading')
      	},
      	normal() {
        	console.log('加载成功，正常展示')
      	},
      	error() {
        	console.log('加载失败')
      	},
      	loading() {
        	console.log('滚动加载')
      	}
      	// .....
    }
  }

  change(state) {
    // 更改当前状态
    let to = this.states[this._currentState][state]
    if(to){
        this._currentState = to
      	this.go()
        return true
    }
    return false
  }

  go() {
    this.actions[this._currentState]()
    return this
  }
}
```

vue 中的使用

```vue
// 使用
mounted(){
  rollingLoad.go()	
}

methods:{
  getData(){
    this.$api.xxxAPI.getXXX().then(res=>{
      if(res.code == 0){
        rollingLoad.change('complete')
      } else {
        rollingLoad.change('failed')
      }
    })
  },
  rolling(){
    rollingLoad.change('loading')
  },
}
```


### 总结
1. 代码中包含大量与对象状态有关的条件语句（if else或switch case），且这些条件执行依赖于该对象的状态
2. 对象的行为取决于它的状态，状态之间存在联系






### 状态模式和策略模式的区别
1.策略类的各个属性之间是平等平行的，它们之间没有任何联系
2.状态机中的各个状态之间存在相互切换，且是被规定好了的。
