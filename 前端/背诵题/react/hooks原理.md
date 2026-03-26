### hooks的使用原则

**只能在函数组件或自定义 Hook 中使用**

**只能在组件顶层调用，不能写在条件/循环/嵌套函数中**

### hooks原理

## React Hooks的原理

React Hooks 的本质是：**利用闭包和链表数据结构，将状态“挂载”在组件对应的 Fiber 节点上。**

### 1. 核心载体：Fiber 上的 memoizedState

在 React 内部，每个函数组件对应一个 `Fiber` 对象。对于函数组件，`Fiber.memoizedState` 并不像类组件那样直接存 State 对象，而是存了一个**由 Hook 对象组成的单向链表**。

**Hook 对象的数据结构：**

```
{
  memoizedState: any,  // 上次渲染时该 Hook 的状态值
  baseState: any,      // 基础状态（用于处理带优先级的更新）
  queue: UpdateQueue,  // 更新队列（环形链表），存放 dispatchAction
  next: Hook | null    // 指向下一个 Hook 节点
}
```

------

### 2. 两个阶段：Mount 与 Update

React 并不是只用一套逻辑处理 Hooks。在 `renderWithHooks` 函数执行时，会根据 `current` 是否为空，为组件分配不同的 **Dispatcher**。

- **Mount (首次挂载):** 使用 `HooksDispatcherOnMount`。此时会创建新的 Hook 对象，并按顺序连接成链表。
- **Update (更新):** 使用 `HooksDispatcherOnUpdate`。此时会**复用**现有的 Hooks 链表，并根据 `queue` 中的更新任务计算出新的 `memoizedState`。

------

### 3. 为什么不能写在条件语句中？

这是面试官最喜欢追问的一点。

**核心原因：** React 寻找 Hook 状态的唯一依据是 **“调用顺序”**。

在更新阶段，React 并不知道你调用的是哪个具体的 Hook，它只是维护一个内部指针（`workInProgressHook`），每执行一个 Hook，指针就移动到 `next`。

- **逻辑崩塌场景：** 如果 Hook 1 被包裹在 `if` 中且此次未执行，React 的指针依然会指向原链表的第一个位置（本该是 Hook 1 的位置），但代码实际运行到了 Hook 2。
- **后果：** Hook 2 拿到了 Hook 1 的状态，导致数据错乱，甚至因为 Hook 数量不匹配直接报错。

------

### 4. 状态更新的秘密：环形链表

你笔记中提到的 `queue.pending` 构成环形链表是一个高级考点。

**为什么要用环形链表？**

- **性能：** 插入更新时，只需要 `pending.next = firstUpdate` 即可完成。由于 `pending` 永远指向最后一个更新，`pending.next` 就是第一个更新。
- **效率：** 无需遍历整个链表就能同时快速定位到“头”和“尾”，实现 $O(1)$ 的插入操作。

------

### 5. useEffect 的特殊处理

`useEffect` 产生的 effect 对象也会组成链表，但它保存在 `fiber.updateQueue` 中。

- **Effect 链表：** 也是环形链表。
- **执行时机：** 在 Render 阶段只是构建链表，真正的执行是在 **Commit 阶段**（LayoutEffect 在 commit 阶段同步执行，Effect 在 commit 阶段后异步执行）。

