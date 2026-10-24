# core 层模板

可直接落地的最小实现。按项目改名、改路径，保留语义。

## dom.js

```js
/** 找不到就抛错：改名的 id 立刻按名字报错，而不是之后报 "cannot read properties of null"。
 * @template {Element} [T=HTMLElement] @param {string} selector @param {ParentNode} [root] @returns {T} */
export function $(selector, root = document) {
  const el = root.querySelector(selector);
  if (!el) throw new Error(`页面里找不到元素：${selector}`);
  return /** @type {T} */ (el);
}

/** @template {Element} [T=HTMLElement] @param {string} selector @param {ParentNode} [root] @returns {T[]} */
export const $$ = (selector, root = document) => /** @type {T[]} */ ([...root.querySelectorAll(selector)]);

/** 委托：root 上监听，来自匹配 selector 的后代时回调（传入匹配到的元素）。
 * @template {keyof HTMLElementEventMap} K
 * @param {Element | Document} root @param {K} type @param {string} selector
 * @param {(event: HTMLElementEventMap[K], target: HTMLElement) => unknown} handler */
export function on(root, type, selector, handler) {
  root.addEventListener(type, (event) => {
    const t = event.target instanceof Element ? event.target.closest(selector) : null;
    if (t instanceof HTMLElement && root.contains(t)) handler(/** @type {HTMLElementEventMap[K]} */ (event), t);
  });
}

const ESC = /** @type {Record<string, string>} */ ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" });
export const esc = (/** @type {unknown} */ s) => String(s ?? "").replace(/[&<>"']/g, (c) => ESC[c]);

/** html`` 或 raw() 产出的标记：原样插入，不再转义。 */
export class Html { constructor(/** @type {string} */ value) { this.value = value; } }
/** 写死的可信标记（如图标 SVG）。绝不传接口或用户数据。 */
export const raw = (/** @type {string} */ m) => new Html(m);

/** @param {unknown} v @returns {string} */
const part = (v) => (v instanceof Html ? v.value : Array.isArray(v) ? v.map(part).join("")
  : v === null || v === undefined || v === false ? "" : esc(v));

/** 插值默认转义；数组拼接；null / undefined / false 渲染为空。 @param {TemplateStringsArray} s @param {...unknown} v */
export function html(s, ...v) { let out = s[0]; v.forEach((x, i) => (out += part(x) + s[i + 1])); return new Html(out); }

export const render = (/** @type {Element} */ el, /** @type {Html} */ c) => void (el.innerHTML = c.value);
export const append = (/** @type {Element} */ el, /** @type {Html} */ c) => el.insertAdjacentHTML("beforeend", c.value);
```

陷阱（写进项目文档）：

- `${n && html`…`}` 在 n 为 0 时渲染出 "0"；数字条件一律用三元。
- `false` 渲染为空，布尔属性值写 `aria-pressed="${String(on)}"`。
- 模板放进属性时必须带引号：`title="${x}"`。
- `style="background:${hex}"` 只放已校验的值（后端校验过的色值）。

## api.js

```js
import { store } from "./store.js";

export class ApiError extends Error {
  /** @param {number} status @param {{ error?: { code?: string, message?: string, details?: any } }} body */
  constructor(status, body) { super(body.error?.message || `HTTP ${status}`); this.status = status; this.code = body.error?.code; this.details = body.error?.details; }
}

/** 统一失败处理；会话失效跳登录页。 @param {Response} res */
export async function failure(res) {
  const body = await res.json().catch(() => ({}));
  if (res.status === 401 && body.error?.code === "login_required") location.href = "/login.html";
  return new ApiError(res.status, body);
}

/** @param {string} path @param {{ method?: string, body?: object | FormData }} [opts] @returns {Promise<any>} */
export async function api(path, opts = {}) {
  const json = opts.body !== undefined && !(opts.body instanceof FormData);
  const res = await fetch(path, { method: opts.method, headers: json ? { "Content-Type": "application/json" } : undefined,
    body: json ? JSON.stringify(opts.body) : /** @type {FormData | undefined} */ (opts.body) });
  if (!res.ok) throw await failure(res);
  return res.json();
}
```

错误码与 `{ error: { code, message, details } }` 外壳按后端约定改。

## store.js

```js
/** @import { Config, Project, Asset } from "./types.js" */

/** 启动时设置 config，打开主对象时设置 project；各模块 bind 之后才读，所以按已设置声明类型。
 * @type {{ config: Config, project: Project }} */
export const store = /** @type {any} */ ({ config: null, project: null });

/** @typedef {object} Events
 * @property {{ from: string | null, to: string }} switch
 * @property {null} project
 * @property {null} refs
 * @property {{ newId?: string }} assets
 * @property {{ running: boolean, finished: boolean }} job */

const bus = new EventTarget();
/** @template {keyof Events} K @param {K} name @param {(d: Events[K]) => void} fn */
export const subscribe = (name, fn) => bus.addEventListener(name, (e) => fn(/** @type {CustomEvent<Events[K]>} */ (e).detail));
/** @template {keyof Events} K @param {K} name @param {Events[K]} detail */
export const emit = (name, detail) => bus.dispatchEvent(new CustomEvent(name, { detail }));

/** 接口返回整个主对象时调用；part 指明只重绘哪块。 @param {Project} p @param {"project" | "refs" | "assets"} [part] */
export function setProject(p, part = "project") { store.project = p; emit(part, part === "assets" ? {} : null); }

/** @param {Asset} a */
export function upsertAsset(a) {
  const list = store.project.assets, i = list.findIndex((x) => x.id === a.id);
  if (i >= 0) list[i] = a; else list.push(a);
  emit("assets", i >= 0 ? {} : { newId: a.id });
}
```

注意：异步请求返回时主对象可能已切换；对"晚到的旧结果"敏感的地方，在回调里比较请求发出时与当前的 id，不一致就丢弃。

## ui.js（节选）

- `toast(msg)`：用 `popover="manual"` 的元素，`showPopover()` 进入顶层，模态对话框打开时也能看见。
- `download(url, el)`：fetch 整个文件后保存；期间按钮 `data-busy` 并把 `[data-label]` 子元素文字换成"打包中…"（保留图标），失败 toast。
- `bindPageUi()`：document 级委托 `a[data-download]`；`details[data-menu]` 点外部或 Esc 关闭。

## types.js

```js
// 只放类型；尽量从后端源码引用，前后端不会各写一份。
/** @typedef {import("../../../src/store/types.js").Project} Project */
/** @typedef {import("../../../src/routes/config.js").ConfigView} Config */
export {};
```

后端没有导出响应类型时，把响应组装抽成函数并导出 `ReturnType` 类型，不改变返回内容。

## tsconfig.web.json

```json
{
  "compilerOptions": {
    "target": "ES2023", "module": "ESNext", "moduleResolution": "Bundler",
    "lib": ["ES2023", "DOM", "DOM.Iterable"],
    "allowJs": true, "checkJs": true, "strict": true, "noEmit": true, "skipLibCheck": true,
    "noUnusedLocals": true, "noUnusedParameters": true
  },
  "include": ["public/js"]
}
```

`typecheck` 脚本串上：`tsc && tsc -p tsconfig.web.json`。登录页这类在鉴权之前加载的脚本不要 import 其他模块（服务端通常只放行这一个文件），并加测试断言它引用的资源未登录也能取到。
