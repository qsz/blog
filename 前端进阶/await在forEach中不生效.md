# 踩坑系列：await 在 forEach 中不生效

最近在做项目的时候遇到一个问题，找了很久总算发现问题所在。在这里记录一下。看下面这个例子。
```js
const awaitFuc = function () {
	return new Promise( resolve => {
			resolve()
	})
}   
const arr = [1,2,3,4,5]
function mapArr(arr) {
    arr.forEach(async (item) => {
	    await awaitFuc()
	    console.log('in forEach')
    })
    console.log('await end')
}
mapArr(arr)
```
一开始，我以为，既然`forEach`中有`async`函数，那么每次当`awaitFuc()`执行完后，才会继续遍历，所以结果是
> in forEach   
> in forEach		
> in forEach		
> in forEach		
> in forEach		
> await end		

然而，运行结果却出乎意料，真实结果为

> await end		
> in forEach		
> in forEach		
> in forEach		
> in forEach		
> in forEach		

发现`awiat`并没有生效。这是为什么呢？

### 模拟forEach实现

为了解释这个现象，我们先来模拟实现`forEach`。`forEach`的定义是

> 用于调用数组的每个元素，并将元素传递给回调函数。

这里我们用`for ... of`来模拟实现一个`myForEach`

```js
Array.prototype.myForEach = function (callback) {
    for(const item of this) {
        callback(item)
    } 
}
```

代替`forEach`实现上述代码

```js
const arr = [1,2,3,4,5]
function mapArr(arr) {
    arr.myForEach(async (item) => {
	    await awaitFuc()
	    console.log('in forEach')
    })
    console.log('await end')
}
mapArr(arr)
```

输出结果与`forEach`一致

> await end		
> in forEach		
> in forEach		
> in forEach		
> in forEach		
> in forEach		

由此可见：

> forEach内部并没有对回调函数做任何异步处理，仍然是同步执行回调函数

### 解决方法

既然`forEach`可以用`for ... of`模拟实现，那么我们就可以用`for ... of`来代替`forEach`。看代码

```js
const arr = [1,2,3,4,5]
async function mapArr2(arr) {
    for(const item of arr) {
        await awaitFuc()
	    console.log('in forEach')
    }
    console.log('await end')
}
```

执行代码，完美输出

> in forEach		
> in forEach		
> in forEach		
> in forEach		
> in forEach		
> await end		

### 引申1 - 为什么 `for ... of`能实现需求

这里涉及到`Iterator（遍历器）`的知识。`for...of`循环内部调用的是数据结构的`Symbol.iterator`方法,该方法返回该对象的默认遍历器。`Iterator `的遍历过程是这样的。

> （1）创建一个指针对象，指向当前数据结构的起始位置。也就是说，遍历器对象本质上，就是一个指针对象。
>
> （2）第一次调用指针对象的`next`方法，可以将指针指向数据结构的第一个成员。
>
> （3）第二次调用指针对象的`next`方法，指针就指向数据结构的第二个成员。
>
> （4）不断调用指针对象的`next`方法，直到它指向数据结构的结束位置
>
> `next`方法返回一个对象，表示当前数据成员的信息。这个对象具有`value`和`done`两个属性，`value`属性返回当前位置的成员，`done`属性是一个布尔值，表示遍历是否结束，即是否还有必要再一次调用`next`方法

我们用代码来模拟` for ... of`的内部机制。

```js
// 定义一个Symbol.iterator方法
function SymbolIterator(arr) {   
    let index = 0;
    const length = arr.length;
    return {
        next: function () {      // next 方法返回一个{value, done}结果的对象
            if(index <  length) {
                return {
                    value: arr[index++],
                    done: false
                }
            }  else {
                return {
                    value: undefined,
                    done: true
                }
            }
        }
    }
}

// 模拟 for ... of
async function mapArr4(arr) {
    const symbolIterator = SymbolIterator(arr);
    symbolIterator.next()
    await awaitFuc()
    console.log('in forEach')

    symbolIterator.next()
    await awaitFuc()
    console.log('in forEach')

    symbolIterator.next()
    await awaitFuc()
    console.log('in forEach')

    console.log('await end')
    
}
mapArr4([1,2,3])
```

输出结果为

> in forEach		
> in forEach		
> in forEach		
> await end		

可以看到`for ... of`内部对`async`函数是链式调用的

#### 参考

[Iterator 和 for...of 循环](http://es6.ruanyifeng.com/?search=forEach&x=0&y=0#docs/iterator#for---of-循环)



### 引申2 - 几种遍历的对比

既然聊到了`forEach`和`for ... of`, 那么我们就做个总结，对比一下常用的几种遍历方式

* `for...in`：

  * 为遍历对象而设计的，遍历的实际上是对象的属性名称。

  * 一个Array数组也是一个对象，数组中的每个元素的索引被视为属性名称，所以我们可以看到使用for...in循环Array数组时，拿到的其实是每个元素的索引
  * 当我们为某个数组手动添加一个属性name的时候，for...in循环会把name属性也包括在内
    
  * 可以 `continue`，` break`

* `for ... of`:
  
  * 如上所示，遍历的是当前位置的元素值
  
  * 可以 `continue`，` break`
  
* `for var`：可以 `continue`，` break`  
* `forEach`: 
  * 不可以 `continue`，` break`。可以`return`，跳过本轮遍历
  * 没有返回值
  
* `map`:

  * 不可以 `continue`，` break`。
  * 有返回值，返回一个新的数组。数组中元素值为`return`的值

* `every`： `return false`时跳出整个循环  
* `some`：`return true`时跳出循环
* `fliter`:   根据`return`返回值是`true`还是`false`决定保留还是丢弃当前元素

