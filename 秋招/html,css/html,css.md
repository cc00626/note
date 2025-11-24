## html5都新增了哪些标签

### 行内元素，块级元素，行内块元素

**行内元素**

1. 不会自动换行，一行撑满才进行换行
2. 设置padding和margin上下无效，水平方向有效
3. span a i

**块级元素**

1. 会自动进行换行，不设置宽度的情况下，默认是100%
2. 可以设置宽度和高度
3. 可以设置四个方向的padding和margin
4. div form h1-h6 p

**行内块元素**

1. 不可以自动换行
2. 可以设置宽度和高度
3. 可以设置四个方向的padding和margin，默认排序是从左到右
4. img 、 table 、 textarea 、 td 表单控件button

### 替换元素/非替换元素

**替换元素**

内容由外部的资源或者标签的属性来定义内容

**非替换元素**

元素的内容直接由标签所定义

### img总的title和alt是什么意思

alt 表示图片加载失败的时候会显示alt中的内容，搜索引擎可以识别到内容，有利于seo

title表示鼠标悬浮的时候，会显示title中的内容

### DOCTYPE标签有什么用

**文档声明类型标签** 告诉当前的浏览器应该以方式来渲染网页，如果不声明则使用兼容模式。向就的浏览器兼容，防止老的浏览器无法工作。

**标准模式** 盒子模型默认是标准盒模型

**兼容模式** 默认是IE盒模型

### css水平居中

1. 有固定宽度，设置margin 0auto
2. 使用flex布局
3. 子元素绝对定位，父元素相对定位。left 和top 移动50%，通过transform 设置
4. text-cell方式

```
.container {
  display: table-cell;
  vertical-align: middle;
  text-align: center;
}
```

### defer和async的区别

默认情况下，浏览器的渲染线程和js引擎是互斥的，scirpt标签会阻塞html的渲染，defer和async都可以让js加载，和html加载一起执行。

其中async是加载完成之后立即执行，defer是加载完成之后，等待html解析完成后再执行。

**浏览器下载加载js是异步的，执行js代码是同步的**

### 什么是bfc

bfc是块级格式化文本上下文,是在页面中独立渲染的一块区域，不受其他区域的影响。

**特点：**

1. BFC中的块级元素上下margin会重叠，会取两者间最大的。
2. BFC中的元素不会和浮动元素重叠，会计算浮动元素的高度，解决浮动问题。
3. 每个元素的左外边距与包含块的左边界相接触（从左向右），即使浮动元素也是如此 。

**如何开启**

1. float开启浮动定位
2. html元素
3. overflow不是visible
4. postion开启固定定位或者绝对定位
5. display的值是inline-block、inline-flex、flex、table-cell、table-caption

### 清除浮动

#### 1.额外标签法

在元素内部下方添加一个空的标签，css设置成clear：both

```
<div class="father">
  <div class="box">box1</div>
  <div class="box">box2</div>
  <div class="box">box3</div>
  <div style="clear: both;"></div>
</div>
```

#### 2.开启BFC

设置overflow：hidden ，缺点 ：会裁剪掉溢出的部分。

#### 3.after为元素

```
.clearfix:after{
//相当于设置一个块级伪元素
content: "";
display: block;
height: 0;
//设置clear: both
clear: both;
visibility: hidden;
}
```

### css的position定位都有哪些

|   |   |   |   |   |
|---|---|---|---|---|
|position|文档流|偏移属性|定位参考|应用场景|
|static|保留|无效|无|默认布局|
|relative|保留|有效|自身原位置|微调位置、父容器定位参考|
|absolute|脱离|有效|最近非 static 祖先|弹窗、浮动、绝对定位子元素|
|fixed|脱离|有效|视口|固定导航、返回顶部|
|sticky|部分保留|有效|最近滚动容器|表头固定、局部导航|

### css如何设置元素隐藏

1. display：none 在构建渲染的时候会跳过属性，但是会存在dom上。事件不可以进行点击。
2. viability：visible 会保留页面的位置，元素不可见，dom中没有删除，事件不可以点击。
3. opacity: 0 元素不可见，不影响布局，会进行事件的监听。

### css设置两栏，三栏布局

#### 两栏布局：

1. 使用float和margin

```
<style>
  .container {
    width: 100%;
  }

  .left {
    width: 100px;
    height: 300px;
    float: left;
    background-color: red;
  }

  .right {
    height: 300px;
    background-color: blue;
    margin-left: 120px;
  }
</style>
</head>

<body>
  <!-- <div class="container"> -->
  <div class="left">left</div>
  <div class="right">
    <div style="width: 100px;height: 100px; background-color: yellow;"></div>
  </div>
  <!-- </div> -->
</body>
```

2. 使用float和BFC

```
  <style>
    .container {
      width: 100%;
    }

    .left {
      width: 100px;
      height: 300px;
      float: left;
      background-color: red;
    }

    .right {
      height: 300px;
      background-color: blue;
      /* margin-left: 120px; */
      overflow: hidden;
    }
  </style>
</head>

<body>
  <!-- <div class="container"> -->
  <div class="left">left</div>
  <div class="right">
    <div style="width: 100px;height: 100px; background-color: yellow;"></div>
  </div>
  <!-- </div> -->
</body>
```

3. 使用flex布局
4. 父元素设置相对定位，左边元素开启绝对定位，并设置固定宽度。右边元素margin-left设置为左边的宽度。
5. 将父级元素设置为相对定位。左边元素宽度设置为200px，右边元素设置为绝对定位，左边定位为200px，其余方向定位为0。

#### 三栏布局：

1. 三个元素设置position ：absolute 左右设置固定宽度，分别设置left：0 left：200px,right:200px right:0

```
  <style>
    /* .container{

    } */
    .left {
      width: 200px;
      position: absolute;
      left: 0;
      background-color: blue;
    }

    .center {
      position: absolute;
      left: 200px;
      right: 200px;
      background-color: red;
    }

    .right {
      width: 200px;
      position: absolute;
      right: 0;
      background-color: yellow;
    }
  </style>
</head>

<body>
  <div class="left">left</div>
  <div class="center">center</div>
  <div class="right">right</div>
</body>
```

2. 使用flex布局，左右固定宽度，中间flex 1
3. 使用浮动布局，中间元素要放在左右元素的下面。

```
.left {
  float: left; width: 300px; height: 600px;
  background-color: blue;
}
.right {
  float: right; width: 300px; height: 600px;
  background-color: blue;
}


.main {
  height: 600px;
  background-color: #55A532; margin: 0 320px;
}
```

### 说一说flex：1的属性代表什么

flex-grow flex-shink flex-basis的缩写

flex-grow定义了项目放大的比例 默认是0

flex-shink定义了项目空间不足缩小的比例 默认是1

flex-basis定义了项目分配空间的初始大小

- **flex-grow: 1** → 可以放大，占用剩余空间。
- **flex-shrink: 1** → 可以缩小，如果空间不够会被压缩。
- **flex-basis: 0%** → 初始大小当成 0（忽略内容本身大小），靠 `grow` 来分配空间。

挑战**flex-basis: 0%** 可能会遇到项目溢出的情况 可以设置flex basis为auto 或者overflow hidden 来进行解决。

### css选择器

！important

内联样式

id选择器

类选择器 属性选择器 伪类选择器

标签选择器 伪元素选择器

通配符选择器

### css动画

- translate 位移，移动自身的距离
- 旋转rotate，将对象进行旋转

**使用transistion**

当元素属性变化的时候进行过度

**使用animation**

使用keyframes定义关键帧

### 伪类选择器和伪元素选择器的区别

在于是否创建了内容

**伪类选择器：**

相当于添加了类名

可以叠加使用

**伪元素选择器：**

相当于添加了一个标签

一个选择器中只能选择一个

### css实现三角形

宽度和高度设置成0，设置四个边框的宽度，并且设置颜色，其中三个设置成透明

**实现直角三角形？**

设置两个边框的宽度，设置left和top的直角位于左下角。

### margin纵向重叠的问题

#### 1.开启bfc

#### 2.为父元素盒子设置border或者padding

### margin负值的问题

margin-top 元素自身向上移动

margin-left 元素自身向左移动

margin-bottom 元素下方元素向上移动

margin-right 元素右面的元素向左移动

### lineheight继承问题

写比例 继承自身的fontsize * lineheight

### display的属性都有哪些？

|   |   |   |
|---|---|---|
|值|特点|应用场景|
|`block`|块级元素，占据整行；可设置宽高|`<div>`<br><br>, `<p>`<br><br>默认值；布局容器|
|`inline`|行内元素，只占内容宽度；宽高无效|`<span>`<br><br>, `<a>`<br><br>；文本、图标|
|`inline-block`|行内块元素；宽高可设置；不会换行|图标+文本混排、按钮|
|`none`|不渲染元素（从文档流移除）|隐藏元素、动态切换显示|
|`list-item`|块级 + 项目符号或编号|`<li>`<br><br>，列表项|
|`run-in`|既可作为块级也可作为行内元素（不常用）|需要特殊排版的文本|

