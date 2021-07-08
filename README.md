该篇文章是 Kai Guo 的 [medium 文章](https://medium.com/@guokai83524/building-redux-from-scratch-e12eb0e484c8) 的翻译。

日常工作、学习中我们在写 React 项目需要全局状态管理时经常会用到 Redux。今天让我们从无到有自己动手写一个 Redux 来更好地理解它是如何工作的。不要害怕，抛开 react-redux，Redux 本身并不复杂。

# APIs

在我们开始动手之前，我们需要总结一下 Redux 的主要 APIs：

1. `createStore(reducer, state, enhancer)`：创建一个 Redux store，返回的 store 对象提供以下 APIs：
   1. `getState()`：获取 store 的 state
   2. `dispatch(action)`：接收 action 对象更新 store state
   3. `subscribe(listener)`：对 store 的更新订阅
   4. `replaceReducer(nextReducer)`：该进阶 API 我们在本文章中不做讨论
2. `combineReducers(reducerObj)`：将多个 reducers 合并成一个 reducer
3. `applyMiddleware(...middlewares)`：接收多个中间件函数并返回一个供`createStore`使用的 enhancer 函数

# Store

## createStore

`createStore` 接受三个参数 `reducer`、`state` 和一个 `enhancer` 函数并且返回一个 store 对象。其中 `state` 和 `enhancer` 都是可选参数：

```js
function createStore(reducer, initialState, enhancer) {
  const store = {};
  let state = initialState;
  const listeners = [];

  store.getState = () => state;
  store.dispatch = (action) => {};
  store.subscribe = (listener) => listeners.push(listener);

  return store;
}
```

## dispatch

`dispatch` 做的事情非常简单，接收一个 `action` 对象，通过 `reducer` 得到新的 state 并且调用所有侦听器：

```js
store.dispatch = (action) => {
  state = reducer(state, action);
  listeners.forEach((listener) => listener());
};
```

## subscribe

Redux 的 `store` 的 `subscribe` API 让我们可以侦听 `state` 更新，并且返回一个解除侦听的 `unsubscribe` 函数：

```js
store.subscribe = (listener) => {
  listeners.push(listener);
  return () => {
    const index = listeners.indexOf(listener);
    listeners.splice(index, 1);
  };
};
```

值得注意的是上面代码中我们并不需要检测`listener`是否存在于 `listeners` 中，因为在该闭包中 `listener` 肯定已经被 push 到 `listeners` 中了。

# combineReducers

在我们调用 `createStore` 创建 store 之前，我们一般都会调用 `combineReducers` 将多个子 `reducer` 合并成一个根 `reducer` 供 Redux 使用。

`combineReducers` 做的事情非常简单，它接收一个键为子状态名，值为子状态 `reducer` 的对象，并返回一个合并后的 `reducer` 函数。

```js
function combineReducers(obj) {
  return (state = {}, action) => {
    const newState = {};
    for (const key in obj) {
      // obj[key] 为子 reducer，接收子状态和 action 对象
      newState[key] = obj[key](state[key], action);
    }
    return newState;
  };
}
```

# applyMiddleware

到目前为止的内容都比较清晰易懂，接下来中间件的内容可能会打破你的这个想法。中间件为第三方提供了扩展的功能。

我们首先设想一个非常简单的 log 中间件，它可以打印每一个 dispatch 的 action 对象并且在状态更新后打印新的状态。如何实现这样一个中间件呢？

最简单直接的方法就是改变我们的 dispatch 函数：

```js
store.dispatch = (action) => {
  console.log('action', action);
  state = reducer(state, action);
  console.log('state', state);
  listeners.forEach((listener) => listener());
};
```

上面的代码可以实现我们的目的，但是将无关的代码添加到 `dispatch` 内部显然是一个 bad idea。我们需要将第三方的逻辑从 `dispatch` 中抽离出来。第二种方法定义了一个嵌套了原始 `dispatch` 的新 `dispatch` 方法：

```js
function loggingDispatch(store, action) {
  console.log('action', action);
  store.dispatch(action);
  console.log('state', store.getState());
}
```

上面这个版本的 `dispatch` 在原始 `dispatch` 周围添加上了 logging 的逻辑来达到我们的目的。但是如果每次我们需要一个中间件的时候就定义一个新的 dispatch 方法的话这是非常不方便的，因为我们可能会使用多个中间件。为了使用多个中间件，每个中间件的 input 和 output 应该是同类型的：

store.dispatch -> **Middleware 1** -> dispatch version 1 -> **Middleware 2** -> dispatch version 2

`store.dispatch` 是最原始的 `dispatch`，每个中间件都接收一个 `dispatch` 并返回一个新版本的 `dispatch`：

```js
function middleware(dispatch) {
  return function newDispatch(action) {
    dispatch(action);
  };
}
```

让我们根据上面的模板重新写一个 logging 中间件：

```js
function logger(dispatch) {
  return function newDispatch(action) {
    console.log('action', action);
    dispatch(action);
    console.log('state', store.getState());
  };
}
```

等等！！！上面的代码中的 `store` 是从哪里来的？我们如何才能获取到这个 `store` 呢？答案很简单，我们在中间件代码外面再嵌套一层函数将 `store` 传入，即创建一个 `middlewareFactory` 函数：

```js
function createLoggerMiddleware(store) {
  return function logger(dispatch) {
    return function newDispatch(action) {
      console.log('action', action);
      dispatch(action);
      console.log('state', store.getState());
    };
  };
}
```

如果我们想要使用多个中间件呢？很简单，我们可以创建一个接收 `store` 和一系列中间件工厂函数的 `decorateDispatch` 函数，并输出一个最终版本的 `dispatch`：

```js
function decorateDispatch(store, middlewareFactories) {
  let dispatch = store.dispatch;
  middlewareFactories.forEach((factory) => {
    dispatch = factory(store)(dispatch);
  });
  return dispatch;
}
```

现在如果我们修改一下 createStore API 的接收参数让它接收一个中间件工厂函数数组作为第三个参数的话，我们就已经实现了一个完整的 Redux：

```js
function createStore(reducer, initialState, middlewareFactories = []) {
  const store = {};
  let state = initialState;
  const listeners = [];

  store.getState = () => state;
  store.dispatch = (action) => {
    state = reducer(state, action);
    listeners.forEach((listener) => listener());
  };
  store.subscribe = (listener) => {
    listeners.push(listener);
    return () => {
      const index = listeners.indexOf(listener);
      listeners.splice(index, 1);
    };
  };

  store.dispatch = decorateDispatch(store, middlewareFactories);
  return store;
}
```

Redux 本身提供了另一个 API applyMiddleware 来实现中间件的功能。

```js
function createStore(reducer, initialState, enhancer) {
  if (typeof enhancer === 'function') {
    return enhancer(createStore)(reducer, initialState);
  }
  ...
}
```

`enhancer` 接收 createStore 并返回一个新版的 createStore。这和上面我们设计中间件的思路非常相似：

```js
function enhancer(createStore) {
  return function newCreateStore(reducer, initialState) {
    createStore(reducer, initialState);
  };
}
```

但是我们如何让 `enhancer` 获取中间件工厂函数列表呢？和中间件的思路一样，我们可以创建一个 `enhancerFactory` 传入中间件工厂函数列表：

```js
function enhancerFactory(...middlewareFactories) {
  return function enhancer(createStore) {
    return function newCreateStore(reducer, initialState) {
      createStore(reducer, initialState);
    };
  };
}
```

眼熟不？上面的 enhancerFactory 就是我们熟悉的 applyMiddleware！

```js
function applyMiddleware(...middlewareFactories) {
  return function enhancer(createStore) {
    return function newCreateStore(reducer, initialState) {
      const store = createStore(reducer, initialState);
      let dispatch = store.dispatch;
      middlewareFactories.forEach((factory) => {
        dispatch = factory(store)(dispatch);
      });
      store.dispatch = dispatch;
      return store;
    };
  };
}
```

## 科里化

还记得我们的 logging 中间件工厂函数吗？

```js
function createLoggerMiddleware(store) {
  return function logger(dispatch) {
    return function newDispatch(action) {
      console.log('action', action);
      dispatch(action);
      console.log('state', store.getState());
    };
  };
}
```

上面的代码和下面的一模一样：

```js
const createLoggerMiddleware = (store) => (dispatch) => (action) => {
  console.log('action', action);
  dispatch(action);
  console.log('state', store.getState());
};
```

## Redux-thunk 中间件

有了上面的知识，我们就可以理解 redux-thunk 这个中间件了：

```js
const createThunkMiddleware = (store) => (dispatch) => (action) => {
  if (typeof action === 'function') {
    return action(dispatch, store.getState());
  }
  return dispatch(action);
};
```
