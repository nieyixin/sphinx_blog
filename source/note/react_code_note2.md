# react源码学习2
本文主要讲react是如何支持Function Components和Hooks的。

## Function Components

以如下的App function component为例：
```
function App(props) {
  return Didact.createElement(
    "h1",
    null,
    "Hi ",
    props.name
  )
}
const element = Didact.createElement(App, {
  name: "foo",
})
```

函数组件相较于直接调用`createElement`方法去渲染，有两点不同：
* 函数组件的fiber结构没有dom节点
* 函数组件的children来自于运行该函数，而非从props中获取

### performUnitOfWork
在之前介绍的`performUnitOfWork`方法中（参考[react源码学习1](https://nyx-blog.readthedocs.io/zh/latest/note/react_code_note.html)），我们会先创建dom节点，然后获取要渲染的子节点：
```
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  const elements = fiber.props.children
  reconcileChildren(fiber, elements)

  ...
}
```

为了支持函数组件，我们需要一些适配。

首先，根据fiber类型是否为function，做不同操作：
```
function performUnitOfWork(fiber) {
  const isFunctionComponent =
    fiber.type instanceof Function
  if (isFunctionComponent) {
    updateFunctionComponent(fiber)
  } else {
    updateHostComponent(fiber)
  }
  
  ...
}
```

`updateHostCompoent`和先前保持一致，在`updateFunctionComponent`中：
```
function updateFunctionComponent(fiber) {
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
```

通过直接调用函数，获取children节点。上例中`fiber.type(fiber.props) = App(props)`，即h1。

有了children以后，`reconcileChildren`跟之前保持一致，无需修改。

### commitWork

同时需要在`commitWork`方法中做一些修改。

因为函数组件没有dom节点，所以需要在寻找parent fiber时，不断回溯，直到parent fiber有dom节点：

```
function commitWork(fiber) {
  if (!fiber) {
    return
  }
​
  let domParentFiber = fiber.parent
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent
  }
  const domParent = domParentFiber.dom
  ...
}
```

这样在后续更新时，才能进行dom操作：
```
...
if (
    fiber.effectTag === "PLACEMENT" &&
    fiber.dom != null
  ) {
    domParent.appendChild(fiber.dom)
  }
...
```

删除节点时，也需要通过递归向下寻找删除fiber节点，确保被删除的fiber节点有dom节点：

```
...
else if (fiber.effectTag === "DELETION") {
    commitDeletion(fiber, domParent)
  }
...

function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom)
  } else {
    commitDeletion(fiber.child, domParent)
  }
}
```

## Hooks

以经典的计数器为例，通过点击实现计数的Counter组件：

```
function Counter() {
  const [state, setState] = React.useState(1)
  return (
    <h1 onClick={() => setState(c => c + 1)}>
      Count: {state}
    </h1>
  )
}
const element = <Counter />
```

具体是通过`React.useState`获取并更新state值：
```
function useState(initial) {
  // TODO
}
```

useState是如何实现state的数据更新的？数据更新时是如何触发重新渲染的？下文将一一解答。

### updateFunctionComponent

在`updateFunctionComponent`方法中，新增`hooks`数组，用于支持该函数组件多次调用`useState`：
```
wipFiber = null
let hookIndex = null
​
function updateFunctionComponent(fiber) {
  wipFiber = fiber
  hookIndex = 0
  wipFiber.hooks = []
  const children = [fiber.type(fiber.props)]
  reconcileChildren(fiber, children)
}
```
同时定义`hoookIndex`索引，用于useState方法。

### useState

当函数组件调用`useState`时，首先检查是否有`oldHook`，如果有则使用oldHook的值，否则则使用初始化的值；然后将这个hook添加到fiber中，并返回这个state。
```
function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  const hook = {
    state: oldHook ? oldHook.state : initial,
  }
​
  wipFiber.hooks.push(hook)
  hookIndex++
  return [hook.state]
}
​
```

`useState`也需要返回一个函数用来更新state的值，因此我们定义一个`setState`函数用来接收一个action。对上述的useState方法进一步修改：
```
function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [],
  }
​   
  const setState = action => {
    hook.queue.push(action)
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    }
    nextUnitOfWork = wipRoot
    deletions = []
  }

  wipFiber.hooks.push(hook)
  hookIndex++
  return [hook.state, setState]
}
​
```
这里将`nextUnitOfWork`赋值为`wipRoot`，这样做是为了重新进入render阶段。即每次更新状态时，都会重新触发渲染，这跟useState的逻辑一致。

那么何时调用action呢？继续做如下修改：

```
function useState(initial) {
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [],
  }
​   
  const actions = oldHook ? oldHook.queue : []
  actions.forEach(action => {
    hook.state = action(hook.state)
  })

  const setState = action => {
    hook.queue.push(action)
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    }
    nextUnitOfWork = wipRoot
    deletions = []
  }

  wipFiber.hooks.push(hook)
  hookIndex++
  return [hook.state, setState]
}
​
```
当调用setState时，由前文可知，会重新触发渲染，此时的`oldHook.queue=[action]`。

在新的render阶段，则会直接调用所有actions，并一个一个赋值给新的state。当函数返回时，`hook.state`已更新成最新值。

> 1. useState是如何实现state的数据更新的？
> 2. 数据更新时是如何触发重新渲染的
> 
> 答：
> 1. useState通过执行action来更新state值。当调用setState时，会触发重新渲染，再次进入函数时，会遍历所有actions，并赋值给state。
> 2. setState方法会将wipRoot赋值为currentRoot，并且设置nextUnitOfWork为wipRoot，这样在workLoop中，会重新进入render阶段。 