---
title: Vue3.0——reactive和ref原理
date: 2021-02-23 10:44:22
cover: 'reactive.png'
tags: Vue
---

# Vue3.0——reactive和ref原理

Vue3采用了新的`Proxy`实现数据拦截，但`Proxy`还是只能代理一层。对于深层的对象需要递归代理，相比2.0在初始化递归，3.0的运行时递归，大大减少了初始化的递归性能消耗。



## Proxy

首先简单了解下`Proxy`，方便后面学习`reactive`和`ref`原理。

> **Proxy** 对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）。

**基础示例：**

```javascript
const handler = {
    // 属性读取操作的捕捉器
    get: function (target, prop) {
    	console.log('getting key: ' + prop);
        return target[prop];
    },
    // 属性设置操作的捕捉器
    set: function (target, prop, value) {
    	console.log('setting key: ' + prop + ', value: ' + value);
    	target[prop] = value;
    	return true;
    }
};

const p = new Proxy({}, handler);
p.a = 1;			// setting key: a, value: 1
p.b = undefined;	// setting key: b, value: undefined

console.log(p.a, p.b);      
// getting key: a
// getting key: b
// 1, undefined
console.log('c' in p, p.c); 
// getting key: c
// false, undefined
```

`Proxy`一起的还有另外一位小伙伴`Reflect`。

`Reflect`对象的方法与`Proxy`对象的方法一一对应，只要是`Proxy`对象的方法，就能在`Reflect`对象上找到对应的方法。这就让Proxy对象可以方便地调用对应的`Reflect`方法，完成默认行为，作为修改行为的基础。也就是说，不管`Proxy`怎么修改默认行为，你总可以在`Reflect`上获取默认行为。

```javascript
const handler = {
    // 属性读取操作的捕捉器
    get: function (target, prop, receiver) {
    	console.log('getting key: ' + prop);
        // ---> Relefct.get
        return Reflect.get(target, prop, receiver);
    },
    // 属性设置操作的捕捉器
    set: function (target, prop, value, receiver) {
    	console.log('setting key: ' + prop + ', value: ' + value);
        // ---> Relefct.set
    	return Reflect.set(target, prop, value, receiver);
    }
};
```

Proxy API具体用法详见官方文档。[MDN文档传送门](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

## 为什么要选用Proxy

 Vue3.0 为什么选用`Proxy`而不是沿用 `Object.defineProperty`呢？

**Proxy 的优势如下:**

- Proxy 可以直接监听对象而非属性；
- Proxy 可以直接监听数组的变化；
- Proxy 有多达 13 种拦截方法,不限于 apply、ownKeys、deleteProperty、has 等等是 Object.defineProperty 不具备的；
- Proxy 返回的是一个新对象,我们可以只操作新的对象达到目的,而 Object.defineProperty 只能遍历对象属性直接修改；
- Proxy 作为新标准将受到浏览器厂商重点持续的性能优化，也就是传说中的新标准的性能红利；

**Object.defineProperty 的优势如下:**

- 兼容性好，支持 IE9，而 Proxy 的存在浏览器兼容性问题,而且无法用 polyfill 磨平，因此 Vue 的作者才声明需要等到下个大版本( 3.0 )才能用 Proxy 重写。



对于监听数组变化我们可以来做一个对比

**Proxy代理数组：**

```javascript
let arr = [1, 1, 2, 3];
let handler = {
	get	(target, prop, receiver) {
		console.log('getting key: ' + prop);
		return Reflect.get(target, prop, receiver);
	},
	set (target, prop, value, receiver) {
		console.log('setting key: ' + prop + ', value: ' + value);
		return Reflect.set(target, prop, value, receiver);
	}
}
let proxyArray = new Proxy(arr, handler);

proxyArray.push(4);
// 往数组push时，length也会随之改变
// getting key: push
// getting key: length
// setting key: 3, value: 4
// setting key: length, value: 4
proxyArray[0] = 0;
// setting key: 0, value: 0
```

**Object.defineProperty **代理数组：

```javascript
const orginalProto = Array.prototype;	// Array原型
const arrayProto = Object.create(orginalProto); // 复制一份原型用于重写
const methodsToPatch = [	// 需要重写的数组内置方法
	'push',
	'pop',
	'shift',
	'unshift',
	'splice',
	'sort',
	'reverse'
];	
methodsToPatch.forEach(method => {
	arrayProto[method] = function () {
		// 执行数组原始方法
		orginalProto[method].apply(this, arguments);
		// do sth 
		console.log('数组改变：', method , ...arguments);
	}
})
		
let arr = [1, 1, 2, 3];
arr.__proto__ = arrayProto;	// 改变__proto__指向
arr.push(4);	// 数组改变： push 4
arr[0] = 0;		// 没有输出
console.log(arr);// [0, 1, 2, 3, 4];
```

上面两种实现数组代理方式对比，实现上`Proxy`来的更便捷，且能监听到通过数组下标改变数组值的形式。而`Object.defineProperty`则是通过重写原型上的方式来监听数组的变化，而且监听不到通过数组下标改变数组值的形式。



## reactive 原理

一些工具函数，后面会一一用到。

```javascript
// 工具方法： 判断是否为对象
const isObject = val => val !== null && typeof val === 'object';
// 工具函数：值是否改变
const hasChanged = (value, oldValue) => value !== oldValue && (value === value && oldValue === oldValue);
// 工具方法:判断当前key是否存在
const hasOwn = (val, key) => Object.hasOwnProperty.call(val, key);
```

首先我们需要一个`reactive`方法用来创建响应式数据

```javascript
// 暴露出去的方法，reactive
function reactive(target) {
	return createReactiveObject(target, baseHandlers);
}
```

`createReactiveObject`方法创建一个Proxy实例

```javascript
// 基础的处理器对象
const baseHandlers = {
	get, set
}
// 创建一个响应式对象
function createReactiveObject(target, baseHandlers) {
	return new Proxy(target, baseHandlers);
}
```

```get/set```拦截器生成

```javascript
// 生成get
function createGetter() {
	return function get(target, key, recevier) {
		const result = Reflect.get(target, key, recevier);
		console.log(`getting key: ${key}`);

    	// 深层代理
    	if (isObject(result)) {
    		return reactive(result);
    	}
        return result
	}
}
// 生成set
function createSetter() {
	return function set(target, key, value, recevier) {
		const oldValue = target[key];
		const hadKey = hasOwn(target, key);
		const result = Reflect.set(target, key, value, recevier);

		if (!hadKey) {
			console.log(`add key：${key}，value：${value}`);
            // trigger(target, 'add' /* ADD */, key, value);
		} else if (hasChanged(value, oldValue)) {
			console.log(`set key：${key}，value：${value}`);
			// trigger(target, 'set' /* SET */, key, value, oldValue);
		}

		return result
	}
}

const get = createGetter();
const set = createSetter();
```

**测试reactive**

```javascript
const proxyObj = reactive({
	id: 1,
	name: 'tab',
	childObj: {
    	age: 24
    }
})

proxyObj.name = 'xxTab';	
// set key：name，value：xxTab
proxyObj.childObj.age = 2;
// getting key: childObj
// set key：age，value：2
```

**总结reactive`原理:**

`reactive`本质上就是一个`Proxy`实例，无非就是在`set`和`get`拦截器上加了一些自身的逻辑。比如`get`拦截器收集依赖，`set`拦截器触发相关依赖。



## ref 原理

还是一样需要一个暴露出来的`ref`方法.

```javascript
function createRef(rawValue, shallow = false) {
	return new RefImpl(rawValue, shallow)
}

function ref(value) {
	return createRef(value)
}
```

`RefImpl`核心类，实现ref的初始化，以及相应的get和set

```javascript
class RefImpl {
	constructor(_rawValue, _shallow = false) {
		this._rawValue = _rawValue;
		this._shallow = _shallow;
        this.__v_isRef = true;
		this._value = _shallow ? _rawValue : convert(_rawValue);
	}
	get value() {
    	// track(toRaw(this), 'get' /* GET */, 'value');
    	return this._value;
	}
	set value(newVal) {
    	if (hasChanged(toRaw(newVal), this._rawValue)) {
        	this._rawValue = newVal;
        	this._value = this._shallow ? newVal : convert(newVal);
        	// trigger(toRaw(this), 'set' /* SET */, 'value', newVal);
        }
    }
}
```

工具方法

```javascript
// 工具方法：判断传入的值是否是一个对象，是的话就用 reactive 来代理
const convert = val => (isObject(val) ? reactive(val) : val);

function toRaw(observed) {
	return (observed && toRaw(observed['__v_raw'])) || observed
}
```

测试ref

```javascript
let num = ref(1)
num.value = 10;

let data = ref([1, 2, 3])
data.value.push(4)
console.log(data.value[3])
```

**总结`ref`原理:**

核心类 `RefImpl` ，我们可以看到在类中使用了经典的 `get/set` 存取器，来进行追踪和触发。
`convert` 方法让我们知道了 ref 不仅仅用来包装一个值类型，也可以是一个对象/数组，然后把对象/数组再交给 `reactive` 进行代理。



## 完整代码

```javascript
// 工具方法： 判断是否为对象
const isObject = val => val !== null && typeof val === 'object';

// 工具函数：值是否改变
const hasChanged = (value, oldValue) => value !== oldValue && (value === value && oldValue === oldValue);

// 工具方法:判断当前key是否存在
const hasOwn = (val, key) => Object.hasOwnProperty.call(val, key);

// 生成get
function createGetter () {
    return function get (target, key, recevier) {
        const result = Reflect.get(target, key, recevier);
        console.log(`getting key: ${key}`);

        // 深层代理
        if (isObject(result)) {
            return reactive(result);
        }
        return result
    }
}

function createSetter () {
    return function set (target, key, value, recevier) {
        const oldValue = target[key];
        const hadKey = hasOwn(target, key);
        const result = Reflect.set(target, key, value, recevier);

        if (!hadKey) {
            console.log(`add key：${key}，value：${value}`);
            // trigger(target, 'add' /* ADD */, key, value);
        } else if (hasChanged(value, oldValue)) {
            console.log(`set key：${key}，value：${value}`);
            // trigger(target, 'set' /* SET */, key, value, oldValue);
        }

        return result
    }
}

const get = createGetter();
const set = createSetter();
// 基础的处理器对象
const baseHandlers = {
    get, set
}
// 创建一个响应式对象
function createReactiveObject (target, baseHandlers) {
    return new Proxy(target, baseHandlers);
}
// 暴露出去的方法，reactive
function reactive (target) {
    return createReactiveObject(target, baseHandlers);
}

// test
const proxyObj = reactive({
    id: 1,
    name: 'tab',
    childObj: {
        age: 24
    }
})

proxyObj.name = 'xxTab';
proxyObj.childObj.age = 2;


// 工具方法：判断传入的值是否是一个对象，是的话就用 reactive 来代理
const convert = val => (isObject(val) ? reactive(val) : val);

function toRaw (observed) {
    return (observed && toRaw(observed['__v_raw'])) || observed
}

class RefImpl {
    constructor(_rawValue, _shallow = false) {
        this._rawValue = _rawValue;
        this._shallow = _shallow;
        this.__v_isRef = true;
        this._value = _shallow ? _rawValue : convert(_rawValue);
    }
    get value () {
        // track(toRaw(this), 'get' /* GET */, 'value');
        return this._value;
    }
    set value (newVal) {
        if (hasChanged(toRaw(newVal), this._rawValue)) {
            this._rawValue = newVal;
            this._value = this._shallow ? newVal : convert(newVal);
            // trigger(toRaw(this), 'set' /* SET */, 'value', newVal);
        }
    }
}

function createRef (rawValue, shallow = false) {
    return new RefImpl(rawValue, shallow)
}

function ref (value) {
    return createRef(value)
}

function shallowRef (value) {
    return createRef(value, true)
}
/**
 * ref test
 */
// let num = ref(1)
// num.value = 10;

// let data = ref([1, 2, 3])
// data.value.push(4)
// console.log(data.value[3])

function isRef (r) {
    return Boolean(r && r.__v_isRef === true);
}

function unRef (ref) {
    return isRef(ref) ? ref.value : ref;
}
```







