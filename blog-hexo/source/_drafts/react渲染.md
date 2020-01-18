---
title: react暇谈系列一——当我们再谈论render时，我们在谈论什么
date: 2019-10-02 19:00:03
tags:
  - react
  - 前端
  - react暇谈
---

# 术语定义

- **render 函数**，本文中指类组件的 render 方法，或者函数组件本身。
- **render**、**渲染**，我会在文中交替的使用这两个词语，但你应该明白，你随时可以把文中的 render 换成 渲染，反之亦然。

# 主旨

本文想要讨论的主题有三个：
- react 的 render 是指什么？
- 组件重渲染的原因
- 如何对render进行优化，以及原理

# react的render到底指什么？

如果你已经接触了 React 一段时间，你经常会看到 render(渲染)，rerender(重渲染)，等各种与渲染相关的字眼，当我作为一个菜鸟时，我时常为这些字眼发愁，尽管我知道这些词语的本意，但是却不明白它们在 React 中代表什么。现在，经过一定量的学习，我觉得自己终于可以看清一部分，借此分享出来，以期帮助别人，同时也是借此梳理下自己掌握的知识。

## 三个渲染阶段

为了理解 render 在 React 中代表什么，我们需要稍微深入下 React 的工作原理，也许你已经知道虚拟DOM，Diff，等各种新奇的词语，不过还是让我们按顺序梳理以下。

React 进行渲染的 **时机** 有两种(其实也可以算一种，其中一种只是另一种的变体)：

- 挂载，React 调用类组件 DidComponentMount 生命周期方法时，或者函数组件的useEffect中回调函数第一次执行的时候，可以认为已经发生了挂载。

- 更新，React 调用类组件的 DidComponentUpdate 的时候，或者函数组件被重新执行时，都可以认为是发生了重渲染。

我们首先来看组件更新的情况，然后再回头看挂载的情况，你会发现，后者和前者的差别其实很小，这也是为什么 React 在函数组件里提供的 useEffect hook 不再区分挂载和更新。

让我们使用经典的计数器例子

```jsx
// 警告：为了代码的精简的，我省略了 useCallback 包裹回调函数的行为
function Counter() {
    const [count, setCount] = useState();

    return (
        <div>
            <span className="count">{count}</span>
            <button onClick={() => setCount(prev => prev + 1)}>INC<button>
        </div>
    );
}
```

在 Counter 挂载后，当你点击了 **INC** 按钮时，就会触发一次 React 的组件的**更新** 或者说是**重渲染**。


这里为了讲解的方便，我自作主张的将 React 的一个完整的渲染周期的分为三个阶段，注意不要联想到生命周期，这里没有任何关系。

三个阶段是：

1. **渲染函数阶段** 或 **render函数阶段**，这个阶段会执行render函数，生成虚拟DOM。

2. **tree diff 阶段**，这个阶段 React 会对比新旧的的虚拟DOM，看看是否有变化发生。

3. **commit 阶段**，**如果确实发生了变化，才会进入这个阶段**，这个阶段会将虚拟DOM中发生的变化同步到真实DOM上。

一般来说，前两个阶段可以视为一个阶段，因为它们都发生在虚拟DOM层面，并且一旦触发渲染函数阶段，则一定会触发 tree diff 阶段，但是 commit 阶段只在虚拟DOM发生变化的时才可以被触发。

在了解组件更新的的流程后，我们对比着更新的流程，回过头来看组件挂载的情况：

- 挂载有虚拟函数阶段吗？

答案: 肯定有！必须要执行 render 函数，生成虚拟DOM。

2. 需要tree diff吗？

答案：这里视场景有不同的答案，不严谨的说，ReactDOM.render() 是没有这个阶段的，因为首次执行还不存在可以进行对比的旧虚拟DOM，会直接跳到 commit 阶段。而其他挂载的情况，被挂载的组件自身也是不会进行 tree diff 的。

这里为了避免误解多说一点，父组件更新导致某个子组件被挂载或卸载，子组件挂载时自身是不会进行 tree diff 的，视情况会直接跳到 commit 阶段或就此结束。但是父组件进行 tree diff 时会用到这个子组件。你可以认为 null 指是一个特殊的虚拟DOM，React 将它作为占位符使用，在进行 tree diff 的时候也有相关的作用。


3. commit阶段

答案：这点毋庸置疑。

## 结论

现在，让我们通过确定 “这个组件发生了渲染/重渲染/无用的重渲染” 是什么意思来解答本小节的问题。

对于 **...发生了渲染/重渲染** 来说都是指阶段123，也就是整个渲染阶段。生成变化后的虚拟DOM，进行 tree diff，将变化 commit 到真实 DOM 上。如果不是在指明讨论虚拟 DOM 的话，将渲染一词局限1、2阶段时没有意义的。

而对于 **发生了无效的重渲染** 来说，是指阶段1、2。一般发生在父组件发生了更新，子组件的props却没有变化，所以子组件生成的虚拟dom没有发生改变，在 tree diff 阶段后直接断掉了，没有进入 commit 阶段。

以后再碰到与 React 渲染相关的问题，可以直接用上文讲解的三个阶段往上套，通过理解上下文，应该可以很容易理解作者想要的强调有关渲染的什么方面。

# 组件 rerender 的原因

这一节我们讨论组件重渲染的原因，至于为什么不讨论渲染的原因，我想这是不自言明的。同时也是为了下一节，讲解组件渲染优化，打下基础。

现在我们列举下组件可能重渲染的原因：

- 组件自身触发重渲染

一般发生在：
- - 类组件setState

- - 函数组件使用 useState Hook

- 父组件导致自身重渲染

- - 这是很常见的情况，实际上大部分 React 组件渲染优化都发生在这里。

- 子孙组件导致自身重渲染

- - 有些人可能对这点有点疑问，这里讲的触发是指父组件将会改变自身状态的回调函数传递给子或组件，一旦被执行，则自身重渲染，严格归类起来一类更新应该属于第一类，不过这里故意区分出来，算是一点咬文嚼字的游戏吧。


这里没有讲 Redux 更新和 Context API 相关的更新，这两个更新看起来特殊，其实也被包含在上面三种情况下。

接下来我们详细讲解下这三种描述的原因：


关于类组件使用setState， 你需要知道：

- setState的参数如果不是null, 即使是个空的object，也会触发组件重渲染，当然这种一般是无效的重渲染。

- 函数组件的 useState 的行为和类组件的显然有区别的，由于 useState 的粒度很小，一般是对单个值进行储存，所以 useState 没有setState(null) 这种不触发重渲染的设置函数，如果你使用 useState 的 setState(null) 只会把值设置成null。所以 React 在useState 做了一点其他的工作，React 的会将新 setStarte 的值和老旧的值就行比较，如果没有发生改变，则不会触发执行渲染函数，与类组件无论如何都会触发渲染函数相比，这算的上是一种优化， 需要注意的是，React 使用引用比较来比对新旧的值，所以如果你使用 useState 来储存对象的话，要注意引用改变会引起重渲染。

# 组件渲染优化

因为一旦一个组件发生了变化，那么其下的所有组件都会被重新渲染的，这是 React 更新的一个原则。所以经常改变处于组件树上层的组件的做法是不被推荐，会导致大量的(无效)重渲染。但是这里需要注意的是，尽管会触发重渲染，但是这些重渲染只停留在虚拟DOM阶段，在经过tree diff 后，React 发现没有改变，不会触摸到真实DOM层面，而 JS 本身是很快的，所以不经优化的 React 程序的速度并没有慢到让人有明显的感知。但为什么 React 的性能依然会出现问题呢？很简单，假如在计数器组件下有一个组件渲染一个1000项的列表，如果不做任何优化，那么每次计数器发生更新，这个列表也会变化，尽管dom没有发生变化，但是每次都会跑一次循环，创建1000个js对象，这同样是不小的开销，再加上tree diff，所以开销依然存在，甚至不少。

一般引起更新的组件自身的重渲染的是必要的，但是非自身引起的重渲染往往就不那么必要。对于 React 来说，一旦组件被重渲染，那么子孙节点间也会被被重新渲染，试着这一个例子

```jsx
function List(props) {
    const {
        items = [],
    } = props;
    return (
        <ul>
            {items.map(item => (
                <li key={item.id}>{item.title}</li>
            ))}
        </ul>
    );
}
function Counter() {
    const { items } = props;
    const [count, setCount] = useState(0);
    reuturn (
        <div>
            <h1>{count}</h1>
            <button onClick={() => setCount(prevCount => prevCount + 1)}>INC</buttonr>
            <List items={items}>
        <div>
    );
}
```

试下看下这个例子，如果不优化，那么 INC 按钮被点击，Counter 组件都会被重渲染，进而引起的 List 的重渲染，但是 items 没有变化，所以 React 在执行渲染函数后，对前后生成的虚拟DOM执行对比，然后中止在 tree diff 阶段，虽然没有进入到 commit 阶段，但是如果items 的长度成百上千的呢，每一次点击都会引起大量的无效渲染，大量无用的js对象被创建，然后销毁，并且因此，React 应用的程序性能也会受到影响，直观的表现出来就是很卡。

React 提供给我们优化组件渲染的方式有三种：

1. 类组件：继承 React.PureComponent
2. 类组件：实现 shouldComponentUpdate 方法
3. 函数组件：React.memo 包裹函数组件

其中前两者的优化方式其实是一样的，shoudComponentUpdate 是 React 提供的一种阻止渲染的方式，在组件挂载后，此后每一次渲染之前 React 都会先调用类组件实例的 shoudComponentUpdate 方法，如果此方法返回 true 则继续渲染，如果是 false，则会停止渲染。表现的行为是，对于类组件，不会执行其 render 方法，对于函数组件，不会执行其本身。

shoudComponentUpdate 方法的参数是新旧的 props，所以我们可以在 shoudComponentUpdate 通过对比新旧 props 是否不同，来决定是否更新的组件。不过，还记着的那句话吗？过早优化是万恶之源。优化本身也是需要消耗性能，过于复杂 shoudComponentUpdate 实现逻辑一样会拖慢的性能。

React 官方的给出的解决方案是浅比较，这算是一种在性能上和命中率相当平衡的选择。通过对比前后 props 对象 key 的数量，key 的名字， key 对应的属性，来判断前后props是否一样，这里之所以叫浅比较，是因为props作为一个对象，这种比较方法只对比对象第一层的属性和值，如果某个属性的值也是一个对象，那么只会比它们的引用，不会比较其内容。

对于类组件来说，凡是继承了 React.PureComponent 的组件，可以认为 React 帮我们实现了浅比较版本的 shouldComponentUpdate 方法。这里稍微提一下，如果你去看 React 源码的话，你会发现在 React.PureComponent 的相关的源码里，并没有 shouldComponentUpdate 实现的细节，仅仅有一个标志的，标志这一个组件是 React.PureComponent 组件，而真正实现的细节在 ReactDOM，当 React 更新时，会先判断一个组件是否是 PureComponent，如果是的话，则会传入浅比较函数作为 shouldComponentUpdate 的逻辑。

当然我们也能自己实现 shouldComponentUpdate 相关的逻辑，不过在我的日常经验中，PureComponent 自带的浅比较实现已经足够了。

React.memo 和 shouldComponentUpdate 的作用是一样的，是 React 提供给函数组件优化的一种方法，memo 函数在函数式编程里是很通常的概念，是一种缓存的手段，对于纯函数来说，相同的参数总是导致相同的输出，所以只要前后的props没变就直接返回上次缓存的结果。React.memo 函数的使用也很简单，第一个参数接受一个函数组件，第二参数解释一个比较函数，可以类比的认为是 shoudComponentUpdate 的实现，如果留空的，React 则会默认使用浅比较来作比较。

接触过函数式的人，可能会误以为 React.memo 是 React 官方实现的函数式编程的 memo 高阶函数，作用是参数输入不变时返回缓存函数的结果，但并不是。React.memo 的返回值是一个特殊的 React 对象。

