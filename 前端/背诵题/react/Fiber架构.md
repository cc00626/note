### 对React-Fiber的理解，它解决了什么问题

React V15 在渲染时，会递归比对 VirtualDOM 树，找出需要变动的节点，然后同步更新它们， 一气呵成。这个过程期间， React 会占据浏览器资源，这会导致用户触发的事件得不到响应，并且会导致掉帧，**导致用户感觉到卡顿**。



为了给用户制造一种应用很快的“假象”，不能让一个任务长期霸占着资源。 可以将浏览器的渲染、布局、绘制、资源加载(例如 HTML 解析)、事件响应、脚本执行视作操作系统的“进程”，需要通过某些调度策略合理地分配 CPU 资源，从而提高浏览器的用户响应速率, 同时兼顾任务执行效率。



所以 React 通过Fiber 架构，让这个执行过程变成可被中断。“适时”地让出 CPU 执行权，除了可以让浏览器及时地响应用户的交互，还有其他好处:

- 分批延时对DOM进行操作，避免一次性操作大量 DOM 节点，可以得到更好的用户体验；
- 给浏览器一点喘息的机会，它会对代码进行编译优化（JIT）及进行热代码优化，或者对 reflow 进行修正。

#### 

**核心思想：**Fiber 也称协程或者纤程。它和线程并不一样，协程本身是没有并发或者并行能力的（需要配合线程），它只是一种控制流程的让出机制。让出 CPU 的执行权，让 CPU 能在这段时间执行其他的操作。渲染的过程可以被中断，可以将控制权交回浏览器，让位给高优先级的任务，浏览器空闲后再恢复渲染。



### React Fiber为什么要推出

React16 之前使用的是 Stack Reconciler，采用递归的方式遍历组件树进行 diff，这个过程是同步且不可中断的，如果组件树很大就会长时间占用主线程，导致页面卡顿。为了解决这个问题，React16 引入了 Fiber 架构，把组件树转换为 Fiber Tree，并把渲染过程拆分为多个可中断的任务单元。React 可以在浏览器空闲时间执行这些任务，从而实现时间切片和优先级调度。Fiber 还将渲染过程分为 render 阶段和 commit 阶段，其中 render 阶段可以被打断，而 commit 阶段是同步执行的。这种架构为 React 的并发渲染、Suspense 和 startTransition 等特性提供了基础。



### React Fiber使用的数据结构

```javascript
function FiberNode(
  tag,
  pendingProps,
  key,
  mode
) {
  // 节点类型
  this.tag = tag

  // 节点key
  this.key = key

  // 组件类型
  this.type = null

  // 真实DOM
  this.stateNode = null

  // Fiber树结构
  this.return = null   // 父节点
  this.child = null    // 第一个子节点
  this.sibling = null  // 兄弟节点
  this.index = 0

  // props
  this.pendingProps = pendingProps
  this.memoizedProps = null

  // state
  this.memoizedState = null

  // 更新队列
  this.updateQueue = null

  // 副作用标记
  this.flags = 0
  this.subtreeFlags = 0

  // 双缓存
  this.alternate = null
}
```

### current Fiber Tree 和 workInProgress Fiber Tree 的区别是什么？

React 在更新 UI 时会维护两棵 Fiber 树。

一棵是 current Fiber Tree，表示当前已经渲染到页面上的 UI 状态；另一棵是 workInProgress Fiber Tree，用来计算下一次更新的 UI。

React 在更新时不会直接修改 current 树，而是基于 current 创建一棵 workInProgress 树，在这棵树上执行 diff 和更新。当 workInProgress 构建完成后，在 commit 阶段统一更新 DOM，并将 workInProgress 设置为新的 current。这样 React 就实现了类似双缓冲的机制，可以在后台构建新的 UI，同时保证当前 UI 的稳定，并且支持中断渲染和并发更新。