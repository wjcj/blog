Redux 所有 API 实现详解
====

本文讲解并实现 Redux 所有 API，包括 [createStore](http://cn.redux.js.org/api/createstore)、[combineReducers](http://cn.redux.js.org/api/combinereducers)、[applyMiddleware](http://cn.redux.js.org/api/applyMiddleware)、[compose](http://cn.redux.js.org/api/compose)、[bindactioncreators](http://cn.redux.js.org/api/bindactioncreators)。所有实现为了简明去除了很多非必要的逻辑，但保持了主要功能完整。本文中的源码可在[这里](https://github.com/wjcj/redux2)查看。

## `createStore`

**使用示例：**

``` jsx
export function counterReducer(state = 0, action) {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
};
const { dispatch, subscribe, getState } = createStore(counterReducer);
const unsubscribe = subscribe(() => console.log(getState()));
dispatch({ type: 'INCREMENT' });
unsubscribe();
```

**实现：**

这里本身没什么难度就不多赘述，看下面👇代码即可。`createStore` 返回 `store`，其中包含以下四个方法。

- `getState()`：返回应用当前的 state。
- `dispatch(action)`：分发 action，这是触发 state 变化的惟一途径。
- `subscribe(listener)`：添加一个变化监听器。
- `replaceReducer(nextReducer)`：替换 store 当前用来计算 state 的 reducer。

``` js
// 保证不会与用户定义的 actionType 重名即可
const ACTION_TYPE_INIT = `@@redux/INIT${Math.random()}`;
const ACTION_TYPE_REPLACE = `@@redux/REPLACE${Math.random()}`;

function createStore(reducer, preloadedState) {
  let currentReducer = reducer;
  const listeners = [];
  let state = preloadedState;

  const getState = () => {
    return state;
  };

  const subscribe = listener => {
    listeners.push(listener);
    // 用于取消订阅
    const unsubscribe = () => {
      const index = listeners.indexOf(listener);
      listeners.splice(index, 1);
    };
    return unsubscribe;
  }

  const dispatch = action => {
    state = currentReducer(state, action);
    listeners.forEach(listener => listener());
  };

  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }
    currentReducer = nextReducer;
    dispatch({ type: ActionTypes.REPLACE } ); // 替换 reducer 后重新初始化 state
    return store;
  }
  // reducer 都设置了默认的 state，调用 dispatch 完成 state 初始化
  dispatch({ type: ACTION_TYPE_INIT });

  const store = { dispatch, subscribe, getState, replaceReducer };
  return store;
};
```

## `combineReducers(reducers)`

应用较大时需要将 reducer 拆成多个 reducer 函数，以实现每个 reducer 负责独立管理 state 的其中一部分。而 `combineReducers` 的作用是允许你把拆分出来的多个 reducer 函数合并成一个最终的 reducer 函数供 `createStore` 使用。

**使用示例：**

``` js
// counter.js
export default function counter(state = 0, action) {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
// todos.js
export default function todos(state = [], action) {
  switch (action.type) {
    case 'ADD_TODO':
      return state.concat([action.text])
    default:
      return state
  }
}

const rootReducer = combineReducers({ todoList: todos, count: counter });
const store = createStore(rootReducer);
store.getState(); // { todoList: [], count: 0 }
```

**实现：**

实现的思路就是最终要返回一个合并后的 reducer 函数（即 `combination`）。这个 `combination` 函数接受到 `action` 后会调用拆分出来的所有 reducer 函数，得到最新的状态（`nextState`）。

``` js
export default function combineReducers(reducers) {
  const reducerKeys = Object.keys(reducers);

  return function combination(state = {}, action) {
    const nextState = {};
    for (let i = 0; i < reducerKeys.length; i++) {
      const key = reducerKeys[i];
      const reducer = reducers[key];
      const previousStateForKey = state[key];
      const nextStateForKey = reducer(previousStateForKey, action);
      nextState[key] = nextStateForKey;
    }
    return nextState;
  };
};
```

## `applyMiddleware(...middleware)`

### middleware（中间件）

Redux 中间件的本质扩展 `dispatch` 的功能，并且返回被增强后的 `dispatch`。我们先修改 `createStore` 中的 `dispatch` 方法，方便查看中间件的运行顺序。

``` js
const dispatch = action => {
  console.log('原始的 dispatch 开始工作了')
  state = reducer(state, action);
  listeners.forEach(listener => listener());
  console.log('原始的 dispatch 结束工作了')
};
```

先自定义两个中间件并使用：

``` js
function middleware1 (store) {
  return next => action => {
    console.log('Middleware 1号增强的 dispatch 开始工作了');
    const returnValue = next(action);
    console.log('Middleware 1号增强的 dispatch 结束工作了');
    return returnValue;
  };
}
function middleware2 (store) {
  return next => action => {
    console.log('Middleware 2号增强的 dispatch 开始工作了');
    const returnValue = next(action);
    console.log('Middleware 2号增强的 dispatch 结束工作了');
    return returnValue;
  };
}
const store = createStore(todosReducer, {}, applyMiddleware(middleware1, middleware2));
```
当我们发起一个 `store.dispatch(action)`，可以看到这样的日志：

```
Middleware 1号增强的 dispatch 开始工作了
Middleware 2号增强的 dispatch 开始工作了
原始的 dispatch 开始工作了
原始的 dispatch 结束工作了
Middleware 2号增强的 dispatch 结束工作了
Middleware 1号增强的 dispatch 结束工作了
```
可以看出多个中间件情况下，`next` 就是被下一个中间件增加后的 `dispatch`，最后一个中间件调用了的即使最原始的 `dispatch`。原始的 `dispatch` 被从右往左的中间件一个个增强。

所以如果只有一个中间件的情况下，`next` 就是最原始的 `store.dispatch`。所以上面代码等效于：

``` js
const store = createStore(todosReducer, {});
const next = store.dispatch;

store.dispatch = action => {
  console.log('Middleware 1号增强的 dispatch 开始工作了');
  next(action);
  console.log('Middleware 1号增强的 dispatch 结束工作了');
}
```

如果存在两个中间件：

``` js
const store = createStore(todosReducer, {});
const next = store.dispatch;

const next2 = action => {
  console.log('Middleware 2号增强的 dispatch 开始工作了');
  next(action); // 原始的 store.dispatch
  console.log('Middleware 2号增强的 dispatch 结束工作了');
}

store.dispatch = action => {
  console.log('Middleware 1号增强的 dispatch 开始工作了');
  next2(action);
  console.log('Middleware 1号增强的 dispatch 结束工作了');
}
```

上面虽然实现了自定义中间的功能，但是不够灵活，因为中间件可能不止2个，顺序需要自由的调换。所以我们就需要增加一个参数支持传入被下一个中间件增强的 `dispatch`： 

``` js
const next = store.dispatch;

const middleware2 = next => action => {
  console.log('Middleware 2号增强的 dispatch 开始工作了');
  const returnValue next(action);
  console.log('Middleware 2号增强的 dispatch 结束工作了');
  return returnValue;
}

const middleware1 = next => action => {
  console.log('Middleware 1号增强的 dispatch 开始工作了');
  const returnValue = next(action);
  console.log('Middleware 1号增强的 dispatch 结束工作了');
  return returnValue;
}

store.dispatch = middleware1(middleware2(next));
```

又考虑到每个中间内部不仅仅是执行下一个 `next`，比如一个 logger 中间件需要通过 `store.getState` 方法获取 state，一些支持异步场景的中间件内部还需要调用 `store.dispatch` ...

``` js
const logger = next => action => {
  return next => action => {
    console.log('即将 dispatch: ', action)
    const returnValue = next(action)
    console.log('dispatch 后的 state: ', store.getState())
    return returnValue
  }
}
```

这些都来自 `store`，所以每个中间件还需要增加一个参数 `mAPI` 以接收这些 API。

``` js
const next = store.dispatch;

const middleware2 = mAPI => next => action => {
  console.log('Middleware 2号增强的 dispatch 开始工作了');
  const returnValue = next(action);
  console.log('Middleware 2号增强的 dispatch 结束工作了');
  return returnValue;
}

const middleware1 = mAPI => next => action => {
  console.log('Middleware 1号增强的 dispatch 开始工作了');
  const returnValue = next(action);
  console.log('Middleware 1号增强的 dispatch 结束工作了');
  return returnValue;
}
const mAPI = store;
const m2 = middleware2(mAPI);
const m1 = middleware1(mAPI);

store.dispatch = m1(m2(next));
```

Redux 中规定中间件中可使用的 API 只有 `getState` 和 `dispatch`。所以改进之后：

``` js
const next = store.dispatch;
// ...

let dispatch = next;
const mAPI = {
  getState: store.getState,
  dispatch: (action, ...args) => dispatch(action, ...args)
}
const chain = [middleware1, middleware2].map(m => m(mAPI)); // [m1, m2]

// 实现：store.dispatch = m1(m2(next))
dispatch = chain.reverse().forEach(m => {
  dispatch = m(dispatch);
});
store.dispatch = dispatch;
```

### applyMiddleware

上面已经实现了中间件的功能，但是还不够优雅，接下来要做的就是把上面的过程收拢到一个 `applyMiddleware` 函数内部，期望的使用方式是：

``` js
const store = createStore(reducer, preloadedState, applyMiddleware(...middlewares));
```

`applyMiddleware(...middlewares)` 的目的是增强 `createStore` 返回的 `dispatch` 方法。所以可以理解为 `applyMiddleware(...middlewares)` 返回 `createStore` 的增强器 `enhancer`，将 `createStore` 传入增强器后返回 `createStore` 的升级版。

``` js
const enhancer = applyMiddleware(...middlewares);
const createStore2 = enhancer(createStore);
```

`createStore` 中增加一段逻辑，当传入 `enhancer` 时返回增强版的 `createStore`：

``` js
// const enhancer = applyMiddleware(middleware1, middleware2);
function createStore(reducer, preloadedState, enhancer) {
  if (typeof applyMiddleware === 'function') {
    const createStore2 = enhancer(createStore);
    return createStore2(reducer, preloadedState);
  }
  // ...
}
```

**实现 `applyMiddleware` ：**

把处理中间件，增加 `dispatch` 的逻辑收拢到 `applyMiddleware` 返回的 `createStore` 增强器中。

``` js
function applyMiddleware(...middlewares) {
  const enhancer = createStore => (reducer, preloadedState) => {
      const store = createStore(reducer, preloadedState);

      const next = store.dispatch;
      let dispatch = next;

      const mAPI = {
        getState: store.getState,
        dispatch: (action, ...args) => dispatch(action, ...args),
      }

      const chain = middlewares.map(middleware => middleware(mAPI));

      chain.reverse().forEach(m => {
        dispatch = m(dispatch);
      });

      return {
        ...store,
        dispatch
      }
    };
  return enhancer;
}
```

## `compose(...functions)`

`compose` 的作用是从右到左把接收到的函数合成为一个最终函数，右边函数的返回值将作为一个参数提供给它左边的函数。即 `compose(funcA, funcB, funcC)` 变为 `compose(funcA(funcB(funcC())))`。

``` js
function compose(...funcs) {
  if (funcs.length === 0) return arg => arg;

  if (funcs.length === 1) return funcs[0];

  return funcs.reduce((a, b) => {
    retrun (...args) => a(b(...args));
  });
}
```

`applyMiddleware` 处理中间件即可复用此逻辑，`store.dispatch = m1(m2(store.dispatch))`，所以 `applyMiddleware` 可以使用 `compose` 修改为：

``` js
function applyMiddleware(...middlewares) {
  const enhancer = createStore => (reducer, preloadedState) => {
      const store = createStore(reducer, preloadedState);

      let dispatch;

      const mAPI = {
        getState: store.getState,
        dispatch: (action, ...args) => dispatch(action, ...args),
      }

      const chain = middlewares.map(middleware => middleware(mAPI));
      dispatch = compose(...chain)(store.dispatch);

      return {
        ...store,
        dispatch
      }
    };
  return enhancer;
}
```

## `bindActionCreators(actionCreators, dispatch)`

`actionCreator` 即创建 action 的函数，比如下面的两个👇。`actionCreator` 的好处是复用创建 action 的逻辑，使用 `dispatch` 时只需 `dispatch(actionCreator(value))`。

``` jsx
export function addTodo(text) {
  return {
    type: 'ADD_TODO',
    text
  }
}
export function removeTodo(id) {
  return {
    type: 'REMOVE_TODO',
    id
  }
}
```

`bindActionCreators` 的唯一使用场景就是当你需要把 actionCreator 传给子组件，但却不想让子组件感觉到 Redux 的存在。

``` js
import { addTodo, removeTodo } from './TodoActionCreators'

class TodoListContainer extends Component {
  constructor(props) {
    super(props)
    const { dispatch } = props; // 由 react-redux 注入的 dispatch

    // 对 actionCreator 绑定 dispatch 方法
    this.boundActionCreators = bindActionCreators({
      addTodo,
      removeTodo,
    }, dispatch);
    console.log(this.boundActionCreators); // { addTodo: Function, removeTodo: Function }
  }

  render() {
    // 由 react-redux 注入的 todos：
    let { todos } = this.props
    return <TodoList todos={todos} {...this.boundActionCreators} />

    // 不使用 bindActionCreators 时的做法：
    // 直接把 dispatch 当作 prop 传递给子组件，但这时子组件需要引入 actionCreator
    // return <TodoList todos={todos} dispatch={dispatch} />;
  }
}
```

**实现 `bindActionCreators` ：**

`bindActionCreators` 的实现很简单，如果我们不想感知到 `dispatch` 和 `actionCreator`，只需要利用闭包实现隐藏：

``` js
function bindActionCreator(actionCreator, dispatch) {
  return function (this, ...args) {
    return dispatch(actionCreator.apply(this, args));
  }
}
// 使用：
const addTodo = bindActionCreator(addTodo, store.dispatch);
addTodo('Use Redux');
```

如果有多个 `actionCreator` 则返回一个 `boundActionCreators` 对象：

``` js
export default function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch);
  }

  if (typeof actionCreators !== 'object' || actionCreators === null) {
    throw new Error('bindActionCreators expected an object or a function.');
  }

  const boundActionCreators = {}
  for (const key in actionCreators) {
    const actionCreator = actionCreators[key];
    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch);
    }
  }
  return boundActionCreators;
}

// 使用：
const boundActionCreators = bindActionCreator({
  addTodo,
  removeTodo,
}, dispatch);
boundActionCreators.addTodo('Use Redux');
```

## 参考

- [redux](http://cn.redux.js.org/introduction/getting-started)
- [redux github](https://github.com/reduxjs/redux)
