可以用于跨组件传值



### useContext 的基本使用

1. 创建context

   ```
   const ThemeContext = React.createContext()
   ```

2. 使用context，提供数据

   ```
   <ThemeContext.Provider value="dark">
       <App />
   </ThemeContext.Provider>
   ```

3. 使用数据

   ```
   function C() {
     const theme = useContext(ThemeContext)
     
     return <div>{theme}</div>
   }
   ```



### 优点

- 解决 props drilling
-  代码更简洁
-  适合全局共享状态
-  降低组件耦合
-  Hook 写法更简单



### 缺点

- Context 更新会导致所有订阅组件重新渲染（性能问题）
-  更新粒度粗
-  不适合高频更新
- 不适合复杂状态管理