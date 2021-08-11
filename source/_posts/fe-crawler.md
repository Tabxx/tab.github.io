---
title: 前端与反爬虫
date: 2019-10-30 18:49:46
cover: 'topbg.jpg'
tags: 安全
---


# 前端与反爬虫

> 爬虫爬取数据主要是从 dom 节点上获取的，那么我们前端可以在 dom 节点上搞点骚操作来干扰爬虫抓取数据

### 

**1.1 字符串穿插**
在 html 字符间插入无关紧要的元素来达到干扰爬虫的效果

```javascript
/**
 * 获取随机中文字
 */
const randomText = () => {
    let _len = 2
    let i = 0
    let _str = ''
    let _base = 20000
    let _range = 1000
    while (i < _len) {
        i++
        var _lower = parseInt(Math.random() * _range)
        _str += String.fromCharCode(_base + _lower)
    }
    return _str
}

const randomStyle = () => {
    let css = [
        'margin:-100px;',
        'top:100%;',
        'border: 1px solid red;',
        'flex-grow: 1;',
        'width: 1000px',
        'color: #fff',
        'overflow: hidden;'
    ]
    let style = 'display:none;'
    let num = Math.floor(Math.random() * 7)
    for (let i = 0; i <= num; i++) {
        style += css[i]
    }
    return style
}
export default {
    bind(el, {
        value
    }) {
        // 字符串穿插
        let fontNum = '';
        let numberArray = value.toString().split('')
        for (let item of numberArray) {
            fontNum += `${item}<span style="${randomStyle()}">${randomText()}</span>`
        }
        el.innerHTML = fontNum;
    }
}
```



**1.2 元素定位覆盖**
根据数据位数生成`位数*2`个标签，然后通过定位将正确数据覆盖在错误数据

```javascript
export default {
    bind(el, {
        value
    }) {
        let { data, w } = value;
        // 定位法
        if (!el.style.position) {
          el.style.position = 'relative';
        }

        // 每一位分割成字符串数组
        let valueArray = data.toString().split('');
        // 干扰个数
        let fakeNum = Math.floor(Math.random() * valueArray.length);
        fakeNum = fakeNum ? fakeNum : 1;
        let newHTML = [];
        let normalStyle = ['font-style: normal', 'background: #252527', 'position:absolute'];
        newHTML = valueArray.map((item, index) => {
          let s = [...normalStyle, ...[`left: ${index * w}px`, `width:${w}px`]].join(';');
          return `<i style="${s}">${item}</i>`;
        });
        for (let i = 0; i <= fakeNum; i++) {
          let n = Math.floor(Math.random() * 9);
          let s = [...normalStyle, ...[`left: ${i * w}px`, `width:${w}px`]].join(';');
          newHTML.unshift(`<i style="${s}">${n}</i>`);
        }
        el.style.width = valueArray.length * w + 'px';
        el.style.height = '12px';
        el.innerHTML = newHTML.join('');
    }
}
```



**1.1 [font-face 拼凑](<https://juejin.im/post/5b6d579cf265da0f6e51a7e0>)**

font-face 定义一套自己的字符集，页面引用对应的 unicode

```css
@font-face {
  font-family: 'iconfont';
  src: url('./font/iconfont.eot');
  src: url('./font/iconfont.eot?#iefix') format('embedded-opentype'), url('./font/iconfont.woff2')
      format('woff2'), url('./font/iconfont.woff') format('woff'), url('./font/iconfont.ttf')
      format('truetype'), url('./font/iconfont.svg#iconfont') format('svg');
}
```

```html
<span>&#xe603;&#xe607;&#xe607;</span>
```

自定义指令 v-fontface

```javascript
const fontface = [
  '&#xe600;',
  '&#xe609;',
  '&#xe607;',
  '&#xe610;',
  '&#xe606;',
  '&#xe605;',
  '&#xe604;',
  '&#xe603;',
  '&#xe602;',
  '&#xe601;'
]
export default {
  bind(el, { value }) {
    let fontNum = ''
    let numberArray = value.toString().split('')
    for (let item of numberArray) {
      fontNum += fontface[item] ? fontface[item] : item
    }
    el.innerHTML = fontNum
  }
}
```

