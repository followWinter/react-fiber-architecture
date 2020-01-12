# React Fiber 架构

## 介绍

React Fiber 是一个正在进行重的的 React 的核心算法的重新实现。

React Fiber 的目标是增加它适合的区域，比如动画，布局，和手势。它的核心特性是**增量渲染**：分割渲染工作到块并扩散到多个帧的能力。

其他关键特性比如暂停，放弃，或者重用的能力作为新的更新到来；给不同类型的更新赋予不同优先级的能力；和新的并发原语。

### 关于这个文档

Fiber 引入了多个新概念，很难仅靠查看代码理解。这个文档作为我跟随 React 项目实现的 Fiber 做的笔记的集合的开始。随着它的进行，我意识到它可能对其他人也是一个很有帮助的资源。

我将尽可能尝试使用最直白的语言，我也会尽可能链接大量外部资源。

请注意我不在 React 团队，并没有向作者询问任何东西。**这不是一个官方文档**。我有征求过 React 团队的成员去评论它的正确。

这个文档还在进程中。**Fiber 是一个正在进行的项目，将会可能在完成之前进行重大重构。** 我在这里记录它的设计也在进行中。提升和建议都是非常欢迎的。

我的目标是在读完整个文档之后，你将会足够理解 Fiber 去[跟随它的实现](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber)，最终甚至可以给 React 贡献。

### 先决条件

我强烈建议你在继续之前先熟悉下面的资源：

- [React 组件，元素，和实例](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - "组件"通常是一个超重的词汇。牢牢掌握它很重要.

- [调和](https://facebook.github.io/react/docs/reconciliation.html) - React 调和算法高级别的描述

- [React 基础理论概念](https://github.com/reactjs/react-basic) - 没有实现负担的 React 概念模型的描述。第一次阅读可能没有什么意义。没关系，它将会随着时间变得有意义。

- [React 设计原则](https://facebook.github.io/react/contributing/design-principles.html) - 注意调度章节。它很好的解释了 React Fiber 的**原因**。

## 回顾

请查阅前置条件章节，如果你还没有。

在我们开始新的事物之前，回顾一些概念。

### 什么是调和

<dl>
  <dt>调和</dt>
  <dd>React 用来比较一棵树和另一棵树，决定那一部分被改变的算法</dd>

  <dt>更新</dt>
  <dd>用于渲染一个 React App 的数据的改变。通常是`setState`的结果。最终导致一个重新渲染。</dd>
</dl>

React 的 API 的核心思想是认为 update 导致整个 app 重新渲染。这允许开发者进行有效声明式，而不是担心怎样去有效的从任意特定状态到其他状态（A 到 B，B 到 C，C 到 A，等等）转移 app 。

实际上，在每一次该改变的时候重新渲染整个 app 只在最细小的 app 发生；在一个真实世界的 app，对于性能来说是禁止的。React 做了优化，重新渲染创建整个 app 的外观并维持好的性能。这些大部分的优化都是**调和**过程的一部分。


调试算法是大众理解的"虚拟 DOM"的背后。一个高级别的描述就像这样：当你渲染一个 React 应用，描述整个 app 的节点被创建保存在内存。然后树被刷新到渲染环境 -- 比如，在浏览器应用场景，它转化为一个 DOM 操作的集合。当 app 被更新（通常通过`setState`），一个新的树被生成。新的树和前一个树对比，计算出哪些操作需要更新到已经渲染的 app。

尽管 Fiber 从头来重写协调，[描述在 React 文档的](https://facebook.github.io/react/docs/reconciliation.html)高层级的算法大部分都一样。关键是：

- 不同的组件类型假设生成不同的子树。React 将不会尝试去对比他们，而是完整替换整棵树。
- 列表的对比通过 key。key 应该是"稳定，可预料，和唯一的"。

### 调和和渲染

DOM 是 React 可以渲染的渲染环境的一种，其他主要目标是原生 IOS 和 Android 视图，通过 React Native。（这也是为什么"虚拟 DOM"是误称）。


React 可以支持这么多的目标是因为它将调和和渲染设计为分开的步骤。协调器负责计算树的哪一部分发生改变的工作；渲染器使用这些信息去真实更新渲染的 app。

这个分离意味着 React DOM 和 React Native 可以使用他们自己的渲染器，共享相同的协调器。

Fiber 实现了协调器。它不是渲染主要关心的，尽管渲染器需要改变去支持（并利用）新的架构。

### 调度

<dl>
  <dt>调度</dt>
  <dd>决定工作何时被执行的进程。</dd>

  <dt>工作</dt>
  <dd>任何必须要被执行的比较。工作通常是一个更新（比如，<code>setState</code>）的结果</dd>
</dl>

React 的[设计理念](https://facebook.github.io/react/contributing/design-principles.html#scheduling)描述的很好，我将只在这里引用它：

> 在当前的实现中，React 递归遍历整棵树树，并在单个 tick 中调用整个更新的树的 render 函数。然而在未来，它可能开始延迟一些更新去避免掉帧。
>
> 这是 React 设计的常见主题。一些流行的库实现"推"方法，在新数据可用时执行计算。然而 React 坚持"pull"方法，这样计算可以被推迟直到必要。
> 
> React 不是通用数据处理库。它是一个用于构建用户接口的库。我们认为它独特的点是知道在 app 中，哪些计算是现在相关的，哪些不是。
> 
> 如果一些东西在屏幕之外，我们可以延迟和它相关的逻辑。如果数据来的比帧率还快，我们可以合并和批量更新。相对于不重要的背景工作（比如渲染只是从网络加载来的新内容），我们可以优先处理来自用户交互的工作（比如点击按钮触发的动画）。

关键点是：

- 在一个 UI，每一个更新被立即应用是不需要的；实际上，这么做很浪费，导致掉帧并降低用户体验。
- 不同类型的更新有不同的优先级 - 一个动画更新需要比数据存储的更新更快的完成。
- 一个基于推的方式需要 app（你，程序员）去决定怎样调度工作。一个基于拉的方式允许框架（React）去灵活为你做这些决定。

React 当前没有充分利用调度的优势；一个更新导致整个子树立即被重新渲染。为了充分利用调用而大修 React 的核心算法是 Fiber 背后的驱动思想。 

---

现在我们准备深入 Fiber 的实现。下一个章节比我们已经讨论的更加技术性。请确保你对前面的材料感觉舒适。

## What is a fiber?

我们将要讨论 React Fiber 的核心架构。Fiber 是比应用开发者通常想的更加底层的概念。如果你在尝试理解它的时候感到沮丧，不要泄气。继续尝试，它最终会有意义。（当你最终理解它，请建议如何提升这个章节。）

让我们开始吧！

---

我们已经建立了一个 Fiber 的主要目标，是让 React 合理利用调度。特别是，我们需要可以做到

- 暂停工作和之后回到工作
- 为不同类型的工作赋予优先级
- 重用前面已经完成的工作
- 如果不再需要，就放弃工作。

为了做到这些，我们首先需要一个方式去将工作打碎成单元。换句话说，这就是 fiber。一个 fiber 表示一个**工作单元**。

为了深入，让我们回到[React 组件是数据的函数](https://github.com/reactjs/react-basic#transformation)，通常解释为：
```
v = f(d)
```

因此，渲染一个 React app 类似于调用一个函数，它的内容包含其他函数调用，等等。这个比喻在思考 fiber 的时候十分有用。

计算机通常跟踪一个程序的执行的方式是使用[调用栈](https://en.wikipedia.org/wiki/Call_stack)。当一个函数被执行，一个新的**栈帧**被添加到栈。这个栈帧表示函数执行的工作。

当处理 UI 的时候，问题是如果太多的工作在同一时刻被执行，它会导致动画掉帧并且看起来抖动。甚至，一些工作可能不需要，如果它被更近的更新取代。这就是 UI 组件和函数比较中断的地方，因为组件通常比函数有更多特定的概念，

更新的浏览器（和 React Native）实现 API 帮助指出这个确切的问题：`requestIdleCallback`调度一个低优先级函数在空闲时调用，`requestAnimationFrame`调度一个高优先级函数在下一个动画帧调用。问题是，为了使用这些 API，你需要一个方式去打破渲染工作到增量单元。如果你只依赖调用栈，它将会保持继续工作，知道栈是空的。

如果我们可以自定义调用栈的行为去优化渲染 UI 不是更好吗？如果我们可以中断未来的调用栈并且手动维护栈帧不是更好吗？

这就是 React Fiber 的目标。Fiber 重新实现了栈，栈们为了 React 组件。你可以认为单个 fiber 就是一个**虚拟栈帧**。

重新实现栈的有点是你可以[在内存保持栈帧](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)并执行他们无论（何时）你想。这是我们需要为调度完成的目标的关键。

除了调度，手动处理栈帧解锁了特性的潜力，比如并发和错误包裹。我们将会在未来的章节覆盖这些话题。

在下一个章节，我们将会查找更多 fiber 的结构。

### fiber 的结构

*注意：随着我们更加具体的了解实现细节，一些可能改变的可能性会增加。请提交一个 PR，如果你注意到任何错误或者过期的信息。*

具体来说，fiber 是一个 JavaScript 对象，包含一个组件的信息，它的输入，它的输出。

一个 fiber 表示一个栈帧，但是它也表示一个组件的实例。

下面是一些属于 fiber 重要的域。（这个列表并不详尽）。

#### `type`和`key`

一个fiber 的 type 和 key 和 React element 的相同域有相同的作用。（实际上，当一个 fiber 从一个 element 创建，这两个域将会直接复制）

fiber 的 type 描述了组件所代表的。对于合成组件，type 是函数或者类组件它自己。对于 host 组件（`div`，`span`，等），type 是字符串。

概念上来说，type 是在栈帧中被跟踪的函数（就像在`v = f(d)`）。


伴随着类似难过，key 在调和的时候去决定是否重用 fiber。


#### `child`和`sibling`

这些域指向其他 fiber，描述了 fiber 的树结构。

child fiber 表示一个组件`render`方法的返回值。所以，在下面的例子：

```js
function Parent() {
  return <Child />
}
```

`Parent` 的 child fiber 是`Child`。

sibling 域表示`render`返回多个子组件（Fiber 的一个新特性！）的场景。

```js
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

child fiber 组成了一个单链表，它的头是第一个 child。所以在这个例子中，`Parent`的 child 是`Child1`，`Child1`的 sibling 是`Child2`。

回到我们的函数比喻，你可以认为一个 child fiber 是一个[尾调用函数](https://en.wikipedia.org/wiki/Tail_call)。

#### `return`

`return` fiber 表示程序在处理完当前 fiber 应该返回的 fiber。它概念上和栈帧的返回地址相同。它也可以认为是 parent fiber。

如果一个 fiber 有多个 child fiber，每一个 child fiber 的返回 fiber 是它的 parent。因此在我们前面一个章节的例子中，`Child1`和`Child2`的 return fiber 是`Parent`。

#### `pendingProps`和`memoizedProps`

概念上，props 是函数的参数。一个 fiber 的`pendingProps`在它开始执行的时候设定，`memoizedProps`在结束的时候设定。

当来临的`pendingProps`和`memoizedProps`相同，它意味着 fiber 之前的输出可以被重用，防止不需要的工作。

#### `pendingWorkPriority`

一个数字指示 fiber 表示的工作的优先级。[ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) 模块列出了不同优先级，和他们所表示的。

`NoWork`是个例外，它是 0，一个大数指示一个低优先级。比如，你可以使用下面的函数检查一个 fiber 的优先级是否和给定级别相同。
```js
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```
*这个函数只是为了描述；不是 React Fiber 代码的一部分。*

调度者使用优先级域去搜索下一个执行的工作单元。这个算法将会在后续章节讨论。

#### `alternate`
<dl>
  <dt>刷新</dt>
  <dd>刷新一个 fiber 是渲染它的输出到屏幕</dd>

  <dt>work-in-progress</dt>
  <dd>一个 fiber 还没有完成；概念上，一个栈帧还没有被返回。</dd>
</dl>

在任何时候，一个组件实例至少有两个 fiber 表示它：current，刷新的 fiber，和 work-in-progress fiber。

current fiber 的 alternate 是 work-in-progress，working-in-progress fiber 的 alternate 是 current fiber。

一个 fiber 的 alternate 是懒创建的，使用一个叫做`cloneFIber`的函数。与其总是创建一个新的对象，`cloneFiber`将会尝试去重用 fiber 的 alternate，如果它存在，最小化分配。

你应该认为`alternate`域是具体实现，但是它经常在代码中出现，以至于它值得在这里讨论。

#### `输出`
<dl>
  <dt>宿主组件</dt>
  <dd>React 应用的叶子节点。他们是渲染环境指定的（比如，在一个浏览器，他们是`div`，`span`，等）。在 JSX，他们使用消协标签名表示，</dd>
</dl>

概念上，一个 fiber 的输出是函数的返回值。

每一个 fiber 最终都有输出，但是输出只有在叶子节点创建，通过**宿主组件**。然后输出被传输到树。

输出是真实给渲染器的，因此他可以输出改变到渲染环境。它的渲染器表示定义输出怎样被创建和更新。

## 更多章节
这就是现在的所有了，但是这个文档现在已经接近完成。更多章节将会描述一个贯穿一个更新的生命周期的算法。话题覆盖包括：

- 调度器怎样找到下一个工作单元去执行
- 优先级是怎样在 fiber 树中传播和跟踪
- 调度器怎样知道何时去暂停和回复工作。
- 工作怎样刷新并且标记为完成
- 副作用怎样（比如生命周期方法）工作
- 协程是什么，它可以怎样用于实现像上下文和布局的功能。

## 相关视频
- [React 下一步是什么（ReactNext 2016）](https://youtu.be/aV1271hd9ew)

