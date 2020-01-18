---
title: 一道ts类型编程题的解答
date: 2019-12-05 12:30:03
tags:
  - ts
---

# 一道 TypeScript 题

## 问题定义

假设有一个叫 `EffectModule` 的类

```ts
class EffectModule {}
```

这个对象上的方法**只可能**有两种类型签名:

```ts
interface Action<T> {
  payload?: T
  type: string
}

asyncMethod<T, U>(input: Promise<T>): Promise<Action<U>>

syncMethod<T, U>(action: Action<T>): Action<U>
```

这个对象上还可能有一些任意的**非函数属性**：

```ts
interface Action<T> {
  payload?: T;
  type: string;
}

class EffectModule {
  count = 1;
  message = "hello!";

  delay(input: Promise<number>) {
    return input.then(i => ({
      payload: `hello ${i}!`,
      type: 'delay'
    });
  }

  setMessage(action: Action<Date>) {
    return {
      payload: action.payload!.getMilliseconds(),
      type: "set-message"
    };
  }
}
```



现在有一个叫 `connect` 的函数，它接受 EffectModule 实例，将它变成另一个一个对象，这个对象上只有**EffectModule 的同名方法**，但是方法的类型签名被改变了:

```ts
asyncMethod<T, U>(input: Promise<T>): Promise<Action<U>>  变成了
asyncMethod<T, U>(input: T): Action<U> 
```

```ts
syncMethod<T, U>(action: Action<T>): Action<U>  变成了
syncMethod<T, U>(action: T): Action<U>
```



例子:

EffectModule 定义如下:

```ts
interface Action<T> {
  payload?: T;
  type: string;
}

class EffectModule {
  count = 1;
  message = "hello!";

  delay(input: Promise<number>) {
    return input.then(i => ({
      payload: `hello ${i}!`,
      type: 'delay'
    });
  }

  setMessage(action: Action<Date>) {
    return {
      payload: action.payload!.getMilliseconds(),
      type: "set-message"
    };
  }
}
```

connect 之后:

```ts
type Connected = {
  delay(input: number): Action<string>
  setMessage(action: Date): Action<number>
}
const effectModule = new EffectModule()
const connected: Connected = connect(effectModule)
```

## 要求

在 [题目链接](https://codesandbox.io/s/o4wwpzyzkq) 里面的 `index.ts` 文件中，有一个 `type Connect = (module: EffectModule) => any`，将 `any` 替换成题目的解答，让编译能够顺利通过，并且 `index.ts` 中 `connected` 的类型与:

```typescript
type Connected = {
  delay(input: number): Action<string>;
  setMessage(action: Date): Action<number>;
}
```

# 解答

```ts

type OriAsyncMethod<T, U> = (input: Promise<T>) => Promise<Action<U>>;
type AsyncMethod<T, U> = (input: T) => Action<U>;

type OriSyncMethod<T, U> = (action: Action<T>) => Action<U>;
type SyncMethod<T, U> = (action: T) => Action<U>;

// 首先根据题目写出对应的类型

type PickFuncKeys<T> = {
  [K in keyof T]: T[K] extends Function ? K : never;
}[keyof T];

type FuncKeys = PickFuncKeys<EffectModule>;

// 取出值为函数的键名，这一步是必须的，如果直接对处理原对象所有的键
// 会得到额外的属性，也就是带上
// count: never;
// age: never;
// 的结果，是无法通过编译的
// 而题目要求结果仅仅具有方法，不具有属性，所以带有额外属性也是错误的，必须仅处理值为函数的键，忽略其他的键名
// type Connected = { 题目要求
//   delay(input: number): Action<string>;
//   setMessage(action: Date): Action<number>;
// };


type Connect = (module: EffectModule) => { // 替换结果
  [K in FuncKeys]: EffectModule[K] extends OriAsyncMethod<infer A, infer B>
  ? AsyncMethod<A, B>
  : EffectModule[K] extends OriSyncMethod<infer A, infer B>
  ? SyncMethod<A, B>
  : never;
}
```