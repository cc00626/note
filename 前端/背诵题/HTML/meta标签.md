### 作用

#### 1.设置字符编码，防止乱码

<meta charset="UTF-8">

#### 2.优化搜索引擎

<meta name="description" content="广州气象灾害监测系统，提供实时降水与预警信息">
<meta name="keywords" content="GIS, 气象, 监测, 广州">

#### 3.Viewport（视口）配置

<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">

| **参数**                 | **含义**               | **建议配置**                                                 |
| ------------------------ | ---------------------- | ------------------------------------------------------------ |
| **`width=device-width`** | 宽度等于设备宽度。     | **必须设置**，否则布局会乱。                                 |
| **`initial-scale=1.0`**  | 初始缩放比例。         | **1.0**，即不缩放。                                          |
| **`maximum-scale=1.0`**  | 最大缩放比例。         | 防止用户双击或多指操作导致页面乱跳。                         |
| **`user-scalable=no`**   | 是否允许用户手动缩放。 | **no**（如果你的 WebGIS 交互依赖手势，建议关掉，由地图框架处理）。 |

