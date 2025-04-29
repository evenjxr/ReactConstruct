# React类组件UpdateQueue中firstBaseUpdate和lastBaseUpdate详解

在React类组件的更新机制中，`firstBaseUpdate`和`lastBaseUpdate`是UpdateQueue结构中的两个关键字段，它们共同维护了一个更新链表，对于理解React的状态更新过程至关重要。本文将详细解析这两个字段的作用、工作原理以及它们在React并发渲染中的重要性。

## UpdateQueue结构回顾

首先，让我们回顾一下类组件的UpdateQueue结构：

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

## firstBaseUpdate和lastBaseUpdate的基本作用

`firstBaseUpdate`和`lastBaseUpdate`这两个字段共同维护了一个单向链表，称为"基础更新链表"(base update list)：

- **firstBaseUpdate**：指向基础更新链表的第一个节点
- **lastBaseUpdate**：指向基础更新链表的最后一个节点

这个链表存储了需要按顺序应用的状态更新，每次渲染时，React会从`baseState`开始，按顺序应用这个链表中的所有更新，计算出最终的状态。

## 更新处理流程

### 1. 更新入队过程

当调用`setState`或`forceUpdate`时，React会创建一个新的Update对象，并将其添加到`shared.pending`链表中：

```typescript
function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  const updateQueue = fiber.updateQueue;
  if (updateQueue === null) {
    // 只有在组件已卸载的情况下才会发生
    return;
  }

  const sharedQueue = updateQueue.shared;
  const pending = sharedQueue.pending;
  
  if (pending === null) {
    // 这是第一个更新，创建一个循环链表
    update.next = update;
  } else {
    // 将新更新添加到循环链表中
    update.next = pending.next;
    pending.next = update;
  }
  
  // pending总是指向最后一个更新
  sharedQueue.pending = update;
}
```

这样，所有通过`setState`创建的更新都会先进入`shared.pending`链表，形成一个环形链表。

### 2. 更新处理过程

在渲染阶段，React会处理这些更新：

```typescript
function processUpdateQueue<State>(
  workInProgress: Fiber,
  props: any,
  instance: any,
  renderLanes: Lanes,
) {
  const queue = workInProgress.updateQueue;
  
  // 获取firstBaseUpdate和lastBaseUpdate
  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;
  
  // 检查是否有新的pending更新
  const pendingQueue = queue.shared.pending;
  if (pendingQueue !== null) {
    // 重置shared.pending
    queue.shared.pending = null;
    
    // 将pendingQueue中的更新转移到baseUpdate链表
    const lastPendingUpdate = pendingQueue;
    const firstPendingUpdate = lastPendingUpdate.next;
    // 断开环形链表
    lastPendingUpdate.next = null;
    
    // 将pending更新附加到baseUpdate链表
    if (lastBaseUpdate === null) {
      firstBaseUpdate = firstPendingUpdate;
    } else {
      lastBaseUpdate.next = firstPendingUpdate;
    }
    lastBaseUpdate = lastPendingUpdate;
    
    // 更新workInProgress和current的updateQueue
    const current = workInProgress.alternate;
    if (current !== null) {
      const currentQueue = current.updateQueue;
      const currentLastBaseUpdate = currentQueue.lastBaseUpdate;
      
      if (currentLastBaseUpdate !== lastBaseUpdate) {
        if (currentLastBaseUpdate === null) {
          currentQueue.firstBaseUpdate = firstPendingUpdate;
        } else {
          currentLastBaseUpdate.next = firstPendingUpdate;
        }
        currentQueue.lastBaseUpdate = lastPendingUpdate;
      }
    }
  }
  
  // 如果存在更新，处理它们
  if (firstBaseUpdate !== null) {
    // 从baseState开始应用更新
    let newState = queue.baseState;
    
    // 跟踪哪些lanes已经处理
    let newLanes = NoLanes;
    
    let newBaseState = null;
    let newFirstBaseUpdate = null;
    let newLastBaseUpdate = null;
    
    let update = firstBaseUpdate;
    do {
      const updateLane = update.lane;
      const updateEventTime = update.eventTime;
      
      // 检查此更新是否有足够的优先级
      if (!isSubsetOfLanes(renderLanes, updateLane)) {
        // 优先级不够，跳过此更新
        // 克隆更新并将其保留在队列中
        const clone = {
          eventTime: updateEventTime,
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
        if (newLastBaseUpdate !== null) {
          // 如果我们已经跳过了一些更新，需要将此更新也添加到新的baseUpdate链表
          const clone = {
            eventTime: updateEventTime,
            lane: NoLane,
            tag: update.tag,
            payload: update.payload,
            callback: update.callback,
            next: null,
          };
          newLastBaseUpdate = newLastBaseUpdate.next = clone;
        }
        
        // 应用更新
        newState = getStateFromUpdate(
          workInProgress,
          queue,
          update,
          newState,
          props,
          instance,
        );
        
        // 如果有回调，将其添加到effects列表
        const callback = update.callback;
        if (callback !== null) {
          workInProgress.flags |= Callback;
          const effects = queue.effects;
          if (effects === null) {
            queue.effects = [update];
          } else {
            effects.push(update);
          }
        }
      }
      
      // 移动到下一个更新
      update = update.next;
      
      if (update === null) {
        // 已处理完所有更新
        pendingQueue = queue.shared.pending;
        if (pendingQueue === null) {
          break;
        } else {
          // 有新的pending更新，将其添加到链表末尾
          // ...类似前面的逻辑
        }
      }
    } while (true);
    
    // 如果没有跳过任何更新，新的baseUpdate链表为空
    if (newLastBaseUpdate === null) {
      newBaseState = newState;
    }
    
    // 更新queue的baseState和baseUpdate链表
    queue.baseState = newBaseState;
    queue.firstBaseUpdate = newFirstBaseUpdate;
    queue.lastBaseUpdate = newLastBaseUpdate;
    
    // 更新workInProgress的状态
    workInProgress.memoizedState = newState;
    workInProgress.lanes = newLanes;
  }
}
```

## firstBaseUpdate和lastBaseUpdate的关键作用

从上面的代码可以看出，`firstBaseUpdate`和`lastBaseUpdate`有几个关键作用：

### 1. 维护更新顺序

React的状态更新是有序的，后面的更新依赖于前面更新的结果。`firstBaseUpdate`和`lastBaseUpdate`确保了更新按照创建的顺序被应用，保证状态计算的正确性。

### 2. 支持优先级调度

在React的并发模式下，更新有不同的优先级。高优先级的更新（如用户输入）需要快速响应，而低优先级的更新（如数据获取）可以延迟处理。

当一个高优先级的渲染打断一个低优先级的渲染时，React可能会跳过一些低优先级的更新。这些被跳过的更新不会丢失，而是被保存在新的`firstBaseUpdate`和`lastBaseUpdate`链表中，等待下一次渲染时处理。

### 3. 确保状态一致性

即使在并发模式下，React也需要确保状态的一致性。`firstBaseUpdate`和`lastBaseUpdate`链表确保了即使某些更新被暂时跳过，它们最终也会被应用，并且按照正确的顺序应用。

## 实际案例分析

让我们通过一个具体的例子来理解`firstBaseUpdate`和`lastBaseUpdate`的工作原理：

```typescript
class Counter extends React.Component {
  state = { count: 0 };
  
  componentDidMount() {
    // 假设这些更新有不同的优先级
    this.setState({ count: 1 }); // 更新A，优先级低
    this.setState({ count: 2 }); // 更新B，优先级高
    this.setState(state => ({ count: state.count + 1 })); // 更新C，优先级低
  }
  
  render() {
    return <div>{this.state.count}</div>;
  }
}
```

在这个例子中，组件挂载后会触发三个状态更新。假设更新B的优先级高于更新A和更新C。

### 初始状态

- `baseState`: `{ count: 0 }`
- `firstBaseUpdate`: `null`
- `lastBaseUpdate`: `null`
- `shared.pending`: 指向一个包含更新A、B、C的环形链表

### 第一次渲染（高优先级）

1. React将`shared.pending`中的更新转移到baseUpdate链表：
   - `firstBaseUpdate`: 指向更新A
   - `lastBaseUpdate`: 指向更新C
   - `shared.pending`: `null`

2. React开始处理更新，但由于只处理高优先级更新：
   - 跳过更新A（优先级低），将其添加到新的baseUpdate链表
   - 应用更新B（优先级高），`newState`变为`{ count: 2 }`
   - 跳过更新C（优先级低），将其添加到新的baseUpdate链表

3. 更新完成后：
   - `baseState`: `{ count: 0 }` (注意这里是更新B之前的状态)
   - `firstBaseUpdate`: 指向更新A的克隆
   - `lastBaseUpdate`: 指向更新C的克隆
   - `memoizedState`: `{ count: 2 }`

### 第二次渲染（处理所有优先级）

1. React处理baseUpdate链表中的所有更新：
   - 从`baseState`开始，应用更新A，`newState`变为`{ count: 1 }`
   - 应用更新B的克隆，`newState`变为`{ count: 2 }`
   - 应用更新C，`newState`变为`{ count: 3 }`

2. 更新完成后：
   - `baseState`: `{ count: 3 }`
   - `firstBaseUpdate`: `null`
   - `lastBaseUpdate`: `null`
   - `memoizedState`: `{ count: 3 }`

这个例子展示了`firstBaseUpdate`和`lastBaseUpdate`如何在优先级调度中保持状态的一致性。即使某些更新被暂时跳过，它们最终也会被正确应用。

## 与函数组件Hook的对比

函数组件中的Hook也有类似的机制，但实现方式不同：

- 在类组件中，`firstBaseUpdate`和`lastBaseUpdate`存在于Fiber节点的updateQueue中
- 在函数组件中，每个Hook（如useState、useReducer）都有自己的`baseQueue`字段，用于存储被跳过的更新

这反映了React团队对状态管理的不同设计理念：类组件采用集中式的状态管理，而函数组件采用分散式的状态管理。

## 总结

`firstBaseUpdate`和`lastBaseUpdate`是React类组件更新机制中的关键字段，它们：

1. **维护更新顺序**：确保状态更新按照创建的顺序被应用
2. **支持优先级调度**：允许高优先级更新"插队"，同时保留低优先级更新
3. **确保状态一致性**：即使在并发模式下，也能保证所有更新最终被正确应用

理解这两个字段的工作原理，有助于深入理解React的状态更新机制，特别是在并发模式下的行为。这也解释了为什么React能够在保持状态一致性的同时，提供流畅的用户体验。