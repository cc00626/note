### 为什么使用

在react中每次组件重新进行渲染，内部的普通变量和函数都会重新创建，分配新的内存地址。

```
// 父组件
const handleClick = () => { console.log('clicked') }; 
return <Child onClick={handleClick} />;
```



在子组件使用了react.mome的时候，每次handleChilck的引用都会变，这样子组件就是强制重新渲染，这时候就需要使用useCallBack锁定函数引用。



useMemo为了保持引用不变和复杂昂贵的计算。 配合 `useEffect` 避免死循环。如果你的 `useEffect` 依赖于一个对象或数组，必须确保这个对象在没有逻辑变化时引用是稳定的，否则 `useEffect` 会在每次渲染后重复触发。



### 区别：useCallback就是useMemo的语法糖



### React.memo

默认情况下，当父组件重新渲染时，React 会**递归地渲染其所有的子组件**，无论子组件的 Props 是否发生了变化。

`React.memo` 通过包裹子组件，让 React 在渲染前先进行一次 **Props 比对**：

- **如果 Props 没变**：直接跳过该组件的渲染，复用上一次的渲染结果（Memoized Result）。
- **如果 Props 变了**：执行正常的渲染流程。

```
const MyComponent = React.memo(function MyComponent(props) {
  /* 使用 props 渲染 */
});
```



`React.memo` 默认执行的是**浅比较**。这意味着它只检查 Props 对象的第一层属性：

- **基本类型**（String, Number, Boolean）：比较**值**是否相等。
- **引用类型**（Object, Array, Function）：比较**内存地址（引用）**是否相同。

> **注意**：这正是为什么你之前问 `useCallback` 的原因。如果你给 `React.memo` 包裹的子组件传了一个普通函数，父组件一变，函数地址就变，`React.memo` 的浅比较就会判定“Props 已改变”，导致缓存失效。



**自定义比较逻辑**

`React.memo` 接受第二个可选参数，允许你手动控制比较逻辑。这在处理复杂的深层对象时非常有用。

```
const areEqual = (prevProps, nextProps) => {
  // 返回 true 则【跳过】重新渲染
  // 返回 false 则【触发】重新渲染
  return prevProps.id === nextProps.id && prevProps.data.version === nextProps.data.version;
};

export default React.memo(MyChildComponent, areEqual);
```

