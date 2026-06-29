# Typesetter · 公众号文章排版工具

一个开箱即用的微信公众号排版工具。左侧写 Markdown，右侧实时预览公众号效果，一键复制带内联样式的 HTML，直接粘贴到公众号编辑器即可发布——无需构建、无需后端、无需安装，打开 `index.html` 就能用。

## 功能特性

- **实时预览** — 编辑器输入即时渲染，模拟 680px 手机阅读宽度
- **10 款精选主题** — 涵盖人文、科技、自然等多种风格，侧边栏一键切换
- **一键复制排版** — 自动将 CSS 内联到每个元素，粘贴到公众号编辑器后样式不丢失
- **同步滚动** — 编辑器与预览区按比例联动，长文写作不迷路（可切换独立滚动）
- **标准 Markdown** — 兼容 GFM 语法，支持标题、加粗、引用、代码块、表格、链接、图片
- **纯前端单文件** — 一个 `index.html` 完成所有工作，不依赖构建工具

## 主题预览

| 主题 | 风格 | 适用场景 |
|------|------|----------|
| 素笺 | 纸笺质感 · 温润日常 | 散文、随笔、生活记录 |
| 墨砚 | 文人气韵 · 深邃克制 | 评论、深度长文 |
| 霜白 | 冷感留白 · 理性纯净 | 学术、严肃论述 |
| 松风 | 山林清气 · 自然呼吸 | 旅行、自然、慢生活 |
| 晚照 | 暮色温柔 · 暖意流淌 | 情感、故事、人文 |
| 青岚 | 山间薄雾 · 蓝灰诗意 | 诗歌、文艺、设计 |
| 极客 | 文档气质 · 结构清晰 | 技术博客、教程 |
| 蓝图 | 架构设计 · 冷静克制 | 系统设计、工程方案 |
| 终端 | 命令行风 · 极客专属 | 编程、极客文化 |
| 星图 | 数据洞察 · 科技分析 | 数据分析、行业观察 |

## 技术栈

- **React 18**（UMD CDN 引入，无构建步骤）
- **marked**（Markdown 解析）
- **原生 CSS**（主题文件独立存放，按需加载）
- 零依赖、零打包——所有逻辑都在 `index.html` 一个文件里

## 项目结构

```
typesetter/
├── index.html              # 应用主文件（React + 全部逻辑）
├── overview.md             # 开发笔记与修复记录
├── style/                  # 主题样式目录
│   ├── sujian.css          # 素笺
│   ├── moyan.css           # 墨砚
│   ├── shuangbai.css       # 霜白
│   ├── songfeng.css        # 松风
│   ├── wanzhao.css         # 晚照
│   ├── qinglan.css         # 青岚
│   ├── jike.css            # 极客
│   ├── lantu.css           # 蓝图
│   ├── zhongduan.css       # 终端
│   └── xingtu.css          # 星图
```

## 快速开始

由于主题 CSS 通过 `fetch` 动态加载，需通过 HTTP 服务器打开（直接双击 `index.html` 会因 `file://` 协议导致 CSS 加载失败）。

任选一种方式启动本地服务器：

```bash
# Python
python -m http.server 8000

# Node（http-server）
npx http-server -p 8000

# PHP
php -S localhost:8000
```

浏览器访问 `http://localhost:8000` 即可使用。

## 使用方法

1. 在左侧编辑器输入或粘贴 Markdown 内容
2. 点击左侧边栏的圆点切换主题，右侧即时预览效果
3. 点击「复制排版」按钮（或按 `Ctrl/Cmd + S`）
4. 打开微信公众号图文编辑器，`Ctrl/Cmd + V` 粘贴，样式完整保留

## 支持的 Markdown 语法

| 语法 | 说明 |
|------|------|
| `# 标题` / `## 标题` / `### 标题` | 三级标题层级，主题自动添加装饰（编号、色块、分隔线等） |
| `**加粗**` / `*斜体*` | 行内强调，主题自定义颜色 |
| `> 引用` | 引用块，带左侧色条与背景 |
| `` `行内代码` `` / ` ```代码块``` ` | 等宽字体，深色背景 |
| `[链接](url)` | 文字超链接 |
| `![图片](url)` | 图片自适应宽度 |
| `---` | 水平分隔线 |
| `\| 表格 \|` | 标准表格语法 |

## 复制排版的实现原理

公众号编辑器只接受**内联样式**（inline style），会剥离 `<style>` 标签和 `class` 属性。因此复制时需把所有 CSS 规则转换为元素的内联 style。本项目采用 CSS 规则解析方案（参考 doocs/md 的 juice 方案）：

```
主题 CSS 字符串
   │
   ▼ parseCSS()   正则解析为 {selector, properties} 规则数组
   │
   ▼ inlineStyles()  querySelectorAll 匹配元素 → setProperty 注入
   │                 ::before/::after → 转为真实 <section> 插入 DOM
   │                 继承性属性（font-family/color 等）下放到每个子元素
   │
   ▼ serialize()   遍历 DOM，读取 element.style 生成 HTML 字符串
   │               （style 值内的双引号转为单引号，避免属性解析冲突）
   │
   ▼ navigator.clipboard.write()   写入剪贴板
```

**关键设计**：

- **不依赖 `getComputedStyle`** — 直接解析 CSS 规则并匹配选择器，避免浏览器默认值混入和伪元素提取不可靠的问题
- **继承性属性下放** — `font-family`、`color`、`line-height` 等只定义在根容器 `.article-body` 上，复制时显式下放到每个子元素，防止公众号剥离根容器后继承链断裂
- **伪元素实体化** — `::before`/`::after`（如标题前的 `◆`、编号 `PART 01`）转换为真实 `<section>` 节点插入 DOM，保留全部声明属性
- **CSS counter 计算** — `counter-reset`/`counter-increment` 在序列化前预计算，伪元素中的 `counter()` 函数被替换为实际数值
- **跳过根容器** — 序列化时只输出 `.article-body` 的子节点，避免公众号把最外层 `<section>` 当作字面文本显示

## 键盘快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl/Cmd + S` | 复制排版 |

## 浏览器兼容性

推荐使用 **Chrome / Edge / Firefox** 最新版。需支持 Clipboard API（`navigator.clipboard.write`），不支持的浏览器会自动回退到 `execCommand('copy')`。

## 自定义主题

主题就是一个标准的 CSS 文件，所有规则以 `.article-body` 为根选择器。新建主题步骤：

1. 在 `style/` 目录下新建 `mytheme.css`
2. 参考 `style/sujian.css` 的结构编写样式
3. 在 `index.html` 的 `THEMES` 数组中添加一条配置：

```javascript
{id:'mytheme', name:'我的主题', color:'#yourcolor', desc:'主题描述'},
```

4. 刷新页面，左侧边栏即可看到新主题

可定义的元素包括：`h1`~`h3`、`p`、`blockquote`、`strong`/`b`、`em`/`i`、`a`、`code`、`pre`、`hr`、`img`、`table`，以及对应的 `::before`/`::after` 伪元素装饰。

## 开发笔记

详细的修复记录与架构决策见 [overview.md](./overview.md)，主要包括：

- 同步滚动的 mouseenter 方案（参考 wanglin2/bytemd）
- 复制样式丢失的三重根因修复（双引号冲突、根容器溢出、继承性属性下放）

## 许可

本项目供个人使用与学习交流。
