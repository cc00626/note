## 背景

在项目的app端已支持发布live图，在审核项目中同样需要支持live图的审核，一般这种格式的图片为3s的视频，如果直接以视频的格式展示，审核人员需要来回查看视频，审核效率低。为了提高审核效率，我们将3s视频中抽取出关键的图片，方便审核人员进行查看。



## 技术选型

### 前端实现

1. 使用video + canvas

- 优点：简单容易实现
- 缺点：依赖于浏览器的video标签的播放，部分的格式存在兼容性问题，比如AVI/WMV，对于长视频可能会造成卡顿的现象。



2. 使用ffmpeg + webassembly

- 优点：不存在兼容性问题，通过webassembly的二进制编码直接转换为图片，
- 缺点：实现复杂，需要安装额外的库

### 后端实现

利用一些第三方库进行实现，可能会对服务器性能造成影响。

因为我们返回的格式中，将live图统一处理成了mp4格式，并且视频一般在3s左右，文件不会太大，所以选择了第一种方案。



## 实现

### extractVideoFrame方法

1. 创建video标签，设置为静音，由于浏览器的播放策略，否则可能会出现无法播放的问题。对于苹果设备要设置禁止全屏播放。
2. 添加视频的链接，通过监听视频的loadeddata事件，这个事件会在第一帧加载完成的时候触发。通过设置时间将视频跳到想要截取的位置。
3. 监听seek事件，会在视频跳转完成的时候进行触发，通过width和height设置比例。创建canvas元素，指定高度和宽度和画面。然后导出baseUrl。

```javascript
export const extractVideoFrame = (
  videoSource: string | File | Blob,
  timeInSeconds: number,
  options: ExtractOptions = {}
): Promise<string> => {
  const { maxWidth = 1920 } = options;

  return new Promise<string>((resolve, reject) => {
    if (timeInSeconds < 0) {
      return reject(new Error('抽帧时间不能小于 0'));
    }
	
    //chua
    const video = document.createElement('video');
    video.muted = true;
    video.playsInline = true;
    video.crossOrigin = 'anonymous';

    let objectUrl: string | null = null;

    if (typeof videoSource === 'string') {
      video.src = videoSource;
    } else if (videoSource instanceof File || videoSource instanceof Blob) {
      objectUrl = URL.createObjectURL(videoSource);
      video.src = objectUrl;
    } else {
      return reject(new Error('videoSource 必须是 File、Blob 或 URL'));
    }

    const cleanup = () => {
      video.pause();
      video.removeAttribute('src');
      video.load();
      if (objectUrl) URL.revokeObjectURL(objectUrl);
    };

    const handleError = (e: Event) => {
      cleanup();
      reject(new Error(`视频加载失败: ${(e as ErrorEvent).message || '未知错误'}`));
    };

    // ✅ 关键：使用 loadeddata + seeked
    video.addEventListener('loadeddata', () => {
      if (timeInSeconds > video.duration) {
        cleanup();
        return reject(new Error(`时间 (${timeInSeconds}s) 超出视频总时长 (${video.duration.toFixed(2)}s)`));
      }
      video.currentTime = timeInSeconds;
    }, { once: true });

    video.addEventListener('seeked', () => {
      // ✅ seek 完成，立即抽帧
      let drawWidth = video.videoWidth;
      let drawHeight = video.videoHeight;
      if (drawWidth > maxWidth) {
        const ratio = maxWidth / drawWidth;
        drawWidth = maxWidth;
        drawHeight = Math.round(drawHeight * ratio);
      }

      const canvas = document.createElement('canvas');
      canvas.width = drawWidth;
      canvas.height = drawHeight;
      const ctx = canvas.getContext('2d');
      if (ctx) {
        ctx.drawImage(video, 0, 0, drawWidth, drawHeight);
        const dataUrl = canvas.toDataURL('image/png');
        cleanup();
        resolve(dataUrl);
      } else {
        cleanup();
        reject(new Error('无法获取canvas上下文'));
      }
    }, { once: true });

    video.addEventListener('error', handleError, { once: true });

    video.load();
  });
};
```



### extractThreeFrames方法

1. 通过监听loadedmetadata事件，获取 视频文件的头信息，得到视频的时长信息。
2. 根据不同的时长进行截取

```javascript
if (duration < 1) {
  // 视频时长小于1秒时，提取0秒、中间点和结束前的帧
  timePoints = [0, duration / 2, Math.max(0, duration - 0.1)];
} else if (duration < 3) {
  // 视频时长小于3秒但大于等于1秒时，在可用范围内均匀分布提取点
  const interval = duration / 4;
  timePoints = [interval, interval * 2, interval * 3];
}

// 确保时间点不超过视频时长
timePoints = timePoints.map(time => Math.min(time, Math.max(0, duration - 0.1)));
```



## 如何实现

在我们的审核系统中，App 端已经支持发布 **Live 图**。
 Live 图本质上是 **大约 3 秒的视频**。

```
//ios
IMG_1234.HEIC  （封面图）
IMG_1234.MOV   （3秒视频）

//android
安卓厂商比较放飞自我：

有的用 .mp4
有的用 .jpg + mp4
有的把视频嵌进图片（比如 Motion Photo）

👉 举例：

Google Pixel：Motion Photo（JPEG + MP4嵌入）
小米/华为：类似结构但实现不同
```



如果在审核系统中直接以视频形式展示，会有两个问题：

 审核人员需要 **反复播放视频** 才能看清内容
 视频播放会降低 **审核效率**

所以我们做了一个优化：

从 3 秒视频中抽取几张关键帧图片，直接展示给审核人员。

这样审核人员 **看图片就能完成审核**，效率会高很多。

我们使用video + canvas进行实现，首先创建一个video元素，并设置静音，设置视频的src，通过监听视频的loadeddata事件，表示视频的第一帧已经加载完成，通过设置currenttime跳转到时间点，然后监听seek事件，表示画面已经触发完成，通过创建canvas元素，然后进行绘制。最后导出base64图片。完成后通过video.pause()清除内存。



### canvas如何设置清晰度？

**为什么会丢失清晰度？**

1. 画布尺寸太小。
2. 没有按照视频的原始分辨率进行绘制
3. 没有考虑设备的像素比

```
const dpr = window.devicePixelRatio || 1

canvas.width = drawWidth * dpr
canvas.height = drawHeight * dpr

canvas.style.width = drawWidth + "px"
canvas.style.height = drawHeight + "px"

ctx.scale(dpr, dpr)
```



这个问题主要是canvas的绘图缓冲区尺寸和屏幕的物理像素不匹配导致的。如果我们只按CSS 大小定义 Canvas 的 `width` 属性，浏览器为了填满高分辨率屏幕的物理像素，会对 Canvas 生成的位图进行**拉伸插值**。

为了保证图片不模糊，我读取video.videoWidth作为canvas的基础像素。

如果不设置宽度和高度，canvas默认的像素是300 * 150，会出现细节消失。



### canvas跨域问题

当你尝试将一个**非同源**（协议、域名、端口任一不同）的资源（如图片或视频）绘制到 Canvas 上时，Canvas 本身是可以显示的，但它会进入**“被污染”（Tainted）**的状态。



**解决方法1**

资源所在的服务器必须返回 `Access-Control-Allow-Origin` 响应头，明确允许你的域名访问。



**方法二**

当你设置了 `crossOrigin = 'anonymous'`，浏览器在发起网络请求时，行为会发生改变：

1. **请求头发起申请**：浏览器会在 HTTP 请求头中自动加入 `Origin: [你的域名]`。这相当于告诉服务器：“我想以跨域安全模式使用这个视频，请问你允许吗？”
2. **服务器响应验证**：服务器收到请求后，如果它配置了 CORS，会返回一个响应头：`Access-Control-Allow-Origin: *`（或你的具体域名）。
3. **浏览器终审**：
   - **如果匹配成功**：浏览器认为这个资源是“受信任”的，它会把视频数据交给 Canvas，且**不会**标记 Canvas 为“受污染”。此时你可以自由调用 `toDataURL`。
   - **如果匹配失败**（服务端没配 CORS）：即便视频在页面上能播，但当你调用 `toDataURL` 时，浏览器会直接报错，因为协商失败了。

```
video.crossOrigin = 'anonymous'; // 必须在设置 src 之前！
```

