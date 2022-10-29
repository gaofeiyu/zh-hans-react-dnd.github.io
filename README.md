> 官方文档：https://react-dnd.github.io/react-dnd/about
> 该文档是个人汉化学习使用，根据自己的使用情况会一点点覆盖汉化内容

# 概述
`React DnD` 与大多数拖放库不同，如果您以前从未使用过它，它可能会令人生畏。但是，一旦您了解了其设计核心的一些概念，它就会开始变得有意义。我建议您在阅读其余文档之前阅读这些概念。

其中一些概念类似于[Flux](http://facebook.github.io/flux/)和[Redux](https://github.com/reduxjs/react-redux)架构。
这不是巧合，因为 `React DnD` 在内部使用 `Redux`。
## 元素和类型(Items and Types)
与 `Flux`（或 `Redux`）一样，`React DnD` 使用数据而不是视图作为事实来源。当您在屏幕上拖动某些东西时，我们不会说正在拖动组件或 `DOM` 节点。相反，我们说正在拖动某种类型的元素(item)。

什么是元素(item)？元素是描述被拖动内容的纯JavaScript 对象。例如，在看板应用程序中，当您拖动卡片时，元素可能看起来像`{ cardId: 42 }`. 在国际象棋游戏中，当您拿起一块棋子时，该物品可能看起来像`{ fromCell: 'C5', piece: 'queen' }`. 将拖动的数据描述为一个普通对象可以帮助您保持组件解耦并且彼此不了解。

那么什么是类型呢？类型是一个字符串（或一个符号），用于唯一标识应用程序中的一整类项目。在看板应用程序中，您可能有一个`'card'`表示可拖动卡片的`'list'`类型和一个用于这些卡片的可拖动列表的类型。在国际象棋中，您可能只有一种`'piece'`类型。

类型很有用，因为随着您的应用程序的增长，您可能希望使更多的东西可拖动，但您不一定希望所有现有的放置目标突然开始对新项目做出反应。这些类型让您可以指定兼容的拖放源和拖放目标。您可能会在应用程序中枚举类型常量，类似于枚举 `Redux` 操作类型的方式。
## 监视器(Monitors)
拖放本质上是有状态的。要么有拖动操作正在进行中，要么没有。要么有当前类型和当前项目，要么没有。这种状态必须存在于某个地方。

React DnD 通过称为监视器的内部状态存储上的一些微小包装器将这种状态暴露给您的组件。监视器允许您更新组件的 props 以响应拖放状态更改。

对于每个需要跟踪拖放状态的组件，您可以定义一个收集函数(`collecting function`)，从监视器中检索它的相关内容。 然后React DnD会负责及时调用您的收集函数并将其返回值合并到组件的`props`中。

假设您想在拖动棋子时突出显示 `Chess` 单元格。组件的收集函数`Cell`可能如下所示：
```typescript
function collect(monitor) {
  return {
    highlighted: monitor.canDrop(),
    hovered: monitor.isOver()
  }
}
```
它指示 React DnD 将所有实例的最新值作为`props`传递`highlighted`和`hovered`给所有`Cell`。
## 连接器(Connectors)
如果后台处理 DOM 事件，但是组件使用 React 来描述 DOM，那么后台如何知道要监听哪些 DOM 节点？
通过连接器。
连接器允许您将预定义角色之一（拖动源、拖动预览或放置目标）分配给`render`函数中的 DOM 节点。

实际上，连接器作为第一个参数传递给我们上面描述的收集函数。让我们看看如何使用它来指定放置目标：
```typescript
function collect(connect, monitor) {
  return {
    highlighted: monitor.canDrop(),
    hovered: monitor.isOver(),
    connectDropTarget: connect.dropTarget()
  }
}
```
在组件的`render`方法中，我们既可以访问从监视器获得的数据，也可以访问从连接器获得的函数：
```typescript
render() {
  const { highlighted, hovered, connectDropTarget } = this.props;

  return connectDropTarget(
    <div className={classSet({
      'Cell': true,
      'Cell--highlighted': highlighted,
      'Cell--hovered': hovered
    })}>
      {this.props.children}
    </div>
  );
}
```
该connectDropTarget调用告诉 React DnD 我们组件的根 DOM 节点是一个有效的放置目标，并且它的悬停和放置事件应该由后台处理。在内部，它通过将`[callback ref](https://reactjs.org/docs/refs-and-the-dom.html#callback-refs)`附加到你给它的 React 元素来工作。连接器返回的函数已被记忆，因此不会破坏`shouldComponentUpdate`优化。
## 拖放源和拖放目标(Drag Sources and Drop Targets)
到目前为止，我们已经介绍了使用 DOM 的后端、由元素(items)和类型(types)表示的数据以及收集(collecting)功能，借助监视器(monitors)和连接器(connectors)，让您您可以描述 React DnD 应该向组件中注入哪些`props`.

但是我们如何配置我们的组件以实际注入这些`props`？我们如何执行副作用以响应拖放事件？React DnD 的主要抽象单元，满足拖动源和放置目标。**它们确实将类型(types)、元素(items)、副作用(effects)和收集函数(collecting functions)与您的组件联系在一起。**

每当您想使组件或其某些部分可拖动时，您需要将该组件包装到拖动源声明中。每个拖动源都注册为特定类型，并且必须实现一个从组件的`props`生成项目的方法。它还可以选择指定一些其他方法来处理拖放事件。拖动源声明还允许您为给定组件指定收集函数。

放置目标与拖动源非常相似。唯一的区别是单个放置目标可以同时注册多个元素类型，而不是生成元素，它可以处理其悬停或放置。
## 后台(Backends)
React DnD 使用[HTML5 拖放 API](https://developer.mozilla.org/en-US/docs/Web/Guide/HTML/Drag_and_drop)。这是一个合理的默认设置，因为它会截取拖动的 DOM 节点并将其用作开箱即用的“拖动预览”。当光标移动时，您不必进行任何绘图，这很方便。此 API 也是处理文件拖入事件的唯一方法。

不幸的是，HTML5 拖放 API 也有一些缺点。它不适用于触摸屏，并且它在 IE 上提供的定制机会比在其他浏览器中少。

这就是为什么在 React DnD中**以可插入的方式实现 HTML5 拖放支持**的原因。你不必使用它。您可以完全基于触摸事件、鼠标事件或其他东西编写不同的实现。这种可插拔的实现在 React DnD中称为后台(backends)。

该库当前附带[HTML backend](https://react-dnd.github.io/react-dnd/docs/backends/html5)，对于大多数 Web 应用程序来说应该足够了。还有一个可用于移动 Web 应用程序的[Touch backend](https://react-dnd.github.io/react-dnd/docs/backends/touch)。

后台的作用类似于 React 的合成事件系统：它们抽象出浏览器的差异，并处理原生 DOM 事件。尽管有相似之处，但 React DnD 后台并不依赖于 React 或其合成事件系统。在后台，所有后台所做的就是将 DOM 事件转换为 React DnD 可以处理的内部 Redux 操作。
## 钩子与高阶组件(Hooks vs Higher-Order Components)
现在您应该对 React DnD 的各种移动部分有所了解：

- 元素对象和类型
- DnD 属于flux方式的状态
- 用于观察 DnD 状态的监视器
- 用于将监视器输出转换为`props`的收集器功能
- 用于将 DnD 状态机附加到视图节点（例如 DOM 元素）的连接器
## 钩子(Hooks)
现代 React 应用程序已经用钩子取代了高阶组件模式。Hooks 是 React 的一个特性，在 16.8 中引入，它允许开发人员编写有状态的函数组件。它们也非常适合管理有状态组件，以及与外部有状态系统交互（*`cough`*像拖放引擎*`cough`*）。

如果您不熟悉 React 钩子，请参阅 React 博客文章，[介绍钩子](https://reactjs.org/docs/hooks-intro.html)。

React-DnD 提供了将组件连接到 DnD 引擎的钩子，并允许您收集监视器状态以进行渲染。

有关基于 hooks 的 API 的概述，请参阅[Hooks 概述](https://react-dnd.github.io/react-dnd/docs/api/hooks-overview)页面。
# DropTargetMonitor
`DropTargetMonitor`是传递给[放置目标](https://react-dnd.github.io/react-dnd/docs/api/use-drop)的收集函数的对象。它的方法让您可以获取有关特定放置目标的拖动状态的信息。绑定到该监视器的特定放置目标在下面称为监视器的所有者(`monitor's owner`)。
## Methods
### canDrop()
如果有拖拽操作正在进行，则返回`true`，所有者的`canDrop()`返回`true`或未定义。
### isOver(otions)
如果正在进行拖动操作，并且指针当前悬停在所有者上，则返回`true`。您可以选择传递`{ shallow: true }`以严格检查是否只有所有者被悬停，而不是嵌套目标。
### getItemType()
返回一个字符串或一个符号，标识当前拖动项的类型。如果没有项目被拖动，则返回`null`。
### getItem()
返回一个表示当前拖动项目的普通对象。每个拖动源都必须通过从其`beginDrag()`方法返回一个对象来指定它。如果没有项目被拖动，则返回`null`。
### getDropResult()
返回一个普通对象，表示最后记录的放置结果。放置目标可以选择通过从它们的`drop()`方法返回一个对象来指定它。当为嵌套目标分派一个`drop()`链时，自下而上，任何从`drop()`显式返回其自身结果的父级都会覆盖先前由子级设置的放置结果。如果在外部调`drop()`则返回`null`。
### didDrop()
如果某个放置目标已处理放置事件，则返回`true`，否则返回`false`。即使目标没有返回丢弃结果，也会`didDrop()`返回`true`. 在内部使用它`drop()`来测试是否有任何嵌套的放置目标已经处理了放置。如果在外部调用`drop()`则返回`false`。
### getInitialClientOffset()
返回当前拖动操作开始时指针的客户端偏移量`{ x, y }`。如果没有项目被拖动，则返回`null`。
### getInitialSourceClientOffset()
返回当前拖拽操作开始时拖拽源组件的根DOM节点的客户端偏移量`{ x, y }`。如果没有项目被拖动，则返回`null`。
### getClientOffset()
在拖动操作正在进行时返回指针的最后记录的客户端偏移量`{ x, y }`。如果没有项目被拖动，则返回`null`。
### getDifferenceFromInitialOffset()
返回上次记录的指针客户端偏移量与当前拖动操作开始时的客户端偏移量之间的差值`{ x, y }`。如果没有项目被拖动，则返回`null`。
### getSourceClientOffset()
根据当前拖动操作开始时的位置和移动差异，返回拖动源组件的根 DOM 节点的投影客户端偏移量`{ x, y }`。如果没有项目被拖动，则返回`null`。
