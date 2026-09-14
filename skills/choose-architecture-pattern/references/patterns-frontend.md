# 前端 / Web 成熟模式

引擎第 4 步识别出项目类型为"前端/Web"时读本包。使用方式：先在下方各维度小节中，按需求信号找到对应行，确认"真实工程问题"；再回到引擎第 5-8 步，用"成熟模式"列的结论驱动方案选择与生产审查。本包是模式地图，不是强制规则——只有场景需要时才采用对应模式。

---

## 服务端状态 (Server State)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "刷新页面数据总是重新请求，等很久" | 缺少缓存层，每次挂载都打完整请求 | Server state cache（TanStack Query / SWR）— staleTime 控缓存窗口；不适用于需要实时强一致的写确认场景 |
| "多个组件都要同一份数据，请求打了好几次" | 同一 key 的请求未去重，缺 deduplication | Server state cache 去重（同一 key 在同一 tick 内只发一次请求）— 不适用于不同生命周期必须各自独立取的场景 |
| "数据改了但页面没更新，要手动刷新" | 写操作后未触发 cache invalidation | Cache invalidation / refetch（mutate 后 invalidate 相关 query key）— 不适用于乐观更新已覆盖最终态、不需要重取的场景 |
| "想把接口返回数据存进 Redux / Zustand 方便全局共享" | 把 server state 当 client state 管理，引入双重来源和同步负担 | Server state cache 即全局来源 — 直接用缓存库的 query key 作为跨组件共享入口，不要把 server state 复制进 Redux/Zustand 形成双重来源；详见下方反模式提示 |

- **Server state cache**：TanStack Query、SWR 等库将远端数据的生命周期（fetching、caching、stale、revalidation）与 UI 解耦。适用于列表、详情、用户信息等任何"从服务端取、展示、偶尔写"的数据。
- **staleTime**：控制数据多久内视为新鲜、跳过重取；refetchOnWindowFocus 等策略控制后台同步频率。两者配合避免过度请求与过度陈旧。
- **Cache invalidation**：写操作成功后调用 `invalidateQueries`（或 SWR `mutate`），让相关缓存标记失效并触发重取，是保证 UI 与服务端一致的最直接手段。

---

## 客户端状态 (Client State)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "这个多步表单/向导流程，步骤之间的跳转逻辑太乱" | UI 流程状态分散，条件分支隐式，难以追踪和测试 | Client state machine（XState / useReducer 显式状态机）— 边界：纯线性、无分支的简单流程用普通 useState 即可，不必引入状态机 |
| "筛选条件/分页/tab 切换后，刷新或分享链接就丢了" | 可分享、可回退的 UI 状态未持久化到 URL | URL as state（searchParams / query string 作为稳定来源）— 不适合存放敏感信息、高频变化的临时状态（如输入框实时内容）或超大 JSON payload |
| "表单提交两次了，数据重复写入" | 提交期间未锁定，用户多次触发写操作 | Form state：提交期锁定（isSubmitting 标志，禁用提交按钮）+ 防重提交 — 不适合不需要防重的幂等只读查询 |
| "字段校验报错时机不对，要么太早要么太晚" | 校验时机策略未明确（onChange / onBlur / onSubmit） | Form state：校验时机分层（onBlur 触发单字段，onSubmit 触发全量）— 边界：实时联动校验（如密码强度）可在 onChange 触发，但要节流 |
| "用户改了表单又取消，原始数据被污染了" | 缺少脏值跟踪，取消操作无法还原 | Form state：脏值跟踪（dirty diff，取消时 reset 到 defaultValues）— 简单只有一两个字段的内联编辑用受控组件 + 本地 state 即可 |

- **Client state machine**：用显式状态节点和转移边（transitions）代替散落的 boolean 标志组合。适用于有明确"阶段"和"事件"的 UI 流程，例如多步向导、上传进度、支付流程。
- **URL as state**：把筛选、分页、激活 tab 等状态序列化进 URL searchParams，使页面可收藏、可分享、支持浏览器前进/后退。注意不要将敏感数据（token、个人信息）或高频临时状态（输入中的文本）放入 URL。
- **Form state**：受控组件将值保存在 React state / 表单库 store 中，适合需要实时校验和脏值跟踪的场景；非受控组件用 ref 读取 DOM 值，适合简单一次性表单。提交期间必须锁定提交入口，配合后端幂等键或请求去重彻底杜绝重复写入。

---

## 数据写入与 UI 一致性 (Mutations & UI Consistency)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "点击操作有延迟，用户感知到卡顿" | 写操作等待服务端响应才更新 UI，感知延迟高 | Optimistic UI with rollback — 仅限冲突可控、失败影响范围小、回滚逻辑明确的场景；不适合金融/库存等强一致写入 |
| "乐观更新了，但接口失败后页面数据不对" | 乐观更新缺少失败回滚路径，UI 与服务端状态不一致 | Optimistic UI with rollback：onError 回调中 rollback 到旧快照，再 invalidate 触发重取 — 边界：回滚逻辑复杂的操作（嵌套列表、批量排序）慎用乐观更新 |
| "提交成功后，列表没有更新" | mutation 成功后未触发 cache invalidation，UI 读的是旧缓存 | mutation 后 invalidate（见 Server state 维度）— 乐观更新与 invalidation 配合：onSuccess 时 invalidate 让缓存最终对齐服务端真实态 |

- **Optimistic UI with rollback**：mutation 触发时立即用预期结果更新本地缓存，同时保留旧快照；若服务端返回失败，在 `onError` 中恢复旧快照，再调用 `invalidateQueries` 重取真实数据。乐观更新的收益是即时反馈，代价是必须维护完整回滚路径。
- **乐观更新 + invalidation 配套**：乐观更新只解决感知延迟，不替代 invalidation——`onSuccess` 时仍应 invalidate 相关 query，确保 UI 最终与服务端一致，不依赖客户端推算的结果。

---

## 渲染与加载 (Rendering & Loading)

| 需求信号（用户的话） | 真实工程问题 | 成熟模式 |
|---|---|---|
| "SEO 不好 / 首屏白屏时间长" | 纯 CSR，HTML 无实质内容，爬虫和 LCP 均受损 | SSR / SSG（Next.js / Nuxt / Remix 等）— 边界：高度动态、用户私有、无 SEO 需求的管理后台仍可用纯 CSR |
| "SSR 后客户端 hydration 报错或内容闪烁" | 服务端渲染结果与客户端首次渲染不一致（hydration mismatch） | Hydration mismatch 隔离：client-only 内容用 `dynamic({ ssr: false })` 或 `<ClientOnly>` 包裹，延迟到客户端渲染 — 不适合需要 SSR 的关键内容 |
| "首次加载 JS 包太大，TTI 慢" | 所有代码打成单包，用户加载了未使用的模块 | 代码分割（Code splitting）：route-level lazy loading 为首选；component-level splitting 用于重型、低频组件（富文本编辑器、图表库）— 过细的分割会引入加载瀑布流，适得其反 |
| "页面切换时连续触发多个 loading，体验跳跃" | 过度分割导致串行加载瀑布（waterfall），多个 Suspense 边界依次挂起 | 合理合并 Suspense 边界：同屏同时需要的资源放在同一 Suspense 下，避免串行挂起；路由级预加载（prefetch）减少切换延迟 |

- **SSR / SSG**：服务端渲染生成含实质内容的 HTML，改善首屏性能和 SEO。SSG 适合内容变化频率低的页面；SSR 适合需要请求级动态数据的页面；ISR 兼顾两者。选型前先确认是否真的有 SEO 或首屏 LCP 需求。
- **Hydration mismatch 隔离**：浏览器专属 API（`window`、`localStorage`、随机值、时间戳）在 SSR 阶段不可用，必须将依赖这些 API 的组件标记为 client-only，防止服务端与客户端渲染结果不一致。
- **Route-level code splitting**：以路由为粒度懒加载，是成本最低、收益最高的分割策略。Component-level splitting 仅在有明确体积/加载时机收益时才引入，过度拆分会导致请求瀑布流。

---

## 反模式提示

- **把 server state 塞进全局 client store**：将接口数据手动同步进 Redux / Zustand / Pinia 等全局 store，制造双重来源（store vs 服务端），引发失效同步、缓存竞争和大量样板代码。server state 应由专用缓存层（TanStack Query / SWR）管理。
- **在客户端裁决鉴权与权限**：仅靠前端隐藏按钮/路由来控制访问，不在服务端验证。任何客户端权限判断只是 UI 提示，不构成安全边界；敏感操作必须在服务端鉴权。
- **乐观更新无回滚路径**：只做乐观更新，不保留旧快照，接口失败后 UI 进入不一致状态且无法自动恢复。每个乐观更新必须配套 `onError` 回滚和 `onSettled` invalidation。
- **把敏感或高频状态序列化进 URL**：将 token、用户 ID、内部业务标识、或高频变化的输入中间态放入 URL，导致信息泄漏、URL 膨胀或浏览器历史记录污染。
- **过度分割导致加载瀑布流**：为追求极致 bundle 体积将同屏所需资源过度拆分，导致多个 Suspense 串行挂起，实际 TTI 反而变长。应以路由为主要分割粒度，按实际加载时序合并 Suspense 边界。

通用推理类反模式见 `anti-patterns.md`；鉴权边界见 `patterns-fullstack.md`。
