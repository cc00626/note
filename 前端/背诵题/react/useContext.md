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

**缺点**

- Context 更新会导致所有订阅组件重新渲染（性能问题）
- Provider嵌套问题

### 配合useReducer实现redux

1. 创建state和reducer

```javascript
// store/reducer.js
export const initialState = { count: 0, user: null };

export function authReducer(state, action) {
  switch (action.type) {
    case 'LOGIN':
      return { ...state, user: action.payload };
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    default:
      return state;
  }
}
```

2. 简历全局context

```javascript
// store/StoreContext.js
import React, { createContext, useReducer, useContext } from 'react';
import { authReducer, initialState } from './reducer';

const GlobalContext = createContext();

export const StoreProvider = ({ children }) => {
  const [state, dispatch] = useReducer(authReducer, initialState);

  return (
    // 将 state 和 dispatch 同时放入 Context
    <GlobalContext.Provider value={{ state, dispatch }}>
      {children}
    </GlobalContext.Provider>
  );
};

// 自定义 Hook，方便组件调用
export const useStore = () => useContext(GlobalContext);
```

3. 在根组件中使用

```javascript
// index.js 或 App.js
import { StoreProvider } from './store/StoreContext';

function App() {
  return (
    <StoreProvider>
      <Navbar />
      <MainContent />
    </StoreProvider>
  );
}
```

4. 使用

```javascript
function Counter() {
  const { state, dispatch } = useStore();

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>加 1</button>
    </div>
  );
}
```

## 