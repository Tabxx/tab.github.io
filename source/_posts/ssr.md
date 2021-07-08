---
title: 性能优化之SSR初体验
date: 2021-07-08 14:37:22
tags: SSR
cover: 'ssr.jpg'
---


##  性能优化之SSR初体验

###  SSR是什么？

> Server Side Render简称SSR。具体指html页面内容通过服务端渲染生成，浏览器接收到服务器响应时直接显示对应的内容就可以了。

![ssr](ssr.jpg)

了解完`SSR`就必须知道和它相爱相杀的`CSR`。`CSR`全称`Client Side Render`，页面上的内容是js动态渲染出来的（`Vue`、`React`等框架），服务端只返回一个html模板，其余渲染在浏览器完成。

![csr](csr.jpg)

`SSR`也不算是一个新的概念，早期的后端`PHP`、`JAVA`就已经支持服务端渲染，美其名曰`模板引擎`，如`phtml`、`jsp`、`jade`等。最近几年前端工程化的日渐成熟，前后端开始分离，将原本的`SSR`模式转变成了`CSR`。同时也带来了一些性能上的缺陷以及`SEO`问题等等。

### SSR优劣势

对比上面两张图`SSR`的优势很明显，在进行到第二阶段下载`js`文件时页面已经呈现，而`CSR`需要到第四阶段可交互时才显示页面。这对于**首屏性能**和**搜索引擎`SEO`**有着天然的好处。

当然`SSR`的劣势不小，对于现有的`Vue`或者`React`项目迁移成本很大；消耗服务器资源；框架生态支持不太完美。

从个人经验上来说，除非对项目首屏性能或者`SEO`有比较高的要求且可以容忍它的副作用时才使用，否则不建议直接使用`SSR`来开发项目。



### SSR 实现的本质-虚拟dom

首先在`SSR`工程中，`Vue`代码（本文以vue举例，其他框架同理）会在客户端和服务端各执行一次。为什么会各执行一次呢？因为页面需要在服务端渲染完成再返回给浏览器，因此就必须在服务端先执行一次。还有一个问题就是服务端是怎么运行`Vue`代码的？这一点你可能觉得很好理解，毕竟`Node`也是基于`JavaScript`开发的，能够运行很正常。但是如果代码中涉及到`dom`的操作在`Node`环境是不支持的，`Node`环境下没有`DOM`的概念，甚至没有`window`这个全局变量的存在。

因此服务端直接操作`Dom`肯定是不现实的，好在`Vue`框架有一个虚拟`Dom`的概念。虚拟`Dom`是真实`Dom`的映射，简单理解为就是一个普通的js对象，我们在用`Vue`操作页面时其实操作的都是这个js对象，而非真实的`Dom`，这就使得`SSR`成为了可能。

经过上述分析，我们就能分析出所谓的服务端渲染就是在服务端完成虚拟`Dom`的组装，浏览器只要将虚拟`Dom`转换成真实`Dom`就可以了。相比`CSR` 的请求数据->组装虚拟`Dom`->渲染，少了两步，少的这两步都在服务端完成，且性能比用户浏览器更好。

那为什么说性能会更好呢？个人觉得主要有两个原因，首先服务器的性能比用户本地系统要好，而且不考虑money的情况下，可以一直加配置。其次服务器请求接口数据，远比用户浏览器发起`http`请求快的多。因为服务器请求是直接请求集群的，没有多余的`DNS`等解析过程，而且不受用户本地网络的影响。



### SSR实战-大数据表格渲染

#### 前期准备

- Vue-cli3
- Nuxt
- koa (用于模拟接口数据)



####  1. 模拟接口数据

模拟数据来自某顺的爱基金数据，接口采用读取本地文件的形式，不采用爬虫的形式实时抓取（保护下数据源的安全）。

```javascript
const Koa = require('koa');
const router = require('koa-router')();
const app = new Koa();
const cors = require('koa-cors');
// 解决跨域问题
app.use(cors());

router.get('/jijin', async (ctx, next) => {
	const fs = require('fs');
    // 读取本地json文件作为接口响应数据
	const data = fs.readFileSync('jijin.json', 'utf-8');
	ctx.body = data;
})

app.use(router.routes());
app.listen(5000);
console.log('app started at port 5000...');
```



#### 2. 基于Vue-cli3  + element UI搭建一个表格

核心代码如下，其余部分代码参考[Vue CLI ](https://cli.vuejs.org/zh/guide/) 和 [Element](https://element.eleme.cn/#/zh-CN/component/installation)

```vue
<template>
  <div id="app">
    <el-table
      :data="tableData"
      height="500px"
      style="width: 100%">
      <el-table-column
        type="index"
        label="序号"
        fixed
        width="100">
      </el-table-column>
      <el-table-column
        prop="code"
        label="代码"
        fixed
        width="100">
      </el-table-column>
      <el-table-column
        prop="SYENDDATE"
        label="更新日期"
        fixed
        width="180">
      </el-table-column>
      <el-table-column
        prop="F005"
        label="F005"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F008"
        label="F008"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F009"
        label="F009"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F010"
        label="F010"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F011"
        label="F011"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F012"
        label="F012"
        width="180">
      </el-table-column>
      <el-table-column
        prop="net"
        label="net"
        width="180">
      </el-table-column>
      <el-table-column
        prop="net1"
        label="net1"
        width="180">
      </el-table-column>
       <el-table-column
        prop="prerate"
        label="prerate"
        width="180">
      </el-table-column>
      <el-table-column
        prop="typename"
        label="类型">
      </el-table-column>
    </el-table>
  </div>
</template>

<script>


export default {
  name: 'App',
  
  data() {
    return {
      tableData: []
    }
  },

  created() {
    this.getData();
  },

  methods: {
    getData() {
      fetch('http://127.0.0.1:5000/jijin').then(res => {
        return res.json();
      }).then(res => {
        const { data } = res.data;
        const MAX_LIST = 200;	// 数据太多控制下数量
        let i = 0;
          
        this.tableData = [];
        for(let key in data) {
          if(i === MAX_LIST) {
            break;
          }
          this.tableData.push(data[key]);
          i++;
        }
      })
    }
  }
}
</script>
```

运行效果：

![运行效果](bt.gif)



#### 3. nuxt 改造表格实现SSR

`nuxt.js` 食用方式见[Nuxt.js - Vue.js 通用应用框架 | Nuxt.js 中文网 (nuxtjs.cn)](https://www.nuxtjs.cn/)

`nuxt`的代码结构与`Vue-cli`差别不大，不需要过多的改动（仅限于简单demo项目，复杂项目迁移是比较麻烦的），所以只需要实现服务端异步获取数据这个逻辑即可。

`nuxt`提供了服务端异步获取数据的方法`asyncData`，但是有局限性只适用于页面级组件，肤浅的理解就是项目`pages`目录下的组件，具体实现如下：

`pages/Index.vue`：

```vue
<template>
  <Tutorial :tableData2="tableData"/>
</template>

<script>
export default {

  data() {
    return {
      tableData: []
    }
  },
  // 异步获取数据，并传递给子组件
  async asyncData() {
    return await fetch('http://localhost:5000/jijin').then(res => {
        return res.json();
      }).then(res => {
        let tableData = [];
        const { data } = res.data;
        let i = 0;
        const MAX_LIST = 200;
        for(let key in data) {
          if(i === MAX_LIST) {
            break;
          }
          tableData.push(data[key]);
          i++;
        }
        return {
          tableData
        }
    })
  }
}
</script>
```

`components/Tutorial.vue`，该组件是在浏览器执行，因为上级页面组件已经在服务端取到了数据，所以不需要调用接口，直接通过`props`获取数据渲染即可。

```vue
<template>
  <div id="app">
    <el-table
      :data="tableData2"
      height="500px"
      style="width: 100%">
      <el-table-column
        type="index"
        label="序号"
        fixed
        width="100">
      </el-table-column>
      <el-table-column
        prop="code"
        label="代码"
        fixed
        width="100">
      </el-table-column>
      <el-table-column
        prop="SYENDDATE"
        label="更新日期"
        fixed
        width="180">
      </el-table-column>
      <el-table-column
        prop="F005"
        label="F005"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F008"
        label="F008"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F009"
        label="F009"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F010"
        label="F010"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F011"
        label="F011"
        width="180">
      </el-table-column>
      <el-table-column
        prop="F012"
        label="F012"
        width="180">
      </el-table-column>
      <el-table-column
        prop="net"
        label="net"
        width="180">
      </el-table-column>
      <el-table-column
        prop="net1"
        label="net1"
        width="180">
      </el-table-column>
       <el-table-column
        prop="prerate"
        label="prerate"
        width="180">
      </el-table-column>
      <el-table-column
        prop="typename"
        label="类型">
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
export default {
  props: ['tableData2'],
}
</script>
```

运行效果：

![运行效果](ssr.gif)



#### 4. 对比结果

- 未优化的表格有很长时间的空态，`SSR`优化后感知不到空态

- `ssr`优化后页面html文件请求速度变慢，但是页面展现数据速度有很大提升
- 性能数据(`finish_time`)没有太大改善，只是用户体验上提升了



### 一些想法

- `SSR`的部署必然涉及到服务器，那么就需要考虑到一些容灾处理，比如`ssr`服务被打爆了如何做降级等
-  旧项目迁移成本很大，耗时耗力，最好有自动迁移的工具或者脚本支持
- 对于没有服务端知识的前端开发来说学习成本略大，需要清楚服务端的一些配置等等
- 容忍不了`SSR`缺点的情况下不建议用`SSR`

![](k.png)


