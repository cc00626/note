### 什么是 Reconciliation？

react有三大核心模块

**Scheduler**（调度器）、**Reconciler**（协调器）和 **Renderer**（渲染器）



**调度器**

当setState触发的时候，react不会立即开始diff，而是先创建一个 **Update** 对象，并请求调度。

**在任务入队**：每个任务都有一个 `expirationTime`（过期时间）。`expirationTime = startTime + timeout`。

**优先级计算**：

- 立即执行（Immediate）：`-1ms`
- 用户交互（UserBlocking）：`250ms`
- 默认（Normal）：`5000ms`
- 低优先级（Low）：`10000ms`

**时间切片**：通过 `MessageChannel` 发送消息，在浏览器的空闲时间（大约 5ms 一个周期）执行 `workLoop`。

```
// 简化后的 Scheduler 循环逻辑
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  currentTask = peek(taskQueue); // 从堆顶取最高优先级任务
  while (currentTask !== null) {
    if (currentTask.expirationTime > currentTime && (!hasTimeRemaining)) {
      break; // 时间片用完了，退出，等下一帧
    }
    const callback = currentTask.callback;
    callback(); // 这里的 callback 就是 Reconciler 的执行入口
    pop(taskQueue);
    currentTask = peek(taskQueue);
  }
}
```



**协调器**

协调器内部维护了两棵树，这是实现“可中断”和“快照恢复”的基础：

- **Current Tree**：当前页面正在显示的 Fiber 树。
- **WorkInProgress Tree (WIP)**：内存中正在构建的树，也就是即将更新的树。

这两个树的节点通过 `alternate` 属性互相引用。当更新完成后，React 只需要交换根节点的指针，WIP 树就变成了 Current 树。

#### A. Render 阶段：深度优先搜索 (DFS)

这是协调器最繁忙的时候。它通过 `workLoopConcurrent` 循环处理 Fiber 节点。

##### 1. beginWork（向下寻找差异）

源码位于 `ReactFiberBeginWork.js`。

- **判断复用**：对比旧 Fiber 和新组件的 `props`、`type`。如果没变且没有更新，直接跳过（Bailout）。
- **调和子节点（Reconcile Children）**：如果变了，调用 `reconcileChildFibers`。
- **执行 Diff**：
  - **单节点**：对比 `key`。`key` 相同则复用旧 Fiber，不同则创建新 Fiber 并标记旧的为 `Deletion`。
  - **多节点**：执行两轮遍历（第一轮找位置没变的，第二轮通过 Map 找位置变了但 `key` 相同的）。
- **打标签 (Flags)**：如果发现节点需要更新，就在 Fiber 对象的 `flags` 属性上打上二进制标记（如 `Placement` | `Update`）。

##### 2. completeWork（向上收集结果）

源码位于 `ReactFiberCompleteWork.js`。

- **构建 DOM**：如果是初次渲染，创建真实的 DOM 实例并挂载子节点。如果是更新，初始化 DOM 属性。
- **冒泡 Flags**：这是最精妙的地方。协调器会将子节点的 `flags` 向上合并到父节点。这样 React 最终只需要检查根节点的 `subtreeFlags`，就能知道整棵树哪里需要动手术。

#### Commit 阶段：同步施工

一旦 `workInProgress` 变量变回 `null`，意味着“找茬”结束。协调器进入 `commitRoot`。此时它会一次性、同步地执行以下子阶段：

1. **Mutation 阶段**：遍历 Effect 链表。调用 `commitPlacement`、`commitUpdate`、`commitDeletion`。这里是真正的 `dom.appendChild` 或 `dom.innerHTML` 发生的地方。
2. **切换树指针**：执行 `root.current = finishedWork`。此时 WIP 树正式转正。
3. **Layout 阶段**：调用 `useLayoutEffect` 的回调。此时浏览器已经拿到了最新的 DOM 结构，但还没涂色渲染。



### diff算法的核心策略

**节点两两对比 ($O(n^2)$)**：

要把旧树的所有节点 $A$ 变成新树的所有节点 $B$，最朴素的方法是把 A 树的每一个节点都去和 B 树的每一个节点比一下，看看谁能复用谁。这步操作的复杂度是 $n \times n = n^2$。

**寻找最小路径 ($O(n)$)**：

在对比完所有节点后，算法还需要遍历这些对比结果，找到一条“代价最小”的转换路径（比如：是移动这个节点便宜，还是删了重建便宜？）。这个搜索和计算的过程通常会再增加一个 $n$ 的维度。



将 $O(n^3)$ 的传统树对比复杂度降低到 **$O(n)$**，React 提出了三条核心策略（启发式算法）

##### **Tree Diff：策略一（分层求异）**

React 假设 **DOM 节点跨层级的移动操作非常少**，因此可以忽略不计。

- **核心逻辑**：只对树进行 **同层级比较**。
- **处理方式**：React 对 Virtual DOM 树进行深度优先遍历，只比较同一父节点下的子节点。
- **异常处理**：如果一个节点从 A 移到了 B（跨层级），React 不会聪明地移动它，而是直接 **销毁 A 下的旧节点，在 B 下重新创建一个新节点**。
- **开发建议**：保持 DOM 结构的稳定性，尽量避免复杂的跨层级嵌套改变。

##### Component Diff：策略二（组件拆解）

React 假设 **相同类型的组件生成相似的结构，不同类型的组件生成不同的结构**。

- **同一类型组件**：按照原策略继续对比子节点（递归 Diff）。
- **不同类型组件**：如果组件从 `<Header />` 变成了 `<Content />`，React 会判定该组件为“脏组件（Dirty Component）”。
  - 直接 **卸载（Unmount）** 旧组件及其下的所有子节点。
  - 重新 **挂载（Mount）** 新组件。
- **优化手段**：开发者可以通过 `shouldComponentUpdate` 或 `React.memo` 手动告诉 React 某组件是否需要进入 Diff 流程。

##### Element Diff：策略三（节点复用）

当同一层级下有一组子节点时，React 提供了三种操作：**INSERT（插入）**、**MOVE（移动）**、**REMOVE（删除）**。

这是 Diff 算法最精华的部分，主要依靠 **`key`** 来实现：

**A. 无 Key 的“就地复用”**

如果没有 `key`，React 只能按顺序对比。

- **例子**：列表 `[A, B, C]` 变成 `[Z, A, B, C]`。
- **结果**：React 会把 A 改成 Z，B 改成 A，C 改成 B，最后新建一个 C。这种“全量更新”性能极差。

**B. 有 Key 的“精准移动”**

有了 `key`，React 能够识别出节点只是换了位置。

- **算法逻辑**：
  1. 遍历新集合，通过 `key` 去旧集合中查找是否存在可复用节点。
  2. 利用 **`lastPlacedIndex`（最后挂载索引）** 机制：如果当前节点在旧集合中的位置比 `lastPlacedIndex` 大，说明它相对靠后，不需要移动；如果比它小，则需要 **向右移动**。
- **优点**：极大减少了 DOM 节点的销毁与创建。



### key 在 Diff 中起什么作用？

**1. 无 Key 时的处理：就地复用（效率低下）**

当一组同级子节点没有 `key` 时，React 会采用**下标对齐**的策略。

- **场景**：列表从 `[A, B, C]` 变为 `[Z, A, B, C]`（在头部插入了 Z）。
- **React 的逻辑**：
  1. 对比第一个位置：旧的是 A，新的是 Z。**类型相同（假设都是 li），直接把 A 的内容改成 Z。**
  2. 对比第二个位置：旧的是 B，新的是 A。**把 B 的内容改成 A。**
  3. 对比第三个位置：旧的是 C，新的是 B。**把 C 的内容改成 B。**
  4. 最后发现多了一个，**新建一个 C。**
- **结果**：明明 A、B、C 都在，但 React 把它们全部“抹掉重写”了。如果有输入框，里面的文字会因为节点复用错误而产生混乱。

------

**2. 有 Key 时的处理：精确匹配（高效复用）**

有了 `key`，React 会建立一个临时 Map，通过 `key` 直接找到旧树中对应的节点。

- **场景**：列表从 `[A, B, C]`（key 分别为 a, b, c）变为 `[Z, A, B, C]`（key 为 z, a, b, c）。
- **React 的逻辑**：
  1. 遍历新列表，看到 `key="z"`，旧列表里没有，**新建 Z**。
  2. 看到 `key="a"`，旧列表里有 `key="a"`，**直接复用旧的 A 节点**，只做位置移动。
  3. 同理，**复用 B 和 C**。
- **结果**：只创建了一个新节点，其余三个节点仅仅是移动了位置，性能大幅提升。

------

**3. Key 的核心算法策略：lastPlacedIndex**

在多节点 Diff 源码（`reconcileChildrenArray`）中，React 使用一个变量 **`lastPlacedIndex`** 来决定节点是否需要移动：

1. **查找**：在新旧对比时，记录当前复用节点在**旧集合**中的索引（`oldIndex`）。
2. **比较**：如果 `oldIndex < lastPlacedIndex`，说明该节点在旧集合中本来在前面，现在跑到后面去了，需要**移动**。
3. **更新**：如果 `oldIndex >= lastPlacedIndex`，说明顺序相对没变，不需要移动，并将 `lastPlacedIndex` 更新为 `oldIndex`。

### diff算法如何处理更新、删除和添加

对于单个fibe节点的更新，直接更就行

对于多个fiber节点的更新，

### 为什么key不能设置成index

