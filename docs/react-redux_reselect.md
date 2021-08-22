react-redux、reselect 使用与原理详解
=====

# react-redux

在未使用 `react-redux` 之前，如果想让所有子组件都可以获取 store，可以通过 `context` 将 store 传递給所有子组件。
但 `context` 的缺陷在于层级中的任意一个组件 `shouldComponentUpdate` 返回 `false`，`context` 将不再继续传递下去。

`react-redux` 的原理是通过 `Provider` 组件将 store 放到自己的 `context` 中，需要使用 `store` 中数组的组件则可以通过 `connect` 与 `context` 进行连接，取出 `context` 中的所需的数据后通过 props 传入组件。

`Provider` 组件实现如下：
``` js
class Provider extends Component {
  static propTypes = {
    store: PropTypes.object,
    children: PropTypes.any
  }

  static childContextTypes = {
    store: PropTypes.object
  }

  getChildContext () {
    return {
      store: this.props.store
    }
  }

  render () {
    return <>{this.props.children}</>
  }
}
// 使用：
<Provider store={store}>
  <App/>
</Provider>
```

### connect

`connect` 的本质是返回一个高阶组件，该高阶组件从 `context` 获取 store，通过 `mapStateToProps` 提取 store 中状态，通过 `mapDispatchToProps` 得到 `action` 的分发函数，并将这些通过 props 传递给子组件。具体实现如下：

``` jsx
export const connect = (mapStateToProps, mapDispatchToProps) => WrappedComponent => {
  return class Connect extends Component {
    static contextTypes = {
      store: PropTypes.object,
    }

    constructor () {
      super();
      this.state = { allProps: {} };
    }

    componentWillMount () {
      const { store } = this.context;
      this._getProps();
      store.subscribe(() => this._getProps());
    }

    _getProps () {
      const { store } = this.context
      const stateProps = mapStateToProps ? mapStateToProps(store.getState(), this.props) : {};
      const dispatchProps = mapDispatchToProps ? mapDispatchToProps(store.dispatch, this.props) : {};
      this.setState({
        allProps: {
          ...stateProps,
          ...dispatchProps,
          ...this.props,
        },
      });
    }

    render () {
      return <WrappedComponent {...this.state.allProps} />
    }
  }
}
```

使用示例：
``` jsx
import { connect } from 'react-redux';

const mapStateToProps = (state, ownProps) => ({ count: state.count });
const mapDispatchToProps = dispatch => {
  return {
    increment: () => dispatch({ type: 'INCREMENT' }),
    decrement: () => dispatch({ type: 'DECREMENT' }),
    reset: () => dispatch({ type: 'RESET' }),
  }
};
export default connect(mapStateToProps, mapDispatchToProps)(Counter);
```

### 缺陷

其实 `react-redux` 仍然有一些缺陷，从上面的实现可知 `store` 中状态只要发生变更就会执行 `this._getProps()`，这就意味着每次更变都会需要执行 `mapStateToProps` 函数来计算提取状态。`mapStateToProps` 相当于 [useSelector](#useSelector) 中 `selector`，即用于从 store 中提取状态的函数，如果 `selector` 的计算比较复杂将带来性能问题，解决问题的思路就是如果 `selector` 接受的参数没有变化则不重新计算，直接使用之前计算的结果，这可以借助 `reselect` 来实现。具体见 [reselect](#reselect) 中的内容。

# react-redux hooks

从 v7.1.0 开始，`react-redux` 提供了一些 `hooks` 用于简化原有 API 的使用，与 TypeScript 的配合也更好。

### useSelector

`useSelector(selector: function, equalityFn?: function)`。

`useSelector` 的作用就是从 store 的 state 中提取状态数据，相当于 `connect` 参数中的 `mapStateToProps`。下面是使用示例：

``` ts
import { useSelector as _useSelector, shallowEqual, TypedUseSelectorHook } from 'react-redux';
interface RootState = { counter: number };
const useSelector: TypedUseSelectorHook<RootState> = _useSelector;

const Counter = () => {
  const counter = useSelector(state => state.counter);
  // const counter = useSelector(state => state.counter, shallowEqual);
  return <div>{counter}</div>
}
```

`selector` 与 `mapStateToProps` 主要区别在于 `selector` 可以返回任何值，而不仅仅是对象。

``` tsx
const selector = state => state.counter;
const counter = useSelector(selector);
```

**返回多个值**

当执行一个 action 后，`useSelector` 默认会使用 `===` 去比较前后两次 `selector` 返回的值，如果变更才会强制重新渲染。如果你需要返回多个值，你可以：

1. 使用多个 `useSelector` ，每个返回一个值。

2. 返回一个包含了多个值的对象。

使用 `connect` 时，它会对 `mapStateToProps` 返回的对象进行浅比较，由于是比较对象中每个值，对象是否变更不会产生影响，从而避免了一些不必要的更新。
但 `useSelector` 默认使用 `===` 去比较两次 `selector` 返回的值，每次返回一个包含多个值（key）的对象将出现问题。这时你就可以传入 `shallowEqualFn` 参数实现对象的浅比较，`react-redux` 已经提供的浅比较函数 `shallowEqual`。

``` tsx
import { shallowEqual, useSelector } from 'react-redux';
const selector = state => {
  todos: state.todos,
  count: state.counter
};
const { todos, count }  = useSelector(selector, shallowEqual);
```

3. 创建一个具有记忆功能（memoization）的 `selector`。

这里使用 [reselect](https://github.com/reduxjs/reselect) 中的 `createSelector` 方法来创建一个记忆化的 `selector`（memoized selector），如果输入 `selector` 的参数与之前调用时相同，则会返回之前计算的值，而不会重新计算，从而避免不必要计算带来的性能问题。`createSelector` 的实现和使用示例将在下文的 [reselect](#reselect) 中介绍。

### useDispatch

`const dispatch = useDispatch(actionCreator());`

`useDispatch` 的作用就是返回一个 `dispatch` 函数。

``` tsx
import { useDispatch } from 'react-redux';
import store from './store';

type DispatchType = typeof store.dispatch;

const Counter = ({ value }) => {
  const dispatch: DispatchType = useDispatch();

  const onIncrement = useCallback(() => dispatch({ type: 'increment' }), [dispatch]);

  return (
    <div>
      <span>{value}</span>
      <button onClick={onIncrement}>+1</button>
    </div>
  )
}
```

### useStore

`const store = useStore()`。`useStore` 返回传递给 `<Provider>` 组件的 store。仅用于需要访问 store 的场景，其他情况下通常使用 `useSelector`。

# reselect 

在 `redux` 或 `react-redux` 场景下，通过定义 `mapStateToProps/selector` 从 store 提取所需的 state，由于 store 的状态变更都会调用 `mapStateToProps/selector` ，如果其中的计算复杂将带来性能问题。`reselect` 的主要功能是可以使用 `createSelector` 方法创建具有记忆功能的 `selector`，它会缓存上一次的计算结果，仅当 `state` 发生变更，才会调用 `selector` 重新计算。

``` js
const memoizationSelector = createSelector(...inputSelectors | [inputSelectors], resultFunc);
```

`createSelector` 接受一个或多个选择器或一组选择器（`selector`），计算它们的值并将它们作为参数传递给 `resultFunc`。`resultFunc` 是一个转换函数，通常用于返回多个 state 值。

使用示例：
``` jsx
import { createSelector } from 'reselect';

const getVisibilityFilter = state => state.visibilityFilter;
const getTodos = state => state.todos;

// createSelector 返回了一个 memoization selector
const getVisibleTodos = createSelector(
  // 一组 selector 👇
  [ getVisibilityFilter, getTodos ],
  // resultFunc 👇
  (visibilityFilter, todos) => {
    switch (visibilityFilter) {
      case 'SHOW_ALL':
        return todos;
      case 'SHOW_COMPLETED':
        return todos.filter(t => t.completed);
      case 'SHOW_ACTIVE':
        return todos.filter(t => !t.completed);
    }
  },
);

// 使用
const mapStateToProps = (state, props) => {
  return { todos: getVisibleTodos(state) };
}
export default connect(mapStateToProps)(Demo);

// 或者
const Demo = () => {
  const todos = useSelector(getVisibleTodos);
  return <TodoList todos={todos} />
}
export default Demo;
```

`createSelector` 的实现并不是很复杂，它允许一次接受多个 `selector`，返回组合后的 `selector`，下面是简化后实现代码：

``` js
// 判断相等的函数
function defaultEqualityCheck(a, b) {
  return a === b
}

// 此方法用于判断前后两次函数的参数是否相等。equalityCheck 默认为 `defaultEqualityCheck`
function areArgumentsShallowlyEqual(equalityCheck = defaultEqualityCheck, prev, next) {
  if (prev === null || next === null || prev.length !== next.length) {
    return false
  }
  const length = prev.length
  for (let i = 0; i < length; i++) {
    if (!equalityCheck(prev[i], next[i])) {
      return false
    }
  }
  return true
}

// 默认的记忆函数。它会缓存上次传入的所有参数和计算的结果，如果前后两次参数不一致才会重新计算，并重新缓存参数和计算结果
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null
  let lastResult = null
  return function () {
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      lastResult = func.apply(null, arguments)
    }

    lastArgs = arguments
    return lastResult
  }
}
 
// 用于创建 `createSelector` 方法。这里省略了 `memoizeOptions` 选项
export function createSelectorCreator(memoize) {
  return (...funcs) => {
    let recomputations = 0
    // createSelector 的最后一个参数是转换函数（resultFunc），用于从 store state 中提取多个值
    const resultFunc = funcs.pop()
    // 传入 createSelector 的多个 selector
    const dependencies = Array.isArray(funcs[0]) ? funcs[0] : funcs
    
    // 使用 resultFunc 具有记忆功能，当参数（传入的 selector 计算的结果）发生变更才会执行 resultFunc，否则返回上一次计算的结果
    const memoizedResultFunc = memoize(() => resultFunc.apply(null, arguments))

    // 用于最终返回的 selector。
    // memoize 的作用是返回具有记忆功能的 selector。在 redux 场景下，state 发生变更才会重新计算，否则返回上一次计算的结果
    const selector = memoize(() => {
      const params = []
      const length = dependencies.length
      // 传入 createSelector 的多个 selector 的计算结果将作为最终返回的 selector 的参数
      for (let i = 0; i < length; i++) {
        params.push(dependencies[i].apply(null, arguments))
      }
      return memoizedResultFunc.apply(null, params)
    })

    return selector
  }
}

export const createSelector = createSelectorCreator(defaultMemoize)
```

# 参考

- [reselect](https://github.com/reduxjs/reselect)
- [react-redux](https://react-redux.js.org/api/hooks)