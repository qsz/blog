# Mutation Observer API 学习

[`MutationObserver`](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver)接口提供了监视对DOM树所做更改的能力

具有一下几个特点

* 异步触发，会等当前所有DOM操作执行完后触发
* 把 DOM 变动记录封装成一个数组进行处理

##  实例化
```js
var observer = new MutationObserver(function (mutations, observer) {
  mutations.forEach(function(mutation) {
    console.log(mutation);
  });
});
```

回调函数接受两个参数，第一个是变动数组`mutations`，第二个是观察器实例

每个mutation有以下几个属性

* addedNodes :               新增的DOM节点
* removedNodes:           删除的DOM节点
* attributeName:            发生变动的属性。如果设置了`attributeFilter`，则只返回预先指定的属性
* attributeNamespace:  已更改属性的名称空间
* nextSibling:                  下一个同级节点, default: null
* previousSibling:           前一个同级节点, default: null
* oldValue:                      变动前的值。这个属性只对attribute和characterData变动有效，如果发生childList变动，则返回null
* target:                           发生变动的DOM节点
* type:                              观察的变动类型（attribute、characterData或者childList)



## 实例方法

### 1.observe() 启动监听

```js
observer.observe(dom, options);
dom: 监听的dom节点
options：配置参数
```

options有以下值：

* childList:  true or false                                    监听子节点的变动（指新增，删除或者更改）
* characterData： true or false                       监听节点内容或节点文本的变动
* subtree：  true or false                                  是否将该观察器应用于该节点的所有后代节点
* attributeOldValue： true or false                 观察attributes变动时，是否需要记录变动前的属性值
* characterDataOldValue： true or false       观察characterData变动时，是否需要记录变动前的值
* attributeFilter： Array                                    需要观察的特定属性

### 2.disconnect()，takeRecords（）

```js
observer.disconnect()    // 停止观察
observer.takeRecords()   // 清除变动记录
```



## 举例说明

MutationObserver监听三种类型的DOM变动，分别为childList(子节点的变动) , attribute(属性的变动), characterData(节点文本的变动)。childList和attribute很容易理解，那么怎样才会触发characterData呢?

```html
html:
<div id='text'>这个是文本节点</div>
<button onclick="setText()">setText</button>

js:
// 触发 characterData
function setText() {
   const textDiv = document.getElementById('text')
   var text = textDiv.childNodes.item(0);    //获取第几个子节点文本值
   text.nodeValue = '改变后的文本'
   text.appendData("加到后面的文本"); 
}

// 监听
var observer = new MutationObserver(function (mutations, observer) {
  mutations.forEach(function(mutation) {
    console.log(mutation);
  });
});

observer.observe(document.documentElement, {
    childList: true,                 
    attributes: true,                
    characterData: true,             
    subtree: true,                   
    attributeOldValue: true,          
    characterDataOldValue: true           
});
```

