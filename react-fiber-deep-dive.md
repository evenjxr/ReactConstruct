# 以 Fiber 为线索，详解 React 底层工作原理

React 的 Fiber 架构是 React 16 引入的核心重构，它彻底改变了 React 的内部实现。下面我将以 Fiber 为线索，详细阐述 React 底层从页面初始化到页面更新的整个过程，以及 Fiber 各个属性的作用和意义，以及 React 如何实现优先级更新。

## 一、Fiber 的基本结构

Fiber 节点是 React 内部工作的基本单位，每个 React 元素都对应一个 Fiber 节点。Fiber 节点的主要结构如下：

```typescript
export type Fiber = {
  // 标记 fiber 的类型
  tag: WorkTag,

  // 唯一标识，特别是在列表中用于区分不同元素
  key: null | string,

  // fiber 类型，对应于函数/类组件、原生DOM等
  elementType: any,

  // 对于组件，type 是函数/类本身
  // 对于原生DOM，type 是字符串（如'div'）
  type: any,

  // 与此 fiber 关联的本地状态
  // 对于类组件，这是组件实例
  // 对于DOM元素，这是DOM节点
  stateNode: any,

  // ===== Fiber 树结构 =====
  // 指向父 fiber
  return: Fiber | null,
  // 指向第一个子 fiber
  child: Fiber | null,
  // 指向下一个兄弟 fiber
  sibling: Fiber | null,
  // 在兄弟节点中的索引
  index: number,

  // ref 字段
  ref: null | (((handle: mixed) => void) & { _stringRef: ?string, ... }) | RefObject,

  // ===== 输入输出 =====
  // 新的 props
  pendingProps: any,
  // 已经处理过的 props
  memoizedProps: any,
  // 更新队列
  updateQueue: UpdateQueue<any> | null,
  // 已经处理过的 state
  memoizedState: any,
  // 依赖项（用于 context, suspense 等）
  dependencies: Dependencies | null,
  // 模式标记（如 StrictMode, ConcurrentMode 等）
  mode: TypeOfMode,

  // ===== 副作用 =====
  // 标记此 fiber 上的副作用
  flags: Flags,
  // 子树中所有 fiber 的副作用标记的合并
  subtreeFlags: Flags,
  // 需要删除的子节点列表
  deletions: Array<Fiber> | null,

  // ===== 调度相关 =====
  // 优先级相关
  lanes: Lanes,
  // 子树中所有 fiber 的优先级的合并
  childLanes: Lanes,
  // 指向 workInProgress 树中对应的 fiber
  // 或指向 current 树中对应的 fiber
  alternate: Fiber | null,
}
```

## 二、通过具体例子跟踪 Fiber 属性变化

为了更直观地理解 Fiber 的工作原理，我们将通过一个简单的计数器组件示例，跟踪 Fiber 属性在整个生命周期中的变化。

### 示例组件

```jsx
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState(state => ({ count: state.count + 1 }));
  }

  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={this.handleClick}>Increment</button>
      </div>
    );
  }
}

ReactDOM.render(<Counter />, document.getElementById('root'));
```

### 1. 初始化阶段 - Fiber 属性的初始状态

当 `ReactDOM.render(<Counter />, document.getElementById('root'))` 被调用时，React 开始创建 Fiber 树。

#### 1.1 创建 RootFiber 和 Counter 组件的 Fiber

首先，React 创建 RootFiber（HostRoot，tag = 3）和 Counter 组件的 Fiber（ClassComponent，tag = 1）：

```
RootFiber {
  tag: 3 (HostRoot),
  stateNode: FiberRoot,
  memoizedState: null,
  updateQueue: {
    baseState: null,
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: { pending: null },
    effects: null
  },
  child: CounterFiber,
  // ...其他属性
}

CounterFiber {
  tag: 1 (ClassComponent),
  type: Counter, // 指向 Counter 类
  stateNode: Counter实例, // 组件实例
  memoizedState: { count: 0 }, // 初始状态
  updateQueue: null, // 初始时没有更新
  pendingProps: {}, // 初始props
  memoizedProps: {}, // 记忆的props
  child: DivFiber, // 指向render返回的div
  return: RootFiber, // 父节点是RootFiber
  flags: 0, // 没有副作用
  lanes: 0, // 没有优先级
  // ...其他属性
}
```

#### 1.2 创建子元素的 Fiber 节点

接下来，React 为 `render()` 方法返回的 JSX 创建对应的 Fiber 节点：

```
DivFiber {
  tag: 5 (HostComponent),
  type: 'div',
  stateNode: HTMLDivElement, // DOM节点
  pendingProps: { children: [PFiber, ButtonFiber] },
  memoizedProps: { children: [PFiber, ButtonFiber] },
  child: PFiber, // 第一个子节点
  return: CounterFiber,
  sibling: null,
  // ...其他属性
}

PFiber {
  tag: 5 (HostComponent),
  type: 'p',
  stateNode: HTMLParagraphElement,
  pendingProps: { children: "Count: 0" },
  memoizedProps: { children: "Count: 0" },
  child: TextFiber,
  return: DivFiber,
  sibling: ButtonFiber,
  // ...其他属性
}

ButtonFiber {
  tag: 5 (HostComponent),
  type: 'button',
  stateNode: HTMLButtonElement,
  pendingProps: { 
    onClick: Counter实例.handleClick, 
    children: "Increment" 
  },
  memoizedProps: { 
    onClick: Counter实例.handleClick, 
    children: "Increment" 
  },
  child: TextFiber2,
  return: DivFiber,
  sibling: null,
  // ...其他属性
}
```

#### 1.3 完成初始渲染

在完成 Fiber 树的构建后，React 进入 commit 阶段，将 Fiber 树转换为实际的 DOM 操作：

1. 创建 DOM 节点
2. 设置属性和事件监听器
3. 将节点插入到 DOM 树中
4. 调用生命周期方法 `componentDidMount`

此时，屏幕上显示了初始 UI，Counter 组件的 Fiber 节点的关键属性如下：

```
CounterFiber {
  memoizedState: { count: 0 },
  memoizedProps: {},
  updateQueue: null,
  flags: 0, // 已清除所有副作用标记
  // ...其他属性
}
```

### 2. 更新阶段 - 点击按钮后 Fiber 属性的变化

当用户点击"Increment"按钮时，触发了 `handleClick` 方法，调用 `this.setState(state => ({ count: state.count + 1 }))`。

#### 2.1 创建 Update 对象并加入 updateQueue

当 `setState` 被调用时，React 会：

1. 创建一个 Update 对象
2. 将其添加到 Counter 组件 Fiber 节点的 updateQueue 中

```
// 创建的Update对象
Update {
  eventTime: 123456789, // 时间戳
  lane: 0b0000000000000000000000000000001, // SyncLane（同步优先级）
  tag: 0, // UpdateState
  payload: (state) => ({ count: state.count + 1 }), // 状态更新函数
  callback: null,
  next: null // 指向链表中的下一个更新
}

// 更新后的CounterFiber
CounterFiber {
  // ...其他属性不变
  updateQueue: {
    baseState: { count: 0 },
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: Update // 指向上面创建的Update对象
    },
    effects: null
  },
  // ...其他属性
}
```

#### 2.2 调度更新

React 会调度一次新的渲染，创建 workInProgress 树（一个新的 Fiber 树）。这个树是通过复制 current 树（当前显示的 Fiber 树）并应用更新来创建的。

对于 Counter 组件，React 会：

1. 创建一个 workInProgress Fiber 节点（通过 `alternate` 字段关联到 current Fiber）
2. 处理 updateQueue 中的更新

```
// workInProgress CounterFiber
workInProgressCounterFiber {
  // 从current复制的属性
  tag: 1,
  type: Counter,
  stateNode: Counter实例,
  return: workInProgressRootFiber,
  alternate: CounterFiber, // 指向current树中的对应节点

  // 待处理的属性
  pendingProps: {},
  memoizedProps: {}, // 尚未更新
  memoizedState: { count: 0 }, // 尚未更新
  updateQueue: {
    baseState: { count: 0 },
    firstBaseUpdate: Update, // 从shared.pending移动过来
    lastBaseUpdate: Update,
    shared: { pending: null }, // 已清空
    effects: null
  },
  // ...其他属性
}
```

#### 2.3 处理 updateQueue

在 render 阶段，React 会处理 updateQueue 中的所有更新，计算新的 state：

```javascript
// 处理updateQueue的简化逻辑
function processUpdateQueue(workInProgress, instance, renderLanes) {
  const queue = workInProgress.updateQueue;
  let newState = queue.baseState;

  // 获取所有更新
  const firstUpdate = queue.firstBaseUpdate;

  // 应用所有更新
  let update = firstUpdate;
  while (update !== null) {
    // 检查优先级
    const updateLane = update.lane;
    if (isSubsetOfLanes(renderLanes, updateLane)) {
      // 应用更新
      const payload = update.payload;
      let partialState;

      if (typeof payload === 'function') {
        // 函数式更新
        partialState = payload.call(instance, newState);
      } else {
        // 对象式更新
        partialState = payload;
      }

      // 合并状态
      newState = Object.assign({}, newState, partialState);
    }

    update = update.next;
  }

  // 更新workInProgress的状态
  workInProgress.memoizedState = newState;

  // 清理updateQueue
  queue.baseState = newState;
  queue.firstBaseUpdate = null;
  queue.lastBaseUpdate = null;
}
```

在我们的例子中，应用更新后：

```
workInProgressCounterFiber {
  // ...其他属性
  memoizedState: { count: 1 }, // 更新后的状态
  updateQueue: {
    baseState: { count: 1 }, // 更新后的基础状态
    firstBaseUpdate: null, // 已清空
    lastBaseUpdate: null, // 已清空
    shared: { pending: null },
    effects: null
  },
  flags: 0b000000000000000000000000000100, // Update标记
  // ...其他属性
}
```

#### 2.4 调用 render 方法并比较子树

使用更新后的 state，React 调用 Counter 组件的 render 方法，获取新的 React 元素树：

```jsx
// 新的React元素树
{
  type: 'div',
  props: {
    children: [
      {
        type: 'p',
        props: {
          children: "Count: 1" // 这里变成了1
        }
      },
      {
        type: 'button',
        props: {
          onClick: Counter实例.handleClick,
          children: "Increment"
        }
      }
    ]
  }
}
```

React 会将这个新的元素树与 current Fiber 树进行比较（Diff 算法），创建或更新子 Fiber 节点。在这个例子中，主要的变化是 `p` 元素的文本内容从 "Count: 0" 变成了 "Count: 1"。

```
// 更新后的PFiber
workInProgressPFiber {
  tag: 5,
  type: 'p',
  stateNode: HTMLParagraphElement, // 同一个DOM节点
  pendingProps: { children: "Count: 1" }, // 新的props
  memoizedProps: { children: "Count: 0" }, // 旧的props
  return: workInProgressDivFiber,
  sibling: workInProgressButtonFiber,
  flags: 0b000000000000000000000000000100, // Update标记
  // ...其他属性
}
```

#### 2.5 完成 render 阶段

在完成整个 workInProgress 树的构建后，React 会标记需要更新的节点（通过 `flags` 字段）。在我们的例子中，Counter 组件和 p 元素都被标记为需要更新。

#### 2.6 commit 阶段 - 应用变更

在 commit 阶段，React 会：

1. 根据 `flags` 执行 DOM 更新操作
   - 对于 p 元素，更新其文本内容为 "Count: 1"
2. 调用生命周期方法
   - 对于 Counter 组件，调用 `componentDidUpdate`
3. 切换 current 树
   - workInProgress 树变成新的 current 树
   - 旧的 current 树变成 alternate

更新完成后，Counter 组件的 Fiber 节点变为：

```
CounterFiber {
  memoizedState: { count: 1 }, // 更新后的状态
  memoizedProps: {}, // 没有变化
  updateQueue: {
    baseState: { count: 1 },
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: { pending: null },
    effects: null
  },
  flags: 0, // 已清除所有副作用标记
  // ...其他属性
}
```

### 3. 连续更新 - 优先级调度的体现

假设用户快速连续点击了"Increment"按钮两次，在第一次更新尚未完成时，第二次点击事件触发。这种情况下，React 的优先级调度机制会发挥作用。

#### 3.1 第一次点击正在处理中

假设第一次点击触发的更新正在 render 阶段，workInProgress 树正在构建中：

```
workInProgressCounterFiber {
  memoizedState: { count: 1 }, // 正在计算中
  updateQueue: {
    baseState: { count: 0 },
    firstBaseUpdate: Update1, // count: 0 -> 1
    lastBaseUpdate: Update1,
    shared: { pending: null },
    effects: null
  },
  // ...其他属性
}
```

#### 3.2 第二次点击触发新的更新

此时，第二次点击触发了新的 `setState` 调用，创建了一个新的 Update 对象：

```
Update2 {
  eventTime: 123456790, // 新的时间戳
  lane: 0b0000000000000000000000000000001, // SyncLane
  tag: 0,
  payload: (state) => ({ count: state.count + 1 }),
  callback: null,
  next: null
}
```

这个新的 Update 被添加到 current Fiber 的 updateQueue 中：

```
CounterFiber {
  // ...其他属性
  updateQueue: {
    baseState: { count: 0 },
    firstBaseUpdate: null,
    lastBaseUpdate: null,
    shared: {
      pending: Update2 // 新的更新
    },
    effects: null
  },
  // ...其他属性
}
```

#### 3.3 合并更新

当 React 继续处理第一次更新时，它会检查是否有新的更新被添加到 shared.pending 中。如果有，它会将这些更新合并到 workInProgress 的 updateQueue 中：

```
workInProgressCounterFiber {
  // ...其他属性
  updateQueue: {
    baseState: { count: 0 },
    firstBaseUpdate: Update1, // 第一次更新
    lastBaseUpdate: Update2, // 第二次更新被附加到链表末尾
    shared: { pending: null }, // 已清空
    effects: null
  },
  // ...其他属性
}
```

#### 3.4 按顺序应用更新

React 会按照更新的添加顺序应用它们，确保状态计算的正确性：

1. 应用 Update1: `{ count: 0 }` -> `{ count: 1 }`
2. 应用 Update2: `{ count: 1 }` -> `{ count: 2 }`

最终的状态是 `{ count: 2 }`。

#### 3.5 优先级调度的体现

如果两次更新具有不同的优先级，例如第一次是低优先级（如 TransitionLane），第二次是高优先级（如 SyncLane），React 会：

1. 中断第一次更新的处理
2. 优先处理第二次高优先级更新
3. 完成后再回来处理第一次更新

这是通过比较 Update 对象的 `lane` 字段和当前渲染的 `renderLanes` 实现的：

```javascript
// 在processUpdateQueue中
if (!isSubsetOfLanes(renderLanes, updateLane)) {
  // 优先级不够，跳过此更新
  // 但保留在队列中，等待下次渲染
  const clone = {
    lane: updateLane,
    tag: update.tag,
    payload: update.payload,
    callback: update.callback,
    next: null,
  };

  // 将跳过的更新添加到新的baseUpdate链表
  if (newLastBaseUpdate === null) {
    newFirstBaseUpdate = newLastBaseUpdate = clone;
    newBaseState = newState;
  } else {
    newLastBaseUpdate = newLastBaseUpdate.next = clone;
  }

  // 更新workInProgress的lanes，以便下次渲染时处理这个更新
  newLanes = mergeLanes(newLanes, updateLane);
} else {
  // 此更新有足够的优先级，应用它
  // ...
}
```

## 七、深入理解 UpdateQueue 中的三个关键字段

在 React 的更新机制中，`updateQueue` 是一个核心数据结构，它负责存储和管理组件的状态更新。其中，`firstBaseUpdate`、`lastBaseUpdate` 和 `shared.pending` 这三个字段扮演着关键角色，它们共同协作，确保更新的正确处理和优先级调度。

### 1. UpdateQueue 的完整结构

首先，让我们回顾一下 `updateQueue` 的完整结构：

```typescript
export type UpdateQueue<State> = {
  baseState: State,                        // 基础状态，作为应用更新的起点
  firstBaseUpdate: Update<State> | null,   // 第一个基础更新
  lastBaseUpdate: Update<State> | null,    // 最后一个基础更新
  shared: {
    pending: Update<State> | null,         // 等待处理的更新
  },
  effects: Array<Update<State>> | null,    // 带有回调的更新列表
};
```

### 2. shared.pending - 更新的入口点

`shared.pending` 是所有新创建的更新的入口点。当调用 `setState` 或 `forceUpdate` 时，React 会：

1. 创建一个新的 `Update` 对象
2. 将其添加到 `shared.pending` 中，形成一个**环形链表**

```javascript
function enqueueUpdate(fiber, update) {
  const updateQueue = fiber.updateQueue;
  const sharedQueue = updateQueue.shared;
  const pending = sharedQueue.pending;
  
  if (pending === null) {
    // 这是第一个更新，创建一个自引用的环形链表
    update.next = update;
  } else {
    // 将新更新添加到环形链表中
    update.next = pending.next;
    pending.next = update;
  }
  
  // pending 始终指向最后一个更新
  sharedQueue.pending = update;
}
```

这里有一个重要的设计：`shared.pending` 总是指向环形链表中的**最后一个更新**，而这个更新的 `next` 指向**第一个更新**。这种设计使得 React 可以在 O(1) 时间内访问到链表的头部和尾部。

![shared.pending环形链表](https://i.imgur.com/XZrANjL.png)

### 3. firstBaseUpdate 和 lastBaseUpdate - 更新的处理队列

`firstBaseUpdate` 和 `lastBaseUpdate` 共同维护一个**单向链表**，称为"基础更新链表"。这个链表存储了需要按顺序应用的状态更新。

在处理更新时，React 会：

1. 将 `shared.pending` 中的环形链表**打断**并转移到基础更新链表中
2. 从 `baseState` 开始，按顺序应用基础更新链表中的所有更新
3. 计算出最终的状态

```javascript
function processUpdateQueue(workInProgress, props, instance, renderLanes) {
  const queue = workInProgress.updateQueue;
  
  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;

  // 检查是否有新的pending更新
  const pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    // 重置shared.pending
    queue.shared.pending = null;
    
    // 获取环形链表的最后一个和第一个更新
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = pendingQueue.next;
    
    // 打断环形链表
    lastPendingUpdate.next = null;
    
    // 将pending更新附加到基础更新链表
    if (lastBaseUpdate === null) {
      firstBaseUpdate = firstPendingUpdate;
    } else {
      lastBaseUpdate.next = firstPendingUpdate;
    }
    lastBaseUpdate = lastPendingUpdate;
  }

  // 处理基础更新链表
  if (firstBaseUpdate !== null) {
    let newState = queue.baseState;
    // ...应用更新的逻辑
  }
}
```

### 4. 三者之间的协作关系

这三个字段在更新过程中的协作关系如下：

1. **更新创建阶段**：
   - 新的更新被添加到 `shared.pending` 环形链表中
   - `current` 树和 `workInProgress` 树共享同一个 `shared` 对象，因此无论更新添加到哪个树，另一个树也能看到

2. **更新处理阶段**：
   - 在处理 `workInProgress` 树时，React 将 `shared.pending` 中的更新转移到 `firstBaseUpdate`/`lastBaseUpdate` 链表
   - 这个转移过程会清空 `shared.pending`，确保更新不会被重复处理
   - 然后按顺序应用 `firstBaseUpdate` 到 `lastBaseUpdate` 链表中的所有更新

3. **优先级调度**：
   - 如果某个更新的优先级不够，它会被跳过，但会保留在新的 `firstBaseUpdate`/`lastBaseUpdate` 链表中
   - 这确保了低优先级更新不会丢失，会在后续渲染中被处理

### 5. 具体示例：不同优先级更新的处理

让我们通过一个具体示例来说明这三个字段如何协作处理不同优先级的更新：

假设我们有三个更新：Update1（高优先级）、Update2（低优先级）和 Update3（高优先级），它们按顺序被添加到 `shared.pending` 中：

```
shared.pending -> Update3 -> Update1 -> Update2 -> Update3 (环形链表)
```

当 React 开始处理这些更新时：

1. **转移更新**：
   ```
   firstBaseUpdate = Update1
   lastBaseUpdate = Update3
   shared.pending = null
   ```
   
   更新链表变成：
   ```
   Update1 -> Update2 -> Update3 -> null
   ```

2. **处理更新**（假设当前渲染优先级只包含高优先级）：
   - 处理 Update1（高优先级）：应用更新，state 从 `{ count: 0 }` 变为 `{ count: 1 }`
   - 遇到 Update2（低优先级）：优先级不够，跳过，但保留在新的基础更新链表中
   - 处理 Update3（高优先级）：由于前面有被跳过的更新，即使这个更新优先级足够，也会被保留在新的基础更新链表中，以确保更新顺序的一致性

3. **更新完成后**：
   ```
   baseState = { count: 1 } // 应用了 Update1 后的状态
   firstBaseUpdate = Update2 // 保留了未处理的更新
   lastBaseUpdate = Update3
   ```

4. **下一次渲染**：
   - 从 `baseState` 开始，再次应用 `firstBaseUpdate` 到 `lastBaseUpdate` 链表中的所有更新
   - 这次会处理 Update2 和 Update3，最终状态变为 `{ count: 3 }`

### 6. 为什么需要这种复杂的设计？

这种设计看似复杂，但它解决了几个关键问题：

1. **并发更新**：允许在当前渲染进行时添加新的更新
2. **优先级调度**：支持跳过低优先级更新，优先处理高优先级更新
3. **更新顺序**：确保状态更新按照创建的顺序应用，保证状态计算的正确性
4. **状态一致性**：通过保留被跳过的更新，确保最终状态的一致性

### 7. 总结

`firstBaseUpdate`、`lastBaseUpdate` 和 `shared.pending` 这三个字段共同构成了 React 的更新处理机制：

- **shared.pending**：新更新的入口点，形成环形链表
- **firstBaseUpdate/lastBaseUpdate**：处理中的更新队列，形成单向链表
- 它们之间的转换过程确保了更新的正确处理和优先级调度

通过这种精心设计的机制，React 能够高效地处理复杂的状态更新场景，包括并发更新、优先级调度和中断恢复，为用户提供流畅的交互体验。

### 4. Fiber 属性在整个过程中的变化总结

通过上面的例子，我们可以看到 Fiber 节点的关键属性在整个过程中的变化：

#### memoizedState
- 初始化：存储组件的初始状态 `{ count: 0 }`
- 更新后：存储更新后的状态 `{ count: 1 }` 或 `{ count: 2 }`
- 作用：反映组件当前的状态，用于渲染和比较

#### updateQueue
- 初始化：为 `null` 或空队列
- 更新触发时：添加 Update 对象到 `shared.pending`
- 处理更新时：将 Update 对象从 `shared.pending` 移动到 `firstBaseUpdate`/`lastBaseUpdate`
- 更新完成后：清空 `firstBaseUpdate`/`lastBaseUpdate`，更新 `baseState`
- 作用：存储和管理组件的所有状态更新

#### flags
- 初始化：通常为 `0`（无副作用）
- 更新过程中：标记需要执行的操作，如 `Update`(0b000000000000000000000000000100)
- 提交后：重置为 `0`
- 作用：指示 React 在 commit 阶段需要对节点执行哪些操作

#### pendingProps 和 memoizedProps
- 初始化：`pendingProps` 设置为初始 props，`memoizedProps` 为 `null`
- 渲染完成后：`memoizedProps` 更新为与 `pendingProps` 相同
- 更新时：`pendingProps` 更新为新的 props
- 作用：比较 props 是否变化，决定是否需要更新

#### lanes
- 初始化：通常为 `0`（无优先级）
- 更新触发时：设置为对应的优先级通道（如 SyncLane）
- 作用：决定更新的优先级，影响调度顺序

#### alternate
- 作用：连接 current 树和 workInProgress 树中对应的 Fiber 节点
- 初始化：`current.alternate = workInProgress; workInProgress.alternate = current;`
- 提交后：交换角色，workInProgress 变为 current，原 current 变为 alternate

## 三、页面初始化过程

### 1. 创建 FiberRoot 和 RootFiber

当我们调用 `ReactDOM.render(<Counter />, document.getElementById('root'))` 时，React 会创建两个重要的对象：

- **FiberRoot**：整个应用的根节点，保存了应用的全局信息
- **RootFiber**：应用渲染内容的根节点，tag 为 HostRoot (3)

```
FiberRoot.current = RootFiber
RootFiber.stateNode = FiberRoot
```

### 2. 首次渲染的工作流程

React 的渲染过程分为两个主要阶段：**render 阶段**和 **commit 阶段**。

#### 2.1 render 阶段（协调阶段）

在 render 阶段，React 会构建一棵新的 Fiber 树（称为 workInProgress 树）：

1. **从 RootFiber 开始**，创建一个对应的 workInProgress fiber 节点
2. **向下遍历**，为每个 React 元素创建对应的 Fiber 节点
3. **对于类组件**：
   - 创建组件实例（`new YourComponent(props)`）
   - 将实例存储在 `fiber.stateNode` 中
   - 调用生命周期方法 `constructor`、`getDerivedStateFromProps`、`componentWillMount`（已废弃）
   - 调用 `render()` 方法获取子元素
   - 为子元素创建 Fiber 节点并连接到当前 Fiber 节点

4. **完成 Fiber 树构建**：
   - 为每个 Fiber 节点标记副作用（`flags`）
   - 收集所有需要执行副作用的 Fiber 节点

#### 2.2 commit 阶段（提交阶段）

render 阶段完成后，React 进入 commit 阶段，将变更实际应用到 DOM 上：

1. **before mutation 阶段**：
   - 调用 `getSnapshotBeforeUpdate` 生命周期
   - 调度 useEffect

2. **mutation 阶段**：
   - 根据 `flags` 执行 DOM 操作（插入、更新、删除）
   - 对于类组件，调用 `componentWillUnmount`（如果节点被删除）

3. **layout 阶段**：
   - 更新 `current` 指针，使 workInProgress 树变为 current 树
   - 对于类组件，调用 `componentDidMount`（首次挂载）或 `componentDidUpdate`（更新）
   - 执行 useLayoutEffect 的回调

## 四、Fiber 中的优先级更新机制

React 的优先级更新是通过 Lanes 模型实现的，这是 React 17 引入的新优先级模型，取代了之前的 ExpirationTime 模型。

### 1. Lanes 模型

Lanes 使用二进制位表示优先级，每个位代表一个特定的优先级通道：

```typescript
export type Lanes = number;
export type Lane = number;

// 优先级常量
export const NoLanes: Lanes = /*                        */ 0b0000000000000000000000000000000;
export const NoLane: Lane = /*                          */ 0b0000000000000000000000000000000;

export const SyncLane: Lane = /*                        */ 0b0000000000000000000000000000001;
export const SyncBatchedLane: Lane = /*                 */ 0b0000000000000000000000000000010;

export const InputDiscreteHydrationLane: Lane = /*      */ 0b0000000000000000000000000000100;
export const InputDiscreteLanes: Lanes = /*             */ 0b0000000000000000000000000011000;

export const InputContinuousHydrationLane: Lane = /*    */ 0b0000000000000000000000000100000;
export const InputContinuousLanes: Lanes = /*           */ 0b0000000000000000000000011000000;

// ...更多优先级常量
```

优先级从高到低大致为：

1. **SyncLane**：同步优先级，最高优先级，如用户输入、点击等
2. **InputDiscreteLanes**：离散输入事件，如点击、按键等
3. **InputContinuousLanes**：连续输入事件，如拖拽、滚动等
4. **DefaultLanes**：默认优先级，如初次渲染
5. **TransitionLanes**：过渡优先级，如 UI 切换动画
6. **RetryLanes**：重试优先级
7. **IdleLanes**：空闲优先级，最低优先级

### 2. 优先级更新的实现

#### 2.1 为更新分配优先级

当创建 Update 对象时，React 会根据触发更新的上下文为其分配一个优先级（Lane）。

#### 2.2 处理不同优先级的更新

在 render 阶段处理 `updateQueue` 时，React 会根据当前渲染的优先级（renderLanes）来决定哪些更新需要处理。

#### 2.3 调度不同优先级的渲染

React 使用 Scheduler 库来调度不同优先级的渲染工作：

1. **高优先级任务**（如 SyncLane）会立即执行，不会被打断
2. **中等优先级任务**会被调度到下一帧或几帧后执行
3. **低优先级任务**会在浏览器空闲时执行

## 五、双缓冲机制

React 使用双缓冲技术来实现高效的更新：

1. **current 树**：当前屏幕上显示的内容对应的 Fiber 树
2. **workInProgress 树**：正在构建的新 Fiber 树

每个 Fiber 节点通过 `alternate` 字段连接到另一棵树中对应的节点。当 workInProgress 树构建完成后，React 会通过一次性操作（称为"commit"）将其变为 current 树。

## 六、总结

通过跟踪 Fiber 属性的变化，我们可以看到 React 内部工作的完整流程：

1. **初始化**：创建 Fiber 树，设置初始状态和属性
2. **更新触发**：创建 Update 对象，加入 updateQueue
3. **状态计算**：处理 updateQueue，计算新的 memoizedState
4. **渲染比较**：基于新状态渲染，比较新旧元素，更新 Fiber 树
5. **提交变更**：根据 flags 执行 DOM 操作，调用生命周期方法
6. **完成更新**：切换 current 树，准备下一次更新

React Fiber 架构通过精心设计的数据结构和算法，实现了以下关键特性：

1. **增量渲染**：将渲染工作分解为小单元，可以被中断和恢复
2. **优先级调度**：通过 Lanes 模型为不同的更新分配优先级，确保重要的更新优先处理
3. **双缓冲渲染**：使用 current 树和 workInProgress 树，实现高效的更新
4. **可中断的协调**：render 阶段可以被中断，不会阻塞主线程
5. **副作用管理**：通过 flags 和 updateQueue 管理组件的副作用

通过深入理解 Fiber 属性的变化过程，我们可以更好地理解 React 的工作原理，编写更高效的 React 应用。