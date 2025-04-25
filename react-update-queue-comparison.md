# React UpdateQueue 结构对比：函数组件 vs 类组件

本文档详细比较React中函数组件和类组件的updateQueue结构差异，帮助深入理解React的状态更新机制。

## 概述

在React Fiber架构中，`updateQueue`是一个关键的数据结构，但在类组件和函数组件中有着完全不同的用途和结构。类组件的updateQueue用于存储状态更新，而函数组件的updateQueue则用于存储副作用。

## 类组件的UpdateQueue结构

### 基本结构

类组件的updateQueue是一个环形链表，用于存储通过`setState`和`forceUpdate`触发的更新：
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

// 更新对象结构
export type Update<State> = {
  eventTime: number,                       // 更新创建的时间
  lane: Lane,                              // 更新的优先级
  tag: UpdateTag,                          // 更新类型标记
  payload: any,                            // 更新内容（新状态或状态计算函数）
  callback: (() => mixed) | null,          // 更新完成后的回调函数
  next: Update<State> | null,              // 指向下一个更新
};

// 更新类型标记
export const UpdateState = 0;              // 普通状态更新
export const ReplaceState = 1;             // 替换状态
export const ForceUpdate = 2;              // 强制更新
export const CaptureUpdate = 3;            // 错误捕获更新
```

### 工作原理

1. **创建更新**：
   - 当调用`this.setState()`或`this.forceUpdate()`时，React创建一个新的Update对象
   - 更新被添加到`shared.pending`链表中

2. **处理更新**：
   - 在渲染阶段，React将`shared.pending`中的更新转移到`firstBaseUpdate`和`lastBaseUpdate`链表
   - 从`baseState`开始，按顺序应用所有更新，计算出最终状态
   - 高优先级更新可以"插队"，低优先级更新会被跳过并保留到下一次渲染

3. **提交更新**：
   - 计算出的最终状态被设置为组件实例的`this.state`
   - 带有回调的更新被收集到`effects`数组中
   - 在提交阶段，执行所有回调函数

### 示例

```typescript
// 类组件中的setState调用
class Counter extends React.Component {
  state = { count: 0 };

  increment = () => {
    // 创建一个更新对象并加入updateQueue
    this.setState(state => ({ count: state.count + 1 }),
      () => console.log('Updated!'));
  };

  render() {
    return <button onClick={this.increment}>{this.state.count}</button>;
  }
}
```

当调用`this.setState()`时，React会：
1. 创建一个`tag`为`UpdateState`的Update对象
2. 将状态计算函数和回调函数存储在Update对象中
3. 将Update对象添加到updateQueue的`shared.pending`链表中

## 函数组件的UpdateQueue结构

### 函数组件Fiber节点的UpdateQueue

函数组件的Fiber节点上的`updateQueue`与类组件完全不同，它**专门用于存储副作用（Effects）**，而不是状态更新。函数组件的状态更新是通过Hook内部的queue来处理的。

```typescript
// 函数组件Fiber节点的UpdateQueue结构
export type UpdateQueue = {
  lastEffect: Effect | null,  // 指向副作用链表的最后一个节点
};

// Effect结构
export type Effect = {
  tag: HookEffectTag,         // 副作用类型标记
  create: () => (() => void) | void,  // 创建函数，返回清理函数或void
  destroy: (() => void) | void,       // 清理函数
  deps: Array<mixed> | null,  // 依赖数组
  next: Effect | null,        // 指向下一个副作用
};

// 副作用类型标记
export const NoEffect = /*             */ 0b00000;
export const UnmountSnapshot = /*      */ 0b00001;
export const UnmountMutation = /*      */ 0b00010;
export const MountMutation = /*        */ 0b00100;
export const UnmountLayout = /*        */ 0b01000;
export const MountLayout = /*          */ 0b10000;
export const MountPassive = /*         */ 0b100000;
export const UnmountPassive = /*       */ 0b1000000;
```
### 函数组件状态更新的处理

函数组件的状态更新不是通过Fiber节点的updateQueue处理的，而是通过各个Hook内部的queue来处理：

```typescript
// Hook结构
export type Hook = {
  memoizedState: any,                      // Hook的状态
  baseState: any,                          // 基础状态
  baseQueue: Update<any, any> | null,      // 基础更新队列
  queue: UpdateQueue<any, any> | null,     // 更新队列
  next: Hook | null,                       // 指向下一个Hook
};

// useState/useReducer的UpdateQueue结构
type UpdateQueue<S, A> = {
  pending: Update<S, A> | null,            // 等待处理的更新
  dispatch: (A) => void | null,            // 分发函数（如setState）
  lastRenderedReducer: ((S, A) => S) | null, // 上次渲染使用的reducer
  lastRenderedState: S | null,             // 上次渲染的状态
};

// 更新对象结构
type Update<S, A> = {
  lane: Lane,                              // 更新的优先级
  action: A,                               // 动作（新状态或状态计算函数）
  eagerReducer: ((S, A) => S) | null,      // 急切计算的reducer
  eagerState: S | null,                    // 急切计算的状态
  next: Update<S, A> | null,               // 指向下一个更新
  hasEagerState: boolean,                  // 是否有急切计算的状态
};
```

### 工作原理

#### 副作用处理（Fiber节点的updateQueue）

1. **收集副作用**：
   - 在函数组件渲染过程中，React收集所有的副作用（如useEffect、useLayoutEffect）
   - 这些副作用形成一个环形链表，通过`next`字段连接
   - 链表的最后一个节点由Fiber节点的`updateQueue.lastEffect`字段引用

2. **执行副作用**：
   - 在提交阶段，React遍历副作用链表
   - 根据副作用的`tag`字段和当前阶段，决定是否执行副作用
   - 执行顺序：先执行所有的卸载操作，再执行所有的挂载操作

#### 状态更新处理（Hook内部的queue）

1. **创建更新**：
   - 当调用`setState`函数（由`useState`返回）或`dispatch`函数（由`useReducer`返回）时，React创建一个新的Update对象
   - 更新被添加到对应Hook的`queue.pending`链表中

2. **处理更新**：
   - 在渲染阶段，React从Hook的`baseState`开始，按顺序应用所有更新
   - 计算出的最终状态被存储在Hook的`memoizedState`中
   - 与类组件类似，高优先级更新可以"插队"

### 示例

```typescript
// 函数组件中的useEffect（影响Fiber节点的updateQueue）
function Example() {
  useEffect(() => {
    console.log('Component mounted or updated');
    return () => {
      console.log('Cleanup before next effect or unmount');
    };
  }, []);

  return <div>Example Component</div>;
}

// 函数组件中的useState（使用Hook内部的queue）
function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    // 创建一个更新对象并加入Hook的queue
    setCount(c => c + 1);
  };

  return <button onClick={increment}>{count}</button>;
}
```

## 关键差异对比

| 特性 | 类组件 | 函数组件 |
|------|--------|----------|
| **Fiber节点的updateQueue用途** | 存储状态更新 | 仅存储副作用（Effects） |
| **状态更新存储位置** | Fiber节点的updateQueue | 各个Hook内部的queue |
| **更新触发** | 通过`this.setState`和`this.forceUpdate` | 通过`useState`返回的setter函数或`useReducer`返回的dispatch函数 |
| **状态存储** | 组件实例的`this.state` | 各个Hook的`memoizedState` |
| **更新回调** | 支持`setState(updater, callback)` | 不直接支持，需使用`useEffect`实现 |
| **批量更新** | 自动批量处理同一事件中的多次`setState` | 自动批量处理同一事件中的多次状态更新 |
| **更新合并** | 浅合并对象状态 | 完全替换状态（不合并） |
| **优先级处理** | 支持优先级调度 | 支持优先级调度 |

## 内部实现差异

### 类组件的实现

类组件的updateQueue直接挂在Fiber节点上，所有状态更新都通过这个队列处理：

```typescript
// 简化的类组件更新处理
function processUpdateQueue(workInProgress, props, instance, renderLanes) {
  const queue = workInProgress.updateQueue;

  // 处理baseUpdate链表中的所有更新
  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;

  // 合并pending更新到baseUpdate链表
  const pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    queue.shared.pending = null;
    // 将pendingQueue合并到baseUpdate链表
    // ...
  }

  // 应用所有更新
  if (firstBaseUpdate !== null) {
    let newState = queue.baseState;
    let newLanes = NoLanes;

    // 遍历更新链表，应用每个更新
    let update = firstBaseUpdate;
    do {
      const updateLane = update.lane;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够，跳过此更新
        // ...
      } else {
        // 应用更新
        const action = update.payload;
        if (typeof action === 'function') {
          // 使用函数计算新状态
          newState = action(newState, props);
        } else {
          // 直接使用对象更新，进行浅合并
          newState = Object.assign({}, newState, action);
        }
      }
      update = update.next;
    } while (update !== null);

    // 更新queue的baseState和baseUpdate
    // ...
  }

  // 设置最终计算的状态
  workInProgress.memoizedState = newState;
}
```

### 函数组件的实现

函数组件有两套不同的机制：

1. **副作用处理**（通过Fiber节点的updateQueue）：

```typescript
// 简化的函数组件副作用处理
function commitHookEffectListUnmount(tag, finishedWork) {
  const updateQueue = finishedWork.updateQueue;
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;

  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;

    do {
      // 检查副作用标记是否匹配当前操作
      if ((effect.tag & tag) !== NoHookEffect) {
        // 执行副作用的清理函数
        const destroy = effect.destroy;
        effect.destroy = undefined;

        if (destroy !== undefined) {
          destroy();
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

2. **状态更新处理**（通过Hook内部的queue）：

```typescript
// 简化的useState Hook更新处理
function updateReducer(reducer, initialArg, init) {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  // 获取当前reducer
  reducer = queue.lastRenderedReducer;

  // 获取baseState和baseQueue
  let baseState = hook.baseState;
  let baseQueue = hook.baseQueue;

  // 合并pending更新到baseQueue
  const pendingQueue = queue.pending;
  if (pendingQueue !== null) {
    queue.pending = null;
    // 将pendingQueue合并到baseQueue
    // ...
  }

  // 应用所有更新
  if (baseQueue !== null) {
    let newState = baseState;
    let update = baseQueue.next;

    // 遍历更新链表，应用每个更新
    do {
      const updateLane = update.lane;
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够，跳过此更新
        // ...
      } else {
        // 应用更新
        const action = update.action;
        newState = reducer(newState, action);
      }
      update = update.next;
    } while (update !== baseQueue.next);

    // 更新hook的baseState和baseQueue
    // ...
  }

  // 设置最终计算的状态
  hook.memoizedState = newState;
  hook.baseState = newBaseState;
  hook.baseQueue = newBaseQueue;

  return [hook.memoizedState, queue.dispatch];
}
```

## 实际影响和最佳实践

### 状态更新与副作用处理的差异

1. **职责分离**：
   - 类组件：单一的updateQueue同时处理状态更新和副作用（通过回调）
   - 函数组件：Fiber节点的updateQueue只处理副作用，状态更新由Hook内部的queue处理

2. **执行时机**：
   - 类组件的状态更新：在渲染阶段处理
   - 函数组件的状态更新：在渲染阶段处理
   - 函数组件的副作用：在提交阶段执行

### 最佳实践

1. **类组件**：
   - 使用函数式更新避免依赖当前状态：`this.setState(state => ({ count: state.count + 1 }))`
   - 利用第二个参数处理更新后的逻辑：`this.setState(newState, callback)`

2. **函数组件**：
   - 合理使用`useEffect`和`useLayoutEffect`处理副作用
   - 使用多个`useState`管理独立的状态片段，而不是一个大对象
   - 使用`useReducer`处理复杂的状态逻辑

## 总结

类组件和函数组件在处理状态更新和副作用方面有着根本的不同：

- **类组件**：使用Fiber节点的updateQueue管理状态更新和副作用回调
- **函数组件**：使用Fiber节点的updateQueue仅管理副作用，而状态更新则通过Hook内部的queue处理

这种设计反映了React团队对函数组件的重新思考，将状态管理和副作用处理进行了更清晰的分离，使得代码更加模块化和可维护。理解这些差异有助于更好地使用React，并在适当的场景选择合适的组件类型和状态管理方式。
