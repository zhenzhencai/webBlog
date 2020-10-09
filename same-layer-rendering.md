# 小程序同层渲染那些事(keng)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86af6e3198fc4f3f9c67e4b80a75701d~tplv-k3u1fbpfcp-zoom-1.image)

# 1. 为什么写这篇文章

近期，电商直播业务热火朝天。许多团队都纷纷转战直播领域，试图抢占市场。

鄙人有幸参与直享直播小程序的业务开发工作，在项目迭代过程中，遇见过许多“百思不得其解”的问题。

比如直播页面最常见的布局是底层为视频画面，顶层为滚动弹幕、点赞、购物袋、抽奖活动、PK 榜等基础功能。在项目上线后，收到一部分的用户反馈，表示仅能看见“纯净模式”版的直播画面，对于弹幕、点赞动画、商品、抽奖活动啥的，统统看不见！
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cba272b6301443f79c61c7f3bf617abc~tplv-k3u1fbpfcp-zoom-1.image)

bug 突如其来，令人措手不及。在笔者苦心钻研（google、baidu 一把梭）下，终于找到了“罪魁祸首”——**「原生组件」**。其实，小程序的内容大多是渲染在 `WebView` 上的，如果把 `WebView` 看成单独的一层，那么由系统自带的这些 「原生组件」 则位于另一个更高的层级。两个层级是完全独立的，因此无法简单地通过使用 `z-index` 控制「原生组件」和非原生组件之间的相对层级。正如下左图所示，非原生组件位于 `WebView` 层，而「原生组件」及 `cover-view` 与 `cover-image` 则位于另一个较高的层级。所以`live-player`组件把其他业务组件都覆盖了，导致部分用户只能看到“纯净模式”。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af0d97de1d1f41029bb0465d2a178f5a~tplv-k3u1fbpfcp-zoom-1.image)

为了解决这个问题，**得通过一定的技术手段把「原生组件」直接渲染到 `WebView` 层级上**，正如上右图所示,此时 **「原生组件层」** 已经不存在，**「原生组件」** 此时已被直接挂载到 `WebView` 节点上。正所谓，工欲善其事，必先利其器。你一定也想知道 **「同层渲染」** 背后究竟采用了什么技术。只有真正理解了「同层渲染」背后的机制，才能更高效地使用好这项能力。

本文后续的篇幅主要讨论「原生组件」存在的意义和风险、「同层渲染」实现的原理，以及分享在实际业务开发趟坑总结出的一些技巧。

# 2. 「原生组件」存在的意义及风险

要想深刻理解「同层渲染」背后的机制，最好先了解一下「原生组件」的“前世今生”，这得从小程序的技术架构说起。

## 2.1 小程序技术架构

一般来说，**渲染界面的技术有三种**：

- 用纯客户端原生技术来渲染。
- 用纯 Web 技术来渲染。
- 介于客户端原生技术与 Web 技术之间的，互相结合各自特点的技术（下面统称 **Hybrid 技术**）来渲染。

由于小程序的宿主是微信，所以**不太可能用纯客户端原生技术来编写小程序**。如果这么做，那小程序代码需要与微信代码一起编包，跟随微信发版本，这种方式跟开发节奏必然都是不对的。

小程序应该像 Web 技术那样，有一份随时可更新的资源包放在云端，通过下载到本地，动态执行后即可渲染出界面。但是，如果用纯 Web 技术来渲染小程序，在一些有复杂交互的页面上可能会面临一些性能问题，这是**因为在 Web 技术中，UI 渲染跟 JavaScript 的脚本执行都在一个单线程中执行**，这就容易导致一些**逻辑任务抢占 UI 渲染的资源**。

最终，小程序**采用类似于微信 JSSDK 这样的 Hybrid 技术**，即界面主要由成熟的 Web 技术渲染，辅之以大量的接口提供丰富的客户端原生能力。同时，每个小程序页面都是用不同的 WebView 去渲染，这样可以提供更好的交互体验，更贴近原生体验，也避免了单个 WebView 的任务过于繁重。小程序的架构图如下图所示。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2da0a49eb65140bd9a57e1d0ac38f39c~tplv-k3u1fbpfcp-zoom-1.image)

小程序的渲染层和逻辑层分别由两个线程管理：

- 渲染层的界面使用 `WebView` 进行渲染，一个小程序存在多个界面，所以渲染层存在多个 `WebView`。
- 逻辑层采用 `JSCore` 线程运行 `JavaScript` 脚本。

这两个线程间的通信经由小程序 `Native` 侧中转，逻辑层发送网络请求也经由 `Native` 侧转发。

如此设计的初衷是为了**管控和安全**，微信小程序阻止开发者使用一些浏览器提供的，诸如跳转页面、操作 DOM、动态执行脚本的开放性接口。将逻辑层与视图层进行分离，**视图层和逻辑层之间只有数据的通信，可以防止开发者随意操作界面，更好的保证了用户数据安全**。

三端的脚本执行环境以及用于渲染非原生组件的环境是各不相同的：

| 运行环境         |     逻辑层     |            渲染层 |
| :--------------- | :------------: | ----------------: |
| Android          |       V8       | Chromium 定制内核 |
| IOS              | JavaScriptCore |         WKWebView |
| 小程序开发者工具 |      NWJS      |    Chrome WebView |

小程序的视图是在`WebView`里渲染的，那搭建视图的方式自然就需要用到`HTML`语言。但是`HTML`语言标签众多，增加了理解成本，而且直接使用`HTML`语言，开发者可以利用`<a>`标签实现跳转到其他在线网页，也可以动画执行`JAVAScript`，前面所提到的为解决管控与安全而建立的**双线程模型**就成摆设了。

因此，小程序设计一套组件框架——**Exparser**。基于这个框架，内置了一套组件，以涵盖小程序的基础功能，便于开发者快速搭建出任何界面。同时也提供了自定义组件的能力，开发者可以自行扩展更多的组件，以实现代码复用。值得一提的是，**内置组件有一部分较复杂组件是用客户端原生渲染的**，以提供更好的性能，这便是本文的主角——**「原生组件」**。

## 2.2 「原生组件」是一把双刃剑

在内置组件中，有一些组件较为特殊，它们并不完全在 Exparser 的渲染体系下，而是由客户端原生参与组件的渲染，这类组件称为 **“「原生组件」”**，这也是小程序 Hybrid 技术的一个应用。比如小程序中的 `camera`、`video`、`live-player`、`canvas`、`map`、`animation-view`、`textarea`、`input`、`cover-view`、`cover-image` 这些都是「原生组件」。

```html
<video src="{{pullUrl}}"></video>
```

上述代码展示的是一个视频组件，这行代码在渲染层开始运行时，会经历以下几个步骤：

1. 组件被创建，包括组件属性会依次赋值。
2. 组件被插入到 DOM 树里，浏览器内核会立即计算布局，此时可以读取出组件相对页面的位置（x, y 坐标）、宽高。
3. 组件通知客户端，客户端在相同的位置上，根据宽高插入一块原生区域，之后客户端就在这块区域渲染界面。
4. 当位置或宽高发生变化时，组件会通知客户端做相应的调整。

可以看出，「原生组件」在`WebView`这一层的渲染任务是很简单，只需要渲染一个占位元素，之后客户端在这块占位元素之上叠了一层原生界面。因此，**「原生组件」的层级会比所有在 WebView 层渲染的普通组件要高**。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5a44ed836c8474a81b0ee3640e9962a~tplv-k3u1fbpfcp-zoom-1.image)

引入「原生组件」主要有 3 个好处：

1. **扩展 Web 的能力**。比如像输入框组件 `input`, `textarea` 有更好地控制键盘的能力。
2. **体验更好，同时也减轻 WebView 的渲染工作**。比如像地图组件 `map` 这类较复杂的组件，其渲染工作不占用 `WebView` 线程，而交给更高效的客户端原生处理。
3. **绕过 setData、数据通信和重渲染流程，使渲染性能更好**。比如像画布组件 `canvas` 可直接用一套丰富的绘图接口进行绘制。

然而，命运惠赠的礼物，早已在暗中标好了价格。「原生组件」并非十全十美。由于「原生组件」脱离在 `webview` 渲染流程外，因此在使用时有以下限制：

1. **「原生组件」的层级是最高的**，页面中的其他组件无论设置 `z-index` 为多少，都无法盖在「原生组件」上。后插入的「原生组件」可以覆盖之前的「原生组件」。
2. 部分 CSS 样式无法应用于「原生组件」。比如无法对「原生组件」设置 CSS 动画，无法定义「原生组件」为`position: fixed`，不能在父级节点使用 `overflow: hidden` 来裁剪「原生组件」的显示区域。
3. 「原生组件」无法在 `scroll-view`、`swiper`、`picker-view`、`movable-view` 中使用，因为如果开发者在可滚动的 DOM 区域，插入「原生组件」作为其子节点，由于「原生组件」是直接插入到 webview 外部的层级，与 DOM 之间没有关联，所以不会跟随移动也不会被裁减。
4. 「原生组件」在`Android`上，字体会渲染为`rom`的主题字体，而`webview`如果不经过单独改造不会使用`rom`主题字体。

## 2.3 「原生组件」限制的局部解决方案

在小程序引入「同层渲染」之前，「原生组件」的层级总是最高，不受 `z-index` 属性的控制，无法与 `view`、`image` 等内置组件相互覆盖，`cover-view` 和 `cover-image` 组件的出现**一定程度上缓解了覆盖的问题**，但这样做，就像是写`css`的时候，写了一堆`!important`，并不是一个优雅的解决方案。

`cover-view` 和 `cover-image` 组件还具有如下限制：

1. 无法覆盖`textarea`、`input`「原生组件」。
2. 只支持基本的定位、布局、文本样式。不支持设置单边的`border`、`background-image`、`shadow`、`overflow: visible`等。
3. `cover-view` 支持 `overflow: scroll`，但不支持动态更新 `overflow`。
4. `cover-view`和`cover-image`的`aria-role`仅可设置为 button，读屏模式下才可以点击，并朗读出“按钮”；为空时可以聚焦，但不可点击。
5. `cover-view`和`cover-image`的子节点如果溢出父节点，容易出现布局错误。
6. 支持`css transition`动画，transition-property 只支持`transform (translateX, translateY)`与`opacity`。
7. 自定义组件嵌套 `cover-view` 时，自定义组件的 slot 及其父节点暂不支持通过 wx:if 控制显隐，否则会导致 `cover-view` 不显示。

随着小程序生态的发展，开发者对「原生组件」的使用场景不断扩大，「原生组件」的这些问题也日趋显现，为了彻底解决「原生组件」带来的种种限制，微信官方对小程序「原生组件」进行了一次重构，引入了 **「同层渲染」**。

# 3 「同层渲染」那些事

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b668555596fb41a384d62a6c06009ce5~tplv-k3u1fbpfcp-zoom-1.image)

为了解决「原生组件」的层级问题，同时尽可能保留`NA组件`（指代「原生组件」）的优势，小程序客户端、前端及浏览内核团队一起制定了一套**解决方案**：由于此方案的控件并非绘制在 `NA` 贴片层，而是绘制在 `WebView` 所渲染的页面中，与其他 `HTML` 控件在同一层级，因此称为 **「同层渲染」**。在支持「同层渲染」后，「原生组件」与其它 「H5 组件」（指代 HTML5 语言编写的 web 组件）可以随意叠加，层级的限制将不复存在。

你一定也想知道 **「同层渲染」** 背后究竟采用了什么技术。只有真正理解了 **「同层渲染」** 背后的机制，才能更高效地使用好这项能力。实际上，小程序的「同层渲染」在 `iOS` 和 `Android` 平台下的实现不同，因此下面分成两部分来分别介绍两个平台的实现方案。

## 3.1 iOS 端「同层渲染」原理

### 3.1.1 专业名词解释

> **WKWebView**: 是 `iOS 8` 之后提供的一款浏览器组件，iOS 端使用 `WKWebView` 进行渲染，`WKWebView` 在内部采用的是**分层的方式**进行渲染。`WKWebView` 会将 `WebKit` 内核生成的 `Compositing Layer`（合成层）渲染成 iOS 上的一个 `WKCompositingView`(「原生组件」的一种)。

> **Compositing Layer**: NA 合成层，内核一般会将多个`webview`内的 DOM 节点渲染到一个 `Compositing Layer` 上，因此合成层与 DOM 节点之间**不存在一对一的映射关系**。

> **WKChildScrollView**: 「原生组件」的一种。当把一个 DOM 节点的 CSS 属性设置为 `overflow: scroll` （低版本需同时设置 `-webkit-overflow-scrolling: touch`）之后，`WKWebView` 会为其生成一个 `WKChildScrollView`，与 DOM 节点**存在映射关系**，这是一个原生的 `UIScrollView` 的子类，也就是说 `WebView` 里的滚动实际上是由真正的原生滚动组件来承载的。`WKWebView` 这么做是为了可以让 iOS 上的 `WebView` 滚动有更流畅的体验。虽说 `WKChildScrollView` 也是「原生组件」，但 `WebKit` 内核已经**处理了它与其他 DOM 节点之间的层级关系，因此你可以直接使用 WXSS 控制层级而不必担心遮挡的问题**。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dbc2c93cae946608b8d53003330e739~tplv-k3u1fbpfcp-zoom-1.image)

### 3.1.2 渲染原理解析

小程序 iOS 端的「同层渲染」也正是基于 `WKChildScrollView` 实现的，「原生组件」在 attached 之后会直接**挂载到预先创建好的 `WKChildScrollView` 容器下**，大致的流程如下：

1. 小程序前端，在 webview 内创建一个 DOM 节点并设置其 CSS 属性为 `overflow: hidden` 且 `-webkit-overflow-scrolling: touch`，生成一个`containerId`，并将这个`WKChildScrollView`的位置信息通知给客户端。
2. 前端通知客户端递归搜索查找到该 DOM 节点对应的原生 `WKChildScrollView` 组件；
3. 将「原生组件」挂载到该 `WKChildScrollView` 节点上作为其子 View；
4. WebKit 内核已经处理了`WKChildScrollView`与对应 DOM 节点之间的层级关系。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1fb15e23a9c8499381fa02be3bf5602c~tplv-k3u1fbpfcp-zoom-1.image)

通过上述流程，小程序的「原生组件」就被插入到 `WKChildScrollView` 了，也即是在 步骤 1 创建的那个 DOM 节点映射的原生 `WKChildScrollView`节点。此时，修改这个 DOM 节点的样式属性同样也会应用到「原生组件」上。因此，「同层渲染」的「原生组件」与普通的 H5 组件表现并无二致。

## 3.2 Android 端「同层渲染」原理

### 3.2.1 专业名词解释

> **chromium**：小程序在 Android 端采用 `chromium` 作为 `WebView` 渲染层，与 iOS 不同的是，Android 端的 `WebView` 是**单独进行渲染**而不会在客户端生成类似 iOS 那样的 `Compositing View` (合成层)，经渲染后的 WebView 是一个完整的视图。

> **WebPlugin**：chromium 支持 WebPlugin 机制，WebPlugin 是浏览器内核的一个插件机制，主要用来**解析和描述`<embed />` 标签**。比如 Chrome 浏览器上的 pdf 预览，它就是基于 `<embed />` 标签实现的。

### 3.2.2 内核渲染流程

内核的渲染流程可分为下面四个步骤。如下如所示：

1. 解析。当内核收到`HTML`数据时，会构建一颗`DOM Tree`，并为每个节点计算样式。
2. 排版。遍历`DOM Tree`，根据样式构建一颗`Layout Tree`。`DOM Tree`和`Layout Tree`上的节点并非一一对应，如果某个 DOM 节点不可见，则不会在`Layout Tree`上。
3. 绘制。为了提升绘制效率，对`Layout Tree`中的节点按照一定的规则分为不同的图层（Layer），这些图层也构成一个树状结构，称之为`Layer Tree`。绘制过程就是遍历`Layout Tree`，将每个节点的内容绘制到其所在的 Layer 上。在 GPU 硬绘模式下，Layer 存储后端是 GPU 中的纹理-`Texture`。
4. 合成。内核的合成模块（CC 层）负责将 Layer 按照一定的顺序合成到一起，交给系统的`FrameBuffer`，最终输出到屏幕上。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/220ea379ed8c4dfb8df9cd319f57bded~tplv-k3u1fbpfcp-zoom-1.image)

### 3.2.3 渲染原理解析

从内核渲染流程可以看出，要实现「原生组件」「同层渲染」，就要将「原生组件」作为一个`Layer`插入到`Layer Tree`中。 如果能够将「原生组件」渲染到内核提供的`Texture`上，就可达到「同层渲染」的目的。Android 端的「同层渲染」就是基于 `<embed />` 标签结合 chromium 内核扩展来实现的, 大致流程如下:

```html
<embed id="web-plugin" type="plugin/video" width="750" height="600" />
```

1. WebView 侧创建一个 `embed` DOM 节点并指定组件类型；
2. chromium 内核会创建一个 WebPlugin 实例，并生成一个 `RenderLayer`；
3. Android 客户端初始化一个对应的「原生组件」；
4. Android 客户端将「原生组件」的画面绘制到步骤 2 创建的 RenderLayer 所绑定的 `SurfaceTexture` 上；
5. 绘制完成后内核收到`SurfaceTexture`内容更新的通知，通知 chromium 内核渲染该 `RenderLayer`；
6. chromium 渲染该 `embed` 节点并上屏。
7. 当「同层渲染」的节点收到事件时，会将事件转发给 Native 组件模块处理。如果 Native 组件不消费事件，内核会再将事件向上冒泡。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46c9d642ec2f45a69c7cc92bc1f758bb~tplv-k3u1fbpfcp-zoom-1.image)

这样就实现了把一个「原生组件」渲染到 WebView 上，这个流程相当于给 WebView 添加了一个外置的插件。

这种方式可以用于 `map`、`video`、`canvas`、`camera` 等「原生组件」的渲染，对于 `input` 和 `textarea`，采用的方案是直接对 chromium 的组件进行扩展，来支持一些 WebView 本身不具备的能力。

对比 iOS 端的实现，Android 端的 **「同层渲染」** 真正将「原生组件」视图加到了 WebView 的渲染流程中且 `embed` 节点是真正的 DOM 节点，理论上可以将任意 WXSS 属性作用在该节点上。Android 端相对来说是更加彻底的「同层渲染」，但相应的重构成本也会更高一些。

## 3.3 「同层渲染」的小缺陷

「原生组件」的 **「同层渲染」** 能力可能会在特定情况下失效，一方面你需要在开发时稍加注意，另一方面「同层渲染」失败会触发 `bindrendererror` 事件，可在必要时根据该回调做好 UI 的 `fallback`。

- 对 Android 端来说，如果用户的设备没有微信自研的 `chromium` 内核，则会无法切换至 **「同层渲染」**，此时会在组件初始化阶段触发 `bindrendererror`。
- iOS 端的情况会稍复杂一些：如果在基础库创建同层节点时，节点发生了 WXSS 变化从而引起 **WebKit 内核重排**，此时可能会出现同层失败的现象。这是因为设置了`-webkit-overflow-scrolling`属性的 div 块（container），在其下层同时会产生一个 div 块（内层 div）作为渲染使用，在 iOS 设备表现是生成的`UIScrollView`会有一个子 view 是`WKCompositingView`，如果改变内层 div 的高度，可能会触发重新渲染，进而导致「原生组件」组件可能被无感知的被移除掉。**解决方法**：应尽量避免在「原生组件」上频繁修改节点的 WXSS 属性，尤其要尽量避免修改节点的 `position` 属性。如需对「原生组件」进行变换，强烈推荐使用 `transform` 而非修改节点的 `position` 属性。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f11c9f2bff37450ba029cdec13d8ad1a~tplv-k3u1fbpfcp-zoom-1.image)

# 4 重温 「原生组件」 的那些坑

如果你仔细阅读前面的篇幅，想必你对「原生组件」会有更深刻的了解，之前在实践开发中遇到的一些问题，也将迎刃而解。接下来，我们一起重温一下「原生组件」 的那些坑。

## 4.1 `<textarea>`组件的`placeholder`穿透问题？

> - **相关问题**：
>
> 1.  [在 Android 手机上，textarea 组件输入的内容显示在固定定位的上面？](https://developers.weixin.qq.com/community/develop/doc/000a024b8c4248fc3f4a54c795b000)
> 2.  [当页面内容超出一屏时用确认框中用了 textarea，textarea 就会跑到上面去造成无法填写异常？](https://developers.weixin.qq.com/community/develop/doc/00064ec80dc4583d9d6a8be1052000)
>
> - **问题描述**：
>   如图所示，页面上滑的时候`<textarea>`组件的`placeholder`会穿透过来。
>   ![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b0b49d334444228b342a207b6a5ef8b~tplv-k3u1fbpfcp-zoom-1.image)
> - **根本原因**：_`<textarea>`组件是「原生组件」，它的层级是最高的，页面中的其他组件无论设置 `z-index` 为多少，都无法盖在「原生组件」上_。

> - **其他问题**：[textarea 在 scroll-view 中会阻止滑动？](https://developers.weixin.qq.com/community/develop/doc/000a2ae2f00938115f7a328fd51000)
> - **根本原因**：_「原生组件」无法在 `scroll-view`、`swiper`、`picker-view`、`movable-view` 中使用，因为如果开发者在可滚动的 DOM 区域，插入「原生组件」作为其子节点，由于「原生组件」是直接插入到 webview 外部的层级，与 DOM 之间没有关联，所以不会跟随移动也不会被裁减_。

解决思路主要有三种：

1. **使用`cover-view`原生组件覆盖`<textarea>`组件**。即将图中被穿透的`view`和`button`使用`cover-view`代替。
2. **通过滑动页面去判断`<textarea>`组件的显示和隐藏**。使用`onPageScroll`函数来获取页面的滚动距离，当滚动距离等于`<textarea>`组件的`top`减去固定到顶部的盒子的距离的时候就让`<textarea>`组件隐藏，或者把`<textarea>`组件的`placeholder`设置为空也是可以解决穿透问题的。
3. **用`view`标签模拟`<textarea>`组件，来避免`<textarea>`组件的`placeholder`穿透问题**。用`view`标签模拟`<textarea>`组件，当点击`view`标签时，隐藏`view`标签显示`<textarea>`组件，并使用`focus`属性让`<textarea>`组件聚焦。通过给`<textarea>`组件设置一个`bindinput`的方式将输入的文字显示在`view`标签里面。通过`bindblur`判断`<textarea>`组件失去焦点，隐藏`<textarea>`组件并让`view`标签显示。

## 4.2 在`scroll-view`中使用`input`，`input`键盘弹出时滚动页面，输入框内容会出现错位问题？

> - **相关问题**：
>
> 1.  [scroll-view 滚动页面，input 键盘弹出时，页面滚动到顶部，输入框内容错位问题。怎么解决？](https://developers.weixin.qq.com/community/develop/doc/00006850f70cc08156ea2c53c56800)
> 2.  [scroll-view + input 输入类容错位以及 input 中出现滚动条？](https://developers.weixin.qq.com/community/develop/doc/000a4008c6c5b87c7eca093a55b400)
> 3.  [在 scroll-view 中 input 的输入内容和 placeholder 文字会产生错位？](https://developers.weixin.qq.com/community/develop/doc/0008466eb545106d5dda506dc5b400)

**问题描述**：如图所示，在较长的表单中，页面可能需要滑动，比如使用`scroll-view`标签，在该标签中嵌入`input`组件，可能会导致输入内容上移错位的问题。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb9694e3e5c74b79a3a2b47904a0ae73~tplv-k3u1fbpfcp-zoom-1.image)

**根本原因**：_**input 组件在 focus 时表现为「原生组件」**，如果开发者在可滚动的 DOM 区域，插入「原生组件」作为其子节点，由于「原生组件」是直接插入到 webview 外部的层级，与 DOM 之间没有关联，所以不会跟随移动也不会被裁减_。

**解决方法**：_需要设置一个状态控制`scroll-view`是否允许滑动，当`Input`获取焦点时，禁止滑动，当`Input`失去焦点时，允许滑动。_

代码如下所示：

- wxml

```html
<scroll-view scroll-y="{{isScroll}}" scroll-into-view="{{intoView}}" style="height: 100vh;">
    <input  type="digit"  bindfocus="bindfocus" bindblur="closeblur"  />
</scroll-view>
```

- js

```js
// 获取焦点事件
bindfocus(e){
    this.setData({
      isScroll:false
    })
},
// 失去焦点事件
closeblur(e) {
    this.setData({
      isScroll:true
    })
}
```

> 基础库`2.10.4`起，`input`组件添加了`always-embed`属性，为 true 时，可以强制 `input` 处于同层状态，默认 focus 时 `input` 会切到非同层状态 (仅在 iOS 下生效)。

因此这个问题，有了第二种解决方法：

- wxml

```html
<scroll-view scroll-y="{{isScroll}}" scroll-into-view="{{intoView}}" style="height: 100vh;">
    <input type="digit"  bindfocus="bindfocus" bindblur="closeblur"  always-embed = true />
</scroll-view>
```

## 4.3 `live-player`组件「同层渲染」失败问题？

据目前后台统计，主要有以下几种机型会触发「同层渲染」失败问题：
+ **HONOR**：HLK-AL00、HLK-AL10、JSN-AL00a、HRY-AL00a
+ **HUAWEI**: ELE-AL00、JNY-AL10、TAS-AN00
+ **VIVO**: vivoX9s、V1813BA
+ **OPPO**: CPH1721、A59m、PBBM00、R11、R11s、R9 Plusm A、R9s Plus
+ **xiaomi**: Redmi 7、MI 8 Lite
+ **iphone**: iphone 6

根据排查，失败的原因可能有三种，一是微信版本过低，二是用户的设备没有微信自研的 `chromium` 内核， 三是「同层渲染」过程中产生了一些不可描述的bug。

所幸，「同层渲染」失败会触发 `bindrendererror` 事件，可在必要时根据该回调做好 UI 的 fallback。简单翻译一下，就是可以使用`cover-view`、`cover-image`组件，重构一套直播间组件（滚动弹幕、点赞、购物袋、抽奖活动、PK 榜等等）作为「同层渲染」失败时 UI 的 fallback。然而，直播间功能复杂多样，改造的成本是十分昂贵的，同时前面也提到过`cover-view`、`cover-image`组件的使用“限制”（详见2.3），想要百分百还原之前的UI设计，存在一定的难度。如何取舍，还得经过多方的评估。

## 4.4 `canvas`组件「同层渲染」那些事？

对于`canvas`组件“踩坑”问题，作者之前写过两篇文章，这里不再赘述。

+ [小程序Canvas实践指南—教你如何轻松避坑](https://juejin.im/post/6867403441174642696)
+ [小程序直播-疯狂点赞canvas动画实现原理解析](https://juejin.im/post/6859702071906533383)

# 5 总结

因为业务开发需要，作者接触 小程序直播业务5个月，总结分享实践中遇到的一些问题。因业务水平有限，在某些问题上解释说明若有出入，还请批评指教！

**最后的最后，欢迎讨论，点个赞再走吧 ｡◕‿◕｡ ～**
