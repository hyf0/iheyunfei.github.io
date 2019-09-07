---
title: 译 - React-Redux 官方 Hooks 文档说明
date: 2019-09-07 11:27:57
tags: 
- 翻译
- react
- react-redux
- 前端
---

> - 来源：[React Redux: Hooks](https://react-redux.js.org/next/api/hooks#usestore)
> - 译者：[塔希](https://github.com/iheyunfei)
> - 协议：CC BY-NC-SA 4.0

# Hooks

React的新 ["hooks" APIs](https://zh-hans.reactjs.org/docs/hooks-intro.html) 赋予了函数组件使用本地组件状态，执行副作用，等各种能力。

React Redux 现在提供了一系列 hook APIs 作为现在 <code>connect()</code> 高阶组件的替代品。这些 APIs 允许你，在不使用 <code>connect()</code> 包裹组件的情况下，订阅 Redux 的 store，和 分发(dispatch) actions。

这些 hooks 首次添加于版本 v7.1.0。

## 在一个 React Redux 应用中使用 hooks

和使用 <code>connect()</code> 一样，你首先应该将整个应用包裹在 <code>&lt;Provider&gt;</code> 中，使得 store 暴露在整个组件树中。

```js
const store = createStore(rootReducer)

ReactDOM.render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById('root')
)
```

然后，你就可以 import 下面列出的 React Redux hooks APIs，然后在函数组件中使用它们。

# <code>useSelector()</code>

```js
const result : any = useSelector(selector : Function, equalityFn? : Function)
```
通过传入 selector 函数，你就可以从从 Redux 的 store 中获取 状态(state) 数据。

> **警告**: selector 函数应该是个[纯函数](https://zh.wikipedia.org/wiki/%E7%BA%AF%E5%87%BD%E6%95%B0)，因为，在任意的时间点，它可能会被执行很多次。

从概念上讲，selector 函数与 [<code>connect</code> 的 <code>mapStateToProps</code> 的参数](https://react-redux.js.org/next/using-react-redux/connect-mapstate)是差不多一样的。selector 函数被调用时，将会被传入Redux store的整个state，作为唯一的参数。每次函数组件渲染时， selector 函数都会被调用。<code>useSelector()</code>同样会订阅 Redux 的 sotre，并且在你每 分发(dispatch) 一个 action 时，都会被执行一次。

尽管如此，传递给 <code>useSelector()</code> 的各种 selector 函数还是和 <code>mapState</code> 函数有些不一样的地方：

- selector 函数可以返回任意类型的值，并不要求是一个 对象(object)。selector 函数的返回值会被用作调用 <code>useSelector()</code> hook 时的返回值。
- 当 分发(dispatch) 了一个 action 时，<code>useSelector()</code> 会将上一次调用 selector 函数结果与当前调用的结果进行引用(===)比较，如果不一样，组件会被强制重新渲染。如果一样，就不会被重新渲染。
- selector 函数不会接收到 <code>ownProps</code> 参数。但是 props 可以通过闭包获取使用(下面有个例子) 或者 通过使用柯里化的 selector 函数。
- 当使用 记忆后(memoizing) 的 selectors 函数时，需要一些额外的注意(下面有个例子帮助了解)。
- <code>useSelector()</code> 默认使用严格比较 <code>===</code> 来比较引用，而非浅比较。(看下面的部分来了解细节)

<small>

译者注: 浅比较并不是指 ==。严格比较 === 对应的是 疏松比较 ==，与 *浅比较* 对应的是 *深比较*。

</small>


> **警告**: 在 selectors 函数中使用 props 时存在一些边界用例可能导致错误。详见本页的 **[使用警告](#使用警告)** 小节。

你可以在一个函数组件中多次调用 <code>useSelector()</code>。每一个 <code>useSelector()</code> 的调用都会对 Redux 的 store 创建的一个独立的 订阅(subscription)。由于 Redux v7 的 批量更新(update batching) 行为，对于一个组件来说，如果一个 分发后(dispatched) 的 action 导致组件内部的多个 <code>useSelector()</code> 产生了新值，那么仅仅会触发一次重渲染。


## 相等比较(Equality Comparisons) 和更新

当一个函数组件渲染时，传入的 selector 函数会被调用，其结果会作为 <code>useSelector()</code> 的返回值进行返回。(如果 selector 已经执行过，且没有发生变化，可能会返回缓存后的结果)

不管怎样，当一个 action 被分发(dispatch) 到 Redux store 后，<code>useSelector()</code> 仅仅在 selector 函数执行的结果与上一次结果不同时，才会触发重渲染。在版本v7.1.0-alpha.5中，默认的比较模式是严格引用比较 ===。这与  <code>connect()</code>  中的不同， <code>connect()</code>  使用浅比较来比较 <code>mapState</code> 执行后的结果，从而决定是否触发重渲染。这里有些建议关于如何使用<code>useSelector()</code>。

对于 <code>mapState</code> 来讲，所有独立的状态域被绑定到一个对象(object) 上返回。返回对象的引用是否是新的并不重要——因为  <code>connect()</code>  会单独的比较每一个域。对于 <code>useSelector()</code> 来说，返回一个新的对象引用总是会触发重渲染，作为 <code>useSelector()</code> 默认行为。如果你想获得 store 中的多个值，你可以：

- 多次调用 <code>useSelector()</code>，每次都返回一个单独域的值

- 使用 Reselect 或类似的库来创建一个记忆化的 selector 函数，从而在一个对象中返回多个值，但是仅仅在其中一个值改变时才返回的新的对象。

- 使用 React-Redux <code>shallowEqual</code> 函数作为 <code>useSelector()</code> 的 <code>equalityFn</code> 参数，如：

```js
import { shallowEqual, useSelector } from 'react-redux'

// later
const selectedData = useSelector(selectorReturningObject, shallowEqual)
```

这个可选的比较函数参数使得我们可以使用 Lodash 的 <code>_.isEqual()</code> 或 Immutable.js 的比较功能。

## useSelector 例子

基本用法:

```js
import React from 'react'
import { useSelector } from 'react-redux'

export const CounterComponent = () => {
  const counter = useSelector(state => state.counter)
  return <div>{counter}</div>
}
```

通过闭包使用 props 来选择取回什么状态：

```js
import React from 'react'
import { useSelector } from 'react-redux'

export const TodoListItem = props => {
  const todo = useSelector(state => state.todos[props.id])
  return <div>{todo.text}</div>
}
```

### 使用记忆化的 selectors 函数

当像上方展示的那样，在使用 <code>useSelector</code> 时使用单行箭头函数，会导致在每次渲染期间都会创建一个新的 selector 函数。可以看出，这样的 selector 函数并没有维持任何的内部状态。但是，记忆化的 selectors 函数 (通过 <code>reselect</code> 库中 的 <code>createSelector</code> 创建) 含有内部状态，所以在使用它们时必须小心。

当一个 selector 函数依赖于某个 状态(state) 时，确保函数声明在组件之外，这样就不会导致相同的 selector 函数在每一次渲染时都被重复创建：

```js
import React from 'react'
import { useSelector } from 'react-redux'
import { createSelector } from 'reselect'

const selectNumOfDoneTodos = createSelector(
  state => state.todos,
  todos => todos.filter(todo => todo.isDone).length
)

export const DoneTodosCounter = () => {
  const NumOfDoneTodos = useSelector(selectNumOfDoneTodos)
  return <div>{NumOfDoneTodos}</div>
}

export const App = () => {
  return (
    <>
      <span>Number of done todos:</span>
      <DoneTodosCounter />
    </>
  )
}
```

这种做法同样适用于依赖组件 props 的情况，但是仅适用于单例的组件的形式

```js
import React from 'react'
import { useSelector } from 'react-redux'
import { createSelector } from 'reselect'

const selectNumOfTodosWithIsDoneValue = createSelector(
  state => state.todos,
  (_, isDone) => isDone,
  (todos, isDone) => todos.filter(todo => todo.isDone === isDone).length
)

export const TodoCounterForIsDoneValue = ({ isDone }) => {
  const NumOfTodosWithIsDoneValue = useSelector(state =>
    selectNumOfTodosWithIsDoneValue(state, isDone)
  )

  return <div>{NumOfTodosWithIsDoneValue}</div>
}

export const App = () => {
  return (
    <>
      <span>Number of done todos:</span>
      <TodoCounterForIsDoneValue isDone={true} />
    </>
  )
}
```

如果， 你想要在多个组件实例中使用相同的依赖组件 props 的 selector 函数，你必须确保每一个组件实例创建属于自己的 selector 函数([这里](https://github.com/reduxjs/reselect#accessing-react-props-in-selectors)解释了为什么这样做是必要的)

```js
import React, { useMemo } from 'react'
import { useSelector } from 'react-redux'
import { createSelector } from 'reselect'

const makeNumOfTodosWithIsDoneSelector = () =>
  createSelector(
    state => state.todos,
    (_, isDone) => isDone,
    (todos, isDone) => todos.filter(todo => todo.isDone === isDone).length
  )

export const TodoCounterForIsDoneValue = ({ isDone }) => {
  const selectNumOfTodosWithIsDone = useMemo(
    makeNumOfTodosWithIsDoneSelector,
    []
  )

  const numOfTodosWithIsDoneValue = useSelector(state =>
    selectNumOfTodosWithIsDone(state, isDone)
  )

  return <div>{numOfTodosWithIsDoneValue}</div>
}

export const App = () => {
  return (
    <>
      <span>Number of done todos:</span>
      <TodoCounterForIsDoneValue isDone={true} />
      <span>Number of unfinished todos:</span>
      <TodoCounterForIsDoneValue isDone={false} />
    </>
  )
}
```

# 被移除的：<code>useActions()</code>

<code>useActions()</code> 已经被移除

# <code>useDispatch()</code>

```js
const dispatch = useDispatch()
```

这个 hook 返回 Redux store 的 分发(dispatch) 函数的引用。你也许会使用来 分发(dispatch) 某些需要的 action。

```js
import React from 'react'
import { useDispatch } from 'react-redux'

export const CounterComponent = ({ value }) => {
  const dispatch = useDispatch()

  return (
    <div>
      <span>{value}</span>
      <button onClick={() => dispatch({ type: 'increment-counter' })}>
        Increment counter
      </button>
    </div>
  )
}
```

在将一个使用了 dispatch 函数的回调函数传递给子组件时，建议使用 useCallback 函数将回调函数记忆化，防止因为回调函数引用的变化导致不必要的渲染。

<small>

译者注：这里的建议其实和 dispatch 没关系，无论是否使用 dispatch，你都应该确保回调函数不会无故变化，然后导致不必要的重渲染。之所以和 dispatch 没关系，是因为，一旦 dispatch 变化，useCallback 会重新创建回调函数，回调函数的引用铁定发生了变化，然而导致不必要的重渲染。

</small>

```js
import React, { useCallback } from 'react'
import { useDispatch } from 'react-redux'

export const CounterComponent = ({ value }) => {
  const dispatch = useDispatch()
  const incrementCounter = useCallback(
    () => dispatch({ type: 'increment-counter' }),
    [dispatch]
  )

  return (
    <div>
      <span>{value}</span>
      <MyIncrementButton onIncrement={incrementCounter} />
    </div>
  )
}

export const MyIncrementButton = React.memo(({ onIncrement }) => (
  <button onClick={onIncrement}>Increment counter</button>
))
```

# <code>useStore()</code>

```js
const store = useStore()
```

这个 hook 返回传递给 <Provider> 组件的 Redux sotore 的引用。

这个 hook 也许不应该被经常使用。 你应该将 <code>useSelector()</code> 作为你的首选。但是，在一些不常见的场景下，你需要访问 store，这个还是有用的，比如替换 store 的 reducers。

This hook should probably not be used frequently. Prefer <code>useSelector()</code> as your primary choice. However, this may be useful for less common scenarios that do require access to the store, such as replacing reducers.

## 例子

```js
import React from 'react'
import { useStore } from 'react-redux'

export const CounterComponent = ({ value }) => {
  const store = useStore()

  // EXAMPLE ONLY! Do not do this in a real app.
  // The component will not automatically update if the store state changes
  return <div>{store.getState()}</div>
}
```

# 自定义 context

<code>&lt;Provider&gt; </code> 组件允许你通过 <code>context</code> 参数指定一个可选的 context。在你构建复杂的可复用的组件时，你不想让你自己的私人 store 与使用这个组件的用户的 Redux store 发生冲突，这个功能是很有用的，

通过使用 hook creator 函数来创建自定义 hook，从而访问可选的 context。

```
import React from 'react'
import {
  Provider,
  createStoreHook,
  createDispatchHook,
  createSelectorHook
} from 'react-redux'

const MyContext = React.createContext(null)

// Export your custom hooks if you wish to use them in other files.
export const useStore = createStoreHook(MyContext)
export const useDispatch = createDispatchHook(MyContext)
export const useSelector = createSelectorHook(MyContext)

const myStore = createStore(rootReducer)

export function MyProvider({ children }) {
  return (
    <Provider context={MyContext} store={myStore}>
      {children}
    </Provider>
  )
}
```

# 使用警告

## 过期 Props 和  "丧尸子组件" 

有关 React Redux 实现一个难点在于，当你以 <code>(state, ownProps)</code> 形式定义 <code>mapStateToProps</code> 函数时，怎么保证每次都以最新的 props 调用 <code>mapStateToProps</code>。version 4 中，在一些边缘情况下，经常发生一些bug，比如一个列表中的某项被删除时， <code>mapState</code> 函数内部会抛出错误。

从 version 5 开始，React Redux 试图保证 <code>ownProps</code> 参数的一致性。在 version 7 中，通过在  <code>connect()</code>  内部使用一个自定义的 <code>Subscription</code> 类，实现了这种保证，也导致了组件被层层嵌套的形式。这确保了组件树深处  <code>connect()</code>  后的组件，只会在离自己最近的  <code>connect()</code>  后的祖先组件更新后，才会被通知 store 更新了。但是，这依赖于每个 <code> connect(</code>) 的实例副高 React 内部部分的 context，随后  <code>connect()</code>  提供了自己独特的 <code>Subscription</code> 实例，将组件嵌套其中，提供一个新的 conext 值给 <code>&lt;ReactReduxContext.Provider&gt;</code>，再进行渲染。

使用 hooks，意味着无法渲染 &lt;ReactReduxContext.Provider&gt;，也意味着没有嵌套的订阅层级。因此，“过期 Props” 和  "丧尸子组件" 的问题可能再次发生在你使用 hooks 而非  <code>connect()</code>  应用中。

详细的说，“过期 Props”可能发生的状况在于：

- 某个 selector  函数依赖组件的 props 来取回数据。
- 在某个 action 分发后，父组件将会重渲染然后传递新的props给子组件
- 但是子组件的 selector 函数在子组件以新props渲染前，先执行了。

取决于使用的 props 和 stroe 当前的 状态(state) 是什么，这可能导致返回不正确的数据，甚至抛出一个错误。

"丧尸子组件" 特别指代下面这种情况：

- 在刚开始，多个嵌套  <code>connect()</code>  后的组件一起被挂载，导致子组件的订阅先于其父组件。

- 一个 action 被 分发(dispatch) ，删除了 store 中的某个数据，比如某个待做事项。

- 父组件会停止渲染对应的子组件

- 但是，因为子组件的订阅先于父组件，其订阅时的回调函数的运行先于父组件停止渲染子组件。当子组件根据props取回对应的数据时，这个数据已经不存在了，而且，如果取回数据代码的逻辑不够小心的话，可能会导致一个错误被抛出。

<code>useSelector()</code> 通过捕获所有 selector 内部因为 store 更新抛出的错误（但不包括渲染时更新导致的错误），来应对"丧尸子组件"的问题。当产生了一个错误时，组件会被强制重渲染，此时，selector 函数会重新执行一次。注意，只有当你的 selector 函数是纯函数且你的代码不依赖于 selector 抛出的某些自定义错误时，这个应对策略才会正常工作。

如果你更想要自己处理这些问题，这里有一些建议，在使用 <code>useSelector()</code> 时，可能帮助你避免这些问题。

- 在 selector 函数不要依赖 props 来取回数据。

- 对于你必须要依赖props，而且props经常改变的情况，以及，你取回的数据可能被删除的情况下，试着带有防御性的 selector 函数。不要直接取回数据，如：<code>state.todos[props.id].name</code> - 先取回 <code>state.todos[props.id]</code>，然后检验值是否存在，再尝试取回 todo.name

- 因为 connect 增添了必要 <code>Subscription</code> 组件给 context provider，且延迟子组件订阅的执行，一直到  <code>connect()</code>  的组件重渲染后，在组件树中，将一个  <code>connect()</code>  的组件置于使用了 <code>useSelector</code> 的组件之上，将会避免上述的问题，只要  <code>connect()</code>  的组件和使用了 hooks 子组件触发重渲染是由同一个 store 更新引起的。

> **注意**：如果你想要这个问题更详细的描述，[这个聊天记录](https://gist.github.com/markerikson/faac6ae4aca7b82a058e13216a7888ec)详述了这个问题，以及 [issue #1179](https://github.com/reduxjs/react-redux/issues/1179).


## 性能

正如上文提到的，在一个 action 被分发(dispatch) 后，<code>useSelector()</code> 默认对 select 函数的返回值进行引用比较 ===，并且仅在返回值改变时触发重渲染。但是，不同于 <code> connect(</code>)，<code>useSelector()</code>并不会阻止父组件重渲染导致的子组件重渲染的行为，即使组件的 props 没有发生改变。

如果你想要类似的更进一步的优化，你也许需要考虑将你的函数组件包裹在 React.memo() 中：

```js
const CounterComponent = ({ name }) => {
  const counter = useSelector(state => state.counter)
  return (
    <div>
      {name}: {counter}
    </div>
  )
}

export const MemoizedCounterComponent = React.memo(CounterComponent)
```

# Hooks 配方

我们精简了原来 alpha 版本的 hooks API，专注于更精小的，更基础的 API。不过，在你的应用中，你可能依旧想要使用一些我们以前实现过的方法。下面例子中的代码已经准备好被复制到你的代码库中使用了。

## 配方：<code>useActions()</code>

这个 hook 存在于原来 alpha 版本，但是在版本 v7.1.0-alpha.4 中，[Dan Abramov](https://github.com/reduxjs/react-redux/issues/1252#issuecomment-488160930) 的建议下被移除了。建议表明了在使用 hook 的场景下，“对 action creators 进行绑定”没以前那么有用，且会导致更多概念上理解负担和增加语法上的复杂度。

<small>译者注：**action creators** 即用来生成 action 对象的函数。</small>

在组件中，你应该更偏向于使用 useDispatch hook 来获得 dispatch 函数的引用，然后在回调函数中手动的调用 dispatch(someActionCreator()) 或某种需要的副作用。在你的代码中，你仍然可以使用[bindActionCreators](https://redux.js.org/api/bindactioncreators) 函数绑定 action creators，或手动的绑定它们，比如 const boundAddTodo = (text) => dispatch(addTodo(text))。

但是，如果你自己想要使用这个 hook，这里有个 复制即可用 的版本，支持将 action creators 作为一个独立函数、数组、或一个对象传入。

```js
import { bindActionCreators } from 'redux'
import { useDispatch } from 'react-redux'
import { useMemo } from 'react'

export function useActions(actions, deps) {
  const dispatch = useDispatch()
  return useMemo(() => {
    if (Array.isArray(actions)) {
      return actions.map(a => bindActionCreators(a, dispatch))
    }
    return bindActionCreators(actions, dispatch)
  }, deps ? [dispatch, ...deps] : [dispatch])
}
```

## 配方：<code>useShallowEqualSelector()</code>

```js
import { useSelector, shallowEqual } from 'react-redux'

export function useShallowEqualSelector(selector) {
  return useSelector(selector, shallowEqual)
}
```
