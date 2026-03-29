## 背景

项目太老，原来的项目使用.net进行开发，在分配权限的时候是为每个账号直接绑定权限。操作繁琐，并且容易出错。我们在重构老的系统，对这个功能进行了优化。



## 实现

基于角色的访问设置了两个不同的权限级别。页面权限，数据权限。

页面权限就是不同的角色能够访问不同的页面，通过静态路由和动态路由进行控制，我们基于项目中分配了5种主要角色

- 运营组长 ---  负责为组内成员分配审核，质检的任务。

- 管理员 --- 超级权限，什么都能干。

- 审核员 --- 负责进行人工审核。

- 质检员 --- 对审核员的审核内容进行质检。二次确认。

- 审核评论员 --- 负责对帖子中的评论进行精选，或者删除。



在用户登录成功的时候，后端会返回用户信息，其中该用户角色下的动态路由表

```
[
  {
    "level": "1",
    "name": "首页",
    "path": "/home"
  },
  {
    "level": "1",
    "name": "审核打标管理",
    "path": "/examine"
  },
  {
    "level": "1",
    "name": "数据管理",
    "path": "/datamanage"
  },
  {
    "level": "1",
    "name": "任务管理",
    "path": "/taskmanage"
  },
  {
    "level": "1",
    "name": "评论管理",
    "path": "/comment"
  },
  {
    "level": "1",
    "name": "系统管理",
    "path": "/system"
  }
]
```

```
const Root = () => {
  // 创建一个有子节点的Route
  const CreateHasChildrenRoute = (route: RoutesType) => {
    return (
      <Route path={route.path} key={route.path}>
        <Route
          index
          element={
            <AuthRoute key={route.path} path={route.path}>
              {route.element}
            </AuthRoute>
          }
        />
        {route?.children && RouteAuthFun(route.children)}
      </Route>
    );
  };

  // 创建一个没有子节点的Route
  const CreateNoChildrenRoute = (route: RoutesType) => {
    return (
      <Route
        key={route.path}
        path={route.path}
        element={
          <AuthRoute path={route.path} key={route.path}>
            {route.element}
          </AuthRoute>
        }
      />
    );
  };

  // 处理我们的routers
  const RouteAuthFun = (routeList: any) => {
    return routeList.map((route: RoutesType) => {
      let element: ReactElement | null = null;
      if (route.children && !!route.children.length) {
        element = CreateHasChildrenRoute(route);
      } else {
        element = CreateNoChildrenRoute(route);
      }
      return element;
    });
  };

  return (
    <BrowserRouter>
      <Routes>{RouteAuthFun(routers)}</Routes>
    </BrowserRouter>
  );
};

```

```
// 无需权限认证的白名单
// 一般是前端的一些报错页
const DONT_NEED_AUTHORIZED_PAGE = ["/unauthorized", "/*"];

const AuthRoute = ({ children, path }: any) => {
  // 该flag用于控制 受保护页面的渲染时机，需要等待useEffect中所有的权限验证条件完成后才表示可以渲染
  const [canRender, setRenderFlag] = useState(false);
  const navigate = useNavigate();
  const menuLists = useAppSelector((state) => state.login.menuLists);
  const menuUrls = menuLists.map((menu) => menu.url);
  const token = localStorage.getItem("access_token") || "";

  // 在白名单中的无需验证，直接跳转
  if (DONT_NEED_AUTHORIZED_PAGE.includes(path)) {
    return children;
  }

  useEffect(() => {
    // 用户未登录
    if (token === "") {
      message.error("token 过期，请重新登录!");
      navigate("/login");
    }

    // 已登录
    if (token) {
      // 已登录需要通过logout来控制退出登录或者是token过期返回登录界面
      if (location.pathname == "/login") {
        navigate("/");
      }

      // 已登录，根据后台传的权限列表做判断
      if (!menuUrls.includes(location.pathname)) {
        navigate("/unauthorized", { replace: true });
      }
    }

    // 当上面的权限控制通过后，再渲染受保护的页面
    setRenderFlag(true);
  }, [token, location.pathname]);

  if (!canRender) return null;
  return children;
};
export default AuthRoute;

作者：SaebaRyo
链接：https://juejin.cn/post/7228572620555288636
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

最后动态路由和静态路由进行拼接，返回该用户角色下的完成的路由，动态生成。



数据的权限，是审核员，运营员对不同的数据有不同的访问权限。我们分了3种数据类型，视频，图文，帖子。

在用户登录成功后，会返回可访问的权限id

1 --- 视频

2 --- 图文

3 --- 帖子



在用户发送请求的时候会在请求前拦截器统一带上这个字段。然后后端第二次校验一遍。

```
// utils/request.js
import axios from 'axios'
import { usePermissionStore } from '@/store/permission'

const service = axios.create({
  baseURL: '/api',
  timeout: 10000
})

// 请求拦截器
service.interceptors.request.use(
  config => {
    const permissionStore = usePermissionStore()

    // 自动添加数据权限过滤
    if (config.method === 'get' && config.params) {
      config.params._dataPermission = {
        type: permissionStore.dataPermission.type,
        scope: permissionStore.dataPermission.scope
      }
    }

    return config
  },
  error => {
    return Promise.reject(error)
  }
)

export default service
```





## 如何管理不同用户的权限

我们在管理员页面有一个，用于分配权限的页面。

1. 角色管理 - 创建、编辑、删除角色
2. 权限树配置 - 勾选角色的权限
3. 用户分配角色 - 给用户分配角色
4. 权限预览 - 预览某个角色能访问的菜单



## 权限变更怎么实时生效

### 1. 路由拦截与重新拉取

当管理员在后台修改了某个角色的权限后，最简单的方式是让受影响的用户在**下一次页面跳转**时感知到变化。

- **实现逻辑：** 在全局路由守卫（如 `useEffect` 监听 `location`）中，每次跳转前调用一次轻量级的 `checkPermission` 接口。

- **React 代码思路：**

  JavaScript

  ```
  useEffect(() => {
    // 每次路由变化时验证当前用户信息
    fetchUserInfo().then(res => {
      const { permissions } = res.data;
      // 如果本地缓存的权限版本与后端不一致，强制重新渲染路由表
      if (JSON.stringify(permissions) !== JSON.stringify(localPermissions)) {
        setLocalPermissions(permissions);
        message.info('您的权限已更新');
      }
    });
  }, [location.pathname]);
  ```

### 2. WebSocket 事件推送

如果业务要求用户在**不刷新、不跳转**的情况下，按钮立刻变灰或消失，则需要长连接技术。

- **实现逻辑：**
  1. 前端与后端建立 WebSocket 连接。
  2. 管理员点击“保存权限”时，后端通过消息队列（如 Redis Pub/Sub）通知所有受影响的在线用户。
  3. 前端收到 `PERMISSION_UPDATE` 消息，触发全局状态（Redux/Zustand）的更新。
- **优点：** 毫秒级延迟。
- **缺点：** 维护长连接有服务器成本。



## 如何实现权限的？

UGC老系统使用.NET, 老系统的权限是直接绑定在账号上，操作繁琐。在使用react重构新系统时，我们使用RBAC模型，从页面权限和数据权限进行了重构，定义了运营组长、管理员、审核员、质检员、审核评论员这五种核心角色。



在页面权限上，我们使用动态路由的方式进行控制，首先我们在前端会有一个完整的路由表，记录全部的路由和对应的组件。

```
export interface RoutesType {
  /** 路由访问路径，不带路由前缀 */
  path: string;
  /** 路由名称 */
  name: string;
  /** 菜单层级，比如一级、二级、三级 */
  level?: number;
  /** 页面是否带布局 */
  layout?: boolean;
  /** 页面是否隐藏面包屑 */
  hideBreadcrumb?: boolean;
  /** 路由是否在菜单中显示 */
  hideInMenu?: boolean;
  /** 路由对应的组件路径，相对 src/pages 的路径 */
  component?: string;
  /** 重定向地址，一般用于 mix 布局的一级菜单 */
  redirect?: string;
  /** 子路由 */
  routes?: RoutesType[];
}

```

在前端用户登录成功后，会返回用户的相关信息和动态路由表，前端拿到动态路由表后，将所有的path提取到一个数组里面，然后对前端的动态路由进行过滤，查看path是否存在后端的接口中，如果存在就返回，不存在将组件替换未NoAccess。

```
import { matchPath } from "react-router-dom"
import NoAccess from "@/components/NoAccess"

const WHITE_LIST = ["/home", "/login", "/logon"]

function hasPermission(routePath, paths) {
  return paths.some(p => {
    return matchPath(
      {
        path: p,
        end: false // 允许匹配子路径
      },
      routePath
    )
  })
}

function generateRoutes(routeMap, paths) {
  function traverse(routes) {
    return routes.map(route => {
      const newRoute = { ...route }

      // 👉 1. 有子路由（父节点）
      if (route.children && route.children.length > 0) {
        newRoute.children = traverse(route.children)

        // ⭐关键：如果子节点里“有一个有权限”，父节点必须保留
        const hasChildPermission = newRoute.children.some(child => {
          return child.element?.type !== NoAccess
        })

        if (hasChildPermission) {
          return newRoute
        }

        // 如果子节点全没权限，父节点也可以 NoAccess（可选策略）
        return {
          ...newRoute,
          element: <NoAccess />
        }
      }

      // 👉 2. 白名单
      if (WHITE_LIST.includes(route.path)) {
        return newRoute
      }

      // 👉 3. 权限判断（支持动态路由）
      if (hasPermission(route.path, paths)) {
        return newRoute
      }

      // 👉 4. 无权限
      return {
        ...newRoute,
        element: <NoAccess />
      }
    })
  }

  return traverse(routeMap)
}



// App.jsx
import { BrowserRouter, Routes, Route } from "react-router-dom"
import routeMap from "@/config/routeMap"
import { generateRoutes } from "@/utils/generateRoutes"
import { usePermissionStore } from "@/store/permission"
import useInitPermission from "@/hooks/useInitPermission"
import AuthRoute from "@/components/AuthRoute"
import PageLoading from "@/components/PageLoading"

export default function App() {
  const { paths } = usePermissionStore()
  const { loading } = useInitPermission()

  if (loading) return <PageLoading />

  const routes = generateRoutes(routeMap, paths)

  return (
    <BrowserRouter>
      <Routes>
        {routes.map(({ path, element }) => (
          <Route
            key={path}
            path={path}
            element={<AuthRoute>{element}</AuthRoute>}
          />
        ))}
      </Routes>
    </BrowserRouter>
  )
}
```

这样拼接完成后就得到最终的路由表。然后进行渲染。



登录的权限怎么进行判断？ 通过封装一个AuthRoute组件，判断有没有token，如果有token，不允许跳转到登录和注册页面，如果没有token，不允许跳转其他页面



数据的权限怎么做 ？ 前端会有一个弹框，可以设置自己要审核的数据类型，会发送到后端，修改角色的信息判断是否有权限。

##  权限管理怎么实时生效？

通过和后端建立websocke连接，当路由状态有变化的时候后端会推送permission：update消息，前端会重新发起请求，更新路由。



### 数据的权限怎么办的

数据的权限，是怎么，数据改个请求就能看到全部的数据，完全由后端来进行决定，在发送token的时候回解析用户的身份，然后看有没有这个权限。



### 按钮的权限怎么弄

```
function usePermission(code) {
  const permissions = usePermissionStore(state => state.buttons)
  return permissions.includes(code)
}
```

