# React内部数据结构详解

本文档旨在深入解析React的内部数据结构，帮助开发者更好地理解React的工作原理和渲染机制。

## 目录

- [Fiber架构](#fiber架构)
- [虚拟DOM (Virtual DOM)](#虚拟dom-virtual-dom)
- [React Element](#react-element)
- [Fiber节点](#fiber节点)
- [Fiber树](#fiber树)
- [更新队列 (Update Queue)](#更新队列-update-queue)
- [Hook数据结构](#hook数据结构)
- [Context数据结构](#context数据结构)
- [Ref数据结构](#ref数据结构)

## Fiber架构

Fiber是React 16引入的新架构，它的核心目标是支持增量渲染和优先级调度。Fiber架构将渲染工作分解为多个小单元，每个单元可以被中断、恢复、复用或丢弃。

### Fiber的主要特点

- **可中断性**：渲染过程可以被中断并在稍后恢复
- **优先级**：不同的更新可以被赋予不同的优先级
- **复用**：之前已完成的工作可以被复用
- **双缓冲**：维护两棵树（current树和workInProgress树）

## 虚拟DOM (Virtual DOM)

虚拟DOM是React的核心概念，它是真实DOM的JavaScript对象表示。React使用虚拟DOM来减少直接操作真实DOM的次数，从而提高性能。

\\\javascript
// 虚拟DOM示例
{
  type: 'div',
  props: {
    className: 'container',
    children: [
      {
        type: 'h1',
        props: {
          children: 'Hello World'
        }
      },
      {
        type: 'p',
        props: {
          children: 'This is a paragraph'
        }
      }
    ]
  }
}
\\\

## React Element

React Element是描述UI的不可变对象，是React应用的最小构建块。

\\\javascript
// React Element结构
{
  $$typeof: Symbol.for('react.element'),
  type: 'div', // 可以是字符串或React组件函数/类
  key: null,
  ref: null,
  props: { /* 组件属性 */ },
  _owner: null
}
\\\

## Fiber节点

Fiber节点是Fiber架构中的基本工作单元，每个React Element对应一个Fiber节点。

\\\javascript
// Fiber节点结构（简化版）
{
  // 实例相关
  tag: 0, // 标记不同组件类型
  key: null,
  elementType: null,
  type: null,
  stateNode: null, // 指向组件实例、DOM节点等
  
  // Fiber树结构
  return: null, // 父Fiber
  child: null, // 第一个子Fiber
  sibling: null, // 下一个兄弟Fiber
  index: 0,
  
  // 工作相关
  pendingProps: null,
  memoizedProps: null,
  updateQueue: null,
  memoizedState: null,
  dependencies: null,
  
  // 副作用相关
  flags: 0,
  subtreeFlags: 0,
  deletions: null,
  
  // 调度优先级相关
  lanes: 0,
  childLanes: 0,
  
  // 替身
  alternate: null // 指向另一棵树中对应的Fiber
}
\\\

## Fiber树

React维护两棵Fiber树：current树和workInProgress树。

- **current树**：当前屏幕上显示内容对应的Fiber树
- **workInProgress树**：正在构建的新Fiber树

当workInProgress树构建完成并提交后，它会变成新的current树。

## 更新队列 (Update Queue)

更新队列是一个环形链表，用于存储组件的状态更新。

\\\javascript
// 更新队列结构
{
  baseState: Object, // 基础状态
  firstBaseUpdate: Update, // 第一个基础更新
  lastBaseUpdate: Update, // 最后一个基础更新
  shared: {
    pending: Update // 等待处理的更新
  },
  effects: Array // 副作用列表
}

// 更新对象结构
{
  tag: UpdateTag, // 更新类型标记
  payload: any, // 更新内容
  callback: Function | null, // 回调函数
  next: Update, // 下一个更新
  priority: Priority // 优先级
}
\\\

## Hook数据结构

Hook是React函数组件中使用的特殊函数，它们允许在函数组件中使用状态和其他React特性。

\\\javascript
// Hook基本结构
{
  memoizedState: any, // Hook的状态
  baseState: any, // 基础状态
  baseQueue: Update | null, // 基础更新队列
  queue: UpdateQueue | null, // 更新队列
  next: Hook | null // 链表中的下一个Hook
}

// 不同类型的Hook有不同的memoizedState结构
// useState Hook
{
  memoizedState: currentState,
  baseState: baseState,
  baseQueue: baseQueue,
  queue: {
    pending: null,
    dispatch: dispatchAction.bind(null, currentlyRenderingFiber, queue),
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: currentState
  },
  next: nextHook
}

// useEffect Hook
{
  memoizedState: {
    tag: HookEffectTag,
    create: () => cleanupFn,
    destroy: cleanupFn,
    deps: [dep1, dep2, ...],
    next: nextEffect
  },
  baseState: null,
  baseQueue: null,
  queue: null,
  next: nextHook
}
\\\

## Context数据结构

Context提供了一种在组件树中共享数据的方式，而无需显式地通过props传递。

\\\javascript
// Context对象结构
{
  $$typeof: Symbol.for('react.context'),
  _currentValue: any, // 当前值
  _currentValue2: any, // 用于secondary renderer
  _threadCount: 0, // 追踪并发渲染器
  Provider: {
    $$typeof: Symbol.for('react.provider'),
    _context: context
  },
  Consumer: {
    $$typeof: Symbol.for('react.context'),
    _context: context
  }
}
\\\

## Ref数据结构

Ref允许访问DOM节点或React元素实例。

\\\javascript
// createRef()创建的ref
{
  current: null // 初始值为null，后续会被设置为DOM节点或组件实例
}

// useRef()创建的ref
{
  memoizedState: {
    current: initialValue
  }
}
\\\

---

通过深入理解这些内部数据结构，开发者可以更好地把握React的工作原理，编写更高效的React应用，并在遇到复杂问题时更容易进行调试和优化。

## 参考资料

- [React官方文档](https://reactjs.org/docs/getting-started.html)
- [React源码](https://github.com/facebook/react)
- [React Fiber架构](https://github.com/acdlite/react-fiber-architecture)