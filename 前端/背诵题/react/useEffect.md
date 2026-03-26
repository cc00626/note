### useEffect的原理

**一、 数据结构：Effect 对象与链表**

当你调用 `useEffect` 时，React 内部会创建一个 **Effect 对象**。这个对象并不仅仅存放在 Hook 链表的 `memoizedState` 中，还会被加入到 Fiber 节点的更新队列里。

**1. Effect 对象结构**

JavaScript

```
{
  tag: number,       // 标记 Effect 类型（如：是 HookPassive 代表 useEffect）
  create: Function,  // 你的回调函数 () => { ... }
  destroy: Function, // 你的销毁函数 return () => { ... }
  deps: Array,       // 依赖项数组
  next: Effect       // 指向组件内下一个 Effect，组成环形链表
}
```

**2. 双重存储**

- **在 Hook 链表中**：`useEffect` 产生的 Hook 对象的 `memoizedState` 存的就是上面这个 Effect 对象。
- **在 Fiber 节点上**：Fiber 有个属性叫 `updateQueue`。React 会把组件内所有的 Effect 连接成一个**环形链表**存放在这里，方便在 Commit 阶段统一遍历。

------

**二、 运行生命周期（重点）**

`useEffect` 的执行分三个阶段：

**1. Render 阶段：收集与对比**

- React 执行函数组件代码，调用 `useEffect`。
- **对比依赖项**：React 会取出旧的 Effect（从 `current` 树中），对比 `deps`。
  - 如果依赖项没变：给新的 Effect 打上“无需执行”的标记。
  - 如果依赖项变了或没有依赖项：打上 `HasEffect` 标记。

**2. Commit 阶段：调度执行**

- **区别于 `useLayoutEffect`**：`useLayoutEffect` 是在 DOM 修改后同步执行的。而 `useEffect` 是**异步**的。
- **流程**：在 Commit 阶段结束前，React 会发起一个**异步调度任务**（通过 `MessageChannel` 或 `setTimeout`）。

**3. 异步执行：先销毁，后创建**

等到浏览器渲染完页面、主线程空闲时，React 开始遍历 `updateQueue` 链表：

1. **销毁**：遍历链表，执行所有标记了更新的 Effect 的 `destroy` 函数（即上一次渲染返回的 `return () => {}`）。
2. **创建**：再次遍历链表，执行所有标记了 `HasEffect` 的 `create` 函数（即当前的 `useEffect` 回调）。



### useEffect 和useLayoutEffect的区别

### 1. 一句话总结区别

- **`useEffect` (异步)：** 浏览器把内容**画到屏幕上之后**才执行。不会阻塞页面渲染，体验更流畅。
- **`useLayoutEffect` (同步)：** 浏览器**计算完布局但还没画到屏幕上**时执行。会阻塞页面渲染，直到它执行完。

------

### 2. 执行时机对比图

1. **React 渲染 (Render)：** 生成虚拟 DOM，计算 Diff。
2. **更新真实 DOM (Commit)：** React 修改了实际的 DOM 节点。
3. **`useLayoutEffect` 执行：** 此时 DOM 已经变了，但浏览器**还没开始重绘**（屏幕上还是旧内容）。
4. **浏览器重绘 (Paint)：** 将最新的 DOM 渲染到屏幕上。
5. **`useEffect` 执行：** 屏幕上已经看到新内容了，它才在后台默默执行。

------

### 3. 详细对比

### ⚡ useEffect (推荐的首选)

- **特点：** 异步、非阻塞。
- **适用场景：** 绝大多数情况。
  - 发送 API 请求。
  - 设置订阅、定时器。
  - 修改不会直接导致视觉闪烁的状态。
- **优点：** 性能好，不会拖慢页面显示的响应速度。

### 🛠️ useLayoutEffect (特殊工具)

- **特点：** 同步、阻塞。
- **适用场景：** 处理 **“视觉闪烁”** 问题。
  - **测量 DOM：** 比如你需要根据一个元素的宽度，去动态计算另一个元素的位置。
  - **防止抖动：** 如果在 `useEffect` 里修改 DOM，用户会先看到位置 A，然后瞬间跳到位置 B（闪烁）。在 `useLayoutEffect` 里改，用户直接看到最终位置 B。
- **缺点：** 如果逻辑很重，会导致页面卡顿（白屏时间变长）。

------

### 4. 经典的“闪烁”案例对比

假设我们要做一个功能：点击按钮，让一个数字从 0 变成 100，并立即随机改变位置。

- **使用 `useEffect` 的过程：**

  1. 用户点击。
  2. 数字变成 100，出现在默认位置（用户看到了 100）。
  3. `useEffect` 执行，修改位置。
  4. 浏览器再次重绘，数字跳到了新位置。

  - **结果：** 用户看到数字**闪**了一下。

- **使用 `useLayoutEffect` 的过程：**

  1. 用户点击。
  2. 数字变成 100，React 更新 DOM。
  3. `useLayoutEffect` 立即执行，测量并修改位置。
  4. 浏览器一次性把“100”和“新位置”画出来。

  - **结果：** 用户直接看到在新位置的 100，**无闪烁**。

------

### 5. 开发建议 (Rule of Thumb)

1. **默认全用 `useEffect`。**
2. 只有当你发现你的代码导致了 **UI 闪烁**（元素跳动、错位）时，才尝试将其替换为 `useLayoutEffect`。
3. **注意：** 如果你在使用 Next.js 等服务端渲染（SSR）框架，`useLayoutEffect` 会在控制台报警告，因为它在服务器端没法测量 DOM。



### useEffect闭包问题

