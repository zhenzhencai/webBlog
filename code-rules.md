# 不以规矩，不能成方圆-彻底搞懂 ESLint 和 Prettier

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/888d3c39cef1418e8b49ec03e95a22bc~tplv-k3u1fbpfcp-watermark.image)

> 代码规范是软件开发领域经久不衰的话题，几乎所有工程师在开发过程中都会遇到，并或多或少会思考过这一问题。随着前端应用的大型化和复杂化，越来越多的前端团队开始重视 `JavaScript` 代码规范。

# 1. 为什么写这篇文章？

2020 年 3 月，懵懂的蔡小真怀揣着对技术的向往无比乐观地进入部门实习，入职当天就收到了来自导师的两份礼物，一是代码仓库链接，二是代码开发规范。机智如斯的蔡小真按照规范里的教程，安装好了`ESLint`、`Prettier`等插件，带上安全帽，拉起小推车，就这样风平浪静地度过了半年的搬砖生涯。然而，赶上最近项目交接，在熟悉代码的过程中，遇见了如下事情，让蔡小真自乱阵脚。

![这里是一个案例录屏，加载可能有点慢哦～](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7800253882974b55853d52d3b4e08b2a~tplv-k3u1fbpfcp-watermark.image)

如图所示，是项目中的某个文件，按照现在比较流行的前端代码规范，这个文件存在两个问题，一是每行代码末尾并未添加分号，二是单行代码长度过长。如果你安装在 IDE 中安装了`Pretter`插件并勾选了`format on save`，那么在你保存文件的时候会按照`Pretter`提供的规则对代码进行格式化。格式化后的文件如图所示，每行代码末尾都加上了分号，且每行的长度都不超过 140 个字符。

事情本该往美好的方向发展，可是如果你同时安装了`ESLint`插件来检查代码质量问题，就会发现大面积地提示`Strings must use singlequote`，即“字符串必须使用单引号”，有强迫症的你肯定会按下右键修复所有的`ESLint`问题，即把字符串的双引号全部修改为单引号。

当你以为大功告成，兴奋地按下`ctrl + s`保存这一成果时，你发现字符串的单引号又变成双引号了？？？why？？？一脸懵逼？？？

如果你是个成熟的码农，你一定知道这是`Prettier`和`ESLint`规则冲突了，但你知道如何高效地修复这种“死循环”问题吗？你知道项目根目录配置和插件配置的优先级问题吗？你了解为了执行统一的前端代码规范安装了无数个`ESLint`、`Prettier`相关的包，每个包的使用意义和具体的配置方法吗？

这便是本文所要阐述的内容。

那么这篇文章适合哪些人阅读呢？

1. 初学者： 这篇文章详细介绍了业内最火的前端代码规范配置教程 `ESLint` + `Prettier` + `husky` + `lint-staged`，让你轻松部署一套趁手的搬砖工具。
2. 老司机：这篇文章以抛砖引玉、承上启下的方式介绍`ESLint`、`Prettier`、`husky`、`lint-staged`等每个功能包及其配套插件的配置方法和使用意义，有益于读者系统性地学习整套前端代码规范，非常适合老司机查缺补漏。
3. 感兴趣的路人： 如果你愿意花十分钟阅读我的文章，并且为我点个赞，那么你一定是个可爱又博学的人。

# 2. ESLint

## 2.1 ESLint 是什么？

首先，我们来看看`ESLint`官网的简介：

> 代码检查是一种静态的分析，常用于寻找有问题的模式或者代码，并且不依赖于具体的编码风格。对大多数编程语言来说都会有代码检查，一般来说编译程序会内置检查工具。

> JavaScript 是一个动态的弱类型语言，在开发中比较容易出错。因为没有编译程序，为了寻找 JavaScript 代码错误通常需要在执行过程中不断调试。像 ESLint 这样的可以让程序员**在编码的过程中发现问题而不是在执行的过程中**。

ESLint 是一款插件化的 `JavaScript` 代码静态检查工具，其核心是通过对代码解析得到的 `AST`（Abstract Syntax Tree，抽象语法树）进行模式匹配，来分析代码达到检查代码质量和风格问题的能力。

ESLint 的使用并不复杂。依照 ESLint 的文档安装相关依赖，可以根据个人/团队的代码风格进行配置，即可通过命令行工具或借助编辑器集成的 ESLint 功能对工程代码进行静态检查，发现和修复不符合规范的代码。如果想降低配置成本，也可以直接使用开源配置方案，例如 [eslint-config-airbnb](https://github.com/airbnb/javascript) 或 [eslint-config-standard](https://github.com/standard/eslint-config-standard)。

## 2.2 ESLint 使用方式

### 2.2.1 ESLint 配置方式

ESlint 被设计为完全可配置的，这意味着你可以关闭每一个规则而只运行基本语法验证，或混合和匹配 ESLint 默认绑定的规则和你的自定义规则，以让 ESLint 更适合你的项目。有两种主要的方式来配置 ESLint：

#### 2.2.1.1 Configuration Comments - 使用注释把 lint 规则嵌入到源码中

使用 JavaScript 注释把配置信息直接嵌入到一个代码源文件中。

新建一个`test.ts`文件，直接在源代码中使用 `ESLint` 能够识别的注释方式，进行 `lint` 规则的定义。如下内容定义使用`console`语法便会报错：

```js
/* eslint no-console: "error" */
console.log('this is an eslint rule check!');
```

命令行运行 `ESLint` 校验，出现报错：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8244c5cd87f740ddab611c4c2b8d7788~tplv-k3u1fbpfcp-watermark.image)

#### 2.2.1.2 Configuration Files - 使用配置文件进行 lint 规则配置

在初始化 ESLint 时，可以选择使用某种文件类型进行`lint`配置，有如下三种选项：

1. JavaScript（`eslint.js`）
2. YAML（`eslintrc.yaml`）
3. JSON（`eslintrc.json`）

另外，你也可以自己在 `package.json` 文件中添加 `eslintConfig` 字段进行配置

### 2.2.2 初始化

如果想在现有项目中引入 ESLint，可以直接运行下面的命令：

```js
# 全局安装 ESLint
$ npm install -g eslint

# 进入项目
$ cd ESLint-test

# 强制初始化 package.json
$ npm init -force

# 初始化 ESLint 配置
$ eslint --init
```

在使用 `eslint --init` 后，会出现很多用户配置项，具体可以参考：[eslint-cli 部分的源码](https://github.com/eslint/eslint/blob/v6.0.1/lib/init/config-initializer.js#L432)。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abe125e6d2f34398b1456af56619cfff~tplv-k3u1fbpfcp-watermark.image)

经过一系列一问一答的环节后，你会发现在你文件夹的根目录生成了一个 `.eslintrc.js` 文件。

`.eslintrc.js` 文件配置如下（这是根据上图所示的选择生成的配置，选择不同，配置不同）：

```js
module.exports = {
	env: {
		// 环境
		browser: true,
		es2021: true,
	},
	extends: [
		// 拓展
		'eslint:recommended',
		'plugin:@typescript-eslint/recommended',
	],
	parser: '@typescript-eslint/parser', // 解析器，本解析器支持Ts
	parserOptions: {
		// 解析器配置选项
		ecmaVersion: 12, // 指定es版本
		sourceType: 'module', // 代码支持es6，使用module
	},
	plugins: [
		// 插件
		'@typescript-eslint',
	],
	rules: {
		// 规则
	},
};
```

下面将详细解释每个配置项的含义。

## 2.3 ESLint 配置项解析

### 2.3.1 parser - 解析器

ESLint 默认使用`Espree`作为其解析器，但是该解析器仅支持最新的`ECMAScript`(`es5`)标准，对于实验性的语法和非标准（例如 `Flow` 或 `TypeScript`类型）语法是不支持的。因此，开源社区提供了以下两种解析器来丰富`TSLint`的功能：

- **[`bable-eslint`](https://github.com/babel/babel-eslint)**： `Babel`是一个工具链，主要用于将 `ECMAScript 2015+`(`es6+`) 版本的代码转换为向后兼容的 `JavaScript` 语法，以便能够运行在当前和旧版本的浏览器或其他环境中。因此，如果在项目中使用`es6`，就需要将解析器改成`bable-eslint`。

- **[`@typescript-eslint/parser`](https://github.com/eslint/typescript-eslint-parser)**：该解析器将 `TypeScript` 转换成与 `estree` 兼容的形式， 允许`ESLint`验证`TypeScript`源代码。如上图 ESlint 初始化时，`Does your project use TypeScript?`选择了`Yes`，所以提示我们安装`@typescript-eslint/parser`包。

### 2.3.2 parserOptions - 解析器选项

除了可以自定义解析器外，`ESLint` 允许你指定你想要支持的 `JavaScript` 语言选项。默认情况下，`ESLint` 支持 `ECMAScript 5` 语法。**你可以覆盖该设置，以启用对 `ECMAScript` 其它版本和 `JSX` 的支持**。

解析器选项可以在 `.eslintrc.*` 文件使用 `parserOptions` 属性设置。可用的选项有：

- `ecmaVersion` -你可以使用 6、7、8、9 或 10 来指定你想要使用的 `ECMAScript` 版本。你也可以用使用年份命名的版本号指定为 2015（同 6），2016（同 7），或 2017（同 8）或 2018（同 9）或 2019 (same as 10)。
- `sourceType` - 设置为 `script` (默认) 或 `module`（如果你的代码是 `ECMAScript` 模块)。
- `ecmaFeatures` - 这是个对象，表示你想使用的额外的**语言特性**:
  - `globalReturn` - 允许在全局作用域下使用 `return` 语句
  - `impliedStrict` - 启用全局 `strict mode` (如果 `ecmaVersion` 是 5 或更高)
  - `jsx` - 启用 `JSX`

设置解析器选项能帮助 `ESLint` 确定什么是解析错误，**所有语言特性选项默认都是 `false`**。

### 2.3.3 env 和 golbals - 环境和全局变量

ESLint 会检测未声明的变量，并发出警告，但是有些变量是我们引入的库声明的，这里就需要提前在配置中声明。每个变量有三个选项，`writable`，`readonly` 和 `off`，分别表示可重写，不可重写和禁用。

```js
{
  "globals": {
    // 声明 jQuery 对象为全局变量
    "$": false, // true表示该变量为 writeable，而 false 表示 readonly
    "jQuery": false
  }
}
```

在 `globals` 中一个个的进行声明未免有点繁琐，这个时候就需要使用到 `env` ，这是对一个环境定义的一组全局变量的预设。

```js
{
  "env": {
    "browser": true,
    "es2021": true,
    "jquery": true // 环境中开启jquery，表示声明了jquery相关的全局变量，无需在globals二次声明
  }
}
```

可选的环境很多，预设值都在[这个文件](https://github.com/eslint/eslint/blob/v6.0.1/conf/environments.js)中进行定义，查看源码可以发现，其预设变量都引用自 [globals](https://github.com/sindresorhus/globals/blob/master/globals.json) 包。

同时，可以在`golbals`中使用字符串 `off` 禁用全局变量来覆盖`env`中的声明。

例如，在大多数 `ES2015` 全局变量可用但 `Promise` 不可用的环境中，你可以使用以下配置:

```js
{
    "env": {
        "es6": true
    },
    "globals": {
        "Promise": "off"
    }
}
```

当然，如果是微信小程序开发，`env`并没有定义微信小程序变量，需要在`globals`中手动声明全局变量，否则在文件中引入变量，会提示报错。声明如下所示：

```js
{
  globals: {
    wx: true,
    App: true,
    Page: true,
    Component: true,
    getApp: true,
    getCurrentPages: true,
    Behavior: true,
    global: true,
    __wxConfig: true,
  },
}
```

### 2.3.4 rules - 规则

ESLint 附带有[大量的规则](https://cn.eslint.org/docs/rules/)，你可以在配置文件的 `rules` 属性中配置你想要的规则。要改变一个规则设置，你必须将规则 ID 设置为下列值之一：

- `off` 或 0：关闭规则
- `warn` 或 1：开启规则，`warn` 级别的错误 (不会导致程序退出)
- `error` 或 2：开启规则，`error`级别的错误(当被触发的时候，程序会退出)

有的规则没有属性，只需控制是开启还是关闭，像这样：`"eqeqeq": "off"`，有的规则有自己的属性，使用起来像这样：`"quotes": ["error", "double"]`。具体内容可以查看[规则文档](https://cn.eslint.org/docs/rules/)。

可以通过`rules`配置任何想要的规则，它会覆盖你在拓展或插件中引入的配置项。

### 2.3.5 plugins - 插件

虽然官方提供了上百种的规则可供选择，但是这还不够，因为官方的规则只能检查标准的 `JavaScript` 语法，如果你写的是 `JSX` 或者 `TypeScript`，ESLint 的规则就开始束手无策了。

这个时候就需要安装 ESLint 的插件，来定制一些特定的规则进行检查。ESLint 的插件与扩展一样有固定的命名格式，以 `eslint-plugin-` 开头，使用的时候也可以省略这个头。

举个例子，我们要在项目中使用`TypeScript`，前面提到过，需要将解析器改为`@typescript-eslint/parser`，同时需要安装`@typescript-eslint/eslint-plugin`插件来拓展规则，**添加的 `plugins` 中的规则默认是不开启的**，我们需要在 `rules` 中选择我们要使用的规则。也就是说 `plugins` 是要和 `rules` 结合使用的。如下所示：

```js
// npm i --save-dev @typescript-eslint/eslint-plugin    // 注册插件
{
  "parser": "@typescript-eslint/parser",
  "plugins": ["@typescript-eslint"],   // 引入插件
  "rules": {
    "@typescript-eslint/rule-name": "error"    // 使用插件规则
    '@typescript-eslint/adjacent-overload-signatures': 'error',
    '@typescript-eslint/ban-ts-comment': 'error',
    '@typescript-eslint/ban-types': 'error',
    '@typescript-eslint/explicit-module-boundary-types': 'warn',
    '@typescript-eslint/no-array-constructor': 'error',
    'no-empty-function': 'off',
    '@typescript-eslint/no-empty-function': 'error',
    '@typescript-eslint/no-empty-interface': 'error',
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-extra-non-null-assertion': 'error',
    ...
  }
}
```

在`rules`中写一大堆的配置来启用`@typescript-eslint/eslint-plugin`插件规则，显然是**十分愚蠢的做法**，这时候`extends`派上了用场。

### 2.3.6 extends - 拓展

`extends` 可以理解为一份配置好的 `plugin` 和 `rules`。比如在 `@typescript-eslint/eslint-plugin` 中我们可以看到根目录的 [`index.js`](https://github.com/typescript-eslint/typescript-eslint/blob/master/packages/eslint-plugin/src/index.ts) 的 `module.exports` 输出为：

```js
import rules from './rules';
import all from './configs/all';
import base from './configs/base';

export = {
  rules,
  configs: {
    all,
    base,
    recommended: {
      extends: ['./configs/base', './configs/eslint-recommended'], // 内容看下面例子
      rules: {...}
    },
    'eslint-recommended': {
      extends: ['./configs/base', './configs/eslint-recommended'],
      rules: {...}
    },
    'recommended-requiring-type-checking': {
     extends: ['./configs/base', './configs/eslint-recommended'],
      rules: {...}
    },
  },
};

// 举例： ./configs/base 文件内容如下：
export = {
  parser: '@typescript-eslint/parser',
  parserOptions: { sourceType: 'module' },
  plugins: ['@typescript-eslint'], // 重点看这里
};
```

通过扩展的方式加载插件的规则如下：

```js
extPlugin = `plugin:${pluginName}/${configName}`;
```

比如：

```js
{
  "extends": ["plugin:@typescript-eslint/recommended"]
}

// 或者：
{
  "extends": ["plugin:@typescript-eslint/eslint-recommended"]
}
```

对照上面的案例，插件名`pluginName` 为 `@typescript-eslint` ，也就是之前安装 `@typescript-eslint/eslint-plugin` 包，配置名`configName`为 `recommended` 或者 `eslint-recommended`，表示前面封装的`recommended`或者 `eslint-recommended` 的`plugin` 和 `rules`，**通过这种方式，就无需添加大量的规则来启用插件**。

`extends` 属性值可以是：

- **指定配置的字符串**: 比如官方提供的两个拓展[eslint:recommended](https://github.com/eslint/eslint/blob/v6.0.1/conf/eslint-recommended.js) 或 [eslint:all](https://github.com/yannickcr/eslint-plugin-react/blob/master/index.js#L108)，可以启用当前安装的 ESLint 中所有的核心规则，省得我们在`rules`中一一配置。
- **字符串数组**：**每个配置继承它前面的配置**。如下所示，拓展是一个数组，ESLint 递归地扩展配置, 然后使用`rules`属性来拓展或者覆盖`extends`配置规则。

```js
{
    "extends": [
        "eslint:recommended", // 官方拓展
        "plugin:@typescript-eslint/recommended", // 插件拓展
        "standard", // npm包，开源社区流行的配置方案，比如：eslint-config-airbnb、eslint-config-standard
    ],
    "rules": {
    	"indent": ["error", 4], // 拓展或覆盖extends配置的规则
        "no-console": "off",
    }
};
```

自此，不知看官是否明白`extends`、`plugins`、`rules`三剑客的关系，如果不明白，建议多读两遍，因为它真的挺重要滴。

## 2.4 在注释中使用 ESLint

前面提到过，除了添加 ESLint 配置文件进行`lint`规则配置外，也可以使用注释把`lint`规则嵌入到源码中。

```js
/* eslint eqeqeq: "off", curly: "error" */
```

在这个例子里，`eqeqeq` 规则被关闭，`curly` 规则被打开，定义为错误级别。

当然，人无完人，`lint`规则也一样，只是一种木得感情的辅助工具。在我们日常开发中，总会生产出一些与`lint`规则八字相冲的代码，频繁地 `error`告警总会让人心烦意乱，这时，禁用`lint`规则就显得十分讨巧。

- 块注释： 使用如下方式，可以在整个文件或者代码块禁用所有规则或者禁用特定规则：

```js
/* eslint-disable */
alert('该注释放在文件顶部，整个文件都不会出现 lint 警告');

/* eslint-disable */
alert('块注释 - 禁用所有规则');
/* eslint-enable */

/* eslint-disable no-console, no-alert */
alert('块注释 - 禁用 no-console, no-alert 特定规则');
/* eslint-enable no-console, no-alert */
```

- 行注释： 使用如下方式可以在某一特定的行上禁用所有规则或者禁用特定规则：

```js
alert('禁用该行所有规则'); // eslint-disable-line

// eslint-disable-next-line
alert('禁用该行所有规则');

/* eslint-disable-next-line no-alert */
alert('禁用该行 no-alert 特定规则');

alert(
	'禁用该行 no-alert, quotes, semi 特定规则'
); /* eslint-disable-line no-alert, quotes, semi*/
```

## 2.5 你必须了解的配置优先级

对于如下结构：

```js
your-project
├── .eslintrc  - 父目录配置A
├── lib - 使用A
│ └── source.js
└─┬ tests
  ├── .eslintrc - 子目录配置B
  └── test.js - 使用B和A， 优先级 B>A
```

有两个`.selintrc`文件，**规则是使用离要检测的文件最近的 `.eslintrc`文件作为最高优先级**。因此，`lib/`目录使用父目录的`.selintrc`作为配置文件，而`tests/`目录使用两个`.selintrc`作为配置文件，且子目录 A 的优先级大于父目录 B 的优先级，如果两个文件有冲突，**子目录 A 配置覆盖父目录 A 配置**。如果只想使用子目 A 录配置，可以在子目录 A 配置`.selintrc`中设置`"root": true`，这样就不会向上查找。

如果在你的主目录下有一个自定义的配置文件 (`~/.eslintrc`，比如在 IDE 中使用 ESLint 插件，就会生成这样一个配置文件) ，如果没有其它配置文件时它才会被使用。因为个人配置将适用于用户目录下的所有目录和文件，包括第三方的代码，当 ESLint 运行时可能会导致问题。

综上所述，**对于完整的配置层次结构，从最高优先级最低的优先级**，如下:

1. 行内配置： 比如`/*eslint-disable*/`、`/*eslint-enable*/`、`/*global*/`、`/*eslint*/`
2. 命令行选项（或 CLIEngine 等价物）：比如 `--global`、`--rule`、`--env`
3. 项目级配置：
   1. 与要检测的文件在同一目录下的 `.eslintrc.*` 或 `package.json` 文件
   2. 继续在父级目录寻找 `.eslintrc` 或 `package.json`文件，直到根目录（包括根目录）或直到发现一个有`"root": true`的配置。
4. 如果不是（1）到（3）中的任何一种情况，退回到 `~/.eslintrc` 中自定义的默认配置（即 IDE 环境安装的 ESLint 插件的配置）。

这也是为什么工友们使用的搬砖工具（IDE、插件配置）不尽相同，但是提交代码时，冲突却很少的原因。因为对于成熟的项目，肯定会在根目录下添加`ESLint`配置文件，这个文件的优先级高于 ide 插件优先级。

# 3. Prettier

## 3.1 Prettier 是什么？

Prettier 是一个有见识的代码格式化工具。它通过解析代码并使用自己的规则重新打印它，并考虑最大行长来强制执行一致的样式，并在必要时包装代码。如今，它已成为解决所有代码格式问题的优选方案；支持 `JavaScript`、 `Flow`、 `TypeScript`、 `CSS`、 `SCSS`、 `Less`、 `JSX`、 `Vue`、 `GraphQL`、 `JSON`、 `Markdown` 等语言，您可以结合 `ESLint` 和 `Prettier`，检测代码中潜在问题的同时，还能统一团队代码风格，从而促使写出高质量代码，来提升工作效率。

## 3.2 ESLint 和 Prettier 的区别？

在格式化代码方面， `Prettier` 确实和 `ESLint` 有重叠，但两者侧重点不同：`ESLint` 主要工作就是检查代码质量并给出提示，它所能提供的格式化功能很有限；而 `Prettier` 在格式化代码方面具有更大优势。所幸，`Prettier` 被设计为易于与 `ESLint` 集成，所以你可以轻松在项目中使两者，而无需担心冲突。

## 3.3 Prettier 使用方式

### 3.3.1 安装使用

最基础的使用方式就是使用 `yarn` 或者 `npm` 安装，然后在命令行使用。

```js
$ npm install --save-dev --save-exact prettier

$ yarn add --dev --exact prettier
```

然后在文件夹中创建配置文件和 `ignore` 文件。关于配置文件支持多种形式：

- 根目录创建`.prettierrc` 文件，能够写入`YML`、`JSON`的配置格式，并且支持`.yaml`、`.yml`、`.json`、`.js`后缀；
- 根目录创建`.prettier.config.js` 文件，并对外`export`一个对象；
- 在`package.json`中新建`Prettier`属性。

上面的工作全部完成就可以使用命令格式化我们的代码了。

```js
npx prettier --write . #格式化所有文件
```

### 3.3.2 与 ESLint 配合使用

`ESLint` 和 `Prettier` 相互合作的时候有一些问题，对于他们**交集**的部分规则，`ESLint` 和 `Prettier` 格式化后的代码格式不一致。导致的问题是：当你用 `Prettier` 格式化代码后再用 `ESLint` 去检测，会出现一些因为格式化导致的 `warning`，当你用`eslint --fix`修复问题，又无法通过`Prettier`校验，导致陷入一开始提到的“死循环问题”。

这种问题的主要解决思路是**在 `ESLint` 的规则配置文件上做文章，安装特定的 `plugin`，把其配置到规则的尾部，实现 `Prettier` 规则对 `ESLint` 规则的覆盖**。

实现方案如下：

```js
// 安装eslint-config-prettier
$ npm install --save-dev eslint-config-prettier

// 在 .eslintrc.* 文件里面的 extends 字段添加：
{
  "extends": [
    ...,
    "已经配置的规则",
+   "prettier",
+   "prettier/@typescript-eslint"
  ]
}
```

前面提到过，`extends`的值为数组，会继承和覆盖前面的配置，**将`Prettier`配置添加到其它拓展的后面**，可以实现 `Prettier` 规则对 `ESLint` 规则的覆盖。

**完成上述两步可以实现的是运行 `ESLint` 命令会按照 `Prettier` 的规则做相关校验，但是还是需要运行 `Prettier` 命令来进行格式化**。为了避免多此一举，万能的社区早就提出了整合上面两步的方案：**在使用 `eslint --fix` 时候，实际使用 `Prettier` 来替代 `ESLint` 的格式化功能**。方案如下：

```js
// 安装eslint-plugin-prettier
$ npm install --save-dev eslint-plugin-prettier

// 在 .eslintrc.* 文件里面的 extends 字段添加：
{
  "extends": [
    ...,
    "已经配置的规则",
+   "plugin:prettier/recommended"
  ],
  "rules": {
    "prettier/prettier": "error",
  }
}
```

这个时候你运行 `eslint --fix` 实际使用的是 `Prettier` 去格式化文件。在`rules`中添加`"prettier/prettier": "error"`，当代码出现`Prettier`校验出的格式化问题，`ESLint`会报错。`eslint-plugin-prettier` 具体详细的配置[点这里](https://github.com/prettier/eslint-plugin-prettier)

### 3.3.3 如何对 Prettier 进行配置

同 ESLint 类似，我们可以使用`prettierrc.js`的方式对`Prettier`进行配置，下面讲解下各个配置的作用。

```js
module.exports = {
	printWidth: 80, //一行的字符数，如果超过会进行换行，默认为80
	tabWidth: 2, //一个tab代表几个空格数，默认为80
	useTabs: false, //是否使用tab进行缩进，默认为false，表示用空格进行缩减
	singleQuote: false, //字符串是否使用单引号，默认为false，使用双引号
	semi: true, //行位是否使用分号，默认为true
	trailingComma: 'none', //是否使用尾逗号，有三个可选值"<none|es5|all>"
	bracketSpacing: true, //对象大括号直接是否有空格，默认为true，效果：{ foo: bar }
	parser: 'babylon', //代码的解析引擎，默认为babylon，与babel相同。
};
```

# 4. 提升效率

## 4.1 VS Code 集成

### 4.1.1 Prettier 插件安装

当我们提交代码时，使用命令行校验文件格式，再回去逐行改动格式，重新提交代码是十分影响效率的行为。这时候`Prettier`插件便派上用场了。我们可以在编辑器中安装`Prettier`插件，这样在保存文件时就能格式化文件。边写边格式化，快得飞起~

这里以 VS Code 为例，首先安装 [`Prettier - Code formatter`](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)插件，然后按快捷键 `command` + `+` (windows 是 `ctrl` + `+`)来打开`setting.json`文件，添加如下代码在 VS Code 中将`Prettier`设置为默认格式化程序：

```js
{
  // 设置全部语言的默认格式化程序为prettier
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  // 设置特定语言的默认格式化程序为prettier
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

想要在保存时自动格式化，我们可以用在 VS Code 的配置文件`setting.json`中添加`"editor.formatOnSave": true`。如下所示：

```js
// 设置全部语言在保存时自动格式化
"editor.formatOnSave": ture,
// 设置特定语言在保存时自动格式化
"[javascript]": {
    "editor.formatOnSave": true
}
```

支持的语言有如下几种：

```js
javascript;
javascriptreact;
typescript;
typescriptreact;
json;
graphql;
```

您可以使用 VS Code 设置来配置 `Prettier` 。 `Prettier` 将按以下优先级读取设置：

1. `Prettier` 配置文件，比如`.prettierrc` 、`.prettier.config.js`。
2. `.editorconfig`文件，用于覆盖用户/工作区设置，具体可了解[`EditorConfig for VS Code`](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig)。
3. Visual Studio 代码设置（分用户/工作区设置）。

注意：如果存在任何本地配置文件（即`.prettierrc`），则将不使用 VS Code 设置。

### 4.1.2 ESLint 插件安装

前面提到过，与 linters 集成的最简单且推荐的方法是让 Prettier 进行格式化，并将 linter 配置为不处理格式化规则。 您可以在 Prettier docs 网站上找到有关如何配置每个 linter 的说明。 然后，您可以像往常一样使用每个扩展插件。

这里以[`ESLint`](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)插件举例，安装该插件，可以将 ESLint 规则集成到 VS Code 中，这样在编程过程中，违背 ESLint 规则会自动提示。当然也可以为`ESLint`启用“保存时自动修复”，并且仍然具有格式和快速修复功能：

```js
"editor.codeActionsOnSave": {
    // For ESLint
    "source.fixAll.eslint": true,
}
```

### 4.1.3 editorconfig for vs code 插件安装

百密必有一疏，并非所有的工程项目都能配置尽善尽美的`ESLint`和`Prettier`，由于每个人本地的 VS Code 代码格式化配置不拘一格，在实际的项目开发中，多多少少会因为格式化问题产生争议。因此需要有一个统一的规范覆盖本地配置，[`editorconfig for vs code`](https://marketplace.visualstudio.com/items?itemName=EditorConfig.EditorConfig)承担起了这个作用，只要在项目工程的根目录文件夹下添加`.editorconfig`文件，那么这个文件声明的代码规范规则能覆盖编辑器默认的代码规范规则，从而实现统一的规范标准。

## 4.2 Husky + lint-staged 打造合格的代码检查工作流

如果你安装前面提到过的插件，在实际开发中可以格式化代码和提示 Lint 规则报错问题，但总有人视“提示”而不见，疯狂地往代码里“下毒”，偷偷摸摸地提交代码，日积月累的，`ESLint`也就形同虚设。

试想如果将代码已经`push`到远程后，再进行扫描发现多了一个分号然后被打回修改后才能发布，这样是不是很崩溃，最好的方式自然是**确保本地的代码已经通过检查才能 push 到远程，这样才能从一定程度上确保应用的线上质量，同时也能够避免 lint 的反馈流程过长的问题**。

那么什么时候开始进行扫描检查呢？这个时机自然而然是本地进行`git commit`的时候，如果能在本地执行`git commit`操作时能够触发对代码检查就是最好的一种方式。这里就需要使用的`git hook`。

这里以最常见的`git hook`工具——`husky`举例：

首先，安装依赖：

```js
npm install -D husky
yarn add --dev husky
```

然后修改 `package.json`，增加配置：

```js
{
  "scripts": {
    "precommit": "eslint src/**/*.js"
  }
}
```

这样，在 `git commit` 的时候就会看到 `pre-commit` 执行了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7bc7883b41d414396e63cfb1d995d59~tplv-k3u1fbpfcp-watermark.image)

针对历史项目，在中途安装代码检查工作流，提交代码时，对其他未修改的“历史”文件都进行检查，一下出现成百上千个错误，估计会吓得立马删掉`eslint`等相关配置，冒出一身冷汗。如上图所示，明明只修改了 A 文件，B、C、D 等文件的错全部都冒出来了。针对这样的痛点问题，就是**每次只对当前修改后的文件进行扫描，即进行`git add`加入到`stage`区的文件进行扫描即可，完成对增量代码进行检查**。

如何实现呢？这里就需要使用到`lint-staged`工具来识别被加入到`stage`区文件。

首先安装依赖：

```js
npm install -D lint-staged
yarn add --dev lint-staged
```

然后修改 `package.json`，增加配置：

```js
"husky": {
  "hooks": {
    "pre-commit": "lint-staged"
  }
},
"lint-staged": {
  "src/**/*.{js,vue}": ["prettier --write", "eslint --cache --fix", "git add"]
}
```

在进行`git commit`的时候会触发到`git hook`进而执行`precommit`，而`precommit`脚本引用了`lint-staged`配置**表明只对`git add`到 stage 区的文件进行扫描**，具体`lint-staged`做了三件事情：

1. 执行`Prettier`脚本，这是对代码进行格式化的;
2. 执行`eslint --fix`操作，进行扫描，对`eslint`问题进行修复；
3. 上述两项任务完成后将代码重新`add`进 stage 区，然后执行`commit`。

# 5 写在最后

通过引入以上这些工具能够在一定程度上保证应用的质量问题，并能达到事半功倍的效果。但归根结底，对代码质量的提升需要自身与内心达成契约，也需要团队之间“志趣相投”。希望读到这里的你能把 Lint 工作流打磨到极致，把更多时间专注在解决真正的问题上，成为真正高效的工程师。
