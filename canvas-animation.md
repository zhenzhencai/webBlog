
# 小程序直播-疯狂点赞Canvas动画实现原理解析

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e63bd28b55c4548b578d67b1eb019aa~tplv-k3u1fbpfcp-zoom-1.image)

近期，电商直播业务热火朝天，直播间有一个很重要的互动：**点赞**。

为了烘托直播间的氛围，直播相对于普通视频或者文本内容，点赞通常有两个特殊需求：

- 点赞动作次数不限制，引导用户疯狂点赞
- 直播间的所有疯狂点赞，都需要在所有用户界面都动画展现出来

我们先来看点赞效果图：

![](static/animation-demo.gif)

从效果图上我们还看到有几点重要信息：

- 点赞动画图片大小不一，运动轨迹也是随机的。
- 点赞动画图片都是先放大再匀速运动。
- 快到顶部的时候，逐渐缩小并消失。
- 收到大量的点赞请求的时候，点赞动画不扎堆，井然有序持续出现。

刚接到这个需求的时候，考虑过**CSS 动画**实现，但是 CSS 需要手动清除节点来防止节点过多而造成的性能问题。同时 CSS 动画在部分 iphone 机型上会偶现“闪烁”现象。综合考虑，选择用**canvas**实现。

下面介绍实现原理和踩过的那些坑。

## 1 Canvas 绘图实现原理

### 1.1 初始化

页面元素上新建 canvas 标签，初始化 canvas。

canvas 上可以设置 width 和 height 属性，也可以在 style 属性里面设置 width 和 height。

- style 中的 width 和 height 分别代表 canvas 这个元素在界面上所占据的宽高，**即样式上的宽高**。attribute 中的 width 和 height 则**代表 canvas 实际像素的宽高**。
- 使用 `Canvas 2D` 可以使微信小程序环境中的 Canvas 与 W3C 标准 Canvas 接口更为接近，因而可以解决之前接口实现不一致引起的 bug。并且，**Canvas 2D 的同层渲染可以解决图表与其他原生组件覆盖层级的问题**。 想要进一步了解同层渲染的原理，可以参考这篇文章——[《小程序同层渲染原理剖析》](https://developers.weixin.qq.com/community/develop/article/doc/000c4e433707c072c1793e56f5c813)。

```html
<canvas id="likestar" type="2d" width="{{realWidth}}" height="{{realHeight}}" style="width: {{ width }}rpx; height: {{ height }}rpx;" class="like-fx wr-class"></canvas>
```

### 1.2 提前加载图片资源

将需要随机渲染的点赞图片，先预加载，获得图片的宽高，如果有下载失败的，则不显示该随机图片即可。

由于 canvas 不是矢量图，而是像图片一样是位图模式的。高 dpi 显示设备意味着每平方英寸有更多的像素。也就是说二倍屏，浏览器就会以 2 个像素点的宽度来渲染一个像素，该 canvas 在 Retina 屏幕下相当于占据了 2 倍的空间，相当于图片被放大了一倍，因此绘制出来的图片文字等会变**模糊**。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/261df720e57747bf9b9d9c1b564ed701~tplv-k3u1fbpfcp-zoom-1.image)

因此，要做 Retina 屏适配，关键是**知道当前屏幕的设备像素比，将 canvas 放大到该设备像素比来绘制，然后将 canvas 压缩到一倍来展示**。

```js
    // 创建canvas上下文
    this.createSelectorQuery()
      .select('#likestar')
      .fields({ node: true })
      .exec(res => {
        if (res[0]) {
          const canvas = res[0].node;
          if (canvas.getContext) {
            this.ctx = canvas.getContext('2d');
            // 缩放canvs画布解决高清屏幕模糊问题
            const dpr = wx.getSystemInfoSync().pixelRatio;
            canvas.width = this.realWidth * dpr;
            canvas.height = this.realHeight * dpr;
            this.ctx.scale(dpr, dpr);
            for (let i = 1; i < 8; i++) {
              const likeImgae = canvas.createImage();
              likeImgae.src = `https://***.**.***.com/mp/like-animation/like${i}.png`;
              likeImgae.onload = () => {
                this.likeImgList.push(likeImgae);
              };
            }
          }
        }
      });
```

### 1.3 创建渲染对象

#### 1.3.1 气泡属性概览

每个点赞气泡对象包含如下参数：

- **id**：每个气泡拥有独立的 ID，便于控制
- **opacity**：点赞气泡上升到顶部时逐渐消失，需要设置透明度
- **pathData**：气泡的运动路径
- **image**：预加载的气泡图片列表中随机选择一张图片作为气泡图
- **factor**: 运动参数，包含该气泡运动的速度和贝塞尔曲线系数
- **width**: 当前气泡的大小

```js
      const anmationData = {
        id: new Date().getTime(),
        opacity: 1, //透明度
        pathData: this.getRandomInt(0, 1)
          ? this.generatePathData()
          : this.generatePathDataReverse(), // 路径
        image: this.likeImgList[this.getRandomInt(0, length - 1)],
        factor: {
          speed: this.getRandom(0.01, 0.014), // 运动速度，值越小越慢
          t: 0, //  贝塞尔函数系数
        },
        width: this.iconWidth * this.getRandom(0.9, 1.1),
      };
```

#### 1.3.2 曲线路径生成

实时渲染图片，使其变成一个连贯的动画，很重要的是：生成**曲线轨迹**。这个曲线轨迹需要是平滑的均匀曲线。
假如生成的曲线轨迹不平滑的话，那看到的效果就会太突兀，比如上一个是 10 px，下一个就是 -10px，那显然，动画就是忽左忽右左右闪烁了。
理想的轨迹是上一个位置是 10px,接下来是 9px，然后一直平滑到 -10px，这样的坐标点就是连贯的，看起来动画就是平滑运行。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f4f9a10b2d34e488233a3b30a370070~tplv-k3u1fbpfcp-zoom-1.image)

canvas 动画中常使用**贝塞尔曲线**来平滑路径，本次动画应用的是四阶贝塞尔曲线，如上图所示。贝塞尔曲线原理可参考下面两篇文章：

- [深入理解贝塞尔曲线](https://juejin.im/post/6844903666361565191)
- [如何理解并应用贝塞尔曲线](https://juejin.im/post/6844903796582121485)

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/448727f449624ff9a57162d65ccd6e6d~tplv-k3u1fbpfcp-zoom-1.image)

首先，随机生成四个点，作为气泡的运动路径，为了保证气泡的随机性，使气泡不扎堆，将每个点的位置范围都设置为一个区间。

```js
    generatePathData() {
    const { realWidth, realHeight } = this;
    const p0 = {
      x: this.getRandom(0.6, 0.7) * realWidth,
      y: realHeight,
    };
    const p1 = {
      x: this.getRandom(-0.2, 0.5) * realWidth,
      y: this.getRandom(0.6, 0.85) * realHeight,
    };
    const p2 = {
      x: this.getRandom(0.7, 1) * realWidth,
      y: this.getRandom(0.25, 0.5) * realHeight,
    };
    const p3 = {
      x: this.getRandom(0.04, 0.7) * realWidth,
      y: this.getRandom(0, 0.15) * realHeight,
    };
    return [p0, p1, p2, p3];
  }
```

通过运动路径四个点的坐标和贝塞尔曲线系数，更新气泡下一步的位置信息[x,y]。

```js
   /**更新气泡的最新运动路径 */
  updatePath(data, factor) {
    const p0 = data[0];
    const p1 = data[1];
    const p2 = data[2];
    const p3 = data[3];

    const { t } = factor;

    /*贝塞尔曲线，计算多项式系数*/
    const cx1 = 3 * (p1.x - p0.x);
    const bx1 = 3 * (p2.x - p1.x) - cx1;
    const ax1 = p3.x - p0.x - cx1 - bx1;

    const cy1 = 3 * (p1.y - p0.y);
    const by1 = 3 * (p2.y - p1.y) - cy1;
    const ay1 = p3.y - p0.y - cy1 - by1;

    const x = ax1 * (t * t * t) + bx1 * (t * t) + cx1 * t + p0.x;
    const y = ay1 * (t * t * t) + by1 * (t * t) + cy1 * t + p0.y;
    return {
      x,
      y,
    };
  }
```

#### 1.3.3 放大缩小淡出

从效果图中可知，点赞动画都是先放大然后匀速运动，在最后四分之一的路程中逐渐缩小并淡出。可以通过控制该气泡的**width**和**opacity**属性来实现。

```js
   const anmationData = this.queue[+key];
      const { x, y } = this.updatePath(
        anmationData.pathData,
        anmationData.factor,
      );
      const { speed } = anmationData.factor;
      anmationData.factor.t += speed;

      let curWidth = anmationData.width;
      if (y > 0.25 * realHeight) {
        curWidth = (realHeight - y) / 2.5;
        curWidth = Math.min(anmationData.width, curWidth);
      } else {
        curWidth = (0.75 + y / realHeight) * anmationData.width;
      }

      let curAlpha = anmationData.opacity;
      curAlpha = y / realHeight;
      curAlpha = Math.min(1, curAlpha);
      this.ctx.globalAlpha = curAlpha;
      this.ctx.drawImage(
        anmationData.image,
        x - curWidth / 2,
        y,
        curWidth,
        curWidth,
      );
```

#### 1.3.4 边界处理和气泡管理

- 当气泡超出画布边界，会产生**截断**的效果，需要删除气泡。
- 同时，当贝塞尔曲线系数大于 1 时，表示该气泡已经走完完整路程，可以删除。
- 及时清理气泡对象可以有效防止内存泄漏。

```js
      // 贝塞尔曲线系数大于1，删除该气泡
      if (anmationData.factor.t > 1) {
        delete this.queue[anmationData.id];
      }
      if (y > realHeight) {
        delete this.queue[anmationData.id];
      }
      if (x < anmationData.width / 2) {
        delete this.queue[anmationData.id];
      }
```

### 1.4 动画绘制原理

#### 1.4.1 点赞气泡生成

通过监听点赞数**count**的变化来生成气泡，用户每点赞一次，或者接收到 IM 推送的点赞数目更新，就生成气泡并放入**渲染实例队列**。

- 用户自己点赞的时候，随机生成 1-3 个气泡，IM 推送新的点赞数目时，随机生成 2-6 个气泡。

```js
  /**点赞个数变化 */
  likeChange(newVal, oldVal) {
    // this.ctx初始化才能触发点赞个数变化
    if (this.ctx && newVal - oldVal > 0 && this.likeImgList.length) {
      // 自己点赞的时候，随机 1-3个气泡，im推送更新的时候，随机2-6个气泡
      const count =
        newVal - oldVal > 5 ? this.getRandomInt(2, 6) : this.getRandomInt(1, 3);
      this.likeClick(count);
    }
  }
```

- 随机生成每个气泡的对象属性，放入渲染实例队列。第一个气泡进入队列，开启点赞动画，平均每 20s 刷新一次图层。
- 这里为了增强气泡运动轨迹的随机性，定义气泡的镜像运动路径**generatePathDataReverse**。
- 为了使同一时间生成的多个气泡**不扎堆重叠**，将气泡的运动速度设为一个区间[0.01,0.014]。
- 为了增强层次感，将气泡的大小也设置为一个区间[0.9,1.1]。

```js
 /**点赞函数，参数 count 表示一次点赞同时出现的气泡数量*/
  likeClick(count) {
    const { length } = this.likeImgList;
    const curId = new Date().getTime();

    for (let i = 0; i < count; i++) {
      const image = this.likeImgList[this.getRandomInt(0, length - 1)];
      const anmationData = {
        id: curId + i,
        timer: 0, // 定时器
        opacity: 1, //透明度
        pathData: this.getRandomInt(0, 1)
          ? this.generatePathData()
          : this.generatePathDataReverse(), // 路径
        image: image,
        factor: {
          speed: this.getRandom(0.01, 0.014), // 运动速度，值越小越慢
          t: 0, //  贝塞尔函数系数
        },
        width: this.iconWidth * this.getRandom(0.9, 1.1),
      };
      if (Object.keys(this.queue).length > 0) {
        this.queue[anmationData.id] = anmationData;
      } else {
        this.queue[anmationData.id] = anmationData;
        this.bubbleAnimate();
      }
    }
  }
```

#### 1.4.2 动画绘制

上一步当**渲染实例队列**放入第一个气泡时，执行**bubbleAnimate**函数，该函数原理如下：

- 每一帧遍历**渲染实例队列**的所有气泡对象，更新该气泡的**位置坐标**、**贝塞尔曲线系数**、**气泡大小**和**透明度**，在画布上绘制该气泡。
- 从**渲染实例队列**删除运动结束和超出画布边界的气泡，一是防止内存泄漏， 二是优化定时器执行次数提升性能。
- 当**渲染实例队列**气泡数量大于 0 时，设置定时器，每 20ms 循环执行**bubbleAnimate**函数，更新队列中所有气泡的运动轨迹，直到**渲染实例队列**清空，删除定时器。
- 当新的点赞气泡进入时，重新执行**bubbleAnimate**函数。

```js
  /**点赞动画 */
  bubbleAnimate() {
    const { realHeight, realWidth } = this;
    Object.keys(this.queue).forEach(key => {
      const anmationData = this.queue[+key];
      const { x, y } = this.updatePath(
        anmationData.pathData,
        anmationData.factor,
      );
      const { speed } = anmationData.factor;
      anmationData.factor.t += speed;

      let curWidth = anmationData.width;
      if (y > 0.25 * realHeight) {
        curWidth = (realHeight - y) / 2.5;
        curWidth = Math.min(anmationData.width, curWidth);
      } else {
        curWidth = (0.75 + y / realHeight) * anmationData.width;
      }

      let curAlpha = anmationData.opacity;
      curAlpha = y / realHeight;
      curAlpha = Math.min(1, curAlpha);
      this.ctx.globalAlpha = curAlpha;
      this.ctx.drawImage(
        anmationData.image,
        x - curWidth / 2,
        y,
        curWidth,
        curWidth,
      );
      // 贝塞尔曲线系数大于1，删除该气泡
      if (anmationData.factor.t > 1) {
        delete this.queue[anmationData.id];
      }
      if (y > realHeight) {
        delete this.queue[anmationData.id];
      }
      if (x < anmationData.width / 2) {
        delete this.queue[anmationData.id];
      }
    });
    if (Object.keys(this.queue).length > 0) {
      // 每20ms刷新一次图层
      this.timer = setTimeout(() => {
        this.ctx.clearRect(0, 0, realWidth, realHeight);
        this.bubbleAnimate();
      }, 20);
    } else {
      this.ctx.clearRect(0, 0, realWidth, realHeight);
      clearTimeout(this.timer);
      this.timer = null;
    }
  }
```

### 1.5 清除渲染实例队列

开发期间曾遇见这样一个 bug，在直播间进行点赞，退出直播间重新进入时，可能导致点赞动画失效。bug 出现的原因在于退出直播间时，没有及时清空**渲染实例队列**中的气泡，这导致队列中的气泡数量恒大于 0，不会执行**bubbleAnimate**函数。

```js
      if (Object.keys(this.queue).length > 0) {
        this.queue[anmationData.id] = anmationData;
      } else {
        this.queue[anmationData.id] = anmationData;
        this.bubbleAnimate();
      }
```

解决方法就是在组件实例被从页面节点树移除时清除定时器并清空**渲染实例队列**的所有气泡。

```js
  // 删除定时器，同时删除剩下的其他节点
  detached() {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
      Object.keys(this.queue).forEach(key => {
        const anmationData = this.queue[+key];
        delete this.queue[anmationData.id];
      });
    }
  }
```

### 1.7 旧版本的动画效果图

此动画已经迭代了三个版本，此处附上旧版本的动画效果图。
![](static/old-animation.gif)

### 最后

新人上路，第一次写文章，多有不足指出，欢迎指教。

源代码会在请示领导后，进行开源。

**最后的最后，欢迎讨论，点个赞再走吧 ｡◕‿◕｡ ～**
