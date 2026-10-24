# 视觉体系：token、组件、布局

"廉价感"多半来自三件事：层级不清（所有按钮一个样）、强调色滥用、数值随手写（十几种字号、几十种 padding）。先定体系，再改页面。

## 1. Token（写在 base.css 开头，其余文件只引用）

```css
:root {
  --sp-1: 4px; --sp-2: 8px; --sp-3: 12px; --sp-4: 16px; --sp-5: 24px; --sp-6: 32px;   /* 间距：4 的倍数 */
  --fs-2xs: 11px; --fs-xs: 12px; --fs-sm: 13px; --fs-md: 14px; --fs-lg: 16px; --fs-xl: 18px;
  --r-xs: 4px; --r-sm: 6px; --r-md: 10px; --r-lg: 14px; --r-full: 999px;
  --h-sm: 28px; --h-md: 32px; --h-lg: 40px;                                           /* 控件高度 */
  --dur: 150ms; --ease: cubic-bezier(0.2, 0.7, 0.3, 1);
  --bg; --surface; --surface-2 /* 凹陷、填充 */; --surface-3 /* hover、虚线 */;
  --ink; --ink-2; --muted; --line;
  --accent; --accent-hover; --accent-ink; --accent-soft;
  --ok / --ok-soft; --bad / --bad-soft; --warn / --warn-soft;
  --shadow-1; --shadow-2; --focus-ring;
}
@media (prefers-color-scheme: dark) { :root { /* 只重定义颜色与阴影 token */ } }
```

- 中性色带一点品牌色相，不用纯灰。
- 文字色在 `--bg`、`--surface`、`--surface-2` 及对应的 soft 底上达到 WCAG AA（4.5:1），徽标的小字尤其要算。
- 暗色不是反相：重新挑 surface 层级，强调色提亮，阴影加深。
- 遵守 `prefers-reduced-motion`：统一关闭动画与过渡。

## 2. 基础组件

| 组件 | 规则 |
|---|---|
| 按钮 | `primary`（实心强调色，每个区域最多一个）/ `secondary`（白底描边）/ 默认（透明，hover 填充）/ `link`；尺寸 small / 默认 / large；hover、按下位移 1px、disabled、`data-busy` 加载态 |
| 输入 | 默认 `--surface-2` 无边框，hover 出边框，focus 变白底 + 强调色边 + 光圈；覆盖 `input:not([type])` |
| 开关 | 用 checkbox + `appearance: none` 画成 switch，不用原生勾选框 |
| 分段控件 | 筛选、页签统一用 `.segmented`，选中项白底 + 阴影 |
| 标签 chip | 默认中性，选中才用 `--accent-soft` + 强调色字 + ✓，避免一片强调色 |
| 徽标 | 色点 + 文字，底色用 soft 色；不只靠颜色区分状态 |
| 菜单 | `<details data-menu>` + `.menu-pop`，菜单项带图标与一行说明，危险项单独分隔并标红；靠近栏右边缘的菜单向左展开（或用 popover 顶层），不被所在栏的 `overflow` 裁掉 |
| 对话框 | 原生 `<dialog>` + `::backdrop` 模糊；入场动画；关闭按钮用图标按钮 |
| toast | popover 顶层，胶囊形，深底浅字 |

焦点：用 `:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px }`，不要用 box-shadow 做焦点环（会被组件自身阴影覆盖）。

## 3. 布局（工具类应用）

- 应用式外壳：吸顶顶栏（半透明 + backdrop 模糊）+ 下方多栏铺满窗口，不设 `max-width` 居中留白。
- 内容固定的栏（设置、表单）用 `clamp()` 定宽；内容随屏幕伸缩的栏（列表、网格、画布）拿走剩余宽度。
- 每栏独立滚动：栏 `display:flex; flex-direction:column; overflow:hidden`，栏体 `flex:1; min-height:0; overflow:auto`。
- 断点下退回整页滚动；此时 sticky 元素的 `top` 要让出吸顶顶栏高度。
- 网格卡片 `repeat(auto-fill, minmax(var(--card-w), 1fr))`，尺寸滑杆只改 `--card-w`（有上下限）；列表视图同样给行高 / 缩略图尺寸一个可调变量，切换视图时滑杆不消失。信息密度靠"显示"菜单（勾选卡片上显示哪些字段）调，默认全显示；卡片内信息区用 `grid-template-columns: minmax(0,1fr)`，徽标 `flex:none`，避免被挤截断。
- 少用边框，多用背景深浅与留白分区；卡片阴影只给需要"浮起"的对象。

## 4. 图标

- 在 HTML 顶部放一个隐藏的 SVG `<symbol>` 图标集（Lucide 风格描边），用 `<svg class="i"><use href="#i-name"/></svg>` 引用；JS 里用 `icon(name)`（内部是 `raw()`，名字只能是常量）。
- 会被 JS 改文字的按钮，把文字包进 `<span data-label>`，只改这个 span，图标不丢。
- 窄屏隐藏按钮文字时，按钮要有 `aria-label`。

## 5. 缩略图与预览

- 所有缩略图可点开全屏预览：缩略图本身是 `<button data-lightbox>`（键盘可达），上面不叠常驻图标，只在 hover 时显示放大光标与覆盖在图上的 outline 描边。
- 预览：深色背景 + 下层模糊；适应窗口时按内容区（扣除 padding）计算；小图整数倍放大并 `image-rendering: pixelated`；可切原始大小、下载、同组切换；Esc 只关预览不连带关下层对话框；快捷键遇到修饰键不拦截。
- 透明图一律放在棋盘格上。

## 6. 截图验收

用无头浏览器（Playwright 驱动本机 Chrome）截图检查：亮色、暗色、约 1440 / 2560 / 390 宽、主要对话框和菜单。注意：

- 切换配色或打开菜单后等过渡结束（约 300–400ms）再拍，否则拍到半透明中间态。
- 改视口尺寸后用新页面重拍，复用页面缩放可能出现重复渲染的假象。
- 用离线 mock 数据，不调用付费服务。
