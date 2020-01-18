---
title: Typescript类型编程-官方高级类型实现解析
date: 2020-101-04 12:30:03
tags:
  - react
  - 前端
  - react暇谈
  - ts
---

本文旨在讲解如何在 TS 中进行类型编程，通过实现 TS 官方提供的类型，并在实现过程中讲解其实现原理，最终掌握类型编程的基本语法，能够实现常见的类型。

注意：本文并非 TS 入门文章，只会讲解 TS 中类型编程相关的知识，因此阅读本文要求你有一定的 TS 使用经验，至少已经阅读完 TS 官方的文档，有过一定的使用经验

# 正文

## Partial\<T\>

```ts
// 目标：将所有属性都变成可读属性
interface People {
  name: string;
  age: number;
}

type foo = MyPartial<People>;

// foo 应该的结果
interface People {
  name?: string;
  age?: number;
}
```

<details>

<summary>
点击查看实现
</summary>

```ts
type MyPartial<OBJ extends object> = {
  [KEY in keyof OBJ]?: OBJ[KEY];
};

// 官方实现

type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

</details>

### 讲解

#### \<T extends object\>

我们首先解释下 \<T extends object\> 什么意思，**extends** 关键字在此的意思是 **受限为**，也就是说类型 T 受限为一个 object 类型，其作用是限制形参 T 的类型，除非特殊情况，我们可以认为 传入的类型 T 必须是 object 的子集，因此传入 string、number 等类型就会报错，特殊情况指传 any、never 等类型时不会报错。官方实现没有使用 extends，因此如果传入 string、number 等类型不会报错。

<!-- 也可以这样认为，对于一个类型为T的 **变量** 其 **值** 可以赋给类型为 **object** 的变量。 -->

```ts
type MyPartial<OBJ extends object> = {
  [KEY in keyof OBJ]?: OBJ[KEY];
};

type foo = MyPartial<string>; //报错
type foo = MyPartial<number>; //报错
type foo = MyPartial<'literal'>; //报错
type foo = MyPartial<'123'>; //报错
```

#### \[KEY in keyof OBJ\]

- **keyof** 被称为 **索引类型查询操作符**，用来获取 OBJ 类型上所有为 public 的属性名，然后形成联合类型

```ts
type keys = keyof People { // => "name" | "age"
  name: string;
  age: number;
}

// 可以认为是

type keys = "name" | "age";

```

**in** 关键字类似于 JS 中的 for ... of 或 for ... in，然后用 **KEY** 表示属性名

```js
for (let KEY of ['name', 'age']) {}
```

#### \?

最后我们在每一个属性上加上?，将原来的属性变为可选属性

```ts
//    [KEY in keyof T]: T[KEY];
// => [KEY in keyof T]?: T[KEY];
```

## Readonly\<T\>

```ts
// 目标：将所有属性都变成只读属性
interface People {
  name: string;
  age: number;
}

type foo = MyReadonly<People>;

// foo 应该的结果
interface People {
  readonly name: string;
  readonly age: number;
}
```

Readonly\<T\> 的实现和 Partial 差不多，没什么讲的地方，当作练习题吧


<details>

<summary>
点击查看答案
</summary>

```ts
type MyReadonly<OBJ extends object> = {
  readonly [KEY in keyof OBJ]: OBJ[KEY];
};
```

</details>


## 