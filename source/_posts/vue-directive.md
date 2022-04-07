---
title: vue自定义指令的魅力
date: 2019-11-11 10:38:31
cover: '2701794-250949a6bda06b20.png'
tags: Vue
---

## vue自定义指令的魅力

默认的内置指令：``v-html``、``v-if``、``v-show``、``v-for``等，除此之外vue允许注册自定义指令。

> 指令格式：v-name:arg.modifier="value"

 ``v-on:click.stop = 'clickFn'（简写@click.stop="clickFn"）``

指令定义函数提供了几个钩子函数：
1. **bind**-只调用一次，指令第一次绑定到元素时调用
2. insert
3. update-当前绑定元素有任意改变时触发
4. componentUpdated
5. **unbind**-只调用一次，指令与元素解绑时调用(dom节点移除时调用，v-if)
![指令生命周期](2701794-250949a6bda06b20.png)



### 滚动条实例
产品需求：默认情况下滚动条不展示，滚动起来才出现滚动条
注册指令
```javascript
Vue.directive('scroll', {
    bind(el, binding){
        // binding = { name, arg, value, modifiers, oldValue }
        
        // scroll-normal类将滚动条设置为不可见
        el.classList.add('scroll-normal')
        el.addEventListener('scroll', () => {
            el.classList.remove('scroll-normal')
            setTimeout(() => el.classList.add('scroll-normal'), 1000)
        }, false)
    },
    unbind(el){
        // .....
    }
});
```

使用指令
```vue
<div class="wrap" v-scroll></div>
```

### 页面埋点
注册指令
```javascript
Vue.directive('log', {
    bind(el, { value, modifiers }){
        if(!value) return;
	    let { event = 'click', data } = value;
        // 事件
        let { stop } = modifiers;
        // 加载完发送埋点
	    if(event === 'onload') {
		    return setTimeout(() => sendLog(data), 0);
	    }
        // 标识，方便其他钩子调用
	    el[ctx] = {
		    event,
		    data,
		    bindFn(e){
              if(stop) e.stopPropagation();
			    setTimeout(() => sendLog(el[ctx] ? el[ctx].data : data), 0);
		    }
	    }

	    el.addEventListener(event, el[ctx].bindFn, false)
    },
    unbind(el){
        if(el[ctx]){
		    el.removeEventListener(el[ctx].event, el[ctx].bindFn, false)
	    }
    }
});
// 发送埋点
const sendLog = (log) => {
    // 处理埋点规则
	// ...
	TA.log(log)
}
```
使用指令
```html
<div v-log="{data: 'msg_log' }">点击埋点</div>
<div v-log="{event: 'onload', data: 'msg_log' }">进页面埋点</div>
```


> PS：需要对纯dom元素进行底层操作


