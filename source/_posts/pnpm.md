---
title: pnpm
date: 2022-04-02 14:49:41
cover: "pnpm.jpeg"
tags: Pnpm Node
---

# pnpm

> pnpm-快速的，节省磁盘空间的包管理工具，是`Node`包管理工具，同时它也是`npm`的替代品，但比`npm`更快。

![pnpm](pnpm.jpeg)
`pnpm`的设计初衷是节约磁盘空间并提升安装速度，也是我使用决定从`npm`转`pnpm`的原因。

当使用 `npm` 或 `Yarn` 时，如果你有100个项目使用了某个依赖（dependency），就会有100份该依赖的副本保存在硬盘上。  而在使用 pnpm 时，依赖会被存储在内容可寻址的存储中，所以：

如果你用到了某依赖项的不同版本，那么只会将有差异的文件添加到仓库。 例如，如果某个包有100个文件，而它的新版本只改变了其中1个文件。那么`pnpm update`时只会向存储中心额外添加1个新文件，而不会因为仅仅一个文件的改变复制整新版本包的内容。
所有文件都会存储在硬盘上的某一位置。 当软件包被被安装时，包里的文件会硬链接到这一位置，而不会占用额外的磁盘空间。 这允许你跨项目地共享同一版本的依赖。
因此，在磁盘上节省了大量空间，这与项目和依赖项的数量成正比，并且安装速度要快得多！


## pnpm现状

2021年`pnpm`的下载量约为2020年的3倍，用户量在急剧上升，说明pnpm确确实实有解决npm/yarn用户的问题：
![2021-pnpm](2021-pnpm.png)


## pnpm性能优势
我们来看一看 benchmark 数据：
![benchmark数据](alotta-files.svg)
具体数据对比：

| action | cache | lockfile | node_modules | npm | pnpm | Yarn | Yarn Pnp|
| :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: |
| install |  |  |  | 1m02s | 12.9s | 16.6s | 23.1s |
| install | &#10004; | &#10004; | &#10004; | 1.8s | 1.2s | 2.3s | n/a |
| install | &#10004; | &#10004; |  | 10.1s | 3.6s | 6.5s | 1.5s |
| install | &#10004; |  |  | 15.2s | 6.2s | 11.1s | 5.9s |
| install |  | &#10004; |  | 26.7s | 10.9s | 11.6s | 17.1s |
| install | &#10004; |  | &#10004; | 2.3s | 1.7s | 6.8s | n/a |
| install |  | &#10004; | &#10004; | 1.8s | 1.2s | 7.3s | n/a |
| install |  |  | &#10004; | 2.3s | 5.3s | 11.8s | n/a |
| update | n/a | n/a | n/a | 1.8s | 9.2s | 15.1s | 28.9s |

从以上数据可以看出，`pnpm`在更方面的性能都要优于`npm`和`yarn`。我们常用到的几种没有node_modules（表格3-5行）的场景下
`pnpm`相对`npm`的性能提升都是接近100%的。

### 性能对比

实测`pnpm`耗时`3282ms`
```javascript
const shelljs = require('shelljs')
const start = +new Date();

shelljs.exec('pnpm install vue');
const end = +new Date();

console.log(`耗时：${end-start}ms`);
```


实测`npm`耗时`4711ms`
```javascript
const shelljs = require('shelljs')
const start = +new Date();

shelljs.exec('npm install vue');
const end = +new Date();

console.log(`耗时：${end-start}ms`);
```

## npm 原理
在讲pnpm原理前，先了解一下npm 的原理，帮助我们更好理解pnpm（毕竟pnpm是npm的进化版总有些许的相似）。

### npm是什么
平时我们用到最多的npm，其实真正执行的时候是它的命令行工具npm-cli，那么它到底是个什么东西呢？我们通过`where npm`找到`npm.cmd`所在的位置，源码如下：
![npmcmd](npmcmd.png)

从上面标红的地方可以看出，`npm`命令最终执行的是`node npm-cli.js`。至于`npm-cli.js`里面的逻辑是什么，就需要研究源码了，本文暂不涉及，主要是看功能成眠的原理。

### npm install的整个流程

![npm install流程流程图](npm-install.jpg)
简单总结一下`npm install`的流程：
1. 发出npm install命令
2. npm 向 registry 查询模块压缩包的网址
3. 下载压缩包，存放在~/.npm目录
4. 解压压缩包到当前项目的node_modules目录


## node_modules 安装方式
node_modules 主要的安装方式包括两种：嵌套安装和扁平化安装

### 嵌套安装
在早期的（npm@3之前）npm版本，`node_modules`采用的是嵌套安装，也可以说是树形安装。什么是嵌套安装呢？顾名思义就是
`node_modules`内部嵌套安装`node_modules`。
举个例子：
> 比如需要安装一个包A，而A依赖于B(v1.0)和C，同时C又依赖于B(v2.0)
那么在`npm@3`之前`node_modules`目录是这样子的：
```
node_modules
└─ A
   ├─ index.js
   ├─ package.json
   └─ node_modules
      └─ B(v1.0)
         ├─ index.js
         └─ package.json
      └─ C
         ├─ index.js
         └─ package.json
         └─ node_modules
            └─ B(v2.0)
            ├─ index.js
            └─ package.json
```
上面结构有两个严重的问题：
1. package中经常创建太深的依赖树，这会导致 Windows 上的目录路径过长问题
2. 当一个package在不同的依赖项中需要时，它会被多次复制粘贴并生成多份文件

### 扁平化安装
为了嵌套安装带来的问题问题，`npm`重新设计了`node_modules`结构并提出了扁平化结构。在npm@3+结构变成如下所示：
```
node_modules
└─ A
   ├─ index.js
   ├─ package.json
   └─ node_modules
└─ B(v1.0)
   ├─ index.js
   └─ package.json
└─ C
   ├─ index.js
   └─ package.json
   └─ node_modules
   └─ B(v2.0)
      ├─ index.js
      └─ package.json
```
从上面结构可以看出，`B(v1.0)`和`C`被提升到了顶层。不过需要注意的是，**只有一个版本会被提升到顶层**，比如`B`其实有两个版本`v1.0`和`v2.0`，但只有`v1.0`被提升，`v2.0`还是嵌套安装的。
那么就有同学要问了，为什么不是`Bv2.0`被提升呢？答案是我也不知道，一切皆有可能，取决于在package.json中的位置。好家伙，这解决了嵌套安装的问题，但好像又没有完全解决。

### npm存在的问题
1. 幽灵安装：由于包被提升到顶层，`package.json`没有声明的包，用户也可以直接引用到包。
2. npm分身：一个包多个版本被引用多次，还是会被重复安装
有如下包结构：
```
node_modules
└─ A
   - X(v1.0)
   - Y(v1.0)
└─ B(v1.0)
   - X(v2.0)
   - Y(v2.0)
└─ C
   - X(v2.0)
   - Y(v2.0)
```
在npm3+上的结构是这样的，可以看出X(v2.0)和Y(v2.0)被重复安装了，其实并没有完全解决包嵌套的问题
```
└─ X (v1.0)
└─ Y (v1.0)
└─ A
└─ B(v1.0)
   - X(v2.0)
   - Y(v2.0)
└─ C
   - X(v2.0)
   - Y(v2.0)
```

## pnpm 原理
pnpm的解决方案就是：网状+平铺的node_modules结构。

用pnpm安装一个Vue试试看是什么样的。
![pnpm install vue](pnpm-vue.png)

打开node_modules目录一看，真的是清爽，顶层没有多余的包，只有三个`.pnpm`、`vue`和`.modules.yaml`。

**.pnpm**：虚拟存储目录。该目录下以平铺的形式储存着所有的包，正常的包都可以在这种命名模式的文件夹中被找到。
```
.pnpm/<organization-name>+<package-name>@<version>/node_modules/<name>

// 组织名(若无会省略)+包名@版本号/node_modules/名称(项目名称)
```

### Store
`pnpm` 会在磁盘根目录创建一个名为`.pnpm-store`的文件夹用于存储包。Mac/linux中默认会设置到`{home dir}>/.pnpm-store/v3`；`windows`下会设置到当前盘的根目录下，比如D盘（D/.pnpm-store/v3）。

到这里你应该会猜到通过`pnpm`安装的包可能最终就是`pnpm-store`里面的，不管是以什么样的方式，复制or链接，反正就是存在千丝万缕的联系。那么真实的答案必须也是如此，毕竟`pnpm`号称可以的节省空间的。

### Links
`pnpm`能做到非常大的性能提升，主要原因是利用了计算机内部的**硬链接(hard link)**和**软链接(symbolic link)**。

#### 硬链接(hard link)

硬链接的概念来自于`Unix`操作系统，它是指将一个文件A指针复制到另一个文件B指针中，文件B就是文件A的硬链接

![硬链接](hard-link.png)

通过硬链接，不会产生额外的磁盘占用，并且两个文件都能找到相同的磁盘内容。
硬链接的数量没有限制，可以为同一个文件产生多个硬链接。

#### 软连接(symbolic link)

符号链接又称为软链接，如果为某个文件或文件夹A创建符号连接B，则B指向A。

![软链接](symbol-link.png)


#### 硬链接和软链接的区别
原理上，硬链接和源文件的inode节点号相同，两者互为硬链接。软连接和源文件的inode节点号不同，进而指向的block也不同，软连接block中存放了源文件的路径名。
实际上，硬链接和源文件是同一份文件，而软连接是独立的文件，类似于快捷方式，存储着源文件的位置信息便于指向。
使用限制上，**不能对目录创建硬链接**，不能对不同文件系统创建硬链接，不能对不存在的文件创建硬链接；可以对目录创建软连接，可以跨文件系统创建软连接，可以对不存在的文件创建软连接。


### 原理

通过一个简单例子还讲解`pnpm`的原理。例如我们有一个`pnpm-project`的项目，依赖于包a，而a依赖于b。

假设我们的项目直接依赖a，安装时pnpm会做下面的处理：
1. 查询依赖关系得到包：a和b
2. 检查a和b是否有缓存
3. 无缓存则对node_modules做如下初始化：
![无缓存安装](no-cache.jpg)

4. 如果有缓存，则从缓存的对应包中使用硬链接放在相应包代码目录下
![使用硬链接](use-hard-link.jpg)   

5. 使用软链接，将每个包的直接依赖放置到自己的目录中。这样做的目的是为了a可以直接读取它相关的依赖。
![使用软链接](use-symbol-link.jpg)

6. node_modules目录中使用软链接，放置直接依赖
![完成](finnal.jpg)

## 如何迁移

`pnpm`使用起来十分简单，之前使用`npm`的可以无缝迁移至`pnpm`。比如`npm install`改为`pnpm install`即可，
其余的命令以此类推，附上官方文档：[传送门](https://pnpm.io/zh/motivation)。

迁移过程也比较简单：
1. 删除`package-lock.json`和`node_modules`
2. 执行`pnpm install`

## 总结
综合来看，`pnpm`是一个比`npm`更优秀的方案，利用软硬链接减少了磁盘空间和下载速度。期待未来`pnpm`能有更多的落地。

<div class="aplayer no-destroy" data-id="热门" data-server="tencent" data-type="search" data-fixed="true" data-mini="true" data-listFolded="false" data-order="random" data-preload="none" data-autoplay="true" muted></div>
