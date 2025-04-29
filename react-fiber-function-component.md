# 以 Fiber 为线索，详解函数组件的 React 底层工作原理

React 的 Fiber 架构是 React 16 引入的核心重构，它彻底改变了 React 的内部实现。本文将以 Fiber 为线索，详细阐述函数组件在 React 底层从页面初始化到页面更新的整个过程，以及 Fiber 各个属性的作用和 React 如何实现优先级更新。

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
  // 对于函数组件，这是null
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
  // 对于函数组件，这是Hook链表的头部
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

对于函数组件，有几个特别重要的字段：

1. **tag**: 值为 0 (FunctionComponent)
2. **type**: 指向函数组件本身
3. **memoizedState**: 存储 Hook 链表的头部
4. **updateQueue**: 存储副作用（Effect）链表

## 二、函数组件的 Hook 机制

函数组件通过 Hook 来管理状态和副作用。每个 Hook 都是一个对象，它们通过 `next` 字段形成一个链表，存储在 Fiber 节点的 `memoizedState` 字段中。

### Hook 的基本结构

```typescript
export type Hook = {
  memoizedState: any, // Hook自己的状态
  baseState: any, // 基础状态
  baseQueue: Update<any, any> | null, // 基础更新队列
  queue: UpdateQueue<any, any> | null, // 更新队列
  next: Hook | null, // 指向下一个Hook
};
```

不同类型的 Hook 有不同的 `memoizedState` 结构：

1. **useState/useReducer**:
   ```typescript
   // useState的memoizedState直接存储状态值
   hook.memoizedState = [state, dispatch];
   
   // 更新队列结构
   type UpdateQueue<S, A> = {
     pending: Update<S, A> | null,
     dispatch: (A) => void | null,
     lastRenderedReducer: ((S, A) => S) | null,
     lastRenderedState: S | null,
   };
   ```

2. **useEffect/useLayoutEffect**:
   ```typescript
   // useEffect的memoizedState结构
   hook.memoizedState = {
     tag: HookEffectTag,
     create: () => (() => void) | void, // 副作用函数
     destroy: (() => void) | void, // 清理函数
     deps: Array<mixed> | null, // 依赖数组
     next: Effect | null, // 指向下一个Effect
   };
   ```

3. **useRef**:
   ```typescript
   // useRef的memoizedState结构
   hook.memoizedState = { current: initialValue };
   ```

4. **useMemo/useCallback**:
   ```typescript
   // useMemo的memoizedState结构
   hook.memoizedState = [value, deps];
   
   // useCallback的memoizedState结构
   hook.memoizedState = [callback, deps];
   ```

## 三、函数组件的初始化过程

### 1. 创建 FiberRoot 和 RootFiber

当我们调用 `ReactDOM.render(<App />, document.getElementById('root'))` 或 `ReactDOM.createRoot(document.getElementById('root')).render(<App />)` 时，React 会创建两个重要的对象：

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
3. **对于函数组件**：
   - 创建一个 Fiber 节点，tag 为 FunctionComponent (0)
   - 将函数组件本身存储在 `type` 字段中
   - 调用函数组件，获取子元素
   - 在调用过程中处理所有的 Hook 调用
   - 为子元素创建 Fiber 节点并连接到当前 Fiber 节点

4. **完成 Fiber 树构建**：
   - 为每个 Fiber 节点标记副作用（`flags`）
   - 收集所有需要执行副作用的 Fiber 节点

#### 2.2 函数组件的 Hook 处理

当 React 调用函数组件时，会进入一个特殊的上下文，使得 Hook 可以正确地与当前的 Fiber 节点关联：

```javascript
// 简化的renderWithHooks函数
function renderWithHooks(current, workInProgress, Component, props) {
  // 设置当前正在渲染的Fiber
  currentlyRenderingFiber = workInProgress;
  
  // 重置Hook链表
  workInProgress.memoizedState = null;
  workInProgress.updateQueue = null;
  
  // 设置Hook上下文（根据是首次渲染还是更新）
  ReactCurrentDispatcher.current = 
    current === null || current.memoizedState === null
      ? HooksDispatcherOnMount  // 首次渲染
      : HooksDispatcherOnUpdate; // 更新渲染
  
  // 调用函数组件，这将触发Hook的调用
  let children = Component(props);
  
  // 重置Hook上下文
  ReactCurrentDispatcher.current = ContextOnlyDispatcher;
  
  return children;
}
```

每当函数组件中调用一个 Hook 函数（如 `useState`、`useEffect` 等），React 会：

1. 创建一个新的 Hook 对象
2. 将其添加到当前 Fiber 节点的 Hook 链表中
3. 根据 Hook 类型执行相应的逻辑

例如，对于 `useState` Hook：

```javascript
// 首次渲染时的useState实现（简化版）
function mountState(initialState) {
  // 创建一个新的Hook对象
  const hook = mountWorkInProgressHook();
  
  // 计算初始状态
  let initialStateForHook = 
    typeof initialState === 'function' 
      ? initialState() 
      : initialState;
  
  // 设置Hook的状态
  hook.memoizedState = initialStateForHook;
  hook.baseState = initialStateForHook;
  
  // 创建更新队列
  const queue = {
    pending: null,
    dispatch: null,
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialStateForHook,
  };
  hook.queue = queue;
  
  // 创建dispatch函数
  const dispatch = (queue.dispatch = dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue
  ));
  
  // 返回状态和dispatch函数
  return [hook.memoizedState, dispatch];
}
```

对于 `useEffect` Hook：

```javascript
// 首次渲染时的useEffect实现（简化版）
function mountEffect(create, deps) {
  // 创建一个新的Hook对象
  const hook = mountWorkInProgressHook();
  
  // 获取依赖项
  const nextDeps = deps === undefined ? null : deps;
  
  // 标记当前Fiber需要执行副作用
  currentlyRenderingFiber.flags |= PassiveEffect;
  
  // 创建Effect对象
  hook.memoizedState = pushEffect(
    HookPassive | HookHasEffect,
    create,
    undefined,
    nextDeps
  );
}

// 将Effect添加到Fiber的updateQueue中
function pushEffect(tag, create, destroy, deps) {
  const effect = {
    tag,
    create,
    destroy,
    deps,
    next: null,
  };
  
  // 将Effect添加到updateQueue链表中
  let componentUpdateQueue = currentlyRenderingFiber.updateQueue;
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  
  return effect;
}
```

#### 2.3 commit 阶段（提交阶段）

render 阶段完成后，React 进入 commit 阶段，将变更实际应用到 DOM 上：

1. **before mutation 阶段**：
   - 调度 useEffect（将它们加入调度队列，但不立即执行）

2. **mutation 阶段**：
   - 根据 `flags` 执行 DOM 操作（插入、更新、删除）
   - 执行 `useLayoutEffect` 的清理函数（如果有）

3. **layout 阶段**：
   - 更新 `current` 指针，使 workInProgress 树变为 current 树
   - 执行 `useLayoutEffect` 的副作用函数
   - 在浏览器绘制后异步执行 `useEffect` 的清理函数和副作用函数

在 commit 阶段结束后，屏幕上会显示新的 UI，首次渲染完成。

## 四、函数组件的更新过程

### 1. 触发更新

函数组件中的更新可以通过以下方式触发：

- 调用 `useState` 返回的 setter 函数
- 调用 `useReducer` 返回的 dispatch 函数
- 父组件重新渲染导致 props 变化
- 上下文（Context）变化

当调用 `setState` 或 `dispatch` 时，React 会：

```javascript
// 简化的dispatchAction函数
function dispatchAction(fiber, queue, action) {
  // 计算过期时间（优先级相关）
  const lane = requestUpdateLane(fiber);
  
  // 创建一个Update对象
  const update = {
    lane,
    action,
    hasEagerState: false,
    eagerState: null,
    next: null,
  };
  
  // 尝试进行急切的状态计算
  if (fiber.lanes === NoLanes && fiber.alternate === null) {
    const currentState = queue.lastRenderedState;
    const eagerState = basicStateReducer(currentState, action);
    update.hasEagerState = true;
    update.eagerState = eagerState;
    
    if (Object.is(eagerState, currentState)) {
      // 状态没有变化，不需要调度更新
      return;
    }
  }
  
  // 将Update对象加入队列
  const pending = queue.pending;
  if (pending === null) {
    update.next = update;
  } else {
    update.next = pending.next;
    pending.next = update;
  }
  queue.pending = update;
  
  // 调度更新
  scheduleUpdateOnFiber(fiber, lane);
}
```

### 2. 更新的工作流程

更新过程与首次渲染类似，也分为 render 阶段和 commit 阶段，但有一些重要区别：

#### 2.1 render 阶段（协调阶段）

1. **从 FiberRoot 开始**，创建 workInProgress 树
2. **复用现有 Fiber 节点**：
   - 通过 `alternate` 字段找到对应的 current Fiber 节点
   - 尽可能复用现有节点，减少创建新节点的开销

3. **处理函数组件的更新**：
   - 再次调用函数组件，获取新的子元素
   - 在调用过程中处理所有的 Hook 更新

4. **Diff 算法**：
   - 比较 current Fiber 的子节点和新的子元素
   - 标记需要插入、更新或删除的节点
   - 为这些节点设置相应的 `flags`

#### 2.2 函数组件的 Hook 更新

当 React 再次调用函数组件时，会使用不同的 Hook 实现（HooksDispatcherOnUpdate）：

```javascript
// 更新时的useState实现（简化版）
function updateState(initialState) {
  // 获取当前Hook
  const hook = updateWorkInProgressHook();
  
  // 获取更新队列
  const queue = hook.queue;
  
  // 处理更新
  const pending = queue.pending;
  if (pending !== null) {
    // 应用所有更新
    const first = pending.next;
    let newState = hook.baseState;
    
    let update = first;
    do {
      // 检查更新的优先级
      const updateLane = update.lane;
      if (isSubsetOfLanes(renderLanes, updateLane)) {
        // 应用更新
        const action = update.action;
        if (update.hasEagerState) {
          newState = update.eagerState;
        } else {
          newState = typeof action === 'function'
            ? action(newState)
            : action;
        }
      }
      
      update = update.next;
    } while (update !== first);
    
    // 更新Hook的状态
    hook.memoizedState = newState;
    
    // 清空更新队列
    queue.pending = null;
  }
  
  // 返回最新状态和dispatch函数
  return [hook.memoizedState, queue.dispatch];
}
```

对于 `useEffect` Hook：

```javascript
// 更新时的useEffect实现（简化版）
function updateEffect(create, deps) {
  // 获取当前Hook
  const hook = updateWorkInProgressHook();
  
  // 获取新的依赖项
  const nextDeps = deps === undefined ? null : deps;
  
  let destroy = undefined;
  
  if (currentHook !== null) {
    // 获取上一次的Effect
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    
    if (nextDeps !== null) {
      // 比较依赖项
      const prevDeps = prevEffect.deps;
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 依赖项没有变化，不需要重新执行副作用
        hook.memoizedState = pushEffect(HookPassive, create, destroy, nextDeps);
        return;
      }
    }
  }
  
  // 依赖项变化，需要重新执行副作用
  currentlyRenderingFiber.flags |= PassiveEffect;
  
  hook.memoizedState = pushEffect(
    HookPassive | HookHasEffect,
    create,
    destroy,
    nextDeps
  );
}
```

#### 2.3 commit 阶段（提交阶段）

commit 阶段与首次渲染类似，但会执行不同的副作用：

1. **before mutation 阶段**：
   - 调度 useEffect 的清理函数（对于依赖项变化的 effect）

2. **mutation 阶段**：
   - 执行 DOM 更新操作
   - 执行 useLayoutEffect 的清理函数（对于依赖项变化的 effect）

3. **layout 阶段**：
   - 更新 `current` 指针
   - 执行 useLayoutEffect 的副作用函数（对于依赖项变化的 effect）
   - 在浏览器绘制后异步执行 useEffect 的清理函数和副作用函数

## 五、函数组件中的优先级更新机制

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

### 2. 函数组件中的优先级更新实现

#### 2.1 为更新分配优先级

当调用 `setState` 或 `dispatch` 时，React 会根据触发更新的上下文为其分配一个优先级（Lane）：

```javascript
function dispatchAction(fiber, queue, action) {
  // 根据当前执行上下文获取优先级
  const lane = requestUpdateLane(fiber);
  
  // 创建一个带有优先级的Update对象
  const update = {
    lane,
    action,
    // ...其他字段
  };
  
  // ...将Update加入队列并调度更新
}

function requestUpdateLane(fiber) {
  // 获取当前模式
  const mode = fiber.mode;
  
  // 根据当前执行上下文决定优先级
  if ((mode & ConcurrentMode) === NoMode) {
    // 非并发模式，使用同步优先级
    return SyncLane;
  }
  
  // 获取当前调度优先级
  const schedulerPriority = getCurrentSchedulerPriority();
  
  // 将调度优先级转换为Lane
  if (schedulerPriority === UserBlockingPriority) {
    return pickArbitraryLane(InputDiscreteLanes);
  }
  if (schedulerPriority === NormalPriority) {
    return pickArbitraryLane(DefaultLanes);
  }
  
  // 其他情况返回默认优先级
  return DefaultLane;
}
```

#### 2.2 处理不同优先级的更新

在函数组件的 Hook 更新过程中，React 会根据当前渲染的优先级（renderLanes）来决定哪些更新需要处理：

```javascript
function updateReducer(reducer, initialArg, init) {
  // 获取当前Hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  // 处理更新队列
  const pending = queue.pending;
  if (pending !== null) {
    // 应用所有更新
    const first = pending.next;
    let newState = hook.baseState;
    
    let update = first;
    do {
      // 检查更新的优先级
      const updateLane = update.lane;
      if (isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级足够，应用更新
        const action = update.action;
        newState = reducer(newState, action);
      } else {
        // 优先级不够，跳过此更新
        // 但保留在队列中，等待下次渲染
      }
      
      update = update.next;
    } while (update !== first);
    
    // 更新Hook的状态
    hook.memoizedState = newState;
    
    // 清空更新队列
    queue.pending = null;
  }
  
  // 返回最新状态和dispatch函数
  return [hook.memoizedState, queue.dispatch];
}
```

#### 2.3 useTransition 和 useDeferredValue

React 18 引入了两个新的 Hook，用于更细粒度地控制更新优先级：

1. **useTransition**：将状态更新标记为非紧急的过渡更新
   ```javascript
   const [isPending, startTransition] = useTransition();
   
   // 使用示例
   function handleClick() {
     startTransition(() => {
       // 这里的状态更新会被标记为低优先级
       setCount(count + 1);
     });
   }
   ```

2. **useDeferredValue**：创建一个延迟更新的值
   ```javascript
   const deferredValue = useDeferredValue(value);
   
   // 使用示例
   function SearchResults({ query }) {
     // query的更新会被延迟，直到高优先级更新完成
     const deferredQuery = useDeferredValue(query);
     // 使用deferredQuery进行搜索
   }
   ```

这两个 Hook 的实现都是基于 Lanes 模型，它们将更新标记为 TransitionLanes 优先级，使得这些更新可以被高优先级更新打断。

#### 2.4 调度不同优先级的渲染

React 使用 Scheduler 库来调度不同优先级的渲染工作：

1. **高优先级任务**（如 SyncLane）会立即执行，不会被打断
2. **中等优先级任务**会被调度到下一帧或几帧后执行
3. **低优先级任务**会在浏览器空闲时执行

Scheduler 使用 `requestIdleCallback` 的 polyfill 实现，通过 `MessageChannel` 在浏览器的每一帧中执行一小部分工作，确保不阻塞主线程，保持 UI 的响应性。

## 六、函数组件中的 Fiber 属性详解

### 1. memoizedState

在函数组件中，`memoizedState` 字段存储的是 Hook 链表的头部，而不是组件的状态对象（与类组件不同）。

每个 Hook 都有自己的 `memoizedState` 字段，用于存储该 Hook 的状态：

- **useState**: `[state, dispatch]`
- **useReducer**: `[state, dispatch]`
- **useEffect**: Effect 对象
- **useRef**: `{ current: value }`
- **useMemo**: `[value, deps]`
- **useCallback**: `[callback, deps]`

### 2. updateQueue

在函数组件中，`updateQueue` 字段主要用于存储副作用（Effect）链表，而不是状态更新（与类组件不同）。

Effect 链表是一个环形链表，每个 Effect 对象包含：

```typescript
type Effect = {
  tag: HookEffectTag, // 副作用类型标记
  create: () => (() => void) | void, // 副作用函数
  destroy: (() => void) | void, // 清理函数
  deps: Array<mixed> | null, // 依赖数组
  next: Effect | null, // 指向下一个Effect
};
```

React 在 commit 阶段会遍历这个链表，执行相应的副作用。

### 3. flags

`flags` 字段用于标记 Fiber 节点上需要执行的操作：

```typescript
// 副作用标记
export const NoFlags = /*                      */ 0b000000000000000000000000000;
export const PerformedWork = /*                */ 0b000000000000000000000000001;
export const Placement = /*                    */ 0b000000000000000000000000010;
export const Update = /*                       */ 0b000000000000000000000000100;
export const PlacementAndUpdate = /*           */ 0b000000000000000000000000110;
export const Deletion = /*                     */ 0b000000000000000000000001000;
export const ContentReset = /*                 */ 0b000000000000000000000010000;
export const Callback = /*                     */ 0b000000000000000000000100000;
export const DidCapture = /*                   */ 0b000000000000000000001000000;
export const Ref = /*                          */ 0b000000000000000000010000000;
export const Snapshot = /*                     */ 0b000000000000000000100000000;
export const Passive = /*                      */ 0b000000000000000001000000000;
// ...更多标记
```

对于函数组件，常见的 flags 包括：

- **Update**：组件需要更新
- **Placement**：组件需要插入到 DOM 中
- **Deletion**：组件需要从 DOM 中删除
- **Passive**：组件有 useEffect 副作用需要执行
- **LayoutEffect**：组件有 useLayoutEffect 副作用需要执行

### 4. lanes

`lanes` 字段用于表示 Fiber 节点的优先级。在函数组件中，当调用 `setState` 或 `dispatch` 时，React 会根据当前的执行上下文为更新分配一个优先级，并将其存储在 Update 对象的 `lane` 字段中。

在处理更新时，React 会比较 Update 的 `lane` 和当前渲染的 `renderLanes`，决定是否应用该更新。

### 5. alternate

`alternate` 字段连接 current 树和 workInProgress 树中对应的 Fiber 节点。这是 React 实现双缓冲渲染的关键机制。

当 workInProgress 树构建完成后，React 会通过一次性操作（称为"commit"）将其变为 current 树，同时更新 `alternate` 指针，为下一次更新做准备。

## 七、函数组件与类组件的区别

函数组件和类组件在 React 内部的处理有一些关键区别：

### 1. 状态管理

- **类组件**：状态存储在组件实例的 `this.state` 中，`memoizedState` 直接引用状态对象
- **函数组件**：状态分散在各个 Hook 中，`memoizedState` 存储 Hook 链表

### 2. 更新队列

- **类组件**：`updateQueue` 存储状态更新
- **函数组件**：`updateQueue` 主要存储副作用，状态更新存储在各个 Hook 的 `queue` 字段中

### 3. 生命周期

- **类组件**：有完整的生命周期方法
- **函数组件**：通过 useEffect 和 useLayoutEffect 模拟生命周期

### 4. 实例

- **类组件**：有组件实例，存储在 `stateNode` 中
- **函数组件**：没有实例，`stateNode` 为 null

### 5. 渲染方式

- **类组件**：调用 `render` 方法获取子元素
- **函数组件**：直接调用函数获取子元素

## 八、总结

通过深入理解函数组件的 Fiber 架构和 Hook 机制，我们可以看到 React 内部工作的完整流程：

1. **初始化**：创建 Fiber 树，调用函数组件，处理 Hook，设置初始状态
2. **更新触发**：调用 `setState` 或 `dispatch`，创建 Update 对象，分配优先级
3. **状态计算**：再次调用函数组件，处理 Hook 更新，计算新的状态
4. **渲染比较**：比较新旧元素，更新 Fiber 树
5. **提交变更**：根据 flags 执行 DOM 操作，调用副作用函数
6. **完成更新**：切换 current 树，准备下一次更新

React 的 Fiber 架构和 Lanes 模型为函数组件提供了强大的基础，支持增量渲染、优先级调度和并发模式，使得 React 应用能够处理复杂的 UI 更新，同时保持良好的用户体验。

通过 Hook 机制，函数组件能够拥有与类组件相同的能力，同时提供更简洁、更灵活的 API，使得状态逻辑更容易复用和测试。这也是为什么 React 团队推荐使用函数组件而不是类组件的原因。