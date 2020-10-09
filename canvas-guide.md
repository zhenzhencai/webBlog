
# 小程序canvas实践指南-教你如何轻松避坑

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3847af69c704170a045607297807dd9~tplv-k3u1fbpfcp-zoom-1.image)

## 1. 什么是 Canvas？

在 **[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API)** 是这样定义 canvas 的：

> `<canvas>` 是 HTML5 新增的元素，可用于通过使用 JavaScript 中的脚本来绘制图形。例如，它可以用于绘制图形、制作照片、创建动画，甚至可以进行实时视频处理或渲染。

Canvas 是由 HTML 代码配合高度和宽度属性而定义出的可绘制区域。JavaScript 代码可以访问该区域，类似于其他通用的二维 API，通过一套完整的绘图函数来动态生成图形。

> 微信小程序从基础库`1.0.0`开始支持 canvas，`2.9.0` 起支持一套新 Canvas 2D 接口（需指定 type 属性），同时支持**同层渲染**，原有接口不再维护。

微信小程序一开始就支持 canvas，但早期的 canvas 存在许多不足，**canvas 层级过高覆盖其他组件的问题**一直令人诟病。`2.9.0`起，小程序发布了一套新的 Canvas 2D 接口，可以支持**同层渲染**，解决了这个“心头大患”。

## 2. 小程序 canvas 应用场景

### 2.1 绘制海报

现阶段小程序内生成活动的分享海报，一般采用以下两种方法：

- **服务端合成**：直接返回给前端图片 URL
- **客户端合成**：客户端利用 canvas 绘制

在当前的业务场景下，客户端合成是优于服务端合成，可以避免造成不必要的 **CPU** 和 **带宽** 浪费。而且后端爸爸会摆事实讲道理地拒绝这个需求，服务端合成，no way！

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a6bf747579c45d1b29867b54fd0deab~tplv-k3u1fbpfcp-zoom-1.image)

### 2.2 绘制动画

现阶段小程序内简易的动画绘制常用的方案主要有以下四种：

|      动画类型      |                             实现原理                             |                    存在缺陷                    |
| :----------------: | :--------------------------------------------------------------: | :--------------------------------------------: |
|   CSS animations   |           使用`CSS渐变`和`CSS动画`来创建简易的界面动画           |          使用`postion`属性且动画从屏幕外移动到可见区域会导致真机上偶现`闪烁`和`抖动`现象         |
| wx.createAnimation |       使用`wx.createAnimation`接口来动态创建简易的动画效果       | 性能不好，出现卡顿，ios 机型页面偶现`闪烁`现象 |
|     关键帧动画     | 使用`this.animate`创建关键帧动画化，具有更好的性能和更可控的接口 |           ios 机型页面偶现`闪烁`现象           |
|      gif 动画      | 将动画生成 gif 文件，使用小程序的`image`或`cover-image`标签展示  |    在真机上出现`锯齿`和`白边`情况，引人诟病    |

以上四种方案，仅能实现`简易`的动画绘制，且在 ios 真机上会偶现`闪烁`和`抖动`现象。而 canvas 通过 JavaScript 脚本来绘制图形，稳定性更强，且能 `cover` 复杂的动画逻辑，比如模拟转盘抽奖、直播间点赞动画、刮刮乐等效果。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0fa36a1849e4fecba6701ac28096130~tplv-k3u1fbpfcp-zoom-1.image)

## 3.canvas 避坑之旅

总而言之，canvas 在微信小程序开发中占据一席之地，但也有许多不得不填的“坑”。
现阶段，并没有同类的文章系统的记录这些问题，本文主要记录在 canvas 开发实践中遇见的问题和解决方案。

### 3.1 小程序 canvas 绘图层级最高问题？

众所周知，小程序当中有一类特殊的内置组件——原生组件，这类组件有别于 WebView 渲染的内置组件，他们是交由原生客户端渲染的。原生组件作为 Webview 的补充，为小程序带来了更丰富的特性和更高的性能，但同时由于脱离 Webview 渲染也给开发者带来了不小的困扰，戳这里了解[原生组件的使用限制](https://developers.weixin.qq.com/miniprogram/dev/component/native-component.html#%E5%8E%9F%E7%94%9F%E7%BB%84%E4%BB%B6%E7%9A%84%E4%BD%BF%E7%94%A8%E9%99%90%E5%88%B6)。

小程序基础库`1.0.0`开始支持的 canvas API 就是原生组件，原生组件的层级总是最高，不受 z-index 属性的控制，无法与 view、image 等内置组件相互覆盖。因此，canvas 绘图往往在最顶层，在实际的开发过程中，会出现透出的问题。如下图所示，点赞动画和购物袋动画都是由 canvas 绘制，当打开商品列表弹窗时，这两个动画会透出：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e8d53e23edc48b68fac1beb6fda612c~tplv-k3u1fbpfcp-zoom-1.image)

- 最初想到解决方法是监听**商品列表弹窗**的打开事件，弹窗打开的时候将点赞动画和购物袋动画移动到屏幕外，弹窗关闭的时候，移进屏幕内。这种方法治根不治本，随着 PM 不断的“加料”，最后需要维护的组件状态越来越多，每增加一个组件都需要考虑 canvas 层级覆盖的问题。
- 第二种方法时使用 cover-view 和 cover-image 等原生组件，能在一定程度上缓解层级覆盖的问题，但是过度的使用原生组件会导致层级不易维护，后续迭代出现更多的 bug。
- 第三种方法是利用 `canvasToTempFilePath` 临时将 canvas 转成图片，然后隐藏 canvas，显示 tempImage 即可。这种方法适用于静态的 canvas 绘图，对于 canvas 动画而言，每 16ms 刷新一次，将 canvas 画布转成图片十分影响性能。

所幸，小程序开发团队也意识到了原生组件带来的种种限制，对小程序原生组件进行了一次重构，引入了「同层渲染」。想要进一步了解同层渲染的原理，可以参考这篇文章——[《小程序同层渲染原理剖析》](https://developers.weixin.qq.com/community/develop/article/doc/000c4e433707c072c1793e56f5c813)。

小程序基础库`2.9.0` 起支持一套新 Canvas 2D 接口（需指定 type 属性），同时支持**同层渲染**。所以对于 Canvas 开发，想要解决**层级覆盖**的问题，最有效的方法是将旧的 API 改造成新的 Canvas 2D API。

### 3.2 为什么字体无法加粗？

微信开放社区有人提问，为啥我做了如下设置，在模拟器上可以加粗，安卓机上加粗却没有效果。

```js
this.ctx.font = '700 14px normal';
```

这行代码有两个问题：

- `font必须包含字体大小和字体族名`,此处缺乏字体族名。
- `font-weight CSS 属性指定了字体的粗细程度。 一些字体只提供 normal 和 bold 两种值。`,为了安全起见，加粗用`bold`。

因此，对代码做出了如下优化：

```js
this.ctx.font = '${fontWeight >= 700 || fontWeight === 'bold' ? 'bold' : 'normal'} ${fontSize}px  ${fontFamily || 'sans-serif'}';
```

### 3.3 Canvas 为啥无法绘制 base64 图片？

在海报绘制的业务场景中，`太阳码`或`二维码`需要用户提供部分参数，由服务端生成图片返回给前端，这时一般不会返回图片的 URL，而是将图片进行 base64 转码后返回给前端。然而，canvas 用户绘图的 API-`drawImage`无法识别 base64 格式。

以下是解决方案：

1. 使用 `wx.base64ToArrayBuffer` 将 base64 数据转换为 ArrayBuffer 数据。
2. 使用 `FileSystemManager.writeFile` 将 ArrayBuffer 数据写为本地用户路径的二进制图片文件。
3. 此时的图片文件路径在 `wx.env.USER_DATA_PATH` 中， `wx.getImageInfo` 接口能正确获取到这个图片资源并 drawImage 至 canvas 上。

```js
const fsm = wx.getFileSystemManager();
const FILE_BASE_NAME = 'tmp_base64src';

const base64src = function(base64data) {
  return new Promise((resolve, reject) => {
    // 写文件时记得去掉base64的头部信息
    const [, format, bodyData] = /data:image\/(\w+);base64,(.*)/.exec(base64data) || [];
    if (!format) {
      reject(new Error('ERROR_BASE64SRC_PARSE'));
    }
    const filePath = `${wx.env.USER_DATA_PATH}/${FILE_BASE_NAME}.${format}`;
    const buffer = wx.base64ToArrayBuffer(bodyData);
    fsm.writeFile({
      filePath,
      data: buffer,
      encoding: 'binary',
      success() {
        resolve(filePath);
      },
      fail() {
        reject(new Error('ERROR_BASE64SRC_WRITE'));
      },
    });
  });
};

export default base64src;
```

### 3.4 小程序 canvas 绘制不了网络图片？

小程序的 `canvas.drawImage`是不支持网络图片的，只支持本地图片。所以，任何的网络图片都需要先缓存到本地，再通过 `drawImage`调用存储的本地资源进行绘制，缓存可以通过 `wx.getImageInfo` 和 `wx.downloadFile` 实现。

- 使用`wx.getImageInfo`获取到图片的临时路径

```js
const ctx = wx.createCanvasContext('myCanvas'); //获取canvas画布对象
wx.getImageInfo({
  src: 'https://******.com/example.png', //网络图片路径
  success: res => {
    const path = res.path; //图片临时本地路径
    ctx.drawImage(path, 0, 0, 100, 100); //绘制画布上的路径
    ctx.draw(true);
  }
});
```

- 使用`wx.downloadFile`获取到图片的临时路径

```js
const ctx = wx.createCanvasContext('myCanvas'); //获取canvas画布对象
wx.downloadFile({
  url: 'https://******.com/example.png', //网络路径
  success: res => {
    const path = res.tempFilePath; //临时本地路径
    ctx.drawImage(path, 0, 0, 100, 100); //绘制画布上的路径
    ctx.draw(true); //绘制
  },
});
```

然而有人会问，为啥我在 ide 上能绘制图片，而真机却拿不到图片。这是因为**微信安全域名**的问题，需要在 **小程序后台 > 设置 > 服务器域名 > downloadFile 合法域名**里设置网络图片的域名。ps.因为域名要求是 https 的, 并且一个月只能修改五次，建议把需要下载的网络图片放在自己的 https 的服务器上，再走个 CDN 什么的。

我猜，还会有人问，为啥设置了安全域名后，在真机上还是无法显示绘图。这里需要考虑图片加载的时间，如果图片还未加载就开始绘制，那么就会报错。可以用 image 的`bindload`事件或者`downloadTask.onProgressUpdate`来监听图片加载过程。

基础库 2.7.0 起，小程序方发布了`Canvas.createImage()`，使用这个 api 可以加载网络图片。使用方法如下：

```js
const tempImgae = canvas.createImage();
tempImage.src = 'https://******.com/example.png';
tempImage.onload = () => {
    // 图片加载完后，可以对 tempImage 随意操作
};
```

### 3.5 多行文字如何实现自动换行?

和 CSS 相比，SVG 以及 canvas 对文字排版的支持很弱。CSS 一个`word-break`能解决的问题，canvas 却不行。

> `CanvasContext.measureText(string text)`用于测量文本尺寸信息。目前仅返回文本宽度 。

canvas 自动换行的实现原理在于`CanvasContext.measureText(string text)`这个 API，可以返回一个 TextMetrics 对象，其中包含了当前上下文环境下 text double 精度的占据宽度，于是我们就可以通过每个字符宽度的不断累加，精确计算哪个位置应该可以换行。

参考代码如下：

```js
  wrapText(
    ctx,
    text: string,
    x: number,
    y: number,
    maxWidth = 300,
    lineHeight = 16,
  ) {
    // 字符分隔为数组
    const arrText = text.split('');
    let line = '';

    for (let n = 0; n < arrText.length; n++) {
      const testLine = line + arrText[n];
      const metrics = ctx.measureText(testLine);
      const testWidth = metrics.width;
      if (testWidth > maxWidth && n > 0) {
        ctx.fillText(line, x, y);
        line = arrText[n];
        y += lineHeight;
      } else {
        line = testLine;
      }
    }
    ctx.fillText(line, x, y);
  }
```

当然，在实际的业务场景中，更多的需求是**实现固定行数文本换行且溢出省略**的功能，实现原理一致，这里不做陈述。

### 3.6 如何实现文字数值排列？

文字竖直排列，英文可以使用`context.rotate()`旋转 90deg 实现，但这对于中文，是完全不适用的。在 CSS 中，我们可以使用`writing-mode`改变文档流的方向，从而实现文字竖排。使用 canvas 实现需要**混合计算逐字排列**，计算规则如下：全角字符竖排，英文数字等半角字符旋转排列。

参考代码如下,源自[张鑫旭-canvas 文本绘制自动换行、字间距、竖排等实现](https://www.zhangxinxu.com/wordpress/2018/02/canvas-text-break-line-letter-spacing-vertical/)：

```js
fillTextVertical(text, x, y) {
    var context = this;
    var canvas = context.canvas;

    var arrText = text.split('');
    var arrWidth = arrText.map((letter) => {
        return context.measureText(letter).width;
    });

    var align = context.textAlign;
    var baseline = context.textBaseline;

    if (align == 'left') {
        x = x + Math.max.apply(null, arrWidth) / 2;
    } else if (align == 'right') {
        x = x - Math.max.apply(null, arrWidth) / 2;
    }
    if (baseline == 'bottom' || baseline == 'alphabetic' || baseline == 'ideographic') {
        y = y - arrWidth[0] / 2;
    } else if (baseline == 'top' || baseline == 'hanging') {
        y = y + arrWidth[0] / 2;
    }

    context.textAlign = 'center';
    context.textBaseline = 'middle';

    // 开始逐字绘制
    arrText.forEach((letter, index) => {
        // 确定下一个字符的纵坐标位置
        var letterWidth = arrWidth[index];
        // 是否需要旋转判断
        var code = letter.charCodeAt(0);
        if (code <= 256) {
            context.translate(x, y);
            // 英文字符，旋转90°
            context.rotate(90 * Math.PI / 180);
            context.translate(-x, -y);
        } else if (index > 0 && text.charCodeAt(index - 1) < 256) {
            // y修正
            y = y + arrWidth[index - 1] / 2;
        }
        context.fillText(letter, x, y);
        // 旋转坐标系还原成初始态
        context.setTransform(1, 0, 0, 1, 0, 0);
        // 确定下一个字符的纵坐标位置
        var letterWidth = arrWidth[index];
        y = y + letterWidth;
    });
    // 水平垂直对齐方式还原
    context.textAlign = align;
    context.textBaseline = baseline;
};
```

### 3.7 canvas 如何绘制圆角矩形？

微信小程序允许对普通元素通过 `border-radius` 的设置来进行圆角的绘制，但有时候在使用 canvas 绘图的时候，也需要圆角,但 canvas 并未提供绘制圆角矩形的 kpi，这时候，就需要“曲线救国”。
首先，了解一下画圆的 api：

> CanvasContext.arc(number x, number y, number r, number sAngle, number eAngle, boolean counterclockwise)

- number x 表示圆心的 x 坐标,
- number y 表示圆心的 y 坐标
- number r 表示圆的半径
- number sAngle 表示起始弧度，单位弧度（在 3 点钟方向）
- number eAngle 终止弧度
- boolean counterclockwise 弧度的方向是否是逆时针

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26ac21ca6f2f4a30b61124823149281b~tplv-k3u1fbpfcp-zoom-1.image)

因此，我们可以先绘制四段圆弧，再利用`closePath`方法会连接路径的特点，即可画出圆角矩形。封装函数如下：

```js
const drawRoundedRect = (ctx, width, height, radius, type='fill') => {
  ctx.moveTo(0, radius);
  ctx.beginPath();
  ctx.arc(radius, radius, radius, Math.PI, 1.5 * Math.PI);
  ctx.arc(width - radius, radius, radius, 1.5 * Math.PI, 2 * Math.PI);
  ctx.arc(width - radius, height - radius, radius, 0, 0.5 * Math.PI);
  ctx.arc(radius, height - radius, radius, 0.5 * Math.PI, Math.PI);
  ctx.closePath();
  ctx[method]();
};
```

当然，仅仅绘制圆角矩形是不够的。实际业务需求中，更多的是，给图片添加圆角。这里，需要用到如下 api：

> CanvasContext.createPattern(string image, string repetition)

- 对指定的图像创建模式的方法，可在指定的方向上重复元图像

```js
const drawRoundRectImage = (ctx, x, y, width, height, radius, image) => {
    //圆的直径必然要小于矩形的宽高
    if (2 * radius > width || 2 * radius > height) {
      return false;
    }
    // 创建图片纹理
    const pattern = ctx.createPattern(obj, "no-repeat");
    cxt.save();
    cxt.translate(x, y);
    //绘制圆角矩形的各个边
    this.drawRoundedRect(ctx, width, height, radius);
    cxt.fillStyle = pattern;
    cxt.fill();
    cxt.restore();
  }
```

### 3.8 Canvas 绘图模糊问题

近期因为业务开发需要，接触了 canvas 动画，在开发中发现**绘制的点赞图标异常模糊**，如下图所示，左图是最初开发时绘制的图标，右图是修复这个问题后绘制的图标，清晰度得到质的飞跃。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/261df720e57747bf9b9d9c1b564ed701~tplv-k3u1fbpfcp-zoom-1.image)

#### 3.8.1 为什么会模糊？

在浏览器的 window 变量中有一个`devicePixelRatio`属性，该属性决定了浏览器会用几个像素点来渲染 1 个像素，举例来说，假设某个屏幕的 devicePixelRatio 的值为 2，一张 100x100 像素大小的图片，在此屏幕下，会用 2 个像素点的宽度去渲染图片的 1 个像素点，因此该图片在此屏幕上实际会占据 200x200 像素的空间，相当于图片被放大了一倍，因此图片会变得模糊。
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b01c47eae4da4caaaf0bdef6afd9d981~tplv-k3u1fbpfcp-zoom-1.image)

从上面的图可以看出，在同样大小的逻辑像素下，高清屏所具有的物理像素更多。普通屏幕下，1 个逻辑像素对应 1 个物理像素，而在 dpr = 2 的高清屏幕下，1 个逻辑像素由 4 个物理像素组成。

相信所有了解过 Canvas 绘图的同行都知道 canvas 绘制的是位图，位图又叫像素图或栅格图，它是通过记录图像中每一个点的颜色、深度等信息来存储和显示图像。具象一点讲，可以将位图想象成一个巨大的拼图，这个拼图有无数的拼块，每个拼块代表了一个纯色的像素点。**理论上，1 个位图像素对应着 1 个物理像素**。但假如说你使用了高清屏，比如苹果的 retina 屏去查看一幅图画，又会是什么样子呢？

假设我们有如下代码，该代码将展示在 iphoneX(Dpr=3)的 retina 屏上：

```html
<canvas width="320" height="150" style="width: 320px; height: 150px"></canvas>
```

其中，style 中的 width 和 height 分别代表 canvas 这个元素在界面上所占据的宽高，**即样式上的宽高**。attribute 中的 width 和 height 则**代表 canvas 实际像素的宽高**。

iphoneX 本身的物理像素为 1125 _ 2436，而设备独立像素为 375 _ 812，这代表着 1 个 css 像素实际由 9 个物理像素构成，canvas 的像素为 320 _ 150，其 css 像素为 320 _ 150，则代表 1 个 css 像素将会由 1 个 canvas 元素构成，这样进行换算，**在 retina 屏幕下，1 个 canvas 像素（或者说是 1 个位图像素）将会填充 9 个物理像素，由于单个位图像素不可以再进一步分割，所以只能就近取色，从而导致图片模糊。**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c8fdbc9eb854a6d9792e37bc7711fab~tplv-k3u1fbpfcp-zoom-1.image)

上图说明位图在 retina 屏幕下是如何填充的，上图中左侧的是在普通屏幕下的显示规则，可以看出有 4 个位图像素点，而右侧的高清屏幕下则有 16 个像素点。由于像素点不可切割的原因，颜色产生了改变。

#### 3.8.2 如何解决绘图模糊问题？

了解了问题出现的原因，解决问题就很容易。要做 Retina 屏适配，关键是让 1 个 canvas 像素和一个物理像素挂等号。

参考代码如下：

```js
this.ctx = canvas.getContext('2d');
// 获取retina屏幕的设备像素比
const dpr = wx.getSystemInfoSync().pixelRatio;
// 根据设备像素比，扩大canvas画布的像素，使1个canvas像素和1个物理像素相等
canvas.width = this.realWidth * dpr;
canvas.height = this.realHeight * dpr;
// 由于画布扩大，canvas的坐标系也跟着扩大，如果按照原先的坐标系绘图内容会缩小
// 所以需要将绘制比例放大
this.ctx.scale(dpr, dpr);
```

### 3.9 Canvas 动画在部分 iphone 机型上绘制过多清空画布问题？

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a13fa925a784d09869a1b64f5810e51~tplv-k3u1fbpfcp-zoom-1.image)

最近接到了如下图所示的挂件和购物袋动画的优化需求。最初的版本使用设计大大们给的 gif 图片，gif 图片在真机上的白边和锯齿问题“遭人诟病”。在设计大大们的“威逼利诱”下，只能考虑动画实现。前面也提到过，CSS 动画在真机上会偶现`闪烁`和`抖动`现象，`wx.createAnimation`和`this.animate`在部分 iphone 机型中无法获取动画周期，页面偶现`闪烁`现象,**比如一个动画周期是 2s，有时候 iphone 机型无法获取这个时间，会在 1s 甚至更短的时间内执行这个动画，造成“闪烁”的效果**。

总而言之，Canvas 动画才是最佳实践。然而小程序的`canvas 2d API`也存在不足，比如图片绘制过多的情况下，会自动清空画布。如下图所示，倒计时的动画执行到第 8 秒的时候，画布突然清空。左边的活动挂件也遇见过同样的问题，画布突然清空。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6779278fece045848eb4ef5fcc3c59fb~tplv-k3u1fbpfcp-zoom-1.image)

网上也有很多类似的问题，比如[“ios 上重复跳转到某页面并用 canvas 画图时会导致运行内存不足或意外退出”](https://developers.weixin.qq.com/community/develop/doc/00008e5e50859800f63a89b2750000), [“canvas 2D 真机不显示，开发工具上无任何问题？”](https://developers.weixin.qq.com/community/develop/doc/0008c4bf5147f869514a9698256000)。总结一下就是，ios 机型上绘制 canvas 过于频繁可能会导致画布清空、小程序崩溃。

排查了这个问题很久，推断出一种原因，可能是动画执行过程中，倒计时文本刷新，导致需要重新绘制图片，**两次绘制的时间间隔太短，导致程序崩溃，画布清空**。优化方法如下：

- 文本不使用 canvas 绘制，canvas 仅绘制挂件图片，文本使用<view>标签，通过 css 布局放置于 canvas 画布上。
- 添加兜底策略，在 canvas 画布底下放置一张静态的挂件图片，如果画布突然清空，显示底下的静态图片。这里需要注意的是，底下的图片需要适当缩小，确保挂件执行动画时，不会透出底下的图片。
  
后续对问题进行跟进，发现 **Canvas频繁的创建和销毁可能导致ios7.0.15之前的版本闪退，这个bug在ios7.0.15已经修复。** 针对这个缺陷，想出了以下两个优化方向：
  
 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad7592ce80054e51928ec1ff4859ec19~tplv-k3u1fbpfcp-zoom-1.image)

+ 对于周期性循环播放的动画，比如活动挂件动画，可以使用`apng`格式的图片替代。**APNG是一种类似GIF的动态图片，开发中使用GIF展现包含透明背景的动态效果往往很糟糕，比如图像边缘的白边、锯齿、色彩失真，用APNG代替GIF可以完美解决上述问题。** 小程序侧支持apng动画，如果部分机型出现兼容性问题，apng会向下兼容显示为第一帧的静态图。
+ **canvas多合一**。将多种业务的canvas组件的绘图逻辑合并到同一个canvas中，这里需要处理许多问题，如组件层级、业务代码的耦合度、绘图帧率等等。

### 3.10 Canvas 2d 常见问题&避坑小技巧

- canvas 2d 暂不支持**真机调试**，请直接使用**真机预览**。
- canvas 2d 在 ide 上的表现效果等同于原生组件，仍然会“透出”。需要**在真机上查看实际的效果**。
- canvas 标签默认宽度 300px、高度 150px。开发时要记得**显式设置 canvas 标签的宽度和高度**。
- 避免设置过大的宽高，在安卓下会有 **crash** 的问题。
- **同一页面中的 canvas-id 不可重复**，如果使用一个已经出现过的 canvas-id，该 canvas 标签对应的画布将被隐藏并不再正常工作。
- canvas 2d 的画布有 4096 **大小限制**, 旧版 canvas 没有。
- Canvas 2D 同层渲染在 Pixel 3 失效，由于**国外渠道的微信版本不支持同层渲染**。
  
## 4 最后

因为业务开发需要，作者接触canvas开发两个月，总结分享实践中遇到的一些问题。因业务水平有限，在某些问题上解释说明若有出入，还请批评指教！
  
**最后的最后，欢迎讨论，点个赞再走吧 ｡◕‿◕｡ ～**
  


