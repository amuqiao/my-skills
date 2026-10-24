# 架构：分层 + 单向数据流

## 分层

```text
public/js/
  core/              基础设施：不认识任何具体功能
    dom.js           $ / $$ / on（事件委托）/ html``（默认转义）/ raw / render / append
    api.js           api()、ApiError、URL 拼装
    store.js         共享数据 + subscribe / emit + setter
    ui.js            toast、菜单、打包下载等全页行为
    types.js         JSDoc 类型（尽量引用后端的类型定义）
    icons.js         icon(name) → 引用 SVG 图标集
  <feature>.js       功能模块：各管一块界面与它自己的界面状态
  <feature>/         功能大时拆子目录（如查看器的多个工具区）
  <pure>.js          纯逻辑：顶层不碰 DOM，可在 Node 里单元测试
  main.js            入口：读配置 → 绑定各模块 → 加载初始数据
```

依赖只能向下：功能模块 → core；纯逻辑模块不依赖 DOM。功能模块之间如需协作，优先走 store 事件；确需直接调用时只能单向（A 调 B，B 不回调 A），并在评审时检查是否形成环。

## 单向数据流

```text
用户操作 → 功能模块 → api() → store.setXxx(...) → emit(事件) → 订阅的模块各自重绘
```

- **store 只放共享数据**：服务端配置、当前打开的主对象（项目 / 文档 / 工作区）、跨模块共享的运行状态。
- **setter 负责发事件**：调用方不再记得"改完要通知谁"。按变化粒度拆事件，例如 `project`（整体替换，全部重绘）、`refs` / `style`（只重绘对应面板）、`assets`（列表变了，可带 `newId` 做入场动画）、`job`（后台任务开始 / 结束）。
- **切换主对象前先发 `switch`**（带 from / to），让依赖旧对象的模块先收尾（断开事件流、清空选择），再加载新对象并发整体事件。
- **事件只描述发生了什么**，不描述该怎么渲染；渲染逻辑留在订阅方。
- **无附带数据的事件，detail 为 null**（`CustomEvent` 的默认值），类型声明写 `null`，不要写 `void`。

## 功能模块的写法

```js
const el = { list: $("#refs"), input: $("#refInput") };   // 顶部集中拿 DOM 引用
const view = { filter: "all", query: "" };                 // 自己的界面状态

function renderList() { render(el.list, html`...`); }      // 只读 store 与本模块状态

export function bindRefs() {                               // 只在入口调用一次
  subscribe("project", renderList);
  subscribe("refs", renderList);
  on(el.list, "click", "[data-remove]", async (_e, b) => {
    setProject(await api(`/api/.../${b.dataset.remove}`, { method: "DELETE" }), "refs");
  });
}
```

规则：

- `subscribe` 与 `on` 只在 `bindXxx()` 里注册一次，不在渲染函数里注册，避免重复绑定。
- 列表交互用事件委托（`on(root, "click", "[data-x]", ...)`），不给每个渲染出来的元素单独绑事件。
- 初始化顺序：先 `bind` 所有模块，再加载初始数据；渲染函数要能处理"主对象尚未加载"（直接返回，等首个整体事件）。

## 用户偏好（localStorage）

视图、排序、尺寸、显示哪些字段这类偏好属于界面状态，留在所属模块，按需存 localStorage：

- key 带版本号（`app.library.v2`）；结构变了就换新 key，不写旧格式兼容。
- 读取经一个纯函数校验：逐项检查类型与取值范围，非法项回到默认值，未知项丢弃；这个函数放纯逻辑模块并写单元测试。
- 读写包在 try/catch 里（隐私模式、禁用存储时会抛错）；读不到就用默认值，这是偏好本身的语义，不算静默兜底。
- 默认值以"信息完整"为准：所有字段显示、不折叠，让用户自己收。

## 时间

- 存储与接口一律 UTC ISO 字符串；给人看的时间只经 `core/time.js` 一类的格式化模块，用 `Intl.DateTimeFormat` 显式传 `timeZone`（项目约定的时区）与 `hourCycle`。
- 后端若也生成给人看的时间（导出文件名、压缩包内时间戳），与前端共用同一个时区常量，并有测试保证两边一致。

## 声明式 data-* 行为

全页通用的行为用"属性 + document 级委托"实现，新增入口只加属性、不写 JS：

- `<a data-download href>`：服务端现打包的下载，点击后显示"打包中…"、防连点、失败提示。
- `<button data-lightbox="组" data-src>`：点击全屏预览，同组 ← → 切换。
- `<details data-menu>`：下拉菜单，点外部或 Esc 关闭。

需要额外能力的调用方（如预览里加"选为参照"按钮）直接调用组件函数（`openLightbox(items, i, { action })`）。

## 什么时候该升级到框架

出现以下情况时，向用户说明并建议迁移到 React / Vue 等：

- 前端超过约 1 万行，或多人长期并行开发。
- 大量相互依赖的表单与派生状态，手写订阅开始变复杂。
- 需要 SSR / SEO、复杂路由、细粒度重渲染性能优化。

本架构与框架同源（store ≈ Redux/Pinia，订阅重绘 ≈ 组件响应式），迁移时 store 与模块边界可以直接对应过去。
