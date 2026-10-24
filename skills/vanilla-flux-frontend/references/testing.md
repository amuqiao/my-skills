# 测试：两层金字塔

```text
          ▲ e2e（Playwright）：主流程、交互、键盘、弹窗层叠      慢，少而关键
         ▲▲▲ 单元（node:test）：转义、store 事件、计算、解析、筛选  快，多
```

| 改动 | 测试 |
|---|---|
| 后端新增逻辑 | 单元测试（沿用后端现有框架与覆盖率门槛） |
| 前端新增纯逻辑 | 放进顶层不碰 DOM 的模块，`node:test` 单元测试 |
| 前端新增交互 / 流程 | 加一条 e2e |
| 纯样式 | 不写测试，截图检查 |

## 单元测试：让前端逻辑能在 Node 里跑

- 纯逻辑模块顶层不得访问 `document` / `window` / `$()`；有 DOM 的模块只调用它们。常见可抽出的：缩放 / 布局计算、表单字段解析与标签、列表筛选与排序、格式化（金额、token、字节）。
- `core/dom.js` 的 `html` / `esc` 与 `core/store.js` 在 Node 里可直接导入（Node 自带 `EventTarget`、`CustomEvent`）。
- TS 项目：主 `tsconfig.json` 开 `allowJs`，测试用 `.ts` 直接 import 前端 `.js`，类型来自 JSDoc。
- 必测：`html` 对 `<>&"'` 的转义、嵌套模板与 `raw()` 不二次转义、数组拼接、`null / undefined / false` 为空而 `0` 会渲染（固化陷阱）；store 每个 setter 发出的事件与 detail（无数据为 `null`）。
- 前后端各算一份的派生值（版本编号、尺寸档位、时区常量、排序规则），在单元测试里同时导入两边的实现交叉比对，防止一边改了另一边没跟上。
- 和时间有关的测试固定输入时刻并断言格式化结果；在非约定时区下也跑一次（如 `TZ=America/New_York npm test`），确认结果不随本机时区变化。
- 订阅无法注销时，整个测试套件共用一个记录器，每个用例取自己触发的那一段快照（`structuredClone`），不要共用可变数组做断言。

## e2e：离线、可重复、不花钱

- **专用服务入口**：写一个测试用的启动脚本，直接挂载应用工厂（如 `createApp()`），不走读取 `.env` 的正式入口；启动前清空所有真实密钥环境变量；数据目录用临时目录；外部服务只保留 mock。
- **Playwright 配置**：`webServer` 启动上面的脚本并等待健康检查；`use.channel: "chrome"` 驱动本机已装的 Chrome，不下载浏览器；`workers: 1`；`trace: "retain-on-failure"`；CI 用 `forbidOnly`。
- **用例隔离**：每个用例通过 API 新建自己的数据（项目 / 工作区），用 `page.addInitScript` 写 localStorage 让页面打开它；不依赖其他用例的状态。
- **页面报错即失败**：`beforeEach` 收集 `pageerror`，`afterEach` 断言为空（不要在事件回调里 throw）。
- **稳定选择器**：用 id、`data-*` 和角色，不依赖样式类和整段文案；界面有意改动时同步改测试并在提交里说明。
- **常见误判**：列表重绘后旧的 locator 集合失效（每次重新取 `.first()`）；异步操作要先等结果出现再断言下一步；排序方向（最新在前）要按真实顺序写用例；并行产出的结果顺序不确定，断言按内容或 `data-*` 找元素，不按位置（`.nth()`）找，否则会偶发失败。
- 先为老功能写好 e2e 并在旧代码上跑通，再开始重构；每个重构提交都要在不含后续改动的状态下单独跑一遍（可先 `git stash` 掉未提交的下一步改动）。

## CI

- `npm run check` = typecheck（含 web 配置）+ lint + 单元测试。
- e2e 在一个 Node 版本上跑即可（如 matrix 的最高版本），GitHub 的 ubuntu runner 自带 Chrome。
- 忽略 `test-results/`、`playwright-report/`（git 与 lint 都忽略）。
