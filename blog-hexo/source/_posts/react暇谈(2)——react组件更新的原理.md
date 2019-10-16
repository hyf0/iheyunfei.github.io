---
title: react暇谈(2)——react组件更新的原理
date: 2019-10-16 14:44:15
tags:
- react
- 前端
- react暇谈
---

# 主旨

这次本文旨在讨论：

- react 组件更新的原理

# 术语定义

- 组件更新：组件进行了重渲染，其渲染阶段至少经过了 render 函数和 tree diff 阶段。

# react 组件更新的原理

首先，我们先阐述下 react 组件更新的策略：**一个组件发生了更新，其子组件也会更新，在不进行优化的情况下，这种更新的传递会持续到组件树的叶子组件，即使某个子组件前后props并没有发生改变**。

然后，让我们确定下什么可以引起组件更新(针对react 16)：

- useState/useReducer hook
- 类组件 this.setState
- 父组件更新(props 变化)

## 1. useState/useReducer hook

由于 useReducer 的 api 过于繁琐，为了叙述的方便，这里使用 useState 举例子。

这一节中，我使用 **setState** 指代 useState hook 返回的 setState 函数，而 **this.setState** 指代类组件的 setState 函数。

关于 useState hook 需要注意的几点：

- hooks 给了函数组件和类组件一样保存状态的能力，自然而然的，既然可以保存状态就应该可以更改状态，useState 返回的 setState 可以用于改变相关的状态，自此函数组件和类组件一样，同样具有了主动发起更新的能力。

- react 官方认为 useState hook 应该用于更加细粒度的值，所以 useState 和基本的值类型数据配合良好，但是如果牵扯 object 类型，就需要作出一些额外的考虑：
    - useState hook 的 setState 和 this.setState 的行为并不一样，其中很重要的一点，对于 this.setState，会对前后的 state 进行合并，所以当我们 this.setState 时，只要写上更改的数据即可，而 useState hook 的 setState 则是替换。所以，当你使用 useState 来保存 object 时，你可能会需要写上大量的 **...** 运算符。

- setState 和 this.setState 阻止更新的行为不一样。不同于 this.setState(null)，setState(null) 依旧会触发更新，并且会把 state 赋值成 null。当你使用 setState 时，react 会将前后的 state 进行 === 比较，如果一样，则不会触发更新。所以如果你使用 useState 保存 object，即使新的 object 中的值没有发生变化，但是由于生成了新的引用，依旧会触发更新。这也是为什么尽量不要使用 useState 来储存 object，对于复杂的本地状态，建议使用 useReducer 来管理。

## 2. 类组件 this.setState

关于 this.setState 需要的注意的是：

- 设置空对象 this.setState({}) 依旧会触发更新 

- this.setState(null) 不会触发更新

- this.setState 同步异步问题：

    - 在事件处理函数，或生命周期函数，react 会对 this.setState 作 batching 优化，此时 this.setState 是异步的。

    - 其他时候，例如 setTimeout 或 promise 中，this.setState 是同步的。


## 3. 父组件更新(props 变化)

这是我想咬文嚼字的一节，在阅读 react 相关文章时，经常会看到 **props变化，导致组件发生了更新**，这听起来彷佛 react 在更新时会对比前后的 props，只有发生了变化时才会更新组件。又或者有过 vue 经验的新人，可能认为 react 对 props 对象进行了监听，所以发生变化时才会更新，然而，以上两种解释都是错误的。

```jsx
function Box(props) {
    const { count } = props;
    return <div>{count}</div>;
}

function Counter() {
    const [count, setCount] = useState(0);

    const [useLessCount, setUseLessCount] = useState(0);

    return (
        <div>
            <Box count={count} />
            <button onClick={() => setCount(prev => prev + 1)}>INC</button>
            <button onClick={() => setUseLessCount(prev => prev + 1)}>useLess</button>
        </div>
    );
}
```

当 INC 按钮被点击时，Box 组件显示的数值加了 1，让我们走一遍这个 **更新** 的流程，看看 props 变化是否导致了组件 **更新**

流程：

- count = 0
- INC 按钮被点击
- Counter 发生更新，React 执行 Counter 函数生成虚拟DOM
- Counter 函数执行中，count 通过 useState 拿到改变后的新值，此时 count = 1
- React 通过 createElement 创建 Box 组件的虚拟DOM，注意此时的 Box 的 props 中 count 被赋值为 1
- React 以更新后的的 props 调用 Box 函数，生成其虚拟 DOM
- Box 组件的虚拟 DOM 进行 tree diff，发现 count 值改变
- 最后 commit 到真实 DOM 上

纵观整个流程，react 一直遵循着一个简单的策略 **组件更新，则更新其所有的子组件**，且不断递归的运用这个策略，直到叶子节点。和 props 本身没有任何关系。也就是说，**子组件的更新是由于父组件更新引起的，而非 props 的变化**

另一个更直接的证据是，点击 useLess 按钮，同样会导致 Box 组件更新，即使 count 的值并没有发生改变。

**props 变化，导致组件发生了更新**，犯了一个典型的因果谬误，A 先发生，然后 B 发生了，所以 A 导致了 B 发生。

这句话的主要问题在于，它会给初学者建立一种错误的心智模型，认为其中有什么 fancy 的 magic 发生，然而本质上只是遵循一个简单更新策略的自然而然的结果。

最后，任何时候，**props 变化，导致组件发生了更新** 这句话都是错误的，但是对于本小节中的例子来说：**props 变化，导致组件内容发生了变化** 是正确的。

既然都谈了这么多，我就多说一点，人在交流时，倘若不对一些模棱两可的词语的意思作出规定，非常容易变成大家自说自话，我不同意你的，你不同意我的，谁也说服了不了谁。我本以为这是技术讨论的常识，然而却发现知乎上每天都在发生这样的对话，很少发现充满价值的讨论。
