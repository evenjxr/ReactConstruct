### 类组件几点感悟

1. class组件的 `fiber.updateQueue.firstBaseUpdate` 和 `fiber.updateQueue.lastBaseUpdate` 可以保证优先级高的先渲染且又保证更新的顺序。
2. class组件在中断更新恢复的核心是，firstBaseUpdate在高优先级处理完之后，每次都会跟pending中的合并。保证了效率和顺序。
3. class组件的生命周期方法在Fiber架构下有了新的执行时机，特别是在并发渲染模式下。


### 函数组件几点感悟

```typescript
export type Hook = {
  memoizedState: any, // Hook自己的状态
  baseState: any, // 基础状态
  baseQueue: Update<any, any> | null, // 基础更新队列
  queue: UpdateQueue<any, any> | null, // 更新队列
  next: Hook | null, // 指向下一个Hook
  dependencies?: Array<any>, // 依赖数组
  type?: string, // Hook类型
};
```

1. 如上memoizedState存储的不同hook的值，并通过next指向下一个hook。
2. baseState是应用更新的起点。
3. baseQueue 和 queue 跟类组件的 firstBaseUpdate 和 shared.pending作用类似，既要保证优先级又要保证顺序。
4. Hook的依赖数组用于比较前后两次渲染的依赖是否发生变化，决定是否需要重新执行。
5. Hook的类型信息存储在memoizedState中，用于区分不同类型的Hook（如useState、useEffect等）。


### React 18新特性

1. 并发渲染（Concurrent Rendering）
   - 允许React同时准备多个版本的UI
   - 通过优先级机制决定哪个版本最终被渲染
   - 支持中断和恢复渲染过程

2. 自动批处理（Automatic Batching）
   - 自动将多个状态更新合并为一次重渲染
   - 适用于所有事件处理器、Promise、setTimeout等
   - 提高性能并减少不必要的渲染
   - 批处理是多次udpate只触发了一个调度任务

3. 新的Hooks
   - useId：生成唯一ID
   - useSyncExternalStore：用于外部数据源的同步订阅
   - useTransition：标记非紧急更新
   - useDeferredValue：延迟更新值


### 任务分级、时间切片、baseUpdate通力合作实现中断可恢复，保证结果一致性

1. React有任务调度和时间切片两种混合模式。
2. 有高优先级的任务(用户输入)插入会中断当前任务并废弃掉workInProgress，直接处理高优先级的fiber子树(scheduleUpdateOnFiber判断)。
3. 时间切片将大任务拆成小片段，每次处理fiber都会判断是否有足够的时间，如果没有则记录当前nextUnitOfWork，等空闲的时候再从nextUnitOfWork开始继续遍历（workLoopConcurrent判断）。

```javascript
while (nextUnitOfWork) {
  nextUnitOfWork = performUnitOfWork(nextUnitOfWork);

  if (hasHigherPriorityTask()) {
    // 立即切换到高优先级任务
    break;
  }

  if (shouldYield()) {
    // 时间片用完，暂停
    break;
  }
}
```

4. 只有当workInProgress完全构建好，才会切换为current并提交到DOM。这样即使渲染被多次中断，用户看到的始终是上一次提交的一致状态。
5. baseUpdate保存了上次被跳过的低优先级的update(t1, t2)，优先处理高优先级update(t3)，commit提交渲染之后，下次更新还是会按顺序t1, t2, t3执行。这保证了顺序一致性，但是t3的计算会迭代，不过副作用不会迭代。baseUpdate只有在中断恢复时候有用，如果直接废弃就没用了。


### 任务类型分类

#### 会完全丢弃当前工作，重新构建渲染树的任务：

- 离散事件触发的更新：点击、键盘输入等用户交互事件
- 使用flushSync的更新：强制同步执行的更新
- 错误边界触发：抛出异常需要渲染备用UI
- 过期的更新：等待太久被提升为高优先级
- Suspense边界解除挂起：数据加载完成需要立即显示

这些任务通常使用SyncLane或DiscreteEventLanes优先级。

#### 可中断恢复的高优先级任务（类似t3）：

- 使用useTransition标记的更新：指定为可中断的过渡更新
- 使用useDeferredValue的更新：延迟值计算
- 在批处理中标记的部分高优先级更新：例如t3案例
- 并发渲染模式下的非紧急更新：优先级高但不需要废弃当前工作

这些任务通常使用TransitionLanes或DefaultLanes优先级。




#### 任务分级 & 时间切片 & 中断恢复

- 高优先级的调度渲染任务中断了低优先级的渲染，废弃到旧的workInProgress, 重新从current复制 workInProgress遍历filber, 低优先级的update会被跳过暂存在baseUpdate上，先处理为高优先级的update, 但为了保证执行顺序高优先级的update也会被挂到baseUpdate，当所有高优先级的update处理完成，commit后互换wip 和 current 两棵树，让用户优先看到高优先级任务的渲染结果， 之后在从current复制出来wip, 处理低优先级的任务，所以低优先级的任务会被执行多次，高优先级update为了保证顺序一致性可能也会被执行第二次，但是副作用不会被执行两次。 高优先级update如果是 setState(s + 10), 会被执行两次导致结果多加了10， react框架并没有根本解决。

- 另一种中断并不是被高优先级的任务中断， 而是此次渲染空余时间不足(通常是5ms)， 自己退出fiber遍历，记录当前workInPrgress正在处理的fiber节点，让出js主线程，让浏览器响应用户的 滑动，点击，放大等事项，等下个frame周期再从断点的fiber节点开始继续遍历处理。

