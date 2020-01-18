---
title: react暇谈系列四——React 触发更新的轨迹
date: 2019-11-04 13:20:42
tags:
  - react
  - 前端
  - react暇谈
---

把大部分 React 源码看了几遍，复杂是真的复杂，函数调用链很长，其中还嵌套着一些递归和间接递归调用，不过在借助网上各种解读文章和实际试验后，终于明白了大部分代码的含义，对 React 底层也有了一些比较清晰的认识。这里记录下看源码中，感到印象深刻的感想，没看之前有很多想当然的想法，看过以后发现大相径庭。

作为一个初学者时，我很自然的认为，哪个组件 **产生** 更新，就从那个组件 **开始** 更新。我以前对于 React 的心智模型：

- 哪个组件 **产生** 更新，就从那个组件 **开始** 更新
- 对于更新的组件，更新其所有子孙组件，即使前后 props 和 state 没有改变
- 判断有无 shouldComponentUpdate，对于返回 false 的，跳过其和子树的更新

在阅读源码后，我才发现原来 React 更新的实际模型和我的心智模型大不一样。

# React 触发更新的轨迹

```javascript
const REACT_TREE = (
  <FiberRoot> // => RootFiber 为了方便理解，对实际情况作了简化，可以认为这是 React 使用一个内部组件，用于一些特殊行为
    <App>
      <Content>
        <List>
          <ListItem />
        </List>
        <Counter />
      </Content>
    </App>
  </FiberRoot>
);
```
我们假设 Counter 组件的 state 发生了变化，React 更新的轨迹会从 Counter 开始，一路向上，在向上的过程中更改对应 fiber 节点的 expirationTime 和 childExpirationTime ，然后把 FiberRoot 加入根节点调度队列，开始从 FiberRoot 节点往下更新，此时 React 会对比 RootFiber state 上前后 children (也就是 ReactDOM.render 时传入 App React元素) 是否一样来决定是跳过更新还是重新调和子节点。

一般来说，除非我们对同一个 DOM 节点，重复调用 ReactDOM.render 方法，否则 FiberRoot 的这次更新肯定是会被跳过的。React 更新行为遵循以下规则：

1. 检测自身的 childExpirationTime，也就是子树的 expirationTime，判断子树是否产生更新
2. 如果产生更新，则重新调和子节点
3. 如果没有则跳过整个子树，处理自身的兄弟节点
4. 如果没有兄弟节点，则更新结束

根据以上规则，来过一遍 Counter 组件的 state 发生变化的更新流程。

- Counter 触发更新

- 一路向上更新祖先节点的 childExpirationTime，注意兄弟节点 List 上的 childExpirationTime 没有变化

- React 从 FiberRoot 开始向下更新

- FiberRoot 没有变化，跳过自身的更新，但是根据自身 childExpirationTime 判断，发现子节点有更新产生

- 开始处理 App 节点，假设 App 也是个类组件，React 在默认情况使用 === 对比其前后 props 和 state，如果没发生变化，则跳过自身的更新。作为类组件，如果实现了 shouldComponentUpdate 和 继承了 PureComponent 会调用相应方法比较，而不是使用 ===。这里提一下 React.memo 和 PureComponent 的实现区别很大，会另开一个小节讲解

- 此时来到 Content 组件，注意 Content 有两个子节点，触发更新 Counter 组件是第二个，根据刚才讲的规则，Content 组件也没发生更新，但是知道有子组件更新了，**但是并不知道是哪一个！**，因为 childExpirationTime 并没有记录是对应着那个子组件

- React 先处理 List 组件，发现没有更新以后，因为其子组件也没有更新，整个子树被全部跳过，然后处理其兄弟节点

- 来到 Counter 组件，发生前后 state 不一样，生成新的 state，打上对应的 effectTag，处理完成后，尝试处理兄弟节点，但是没有，于是结束更新

值得注意的点：

- 为了方便描述理解，对于实际更新规则作了些许简化

- 可以认为 React 更新的轨迹是一个倒U，从下到上，再从上到下

- 不论任何层级产生的更新，都是从根节点 FiberRoot 往下找

- 这个阶段，被 React 称为 render 阶段，注意不要认为是 render 到屏幕上！此时所有的操作都是虚拟 DOM 层面上的变化，不过这样说不严谨，因为根据不同的更新，React 会创建不同 DOM 节点，只是没有 commit 到屏幕上而已。应该说，到此时，对于已经显示出来的 DOM 节点来说，没有发生任何变化，只有等到 commit 操作后，才会发生实际的变化

- React 会把需要更新组件对应的 fiber 节点挂载到 RootFiber 上，等到 commit 阶段，可以直接 commit 发生变化的组件，不会发生二次寻找

