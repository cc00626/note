## useRef的原理

### 1. 本质：逃离渲染的“保险箱”

`useRef` 的本质是一个在组件整个生命周期内**保持引用不变**的普通 JavaScript 对象。

- **数据结构：** 永远是 `{ current: ... }`。
- **核心逻辑：** 无论组件重新渲染多少次，`useRef` 返回的永远是同一个内存地址的对象。
- **关键特性：** 修改 `ref.current` **不会**触发组件重新渲染（Re-render）。

------

### 2. 内部原理 (Fiber 级别)

在 React 内部，`useRef` 的工作机制如下：

1. **挂载期 (Mount)：** React 创建一个 `{ current: initialValue }` 对象，并将其存储在该组件对应的 **Fiber 节点**的 `memoizedState` 属性中。
2. **更新期 (Update)：** 每次函数组件重新执行时，React 只是简单地从 Fiber 中取出同一个对象并返回。
3. **对比机制：** * `useState` 修改值会调用 `dispatchAction` 触发调度更新。
   - `useRef` 修改值只是纯粹的 JavaScript 对象属性赋值，React 引擎完全感知不到变化。

------

### 3. 两大核心应用场景

#### 场景 A：访问 DOM 节点

这是最常用的场景，用于直接操作浏览器原生的 DOM 元素。

```
const inputRef = useRef(null);

const focusInput = () => {
  // 直接操作 DOM 属性
  inputRef.current.focus();
};

return <input ref={inputRef} />;
```

- **赋值时机：** React 会在 **Commit 阶段**（DOM 挂载后）将 DOM 元素赋给 `current`。
- **清理时机：** 组件卸载时，React 会自动将 `current` 设为 `null`。

#### 场景 B：跨渲染周期的“记忆存根”

用于存储那些不需要反映在 UI 上的变量，避免因为闭包陷阱拿到过时的值。

- **存储定时器：** `timerRef.current = setInterval(...)`
- **记录上一次的 Props：** 在 `useEffect` 中更新 Ref，可对比新旧值。
- **防止竞态请求：** 存储一个标记位，判断异步回调是否该执行。

