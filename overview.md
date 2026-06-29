# Typesetter 排版工具 — v2 重写

## 同步滚动修复

### 根因
前两次修复使用 setTimeout/rAF 方向锁，均不可靠——scroll 事件异步触发时序在不同浏览器中不一致，导致锁要么过短（回环反弹）、要么过长（丢失同步）。

### 方案（参考 wanglin2/bytemd 的业内标准做法）
**mouseenter 跟踪活动面板**：
- 编辑器 textarea 添加 `onMouseEnter` → 设置 `activePanel = 'editor'`
- 预览区 `.preview-wrap` 添加 `onMouseEnter` → 设置 `activePanel = 'preview'`
- 默认值为 `'editor'`（支持键盘滚动场景）
- 滚动时仅活动面板触发同步，被动面板的 scroll 事件被忽略

### 为什么这个方案可靠
- 无需锁、无需 timer、无需 rAF — 纯声明式
- mouseenter 是浏览器原生事件，触发时机确定
- 程序化设置 scrollTop 触发的 scroll 事件不影响 activePanel

## 复制样式修复

### 根因
`getComputedStyle` 方法缺陷：
- 返回浏览器默认值混入，过滤逻辑极易误杀/漏杀
- 伪元素样式提取不可靠（`getComputedStyle(el, '::before')` 在不同浏览器中行为不一致）
- 手动维护属性白名单遗漏关键 CSS 属性

### 方案（参考 doocs/md 的 juice CSS 内联方案）
**CSS 规则解析 + 选择器匹配 + DOM 内联**：
1. `parseCSS()` — 用正则解析主题 CSS 字符串 → 结构化的规则数组（选择器 + 属性映射）
2. `inlineStyles()` — 对 cloneNode 的 DOM 用 `querySelectorAll` 匹配规则，通过 `element.style.setProperty()` 直接注入声明值
3. `makePseudo()` — 将 `::before`/`::after` 规则转换为真实 `<span>` 插入 DOM
4. `serialize()` — 遍历修改后的 DOM，通过 `element.style` 读取内联样式生成 HTML 字符串

### 关键优势
- 不依赖浏览器渲染管线，不依赖 getComputedStyle
- 精准匹配：parser 直接读取 `.article-body h1 { ... }` 并匹配到 `<h1>` 元素
- 伪元素完整转换：`content:'◆'` 带所有声明属性（display/width/height/background/position 等）
- 没有属性白名单遗漏问题 — 所有 CSS 声明都原样转为内联

## 改动文件
- `index.html`：同步滚动 useEffect、handleCopy 函数、activePanelRef、mouseEnter 事件

## 复制粘贴丢失样式修复（v2.1）

### 用户反馈
1. 复制到公众号编辑器后**字体样式丢失**（font-family、color、line-height、font-size、letter-spacing 全部消失）
2. 文章开始位置**溢出 `<section>` 字面文本**

### 根因 1：style 属性双引号冲突（致命）
`serialize()` 输出 `style="font-family:"Noto Serif SC",..."`，font-family 值内部的双引号与 HTML 属性包裹的双引号冲突。HTML 解析器在 `font-family:"` 处认为 style 属性提前结束，**所有位于 font-family 之后的 CSS 声明（color、line-height、font-size、letter-spacing 等）被浏览器当作无效属性丢弃**。

公众号编辑器接收到的 HTML 实际只剩 font-family 之前的部分声明，所以字体相关样式全部丢失。

### 根因 2：根容器 `<section>` 溢出
`handleCopy` 序列化的是 `.article-body` 这个 `<section>` 元素本身，最终 HTML 是 `<section style="...">...</section>`。公众号编辑器对最外层 `<section>` 根容器的处理不一致，部分版本会把它当作字面文本显示。

### 根因 3：继承性属性未下放
所有主题 CSS 的 `font-family`、`color`、`line-height`、`font-size`、`letter-spacing` 都只定义在 `.article-body` 根容器上，子元素通过 CSS 继承获得。但 `el.style.getPropertyValue()` 不返回继承值，序列化后子元素内联 style 里没有这些属性。公众号编辑器剥离根 `<section>` 后，继承链断裂，所有继承样式全部丢失。

### 修复方案
1. **serialize 输出 style 时把值内的双引号替换为单引号**（CSS 规范允许两种引号）
2. **handleCopy 序列化时跳过最外层 `<section>` 根容器**，只输出子节点
3. **inlineStyles 末尾下放继承性属性**：收集 `.article-body` 上的 font-family/color/line-height/font-size/letter-spacing/text-align/word-spacing/text-indent/font-weight/font-style/white-space/text-transform，遍历所有子元素，仅当子元素未显式定义时显式设置继承值

### 验证
通过 jsdom 离线测试：
- 17/17 块级元素（p/h1/h2/h3/blockquote/li）全部包含 font-family、color、line-height、font-size、letter-spacing
- HTML 不再以 `<section class="article-body">` 包裹
- font-family 值用单引号，HTML 属性结构完整

### 改动文件
- `index.html`：
  - `inlineStyles()` 末尾新增继承性属性下放逻辑
  - `serialize()` 输出 style 时把 `"` 替换为 `'`
  - `handleCopy()` 序列化时遍历 `clone.childNodes` 而非序列化 `clone` 本身