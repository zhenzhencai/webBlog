# 小程序直播-评论弹幕是如何“练”成的？

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8384401950a34d52a7b95ea116290bb3~tplv-k3u1fbpfcp-watermark.image)

# 1 前言
近期，电商直播业务热火朝天，为了烘托直播间的氛围，达到主播与观众的有效正反馈，_弹幕_、_点赞_、*用户行为*提示三件套成为一个标准的直播间必不可少的装备。两个月前，作者曾解析了点赞动画的实现原理（文章戳[这里](http://km.oa.com/articles/show/468146)），虽文笔粗糙，但在总结中，对技术原理有了更加透彻的理解。恰逢近期沉淀直播间业务，重新封装了弹幕组件，对代码的设计有了新的理解，总结成一篇文章，与君共享。

提起**弹幕(dànmù)**，大家都会想到「视频弹幕」。视频弹幕是指网友们在观看视频的同时参与评论，即所谓“即时反馈”， 评论以飞行形式横穿屏幕，视觉效果类似多发密集的子弹飞速而过，故称之为“弹幕”。“弹幕”最大的特点就是允许受众在观看直播的同时将评论内容发送到服务器与直播同步播放，这可以让观众的反馈瞬间产生，与主播发生**即时互动**，甚至可以形成**隔空对话**。

随着手机竖屏直播时代的到来，弹幕也从原本的“横穿飞行”，衍生出一种新的模式——**在屏幕左下方竖向滚动**。实际效果如下图所示。

![](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e6d8744d3024a6a8f3189ded3ce3ecf~tplv-k3u1fbpfcp-zoom-1.image)

从效果图上我们还看到有几点重要信息:

- **历史弹幕上屏**：为了活跃气氛，观众初次进入直播间可以观看前 30 条历史弹幕。
- **即时弹幕消息上屏**：即时收到的弹幕消息从底部上屏，并实现自动滚动。
- **个人评论消息上屏**：个人评论的消息“优先”上屏，不受即时消息堆积影响。
- **系统提示消息上屏**：系统提示消息分两种，一种是固定提示，固定在弹幕列表头部，不会消失。另外一种是主播操作产生的临时提示，跟弹幕消息一起，超过数量限制时，就会被清理。

看似简单，实现的过程中却需考虑如下几点：

- **即时消息堆积问题**： 在李佳琦、薇娅等超级主播的直播间，每秒接收到的消息数以千计，如果不做“屏控”处理，且有计划地过滤掉“低质”弹幕，而是一股脑地将消息上屏，弹幕便会飞速滚动，那么观众和主播都无法有效地获取内容信息。
- **上屏弹幕堆积问题**： 随着时间的推移，上屏弹幕数量逐渐累积，如果不及时清理陈旧的`DOM`节点，会导致系统卡顿，甚至程序崩溃。
- **用户互动体验优化问题**： 当用户滚动弹幕时，需要暂停弹幕上屏逻辑，便于用户进行操作；当弹幕滚动到底部时，自动恢复弹幕上屏逻辑；长期暂停的弹幕需要设有自动恢复机制，等等体验问题都是需要优化的内容。
- **历史弹幕上屏问题**： 如若 30 条历史弹幕一窝蜂上屏，弹幕会飞速滚动，给用户极差的体验观感，需要制定策略实现分布式上屏。

# 2 千里之行，始于布局

## 2.1 组件的 data 与 properties

一口吃不成胖子，先从效果图显示的布局入手，思考封装这个组件，需要传入哪些参数，哪些是通过`properties`由父组件传入，哪些通过`data`来维护，与视图进行通信。

```js
  data = {
    toLast: 'item0', // 弹幕滚动索引
    realBarrage: [], // 实际上屏的弹幕
  };
  properties = {
    barrageHeight: {
      type: Number,
      value: 500,
    }, // 弹幕容器高度
    config: {
      type: Object,
      value: DEFAULT_CONFIG,
    }, // 弹幕配置
    tagConfig: {
      type: Array,
      value: TAG_DEFAULT_CONFIG,
    }, // 标签配置信息
    systemHint: {
      type: String,
      value: DEFAULT_SYSTEM_HINT,
    }, // 系统提示信息
  };
```

先来谈谈`properties`属性：

- **barrageHeight**： *Number*类型，弹幕容器高度，可动态调节，如下图所示，超级弹幕出现时，需要缩小弹幕高度。

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f7cc946b33c46ad826b0d5c36824ddd~tplv-k3u1fbpfcp-watermark.image" alt="弹幕高度变化图片示例" width="600"  align="top" />

- **systemHint**：*String*类型，系统提示信息内容，长期存在于弹幕列表头部，默认配置信息如下。

```js
const DEFAULT_SYSTEM_HINT =
  '系统提示：欢迎来到直播间！直享倡导绿色直播，文明互动，购买直播推荐商品时，请确认购买链接描述与实际商品一致，避免上当受骗。如遇违法违规现象，请立即举报！';
```

- **config**：*Object*类型，弹幕配置信息, 可配置参数如下：

```js
const DEFAULT_CONFIG = {
  INTERVAL: 300, // 刷新频率，默认300ms
  BARRAGE_MAX_COUNT: 50, // 上屏弹幕的最大数量
  POOL_MAX_COUNT: 50, // 弹幕池（未上屏）弹幕上限
  BARRAGE_MAX_FRAME: 6, // 屏控处理，每次同时上屏弹幕的数量
  SLEEP_TIME: 5000, // 弹幕休眠时间，默认5000ms
  CHECK_SLEEP: true, // 是否休眠，休眠超过 SLEEP_TIME ，则开启自动滚动
};
```

- **tagConfig**：*Array*类型，标签配置信息，主要用于自定义化标签的样式和内容，当前的默认配置如下：

```js
const TAG_DEFAULT_CONFIG = [
  {
    bgColor: 'linear-gradient(to right, #fb3e3e, #ff834a)',
    tagName: '主播',
  },
  {
    bgColor: 'linear-gradient(to top, #ffb365, #ff8c17)',
    tagName: '号主',
  },
  {
    bgColor: 'linear-gradient(to left, #8bb1ff, #5195ff)',
    tagName: '粉丝',
  },
];
```

然后，再谈谈`data`属性：

- **realBarrage**: *Array*类型，实际上屏的弹幕列表，分两种，一种是评论消息，一种是系统消息。弹幕列表中每一项包含的属性类型说明如下：

```js
interface queueI {
  name?: string; // 用户昵称
  comment?: string; // 评论内容
  isMe?: boolean; // 是否自己发送的弹幕
  tagIndex?: number; // 标签索引，可自定义，默认情况 0-主播，1-号主，2-粉丝
  systemInfo?: string; // 系统提示消息，如果存在这个属性，则前面四个属性无需存在
}
```

**其中，`name`、`comment`为普通评论消息的必须属性，`systemInfo`为系统消息的必须属性，两者互斥。** 弹幕数据`tagIndex`属性与标签配置信息中`TAG_DEFAULT_CONFIG`数组的索引一一对应，默认情况 0-主播，1-号主，2-粉丝。如果修改标签配置信息，那么`tagIndex`属性值的对应关系也要重新梳理。

- **toLast**: *String*类型，弹幕索引，随着新弹幕不断上屏，弹幕列表需要实现自动滚动，小程序提供的`<scroll-view>`组件的`scroll-into-view`属性支持滚动到对应的子元素 id，所以需要维护`toLast`来指定。

## 2.2 开始布局

知道了每个`data`与`properties`的含义，可以根据设计稿，对布局一把梭了。为了实现可滚动视图区域，组件外层使用`<scroll-view>`包裹。用到的属性如下表所示：

| 属性                  | 类型          | 说明                                                                                          | 默认值  | 必填 |
| --------------------- | ------------- | --------------------------------------------------------------------------------------------- | ------- | ---- |
| scroll-y              | boolean       | 允许纵向滚动                                                                                  | _false_ | 否   |
| scroll-with-animation | boolean       | 在设置滚动条位置时使用动画过渡                                                                | _false_ | 否   |
| scroll-into-view      | string        | 值应为某子元素 id（id 不能以数字开头）。设置哪个方向可滚动，则在哪个方向滚动到该元素          | -       | 否   |
| lower-threshold       | number/string | 距底部/右边多远时，触发 scrolltolower 事件                                                    | 50      | 否   |
| bindscrolltolower     | eventhandle   | 滚动到底部/右边时触发                                                                         | -       | 否   |
| bindscroll            | eventhandle   | 滚动时触发，event.detail = {scrollLeft, scrollTop, scrollHeight, scrollWidth, deltaX, deltaY} | -       | 否   |
| bindtouchstart        | eventhandle   | 手指触摸动作开始                                                                              | -       | 否   |
| bindtouchend          | eventhandle   | 手指触摸动作开始                                                                              | -       | 否   |

容器内的子元素包含两部分，一部分的直播间提示消息，放在滚动列表的头部，永远不会被清理。另外一部分是即时消息，分为评论消息和系统消息，放在`realBarrage`中，达到数量最大限制时会被清理。

视图布局代码如下所示：

```html
<scroll-view class="live-barrage wr-class" style="max-height:{{barrageHeight}}rpx;" scroll-y="true" scroll-with-animation="true" scroll-into-view="{{toLast}}" bindtouchstart="barrageTouchStart" bindtouchend="barrageTouchEnd" lower-threshold="100" bindscroll="handleScrollBarrageContainer" bindscrolltolower="handleScrollBottom">
  <!-- 直播间提示消息 -->
  <view class="live-barrage-system system-class" id="item0">{{systemHint}}</view>
  <!-- 弹幕列表 -->
  <view class="{{item.systemInfo? 'live-barrage-system' : 'live-barrage-item item-class'}}" wx:for="{{realBarrage}}" wx:for-item="item" wx:key="index" id="item{{index+1}}">
    <!-- 弹幕消息 -->
    <view class="live-barrage-item-content" wx:if="{{!item.systemInfo}}">
      <view class="live-barrage-item-content-tag" wx:if="{{item.tagIndex >= 0 &&  tagConfig[item.tagIndex]}}" style="background-image: {{tagConfig[item.tagIndex].bgColor}}">
        {{tagConfig[item.tagIndex].tagName}}
      </view>
      <view class="{{item.isMe ? 'live-barrage-item-content-name-me' :  'live-barrage-item-content-name'}}">
        {{item.name}}:\t
      </view>
      {{item.comment}}
    </view>
    <!-- 系统消息 -->
    <view wx:else>{{item.systemInfo}}</view>
  </view>
</scroll-view>
```

# 3 封装一个类

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c1a6fe66d004681849ed1f97b8f5c58~tplv-k3u1fbpfcp-watermark.image)

根据需求，对功能进行思考和拆分，抽象出一个通用类，此处定义为`QueueBarrage`。这个类维护两个优先级队列，一个是弹幕池，一个是上屏弹幕。服务器推送的弹幕消息进入弹幕池，池子里的弹幕需要进行过滤重排、溢出处理。设置轮询机制，对弹幕进行分批次上屏，同时上屏弹幕也设有溢出处理策略。自己发送的弹幕不走弹幕池，直接上屏。

## 3.1 类的属性定义和构造器

下列代码是类的属性定义和构造器。

- **queueList**：弹幕池，包括所有未上屏的弹幕。
- **barrageList**：上屏的弹幕列表。
- **changeCallback**：弹幕上屏回调，弹幕上屏逻辑是业务逻辑，需要以回调的形式传入。
- **config**: 弹幕配置信息，通过创建类实例传参，可以覆盖默认配置信息。

```js
export default class QueueBarrage {
  queueList: queueI[]; // 弹幕池，包括所有弹幕
  barrageList: queueI[]; // 上屏弹幕
  changeCallback; // 弹幕上屏回调
  config; // 配置信息
  isPaused: boolean; // 弹幕是否暂停
  isActive: boolean; // 弹幕是否休眠
  timer;
  private checkActiveTimer;
  constructor(config = {}) {
    this.config = { ...defaultConFig, ...config };
    this.queueList = [];
    this.barrageList = [];
    this.isPaused = false;
    this.isActive = false;
    this.flush();
  }
}
```

## 3.2 接收到的消息进入弹幕池

前面提到过，在李佳琦、薇娅等超级主播的直播间，每秒接收到的消息数以千计，如果直接一股脑地将消息上屏，弹幕便会飞速滚动，那么观众和主播都无法有效地获取内容信息。因此，需要维护一个「弹幕池」，能对弹幕进行优先级排序，有效地过滤掉“低质”弹幕，同时对池子的容量做限制，添加“溢出”处理策略。

代码实现如下所示，将消息放入弹幕池，如果弹幕数量超出最大数量限制，则对弹幕进行过滤，同时删除超出的那部分弹幕。当然，历史弹幕信息为了保证上下文逻辑的严谨性，是无需进行优先级排序的，所以只需要截取超出的部分。为了区分两种弹幕类型，本函数的第二个参数`isFilter`用来控制是否进行过滤。

```js
  barrageEnterQueue(queue: queueI[], isFilter = true) {
    queue.forEach(v => {
      this.queueList.push(v);
    });
    // 进入直播间历史弹幕不进行过滤
    if (this.queueList.length > this.config.POOL_MAX_COUNT) {
      if (isFilter)
        this.queueList = filter(this.queueList, this.config.POOL_MAX_COUNT);
      else
        this.queueList.splice(
          0,
          this.queueList.length - this.config.POOL_MAX_COUNT,
        );
    }
    if (!this.isPaused) {
      !this.timer && this.flush();
    }
  }
```

那么过滤规则是怎么样的呢？这得视业务情况而定，下面贴出本业务的弹幕过滤逻辑：

- 无意义弹幕的权重降低 0.5
- 短弹幕的权重降低 0.2
- ~~自己发送的弹幕的权重提高 1.0~~

这里需要注意的是，自己发送的弹幕的权重优先级是最高的，可以走弹幕池，通过过滤提高优先级，但这需要一定的时间消耗。不走弹幕池，直接上屏是比较合理的实现方案。

```js
const CONTENT_FIELD = 'comment';
const REG_MEANINGLESS = /^[\d\s\!\@\#\$\%\^\&\*\(\)\-\=]+$/;
// 权重计算规则
const rules = [
  function meaningless(this: any, weight) {
    return this[CONTENT_FIELD] && REG_MEANINGLESS.test(this[CONTENT_FIELD])
      ? weight - 0.5
      : weight;
  },
  function barrage2short(this: any, weight) {
    return this[CONTENT_FIELD] && this[CONTENT_FIELD].length < 3
      ? weight - 0.2
      : weight;
  },
];
```

接下来，便可根据权重计算规则对弹幕进行排序且筛选。第一个参数是弹幕列表，第二个参数是最大数量限制。

```js
export function filter(barrages, limit) => {
  barrages.forEach(barrage => {
    barrage.weight = rules.reduce((weight, rule) => {
      return rule.call(barrage, weight);
    }, 1);
  });
  return barrages
    .sort((a, b) => b.weight - a.weight)
    .slice(barrages.length - limit);
};
```

## 3.3 轮询上屏

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f087117c6cdb4cb59b2012db23863dda~tplv-k3u1fbpfcp-watermark.image)

将消息放入弹幕池后，接下来的操作便是轮询从弹幕池里取固定数量的弹幕上屏。实现逻辑如下所示：

- 第一步，从弹幕池里取出固定数量为`BARRAGE_MAX_FRAME`（默认为 6）的弹幕，添加到`barrageList`（上屏弹幕列表）中。如上效果图所示，6 条弹幕同时上屏，正好在一个屏幕高度内，不影响用户获取内容信息。
- 第二步，对`barrageList`列表做溢出处理，超出最大数量限制`BARRAGE_MAX_COUNT`（默认为 50），删除列表头部超出数量的弹幕。这样可以将上屏弹幕的数量维持在一个最大阈值内。如上效果图所示，弹幕列表数量超出阈值，头部的弹幕已经被清理，用户只能获取最新的 50 条弹幕。
- 第三步，`barrageList`得到更新后，需要执行`changeCallback`回调做真正的上屏处理，这个回调涉及业务逻辑操作，需要在类实例化的时候进行赋值。
- 使用`setTimeout`实现轮询机制，每`INTERVAL`ms（默认为 300）执行如上 3 步操作，如果上屏弹幕列表为空，表示没有新的弹幕消息，则不执行上屏回调处理。需要注意的是，这里用`setTimeout`模拟`setInterval`实现轮询，这是因为当回调函数的执行被阻塞时，`setInterval`会产生回调堆积。

```js
 private flush() {
    this.timer = setTimeout(() => {
      if (this.queueList.length > 0) {
        // 从弹幕池中取弹幕
        this.barrageList = [
          ...this.barrageList,
          ...this.queueList.splice(0, this.config.BARRAGE_MAX_FRAME),
        ];

        // 判断上屏弹幕是否超过最大限制，如超过，删除旧弹幕
        if (this.barrageList.length > this.config.BARRAGE_MAX_COUNT) {
          this.barrageList.splice(
            0,
            this.barrageList.length - this.config.BARRAGE_MAX_COUNT,
          );
        }
        // 弹幕上屏
        this.barrageList.length > 0 &&
          this.changeCallback &&
          this.changeCallback(this.barrageList);
      }
      this.flush();
    }, this.config.INTERVAL);
  }

// 弹幕上屏回调函数赋值
  emitQueueChange(cb) {
    this.changeCallback = cb;
  }
```

## 3.4 自己发的弹幕直出

用户自己发送的弹幕，不走弹幕池过滤，无视网络质量，**无条件优先上屏**。代码如下所示：

```js
  // 自己的弹幕直出
  barrageEnterQueueSelf(queue: queueI) {
    this.barrageList.push(queue);
    this.changeCallback && this.changeCallback(this.barrageList);
  }
```

## 3.5 轮询控制系统

- 当用户滑动弹幕列表时，为了方便用户执行截图、赋值、搜索等操作，弹幕上屏逻辑需要暂停。
- 当用户将列表滚动到底部时，恢复轮询。
- 同时需要设置重启机制，当弹幕暂停时间超过`SLEEP_TIME`ms（默认为 5000），且用户不在执行其它操作，弹幕恢复轮询。

具体的处理逻辑如下所示：

```js
// 检查弹幕是否激活
  setActiveAndAutoRestart() {
    // 如果没有开启
    if (!this.config.CHECK_SLEEP) return;
    if (this.checkActiveTimer) {
      clearTimeout(this.checkActiveTimer);
    }
    this.checkActiveTimer = setTimeout(() => {
      this.restart();
    }, this.config.SLEEP_TIME);
  }

  // 弹幕暂停滚动
  pause() {
    if (this.isPaused) return;
    this.isPaused = true;
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }

    this.setActiveAndAutoRestart();
  }

  // 弹幕重新开始滚动
  restart() {
    if (!this.isPaused) return;
    this.isPaused = false;
    this.queueList.length && this.flush();
  }
```

# 4 组件方法详解

## 4.1 组件生命周期操作

- **attached**，组件挂载时，创建`Barrage`类实例，设置弹幕上屏的实际回调方法，通过`toLast`值指定列表滚动到该元素，通过更新`realBarrage`数据，来更新实际上屏的数据。
- **detached**，组件销毁时，清除`Barrage`类实例,同时对定时器进行清理。

```js
  attached() {
    // 创建Barrage实例
    this.barrage = new Barrage(this.properties.config);
    // 初始化轮询回调方法
    this.barrage.emitQueueChange(data => {
      this.setData({
        toLast: `item${data.length}`,
        realBarrage: data,
      });
    });
    // 校准弹幕容器高度
    this.checkContainerClientHeight();
  }

  detached() {
    this.barrage && this.barrage.destroy();
  }
```

## 4.2 三种角色的弹幕消息进入弹幕池

父组件可以通过 id 获取弹幕组件的实例，从而调用其封装的方法。用法如下所示：

```html
<wr-live-barrage id="wr-live-barrage"></wr-live-barrage>
```

```js
  // 获取弹幕组件实例
  getBarrageContext() {
    if (!this.barrageContext) {
      (this.barrageContext as any) = this.selectComponent('#wr-live-barrage');
    }
    return this.barrageContext;
  }

  // 调用方法示例
  handleSendBarrage() {
    (this.getBarrageContext() as any).sendBarrageBySelf(data);
  }
```

弹幕组件暴露的三个方法如下表格所示：

| API                   | 说明                                                                 | 参数                    | 默认值    |
| --------------------- | -------------------------------------------------------------------- | ----------------------- | --------- |
| multiPushBarrage      | 初始化直播间时，填充数十条历史弹幕数据，组件已经优化，实现分布式上屏 | _queueI[]_              | -         |
| sendBarrageEnterQueue | 将接收到的弹幕消息放入弹幕池, 第二个参数表示是否执行弹幕过滤规则     | <_queueI[]_, _Boolean_> | <-, true> |
| sendBarrageBySelf     | 自己发送的弹幕消息直接上屏                                           | _queueI_                | -         |

三个方法的代码如下所示，都是将弹幕放入弹幕池，唯一的区别是自己发送的弹幕时，需要恢复轮询机制。

```js
 // 自己发弹幕
  sendBarrageBySelf(barrage) {
    this.barrage.barrageEnterQueueSelf(barrage);
    this.barrage.restart();
  }

  // 即时消息弹幕填充
  sendBarrageEnterQueue(list, isFilter = true) {
    this.barrage.barrageEnterQueue(list, isFilter);
  }

  // init直播间的时候，历史弹幕填充
  multiPushBarrage(list) {
    this.sendBarrageEnterQueue(list, false);
  }
```

## 4.3 用户行为处理

### 4.3.1 监听滚动到底部

实现监听滚到到底部，需要先来了解一下浏览器的`scrollHegiht`、`scrollTop`、`clientHegiht`三个属性。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6af0eaea1c8046f2afe1977d62485e82~tplv-k3u1fbpfcp-watermark.image)

- **scrollHegiht**: 文档内容实际高度，包括超出视窗的溢出部分。
- **scrollTop**: 滚动条滚动距离。
- **clientHeight**: 窗口可视范围高度。

当 `clientHeight` + `scrollTop` >= `scrollHeight` 时，表示已经抵达内容的底部了，可以加载更多内容。

### 4.3.2 滚动事件监听

列表滚动时，由于当前实际上屏的弹幕列表数量和内容（内容长度不一，有的 1 行，有的 2 行，有的 3 行）发生变化，需要先校准容器的实际高度，方便后续做滚动到底部判断。同时设置弹幕激活策略，静止 n 秒后恢复滚动。

```js
  // 滚动容器会计算高度,并激活弹幕自动恢复
  handleScrollBarrageContainer({ detail }) {
    this.containerScrollHeight = detail.scrollHeight;
    this.checkContainerClientHeight();
    this.barrage.setActiveAndAutoRestart();
  }

    // 校准弹幕容器高度
  checkContainerClientHeight() {
    if (!this.containerClientHieghtChecked) {
      wx.createSelectorQuery()
        .in(this as TrivialInstance)
        .select('.live-barrage')
        .fields(
          {
            size: true,
          },
          res => {
            if (res) {
              const { height = 0 } = res;
              if (height === this.containerClientHeight) {
                this.containerClientHieghtChecked = true;
              }
              this.containerClientHeight = height;
            }
          },
        )
        .exec();
    }
  }
```

列表滚动到底部，激活弹幕上屏逻辑。

```js
  // 滚动到底部，重新开启刷新策略
  handleScrollBottom() {
    if (!this.barrageTouch) {
      this.barrage.restart();
    }
  }
```

### 4.3.3 touch 事件监听

用户触摸屏幕动作开始时，暂停弹幕上屏逻辑。

```js
  barrageTouchStart() {
    this.barrageTouch = true;
    this.barrage.pause();
  }
```

用户触摸动作结束时，判断是否滚动到底部，如果是的话，恢复弹幕上屏逻辑

```js
  // 判断弹幕是否到底
  barrageTouchEnd() {
    this.barrageTouch = false;
    wx.createSelectorQuery()
      .in(this as TrivialInstance)
      .select('.live-barrage')
      .fields(
        {
          scrollOffset: true,
        },
        ({ scrollTop }) => {
          if (
            this.containerClientHeight + scrollTop + 5 >=
            this.containerScrollHeight
          ) {
            this.begainScroll = true;
            this.barrage.restart();
          }
        },
      )
      .exec();
  }
```

# 5 总结

以上便是我对弹幕组件设计的理解，花了一个周末，时间略微仓促，写作水平有限，在某些问题上解释说明若有出入，还请批评指教！

**最后的最后，欢迎讨论，点个赞再走吧 ｡◕‿◕｡ ～**
