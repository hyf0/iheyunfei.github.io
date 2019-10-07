---
title: 浅谈可取消的Promise
date: 2019-09-20 08:30:56
tags:
- js
- 编程
- 前端
---

最近再尝试实现CSSTransition组件，其中有个需求是每次in属性的变化会导致className的变化，为了增加对应的效果，必须保证不同的className在一段时间范围内，按照特顺序进行显示。

我尝试使用promise来控制顺序，效果很不错，几乎解决了问题，但是，由于用户行为的不确定性，比如疯狂点击toggle按钮，in属性的可能在短时间内大量变化，从而触发大量的回调函数，使得className的变化顺序变得相当混乱。

于是乎，自然而然的，每次触发回调前，要先取消掉先前可能存在的回调函数，这个需求有点像多tab触发网络请求的场景，每次点击tab都会触发一个网络请求，为了不让界面显示老旧的数据(因为异步的原因，新旧请求返回数据的时间顺序是不确定的)，必须取消掉先前的请求。

此外，另一个难点在于链式的回调调用，你必须保证在取消后，不论回调链执行到哪里，都不会再被执行了。

- 没考虑取消的版本

```ts
if (isIn) { // 每次点击，判断变化的isIn属性，触发相应的回调
    setClassName(`${initClassNameRef.current} enter`)
      .then(() => setClassName(`${initClassNameRef.current} enter enter-active`))
      .then(() => wait(timeout))
      .then(() => setClassName(`${initClassNameRef.current} enter-done`));
    } else {
    setClassName(`${initClassNameRef.current} exit`)
      .then(() => setClassName(`${initClassNameRef.current} exit exit-active`))
      .then(() => wait(timeout))
      .then(() => setClassName(`${initClassNameRef.current} enter-done`));
    }
```

问题很明显，一旦用户多次点击按钮(非常可能)，className值变化顺序就不确定了。

在查看了网上各种取消Promise的方案后，我放弃了引入polyfill库的方案，还有一些方案看着很不直观，必须在完全了解js代码执行顺序的情况下，才能明白为什么这样是可以取消的。最后，我尝试实现了一个简单，易于理解的版本。

```ts
type PormiseMaker = (prevPromiseValue: any) => Promise<any>;

interface ICancelToken {
  cancel: () => void;
  finally: (callback: () => void) => void;
}

function cancelablePromiseChain(...promiseMakers: Array<(prevValue: any) => Promise<any>>): ICancelToken {
  let isCanceled = false;
  let finallyCallback: undefined | (() => void);
  const runner = (async function runner() {
    let prevResult;
    if (isCanceled) {
      if (typeof finallyCallback === 'function') finallyCallback();
      return;
    }
    for (const promiseMaker of promiseMakers) {
      if (isCanceled) {
        if (typeof finallyCallback === 'function') finallyCallback();
        return;
      }
      prevResult = await promiseMaker(prevResult);
    }
  }());
  return {
    cancel() {
      isCanceled = true;
    },
    finally(callback) {
      finallyCallback = callback;
    }
  }
}
```

<code>cancelablePromiseChain</code> 接受一个或多个返回Promise的函数，然后按顺序调用，并且会将先前调用得到返回值作为参数，传递给下一个PromiseMaker函数，这个函数模拟了Promise链式调用，然后增加了中断调用的能力。

注意，finally函数，不仅仅是一个语法糖，你不可以在<code>cancelablePromiseChain</code>的最后一个参数写一个PromiseMaker，然后期待它的行为会和finally一样，finally最重要的在于，它是 **同步的**，这保证了一旦回调链被取消或完成，finally回调被同步的立刻调用进行清理工作，如果是异步就会造成无法预料的错误。很简单的例子，如果是异步的，老的清理函数，可能后于新清理函数完成。

- 示例

```js
let lastFetchCancelToken = null;
fetchButton.on('click', () => {
    if (lastFetchCancelToken != null) = lastFetchCancelToken.cancel();
    lastFetchCancelToken = cancelablePromiseChain(
        () => fetchPost(),
        postData => updateView(postData),
    );
    // 试着想想，如果finall是异步的，你能肯定finally回调设置的lastFetchCancelToken，是自己那轮请求对应的lastFetchCancelToken吗？
    lastFetchCancelToken.finally(() => lastFetchCancelToken = null);
});
```

-  引入可取消后的版本

```ts
const lastRoundTranstionCancelTokenRef = useRef<ICancelToken | null>(null);
if (isIn) {
        if (lastRoundTranstionCancelTokenRef.current != null) lastRoundTranstionCancelTokenRef.current.cancel();
        lastRoundTranstionCancelTokenRef.current = cancelablePromiseChain(
          () => setClassName(`${initClassNameRef.current} enter`),
          () => setClassName(`${initClassNameRef.current} enter enter-active`),
          () => wait(timeout),
          () => setClassName(`${initClassNameRef.current} enter-done`),
        );
        // clear
        lastRoundTranstionCancelTokenRef.current.finally(() => lastRoundTranstionCancelTokenRef.current = null);

      } else {
        if (lastRoundTranstionCancelTokenRef.current != null) lastRoundTranstionCancelTokenRef.current.cancel();
        lastRoundTranstionCancelTokenRef.current = cancelablePromiseChain(
          () => setClassName(`${initClassNameRef.current} exit`),
          () => setClassName(`${initClassNameRef.current} exit exit-active`),
          () => wait(timeout),
          () => setClassName(`${initClassNameRef.current} exit-done`),
        );
        // clear
        lastRoundTranstionCancelTokenRef.current.finally(() => lastRoundTranstionCancelTokenRef.current = null);
      }
```


这里说句题外话，类似本文探讨的这种需求，最好的且简单的解决方案是rxjs，然而我实在不想因为这一个简单的需求，引入整个rxjs库，就放弃了。