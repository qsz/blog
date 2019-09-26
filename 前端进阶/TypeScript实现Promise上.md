# 从TypeScript实现Promise到Event Loop（上）

`Promise `是异步编程的一种解决方案，比传统的解决方案——回调函数和事件——更合理和更强大。`Promise`的实现基于`Promises/A+`规范

**本系列将学习**

* [基于typeacript实现Promise](前端进阶/TypeScript实现Promise上.md)
* [引伸到Event Loop](前端进阶/TypeScript实现Promise中.md)
* Event Loop与浏览器渲染机制


## Promises/A+介绍

`Promises/A+`规范参考：

* [英文地址](https://promisesaplus.com/)

* [中文翻译](http://malcolmyu.github.io/malnote/2015/06/12/Promises-A-Plus/)

接下来，对规范中重要的几个地方做下介绍

以下实现以 **`TP/A+`**区分

* **术语**

  * **Promise**：`promise `是一个拥有 `then` 方法的对象或函数，其行为符合本规范

  * **thenable**：是一个定义了 `then` 方法的对象或函数

    由此可知`Promise`是`thenable`的一种

* **1.  Promise的状态**

  三种状态`pending(等待态)`,  `fulfilled(执行态)`，` rejected(拒绝态)`

  状态变为`fulfilled`时，会有`value`值

  状态变为`rejected`时，会有`reason`值

* **2.  then**方法

  * 2.1  有两个可选参数`onFulfilled` 和 `onRejected` 

  * 2.2  如果`onFulfilled` 和 `onRejected` 不是方法，需要被忽略。且将当前`Promise`的值穿透到下一个`Promise`

    何为穿透？用代码说明

    ```js
    const pp = new Promise(function(resolve, reject){
      resolve(1)
    })
    // 当`onFulfilled`是一个方法的时候
    const pp1 = pp.then(function(res) {
      // TODO
    })
    const pp2 = pp1.then(function(res) {
      console.log('pp2', res)
    })
    // 当`onFulfilled`不是一个方法的时候
    const pp3 = pp.then('')
    const pp4 = pp3.then(function(res) {
      console.log('pp4', res)
    })
    ```

    上述代码运行结果

    > pp2 undefined
    >
  > pp4 1
  
    由此可见，当pp3中`onFulfilled`不是一个方法时，pp中的value穿透到了pp4中
  
  * 2.3  `then`必须返回一个`promise`对象

    ```js
    promise2 = promise1.then(onFulfilled, onRejected)
    ```

    当`onFulfilled` or`onRejected`抛出异常, `promise2`状态变为`rejected`，并将`Error`作为`reason`值
    
  * 2.4  当`onFulfilled` or`onRejected`返回一个值`x`，则执行` Promise Resolution Procedure: [[Resolve]](promise2, x)`，即类似执行以下伪代码：

    ```js
    promise2 = new Promise(function(resolve){
       resolve(x)
    })
    ```

* **3.  `Promise Resolution Procedure`(Promise解决过程)**

  **Promise 解决过程** 是一个抽象的操作，其需输入一个 `promise2` 和一个值`x`（`onFulfilled` or`onRejected`返回值），我们表示为 `[[Resolve]](promise2, x)`

  `x`需要满足一下条件

  * 3.1  若`x`和`promise2`相等，抛出`抛出TypeError`异常
  
  * 3.2  若`x`是一个`Promise`对象，则`promise2`沿用`x`的状态和值
  
  * 3.3  若 `x`是对象或函数
    
    * 3.3.1  `x.then`替代`promise2.then`
    
    * 3.3.2  `x.then`抛出错误`Error`, 则`promise2`状态变为`rejected`, `reason`值变为`Error`
    
    * 3.3.3  若`x.then`是一个函数,  执行`x.then(resolvePromise, rejectPromise)`
    * 3.3.3.1  如果 `resolvePromise` 以值 `y` 为参数被调用, 则继续运行 `[[Resolve]](promise, y)`
      
    * 3.3.3.2  如果`rejectPromise`以值据因 `r` 为参数被调用, 则`promise2`状态变为`reject`,`reason`值为`r`
      
    * 3.3.3.3  如果 `resolvePromise` 和 `rejectPromise` 均被调用，或者被同一参数调用了多次，则优先采用首次调用并忽略剩下的调用
    
    * 3.3.4   若`x.then`不是一个函数，`promise2`状态变为`fulfilled`, `value`值为`x`
    
      举例说明
    
      ```js
      const pp = new Promise(function(resolve, reject){
          reject(1)
      })
      // `onRejected`返回 `x` 为 {then: ''}, `x.then`不是一个函数
      const pp1 = pp.then('', function(res) {
          return {
              then: ''
          }
      })
      const pp2 = pp1.then(function(res) {
          console.log('res:', res)
      })
      ```
    
      上述代码运行结果为
    
      > res: {then: ""}
    
   * 3.4  若 `x`不是一个对象或方法, `promise2`状态变为`fulfilled`, `value`值为`x`
  
* **4.注释**

     实践中要确保 `onFulfilled` 和 `onRejected` 方法异步执行，且应该在 `then` 方法被调用的那一轮事件循环之后的新执行栈中执行。这个事件队列可以采用“宏任务（macro-task”机制或者“微任务（micro-task”机制来实现
     

接下来，我们就根据`Promises/A+`规范来实现一个`TypeScript`版本的`Promise`



## 实现TsPromise

一步一步来

#### 首先实现一个简易的Promise

根据**`TP/A+ 1`**定义`PromiseState`的类型

```js
// 定义state的类型
enum PromiseState {
    PENDING = 'pending',      // 等待态
    FULFILLED = 'fulfilled',  // 执行态
    REJECTED = 'rejected',    // 拒绝态
}

// 定义Resolve和Reject函数类型
type Resolve = (value: any) => void;
type Reject = (reason: any) => void;
```

定义 `TsPromise`构造函数

```javascript
class TsPromise {
    private state: PromiseState = PromiseState.PENDING; // 初始状态pending
    private value: any = null;                          // fulfilled时候的返回值
    private reason: any = null;                         // rejected时候的返回值
  
  	constructor(fn: (resolve: Resolve, reject: Reject) => any) {
        try {         // 为了捕获报错
            fn(this.resolve, this.reject);
        } catch (e) { // 如果捕获发生异常，直接调失败，并把参数传进去
            this.reject(e);
        }
    }
    private readonly resolve: Resolve = (value: any): void => {
        if (this.state === PromiseState.PENDING) {
            this.state = PromiseState.FULFILLED;
            this.value = value;
        }
    };
    private readonly reject: Reject = (reason: any): void => {
        if (this.state === PromiseState.PENDING) {
            this.state = PromiseState.REJECTED;
            this.reason = reason;
        }
    };
   /**
   * then 方法
   * @param TsPromise 
   * @param onFulfilled 
   * @param onRejected 
   */
    public then(this: TsPromise, onFulfilled: any, onRejected: any) {
        if (this.state === PromiseState.FULFILLED) {
            onFulfilled(this.value)
        }
        if (this.state === PromiseState.REJECTED) {
            onRejected(this.reason)
        }
    }
}
```

尝试一下这个`TsPromise`

```javascript
const tsPromise = new TsPromise(function(resolve, reject) {
  resolve(1)
})
tsPromise.then(function(res: any){
  console.log(res)
})
// 打印结果 > resolve: 1
```

上述代码成功执行了`then`中的回调函数，并打印出`resolve`中的返回值 1

但上述代码存在问题，接下尝试改一下调用方式，改为异步执行

```javascript
const tsPromise = new TsPromise(function(resolve, reject) {
  setTimeout(function(){
    resolve(1)
  },0)
})
tsPromise.then(function(res: any){
  console.log('resolve: ', res)
})
// 打印结果 > 
```

可以看到，当把`resolve(1)`异步执行时，`then`回调中并不能准确拿到`Promise`的状态

这是由于`resolve(1)`异步执行, `then`回调会先于`resolve`,此时，`then`中的状态依然是`pending`, 所以不会执行`onFulfilled`函数

因此修改我们的`TsPromise`，以支持异步操作，

增加任务队列`deferreds: Array<Handler> = []`

```javascript
class TsPromise {
  
  ...
  private deferreds: Array<Handler> = [];   // 增加一个任务队列
  ...
  private readonly resolve: Resolve = (value: any): void => {
    if (this.state === PromiseState.PENDING) {
      this.state = PromiseState.FULFILLED;
      this.value = value;
      this.finale();
    }
  };
  private readonly reject: Reject = (reason: any): void => {
    if (this.state === PromiseState.PENDING) {
      this.state = PromiseState.REJECTED;
      this.reason = reason;
      this.finale();
    }
  };
  /**
   * 执行任务队列
   * @param this TsPromise
   */
  private readonly finale = (): void => {
    this.deferreds.forEach((deferred) => {
      if (this.state === PromiseState.FULFILLED) {
         deferred.onFulfilled(this.value)
      }
      if (this.state === PromiseState.REJECTED) {
         deferred.onRejected(this.reason)
      }
    });
    this.deferreds.length = 0;
  };
  public then(this: TsPromise, onFulfilled: any, onRejected?: any) {
    // 当前的promise还是pending状态，则把deferred放到当前promise的任务队列里
    if (this.state === PromiseState.PENDING) { 
       const deferred = new Handler(onFulfilled, onRejected)
       this.deferreds.push(deferred);
       return;
    }
    if (this.state === PromiseState.FULFILLED) {
       onFulfilled(this.value)
    }
    if (this.state === PromiseState.REJECTED) {
       onRejected(this.reason)
    }
  }
} 

// Handler表示单个异步任务
class Handler {
    onFulfilled: any;
    onRejected: any;
    constructor(onFulfilled: any, onRejected: any) {
        this.onFulfilled = onFulfilled
        this.onRejected = onRejected
    }
}  
```

尝试异步调用

```javascript
const tsPromise = new TsPromise(function(resolve, reject) {
  setTimeout(function(){
    resolve(1)
  },500)
})
tsPromise.then(function(res: any){
  console.log('异步resolve: ', res)
})
// 打印结果 > 异步resolve: 1
```

ok，成功执行了异步任务。简易的`Promise`已经完成。但我们的代码还不够健壮，真正的`Promise`还需要对返回值的各种情况做判断等等

#### 按照规范完善TsPromise

接下来我们结合规范将简易`TsPromise`逐步完善

* 完善`then`方法，以支持链式调用

  > 根据 `TP/A+ 2.3: ` `then`必须返回一个`promise`对象 

  ```js
  
  public then(this: TsPromise, onFulfilled: any, onRejected?: any): TsPromise {
    const promise2 = new TsPromise(function(){});
    // 将onFulfilled，onRejected缓存到队列中
    this.handle(new Handler(onFulfilled, onRejected, promise2));
    // 返回promise对象
    return promise2; 
  }
  ```

* 完善`Handler`类

  > 根据`TP/A+ 2.3:` 如果`onFulfilled` 和 `onRejected` 不是方法，需要被忽略。且将当前`Promise`的值穿透到下一个`Promise`

  ```javascript
  // 单个异步任务
  class Handler {
      onFulfilled: any;
      onRejected: any;
      promise: TsPromise;
      constructor(onFulfilled: any, onRejected: any, promise: TsPromise) {
          // 如果onFulfilled不是一个方法，则将value返回给下一个promise
          this.onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : function (value: any) { 
              return value;
          };
          // 如果onRejected不是一个方法，则抛出异常，并在下一个promise中的reject中捕获
          this.onRejected = typeof onRejected === 'function' ? onRejected : function (err: any) {       
              throw err;
          };
          this.promise = promise;
      }
  }
  ```

* 实现`handle`方法
  ```js
  private readonly handle = (deferred: Handler) => {
          // 当前的promise还是pending状态，则把deferred放到当前promise的任务队列里
          if (this.state === PromiseState.PENDING) { 
              this.deferreds.push(deferred);
              return;
          }
          // 当前的promise状态改变了,异步执行
          setTimeout(() => {
              const cb: (value: any) => any = this.state === 'fulfilled' ? deferred.onFulfilled : deferred.onRejected;
              const value = this.state === 'fulfilled' ? this.value : this.reason;
              const x = this.tryCallValue(deferred, cb, value);
              // 执行下一个promise2.then的回调函数
              this.handleResolved(x, deferred);
          }, 0);
  };
  // 根据`TP/A+ 2.4`
  // 获取 onFulfilled or onRejected执行后的值，下一个promise2.then里的回调函数 参数需要用x。 
  private readonly tryCallValue = (deferred: Handler, fn: (value: any) => any, value: any) => {
          try {
              return fn(value);
          } catch (err) {  // 如果捕获发生异常，直接调失败，并把参数传进去
              deferred.promise.reject(err);
              return this.IS_ERROR;
          }
  };
  ```
  
* 最关键的部分  `Promise Resolution Procedure`**(Promise解决过程)**，我们在`handleResolved`函数中实现。`handleResolved`接受两个值：上一步`onFulfilled or onRejected`执行后的返回值 `x`, 以及下一个异步任务`defered`。
  首先改写下前面的`resolve`。我们不知`onFulfilled or onRejected`执行后的返回值 `x`做判断，初始化时`Promise`中的传参数也需要判断

  ```javascript
  private readonly resolve: Resolve<T> = (value): void => {
          const defered = new Handler('', '', this);
          // Promise Resolution Procedure
          this.handleResolved(value, defered);
  };
  
  // 提供一个真正的resolve方法--doResolve，用于修改Promise的状态
  private readonly doResolve: Resolve<T> = (value): void => {
          if (this.state === PromiseState.PENDING) {
              this.state = PromiseState.FULFILLED;
              this.value = value;
              this.finale();
          }
  };
  ```

  实现`handleResolved`方法

  ```javascript
   private readonly handleResolved = (x: any, defered: Handler): void => {
        if (x !== this.IS_ERROR) {       // 如果不报错
            // 如果 onFulfilled 或者 onRejected 返回一个值 x ，则运行下面的 Promise 解决过程
            // 根据 `TP/A+ 3.1` 若`x`和`promise2`相等，抛出`抛出TypeError`异常
            if (x === defered.promise) { 
                return defered.promise.reject(new TypeError('返回值不能为本身'));
            }
            // 根据 `TP/A+ 3.3.3.3` resolvePromise 和 rejectPromise采用首次调用
            // called用于表示调用过后便不再调用
            let called: Boolean = false;
            if (x && (typeof x === 'object' || typeof x === 'function')) { 
                // `TP/A+ 3.3` 如果 x 是一个对象或者方法
                try {
                    // `TP/A+ 3.3.1`  x.then替代promise2.then
                    const then = x.then;   
                    if (typeof then === 'function') {  
                        // `TP/A+ 3.3.3` 执行x.then(resolvePromise, rejectPromise)
                        const resolvePromise = (y: any) => {
                            if (called) return; // 如果调用过就return掉
                            called = true;
                            // `TP/A+ 3.3.3.1`  如果 resolvePromise 以值 y 为参数被调用, 则继续运行 [[Resolve]](promise, y)
                            this.handleResolved(y, defered); // 递归调用
                        };
                        const rejectPromise = (err: any) => {
                            if (called) return; // 如果调用过就return掉
                            called = true;
                          // `TP/A+ 3.3.3.2`  如果rejectPromise以值据因 r 为参数被调用, 则promise2状态变为reject,reason值为r
                            defered.promise.reject(err);
                        };
                        then.call(x, resolvePromise, rejectPromise);
                    } else {
                        // `TP/A+ 3.3.4`若不是一个函数，promise2状态变为fulfilled, value值为x
                        defered.promise.doResolve(x);
                    }
                } catch (e) {
                    // `TP/A+ 3.3.2` x.then抛出错误Error, promise2状态变为rejected, reason值变为Error
                    if (called) return; // 如果调用过就return掉
                    called = true;
                    return defered.promise.reject(e); 
                }
            } else {
              // `TP/A+ 3.4`若 x不是一个对象或方法, promise2状态变为fulfilled, value值为x
              defered.promise.doResolve(x);
            }
        }
  };
  ```


到这里为止，我们已经实现了符合`Promises/A+`规范的基本的`Promise`。完整的`Promise`还拥有以下API：实例方法`catch`、`finally`，静态方法`resolve`、`reject`、`all`、`race`。接下来我们一一实现

#### 实现完整的Promise

* `catch`:  用于捕获错误。即返回一个**`thenable`**类型的对象，用于处理`rejected`状态
  
  ```javascript
   public catch(this: TsPromise, onRejected: any): TsPromise {
        return this.then(null, onRejected);
   }
	```

* `finally`: 指定不管 `Promise` 对象最后状态如何，都会执行的操作。因此在`onFulfilled`和`onRejected`都执行回调函数
	```js
    public finally(f: any): TsPromise {
       return this.then(function(value: any) {
                f();
                return value;
            }, function(reason: any) {
                f();
                throw reason;
            });
    }
  ```

* `resolve`:  即返回一个`fulfilled`状态的`Promise`实例， 可将现有对象转为 Promise 对象

  ```javascript
  static resolve = (value: any): TsPromise => {
     return new TsPromise(resolve => resolve(value));
  };
  ```

  > static 在TypeScript中用于表示静态属性

* `reject`:  返回一个新的 `Promise` 实例，该实例的状态为`rejected`

  ```js
  static reject = (reason: any): TsPromise => {
    return new TsPromise((resolve, reject) => reject(reason));
  };
  ```

* `race`

  首先看一下`race`的使用

  > ```javascript
  > const p = Promise.race([p1, p2, p3]);
  > //上面代码中，只要p1、p2、p3之中有一个实例率先改变状态，p的状态就跟着改变。那个率先改变的 Promise 实例的返回值，就传递给p的回调函数
  > ```
  
  这里有几点说明一下
  
  * `race`方法参数中如果不是 Promise 实例，就会先调用`resolve`方法，将参数转为 Promise 实例
  * 参数中只要有一个实例状态改变，Promise的状态就会改变( 无论是成功态还是失败态 )

  实现代码
  ```javascript
  static race = (promises: Array<any>): TsPromise => {
          return new TsPromise(function (resolve, reject) {
              for (const promise of promises) {
                  //  转化为Promise 实例
                  const tsPromise = TsPromise.resolve(promise);
                  tsPromise.then(resolve, reject);
              }
          });
  };
  ```

* `all`

  `all`方法的使用如下：

  > ```javascript
  > const p = Promise.all([p1, p2, p3]);
  > // p的状态由p1、p2、p3决定，分成两种情况
  > // (1) 只有p1、p2、p3的状态都变成fulfilled，p的状态才会变成fulfilled，此时p1、p2、p3的返回值组成一个数组，传递给p的回调函数
  > // (2) 只要p1、p2、p3之中有一个被rejected，p的状态就变成rejected，此时第一个被reject的实例的返回值，会传递给p的回调函数
  > ```

  实现该方法
  ```javascript
  static all = (promises: Array<any>): TsPromise => {
          return new TsPromise(function (resolve, reject) {
              const length: number = promises.length;
              let i: number = 0;
              let results: Array<any> = [];
              function loop() {
                  // 当promises都变为fulfilled, 执行resolve， 并将所有的结果放入数组中
                  if (++i === length) {   
                      resolve(results);
                  }
              }
              for (const promise of promises) {
                  TsPromise.resolve(promise).then(function(res: any) {
                      results.push(res);
                      loop();
                  }, function(err: any) { 
                      // 当有一个promises都变为rejected, reject
                      reject(err);
                  });
              }
          });
  };
  ```

#### TsPromise完整代码
到此为止，我们的**TsPromise**已经实现，完整代码如下

```javascript
// 定义state的类型
enum PromiseState {
    PENDING = 'pending',
    FULFILLED = 'fulfilled',
    REJECTED = 'rejected',
}

// 定义Resolve和Reject接口
type Resolve<T> = (value?: T) => void;
type Reject = (reason?: any) => void;

// 定义回调函数
type Callback<T> = (resolve?: Resolve<T>, reject?: Reject) => void;

// 定义thenable
interface Thenable<T> {
    then<TResult>(onFulfilled?: T | Resolve<T> | undefined | null, onRejected?: Reject | undefined | null): Thenable<TResult>;
}


// 实现thenable
class TsPromise<T = any> implements Thenable<T> {
    static resolve = (value?: any): TsPromise => {
        return new TsPromise(resolve => resolve(value));
    };
    static reject = (reason?: any): TsPromise => {
        return new TsPromise((resolve, reject) => reject(reason));
    };
    static race = (promises: Array<any>): TsPromise => {
        return new TsPromise(function (resolve, reject) {
            for (const promise of promises) {
                const tsPromise = TsPromise.resolve(promise);
                tsPromise.then(resolve, reject);
            }
        });
    };
    static all = (promises: Array<any>): TsPromise => {
        return new TsPromise(function (resolve, reject) {
            const length: number = promises.length;
            let i: number = 0;
            let results: Array<any> = [];
            function loop() {
                if (++i === length) {   
                    // 当promises都变为fulfilled, 执行resolve， 并将所有的结果放入数组中
                    resolve(results);
                }
            }
            for (const promise of promises) {
                TsPromise.resolve(promise).then(function(res: any) {
                    results.push(res);
                    loop();
                }, function(err: any) {  
                    // 当有一个promises都变为rejected, reject
                    reject(err);
                });
            }
        });
    };

    private state: PromiseState = PromiseState.PENDING; // promise的状态, 初始值PENDING
    private value: any = null;                // fulfilled时候的返回值
    private reason: any = null;               // rejected时候的返回值
    private deferreds: Array<Handler> = [];   // 任务队列
    private readonly IS_ERROR: {} = {};                // 报错时的返回值

    constructor(fn: Callback<T>) {
        try {         // 为了捕获报错
            fn(this.resolve, this.reject);
        } catch (e) { // 如果捕获发生异常，直接调失败，并把参数穿进去
            this.reject(e);
        }
    }

    public then(this: TsPromise, onFulfilled?: any, onRejected?: any): TsPromise {
        // 定义一个空的回调函数
        function noop(): void {}
        const promise2 = new TsPromise(noop);
        this.handle(new Handler(onFulfilled, onRejected, promise2));
        return promise2;
    }

    public catch(this: TsPromise, onRejected: any): TsPromise {
        return this.then(null, onRejected);
    }

    public finally(f: any): TsPromise {
        return this.then(function(value: any) {
            f();
            return value;
        }, function(reason: any) {
            f();
            throw reason;
        });
    }

    private readonly resolve: Resolve<T> = (value): void => {
        const defered = new Handler('', '', this);
        this.handleResolved(value, defered);
    };
    private readonly doResolve: Resolve<T> = (value): void => {
        if (this.state === PromiseState.PENDING) {
            this.state = PromiseState.FULFILLED;
            this.value = value;
            this.finale();
        }
    };
    private readonly reject: Reject = (reason): void => {
        if (this.state === PromiseState.PENDING) {
            this.state = PromiseState.REJECTED;
            this.reason = reason;
            this.finale();
        }
    };
    /**
     * 执行任务队列
     * @param this TsPromise
     */
    private readonly finale = (): void => {
        this.deferreds.forEach((deferred) => {
            this.handle(deferred);
        });
        this.deferreds.length = 0;
    };
    /**
     * 处理 onFulfilled 与 onRejected
     * @param this TsPromise
     * @param deferred Handler
     */
    private readonly handle = (deferred: Handler) => {
        if (this.state === PromiseState.PENDING) { 
            // 当前的promise还是pending状态，则把deferred放到当前promise的任务队列里
            this.deferreds.push(deferred);
            return;
        }
        // 当前的promise状态改变了
        setTimeout(() => {
            const cb: (value: any) => any = this.state === 'fulfilled' ? deferred.onFulfilled : deferred.onRejected;
            const value = this.state === 'fulfilled' ? this.value : this.reason;
            // 获取 onFulfilled or onRejected执行后的值,下一个promise2.then里的回调函数 参数需要用x
            const x = this.tryCallValue(deferred, cb, value);
            // 执行下一个promise2.then的回调函数
            this.handleResolved(x, deferred);
        }, 0);
    };
    /**
     * 获取回调函数的返回值
     * @param this TsPromise
     * @param deferred Handler
     * @param fn 回调函数
     * @param value 回调函数参数
     */
    private readonly tryCallValue = (deferred: Handler, fn: (value: any) => any, value: any) => {
        try {
            return fn(value);
        } catch (err) {  // 如果捕获发生异常，直接调失败，并把参数传进去
            deferred.promise.reject(err);
            return this.IS_ERROR;
        }
    };
    /**
     * 处理onFulfilled or onRejected执行后的值，并进行下一个promise2的then
     * @param this TsPromise
     * @param x onFulfilled or onRejected执行后的值
     * @param defered Handler
     */
    private readonly handleResolved = (x: any, defered: Handler): void => {
        if (x !== this.IS_ERROR) {       
            // 如果不报错
            // 如果 onFulfilled 或者 onRejected 返回一个值 x ，则运行下面的 Promise 解决过程
            // 根据 `TP/A+ 3.1` 若`x`和`promise2`相等，抛出`抛出TypeError`异常
            if (x === defered.promise) {
                return defered.promise.reject(new TypeError('返回值不能为本身'));
            }
            // 根据 `TP/A+ 3.3.3.3` resolvePromise 和 rejectPromise采用首次调用
            // called用于表示调用过后便不再调用
            let called: Boolean = false;
            if (x && (typeof x === 'object' || typeof x === 'function')) { 
                // `TP/A+ 3.3` 如果 x 是一个对象或者方法
                try {
                    // `TP/A+ 3.3.1`  x.then替代promise2.then
                    const then = x.then;
                    if (typeof then === 'function') {
                        // `TP/A+ 3.3.3` 执行x.then(resolvePromise, rejectPromise)
                        const resolvePromise = (y: any) => {
                            if (called) return; // 如果调用过就return掉
                            called = true;
                            // `TP/A+ 3.3.3.1`  如果 resolvePromise 以值 y 为参数被调用, 则继续运行 [[Resolve]](promise, y)
                            this.handleResolved(y, defered); // 递归调用
                        };
                        const rejectPromise = (err: any) => {
                            if (called) return; // 如果调用过就return掉
                            called = true;
                            // `TP/A+ 3.3.3.2`  如果rejectPromise以值据因 r 为参数被调用, 则promise2状态变为reject,reason值为r
                            defered.promise.reject(err);
                        };
                        then.call(x, resolvePromise, rejectPromise);
                    } else {
                        // `TP/A+ 3.3.4`若不是一个函数，promise2状态变为fulfilled, value值为x
                        defered.promise.resolve(x);
                    }
                } catch (e) {
                    // `TP/A+ 3.3.2` x.then抛出错误Error, promise2状态变为rejected, reason值变为Error
                    if (called) return; // 如果调用过就return掉
                    called = true;
                    return defered.promise.reject(e);
                }
            } else {
                // `TP/A+ 3.4`若 x不是一个对象或方法, promise2状态变为fulfilled, value值为x
                defered.promise.resolve(x);
            }
        }
    };
}

// 单个异步任务
class Handler {
    onFulfilled: any;
    onRejected: any;
    promise: TsPromise;
    constructor(onFulfilled: any, onRejected: any, promise: TsPromise) {
        this.onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : function (value: any) {  
            return value;
        };
        this.onRejected = typeof onRejected === 'function' ? onRejected : function (err: any) {      
            throw err;
        };
        this.promise = promise;
    }
}
```

#### 测试我们的TsPromise

利用[promises-aplus-tests](https://github.com/promises-aplus/promises-tests)可以测试我们的`TsPromise`，看看是否真的符合`Promise/A+`规范

1. 安装`promises-aplus-tests`: `npm install --save-dev promises-aplus-tests`

2. 编译我们的TsPromise：`tsc TsPromise.ts`

3. 给编译完成后的`TsPromise.js`加上如下代码 

```js
TsPromise.deferred = function() {
    let defer = {};
    defer.promise = new TsPromise((resolve, reject) => {
        defer.resolve = resolve;
        defer.reject = reject;
    });
    return defer;
};
try {
    module.exports = TsPromise;
} catch (e) {}

export default TsPromise;
```

4. `package.json`添加
```json
"scripts": {
    "promises": "promises-aplus-tests TsPromise.js"
},
```

5. 运行`npm run promises `， 没有报错，输出如下结果，完成！

![promises-aplus-tests](./files/promises-aplus-tests.png)

### 小结

`TypeScript`实现`Promise`到此结束。在[promise/A+ 规范中](https://promisesaplus.com/)有提到 `“宏任务（macro-task）”`机制和`“微任务（micro-task）”`。下一篇就将学习一下相关知识—[`Event Loop`](前端进阶/TypeScript实现Promise中.md)

**参考资料**

* [promise源码](https://github.com/then/promise)

  
