### 类组件几点感悟

1. class组件的 `filber.updateQueue.shared.pending`是两个树共享的。所以在渲染的时候pending 也是可以新增的
2. class组件的 `filber.updateQueue.firstBaseUpdate 和 lastBaseUpdate` 可以保证优先级高先渲染且又保证的更新的顺序。
3. class组件在中断更新恢复的核心是，firstBaseUpdate在高优先级处理完之后，每次都会跟 pending中的合并。保证了效率和顺序。


### 函数组件几点感悟

```typescript
export type Hook = {
  memoizedState: any, // Hook自己的状态
  baseState: any, // 基础状态
  baseQueue: Update<any, any> | null, // 基础更新队列
  queue: UpdateQueue<any, any> | null, // 更新队列
  next: Hook | null, // 指向下一个Hook
};
```

1. 如上memoizedState存储的不同hook的值, 并通过next指向下一个hook
2. baseState是应用更新的起点
3. baseQueue 和 queue 跟类组件的 firstBaseUpdate shared.pending作用类似，既要保证优先级又要保证顺序
