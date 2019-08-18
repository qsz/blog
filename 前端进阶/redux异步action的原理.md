# redux异步action的原理解析

我们知道`Redux`的数据流是单向的，其过程为：

1. 用户发出`Action`，
2. 利用`store.dispatch(action)`，调用`Reducer`，接受两个参数，当前`state`(`currentState`)和用户发起的`action`，根据`action`中定义的规则，改变`state`。
3. `state`改变后，调用监听函数`store.subscribe(listener)`，更新试图。`listener`则为具体的更新试图方法。在`React`中可以是`setState`方法

### 为何是同步数据流

`dispatch`用于提交`action`，主要代码如下。

```js
function dispatch(action){
  	if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
          'Use custom middleware for async actions.'
      )
    }
  	...
  	try {
			isDispatching = true
			currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }
 		...
}	
```

可以看到`dispatch`中对`action`做了判断，通过`isPlainObject`，如果`action`不是一个对象，是会抛出错误的。同时在`currentReducer`方法（即我们在`createStore`传入的`reducer`）中也限制了`action`必须为一个对象。因此默认的`redux`是同步的数据流。

### 异步数据流

在实际应用中，我们一般引入中间件`redux-thunk`，来提交异步`action`

```js
import thunk from 'redux-thunk';
import {createStore, applyMiddleware} from 'redux';
...
const store = createStore(reducers, {}, applyMiddleware(thunk));
```

#### 中间件实现

##### createStore 中引入 applyMiddleware

```js
function createStore(reducer, preloadedState, enhancer) {
  ...
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }
  ...
}
```

可见`createStore`方法有三个参数，如果第三个参数是函数，`createStore`会将`createStore`,`reducer`和初始`state`(`preloadedState`)传给`enhancer`，并将其返回

#####  applyMiddleware实现

上面`createStore`的第三个参数就是`applyMiddleware`

`redux`中`applyMiddleware`用于引入中间件，看代码

```js
function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

`applyMiddleware`参数是中间件数组，并返回函数，返回函数接受`createStore`，并在此返回一个函数。根据`createStore`中的实现

>  enhancer(createStore)(reducer, preloadedState)

可知其参数为`reducer`和`preloadedState`。

由代码可知`applyMiddleware`函数最终会返回 `store`对象和经过改造后的`dispatch`。

`applyMiddleware`借助了`compose`方法改造`dispatch`，实现很巧妙，看代码

##### compose代码

```js
function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

`compose`函数利用`reduce`方法，依次执行数组中每个元素，并将其返回结果作为下一个元素的参数，最终达到以下效果

> function a(params) { return ...}
>
> function b(params) { return ...}
>
> function c(params) { return ...}
>
> function(...args) {
>
> ​	return f(a(b(c())))
>
> }

#### redux-thunk实现

由`applyMiddleware`代码中的这一部分，

```js
const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
}
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch)
```

我们也可以看出`redux`中间件的基本实现格式:

1. 接受 `{ getState, dispatch }`两个参数
2. 第一次的返回函数接收`store`对象初始的`dispatch`方法
3. 最后返回改造后`dispatch`方法，同样接收一个`action`参数

接下来我们看`redux-thunk`的代码，代码很简洁

```js
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}
```

由上面的总结可知，这里的`next`就是`store.dispatch`。`redux-thunk`对传入的参数`action`做判断，如果`action`是一个函数，就执行`action`函数，并将`dispatch, getState`依次传入。若不是函数，就按初始的`store.dispatch`执行`dispatch(action)`

因此，经过改造后，`dispatch`可以接收函数作为参数，该函数的参数为`redux`提供的方法：`dispatch, getState`

```js
function ajaxInfoSuccess(dispatch, getState) {
  ajax('/info').then((data) => {
    	dispatch({type: 'infoSuccess', data: data})
 	})
}

store.dispatch(ajaxInfoSuccess)
```

因此，就实现了异步`action`



### 为何要用异步action

上面提到经过`redux-thunk`包装后，可以实现异步`action`。但用同步`action`也能达到一样的效果

```js
ajax('/info').then((data) => {
  store.dispatch({type: 'infoSuccess', data: data})
})
```

那么为什么要用异步的方式呢？

我的理解是这样的。

1. 首先，很直观的一点，异步的方式提供了第二个参数`getState`可以方便我们快速拿到当前`redux`的`state`	数据。使用过`vuex`的同学应该知道，`vuex`也提供了`dispatch(action)`，用于异步提交`muataion`。在`vuex`的`action`函数中，传入了一个对象，包含了当前模块下的 `dispatch`、`commit`、`getters`、`state`，以及全局的 `rootState` 和 `rootGetters`，由此可以在参数中获取到当前模块的一些方法和状态。这点比同步的方式更快捷。
2. 第二点，使用异步的方式可以让`action`功能更明确。我们在`dispatch(action)`时候，不需要关心是否有异步的操作。当需要提交`type: infoSuccess`时，直接`store.dispatch(ajaxInfoSuccess)`，而不是等异步回调后再`store.dispatch(...)`。使用者不需要关心改变至`infoSuccess`内部的实现，不需要考虑同步或者异步。同时，当我们的`ajax('/info')`需要改变时，只需要更改`action`内部的代码即可，而不需要在每个调用`ajax('/info')`的地方去更改。