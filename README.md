# xq-click-captcha

文字点选验证码的**前端组件**，TypeScript + SCSS 实现，UI 基于 **Bootstrap 5.3**。用户按提示顺序点击图片中的汉字，组件采集点击坐标与耗时并上报检验。

配套后端：[`xqkeji/xq-app-admin`](https://github.com/xqkeji/xq-app-admin)，由它提供两个接口：`/admin/user/captchadata`（GET，生成验证码数据）与 `/admin/user/captchacheck`（POST，检验）。本包只负责交互与上报，宿主按这两个地址配置即可；滑动拼图版是独立项目 `xq-slide-captcha`，两者互不依赖。

组件弹窗使用 Bootstrap 的 `modal` / `modal-dialog` / `modal-content` / `btn` / `btn-close` / `spinner-border` / `alert` 等组件，显隐、遮罩、ESC 关闭、焦点管理与 body 滚动锁定全部交给 Bootstrap 的 Modal JS；主题走 Bootstrap 的 `data-bs-theme` 属性与 `--bs-*` 设计令牌，不另建色板。

## 环境要求

- Node.js `^20.19.0 || >=22.12.0`（Vite 8 的 engines 要求；本仓库在 v22.22.2 上验证）
- 组件库构建：unbuild + esbuild + sass + clean-css-cli；演示页构建：Vite 8
- 浏览器：支持 `fetch`、`AbortController`、`matchMedia`、`aspect-ratio` 的现代浏览器
- Bootstrap 5.3 CSS：宿主页面先加载它，再加载本包的 `dist/css/`（组件样式只补充布局规则）

## 快速开始

```bash
npm install
npm run build      # 组件库产物 → dist/
npm run html       # 演示页产物 → html/
npm run dev        # 开发服务器，演示页在 http://localhost:5173/
npm run typecheck  # tsc --noEmit
```

| 命令 | 说明 |
| --- | --- |
| `npm run build` | 构建可发布组件库：`dist/index.mjs`、`index.cjs`、`index.d.ts`，`dist/js/` 下的浏览器 JS（含压缩版），`dist/css/` 下的样式（含压缩版） |
| `npm run html` | 只构建演示页到 `html/`，可被浏览器直接打开 |
| `npm run dev` | 启动 Vite 开发服务器 |
| `npm run typecheck` | 只做类型检查，不产出文件 |
| `npm run preview` | 预览 `html/` 里的演示页 |

两种构建互不覆盖：`build` 只写 `dist/`，`html` 只写 `html/`。`dist/` 是唯一发布目录（`files: ["dist"]`），不含演示页、演示后端或第三方素材；`html/` 不参与发布。

## 安装与引入

```bash
npm i xq-click-captcha
```

```ts
import { ClickCaptcha } from 'xq-click-captcha';
import 'bootstrap/dist/css/bootstrap.min.css';
import 'xq-click-captcha/dist/css/xq-click-captcha.min.css';
```

`bootstrap` 是本包的运行时依赖：模块构建（ESM/CJS）把它列为 external，由宿主打包器自行解析；浏览器 IIFE 构建会把所需 Modal 逻辑内联进去。`dist/css/` 只补充 Bootstrap 未覆盖的布局规则，两种引入方式都请**先加载 Bootstrap 5.3 的 CSS**，再加载组件 CSS。

不用打包器时，把 `dist/css/` 与 `dist/js/` 拷到站点静态目录，并**先引入 Bootstrap 5.3 的 CSS**（IIFE 脚本已内联 Modal JS）；页面级实例自动挂到 `window.xqClickCaptcha`。

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <link rel="stylesheet" href="/static/bootstrap/css/bootstrap.min.css" />
  <link rel="stylesheet" href="/static/xq-click-captcha/css/xq-click-captcha.min.css" />
</head>
<body>
  <form action="/login" method="post"><!-- 不要加 novalidate -->
    <input name="username" />
    <input name="password" type="password" />
    <!-- 触发元素：点击后弹出 Bootstrap 模态框 -->
    <div class="xq-captcha"></div>
    <!-- 守卫输入框：未通过验证时由浏览器原生拦截提交 -->
    <input type="text" name="xq_captcha_token" class="xq-captcha-guard visually-hidden"
           tabindex="-1" autocomplete="off" data-xq-guard-text="请先完成人机验证" />
    <button type="submit">登录</button>
  </form>

  <script src="/static/xq-click-captcha/js/xq-click-captcha.min.js"></script>
  <script>
    xqClickCaptcha
      .init({
        dataUrl: '/admin/user/captchadata',
        checkUrl: '/admin/user/captchacheck',
        inputName: 'xq_captcha_token'
      })
      .onSuccess(function (token) {
        console.log('验证通过，令牌：', token);
      })
      .onFail(function (message) {
        console.warn('验证失败：', message);
      });
  </script>
</body>
</html>
```

路径按实际静态目录调整。脚本方式下**不用设置 `transport`**：全局实例的默认传输层就是内置 `FetchTransport`，会请求上面的 `dataUrl` / `checkUrl`（默认即 xq-app-admin 的两个地址）。要自定义传输层请用模块方式传入。验证通过后组件会写入令牌字段，由后端二次校验；令牌字段支持「宿主页静态守卫」和「组件动态插入」两种模式，见下方「表单提交守卫」。要自定义实例可用模块方式的 `installGlobal()`。

## 集成到宿主页面

```html
<form action="/login" method="post">
  <input name="username" />
  <input name="password" type="password" />
  <!-- 触发元素：统一使用 .xq-captcha，可匹配多个 -->
  <div class="xq-captcha"></div>
  <!-- 可选：静态守卫输入框，启用 HTML5 原生拦截（推荐） -->
  <input type="text" name="xq_captcha_token" class="xq-captcha-guard visually-hidden"
         tabindex="-1" autocomplete="off" data-xq-guard-text="请先完成人机验证" />
  <button type="submit">登录</button>
</form>
```

### 表单提交守卫

组件本身不拦截表单提交。若希望「验证没通过时表单提交不了、并给出原生提示」，在表单里预置上面的守卫输入框即可——它用 Bootstrap `visually-hidden` 隐藏但可聚焦，组件通过 `setCustomValidity` 控制它：未通过时带上 `data-xq-guard-text` 的提示让浏览器拦截提交，通过后清空提示并写入令牌。要点：

- **表单不要加 `novalidate`**，否则原生验证失效。
- 守卫输入框按 `inputName`（默认 `xq_captcha_token`）在触发器所属表单内查找，多实例各自在自己的表单里不会串写。
- `backend_only` 模式通过后无令牌，守卫仍会放行，但不写值；安全仍然靠后端会话校验兜底。
- 若页面不放守卫输入框，组件回退到旧行为：通过时动态插入 `type=hidden` 字段，不提供原生拦截——此时请自行在 submit 里用 `captcha.result()` 判断，并由后端二次校验兜底。

```ts
import { ClickCaptcha } from 'xq-click-captcha';
import 'xq-click-captcha/dist/css/xq-click-captcha.min.css';

const captcha = new ClickCaptcha({
  dataUrl: '/admin/user/captchadata',
  checkUrl: '/admin/user/captchacheck',
  inputName: 'xq_captcha_token',
  theme: 'auto',
});

captcha
  .onSuccess((token) => { /* dual 模式拿到一次性令牌，已自动写入隐藏输入框 */ })
  .onFail((message) => { /* 展示或上报失败原因 */ })
  .init();
```

`init()` 可重复调用：会先解绑旧监听再重新绑定，不会叠加。单页应用卸载组件时调用 `dispose()`，释放监听、计时器、Bootstrap Modal 实例与弹窗 DOM。

`src/captcha/index.ts` 另外导出一个页面级默认实例 `clickCaptcha`（弹窗 DOM 是单例，一个页面共用它即可），以及 `installGlobal()`——显式挂到 `window.xqClickCaptcha`，供不用模块打包器的宿主页面直接取用。

## 配置项

默认值唯一真源是 [`src/captcha/core/options.ts`](src/captcha/core/options.ts) 的 `DEFAULT_OPTIONS`，`init()` / 构造函数传入的对象浅覆盖同名键（`copyright` 单独深合并一层）。

| 键 | 默认 | 说明 |
| --- | --- | --- |
| `handleDom` | `.xq-captcha` | 触发元素选择器，可匹配多个 |
| `dataUrl` | `/admin/user/captchadata` | 生成验证码数据的接口（GET，xq-app-admin） |
| `checkUrl` | `/admin/user/captchacheck` | 检验验证码的接口（POST，xq-app-admin） |
| `placeholder` | `点击按钮进行验证` | 触发元素初始文案 |
| `hintText` | `请依次点击指定字符` | 提示条兜底文案（后端下发 `hint` 时优先用后端的） |
| `successText` | `✓ 验证成功` | 通过后触发元素文案 |
| `failText` | `验证失败，请重试` | 兜底失败文案（后端有 message 时优先用后端的） |
| `showClose` | `true` | 是否显示 Bootstrap `btn-close` 关闭按钮 |
| `showRefresh` | `true` | 是否显示 Bootstrap `btn` 刷新按钮 |
| `showRipple` | `true` | 是否显示触发按钮上的呼吸点动画 |
| `theme` | `auto` | `auto` / `light` / `dark`，写入 `data-bs-theme` |
| `inputName` | `xq_captcha_token` | 令牌字段名，守卫输入框与动态输入框都用它；**留空则不写入** |
| `autoInsertInput` | `true` | 回退模式下是否自动插入 `type=hidden` 输入框（守卫模式不受此影响） |
| `guardText` | `请先完成人机验证` | 守卫输入框未通过时的提示；页面元素的 `data-xq-guard-text` 优先 |
| `timeout` | `10000` | 请求超时毫秒数 |
| `copyright` | `{ text: '© xqkeji.cn', url: 'https://xqkeji.cn/', title: 'xqkeji.cn' }` | 弹窗底部版权行 |
| `transport` | 内置 `FetchTransport` | 自定义传输层，演示页注入本地模拟后端时使用 |

`transport` 是一个单方法接口，实现它就能接管全部请求（例如走宿主自己的 axios 实例）：

```ts
interface Transport {
  /** 2xx 时 resolve 响应文本，其余情况 reject HttpError。 */
  request(method: 'GET' | 'POST', url: string, data?: Record<string, string>): Promise<string>;
}
```

**默认不用管 `transport`**：`init()` 不传它时，`createTransport()` 会创建一个 `FetchTransport(timeout)`，直接请求上面两个默认地址。要显式指定或调整超时：

```ts
import { ClickCaptcha, FetchTransport } from 'xq-click-captcha';

const captcha = new ClickCaptcha();
captcha.init({
  dataUrl: '/admin/user/captchadata',
  checkUrl: '/admin/user/captchacheck',
  transport: new FetchTransport(10000),
});
```

演示页把开关放在 [`src/main.ts`](src/main.ts) 顶部，一处同时切换组件请求和表单二次验证：

```ts
// src/main.ts
const USE_REAL_BACKEND = true; // 默认 false = 浏览器内模拟后端
const backend: Transport = USE_REAL_BACKEND
  ? new FetchTransport(10000)
  : new MockBackend();
```

内置 `FetchTransport` 用 `application/x-www-form-urlencoded` 提交、`credentials: 'same-origin'` 带上同源 Cookie（答案存在后端会话里），超时用 `AbortController` 中断。`HttpError.status` 沿用上游语义：`-1` 超时、`0` 网络错误、其余为 HTTP 状态码。

## 实例 API

| 方法 | 说明 |
| --- | --- |
| `init(options?)` | 应用配置、构建弹窗、绑定触发元素。返回 `this` |
| `show()` / `hide()` | 打开 / 关闭 Bootstrap 模态框（打开时自动拉取一轮新图） |
| `refresh()` | 换一轮新的点选验证码 |
| `reset()` | 回到未验证状态，清掉令牌与隐藏输入框。返回 `this` |
| `result()` | 本轮是否已通过 |
| `getToken()` | 一次性令牌，未通过或 `backend_only` 时为 `null` |
| `getCaptchaType()` | 恒为 `click`（保留方法以兼容统一调用口径） |
| `getClickPoints()` | 本轮采集到的点击点（图片像素坐标 + 耗时） |
| `setTheme(theme)` | 运行时切换主题。返回 `this` |
| `dispose()` | 释放监听、计时器、Bootstrap Modal 实例与弹窗 DOM |

## 回调

全部返回 `this`，可链式调用。

| 方法 | 签名 | 触发时机 |
| --- | --- | --- |
| `onSuccess` | `(token: string \| null) => void` | 检验通过。dual 首次通过时 token 非空 |
| `onFail` | `(message: string) => void` | 检验失败或请求出错 |
| `onShow` / `onHide` | `() => void` | 弹窗显示 / 隐藏 |
| `onClose` | `() => void` | 用户主动关闭（`btn-close`、ESC 或点击遮罩） |
| `onRefresh` | `(type, switched) => void` | 换图完成。`type` 恒为 `click`，`switched` 恒为 `false`（签名保留以兼容统一口径） |
| `onReset` | `() => void` | 复位完成 |

## 接口契约

默认即 `xq-app-admin` 提供的 `dataUrl = /admin/user/captchadata` 与 `checkUrl = /admin/user/captchacheck`；若宿主把它们挂在别的路径下，用配置项覆盖即可。

### GET `dataUrl`

查询参数：`t`（时间戳）、`_s`（随机串）用于破坏缓存。

响应体必须是 JSON，且 `success` 为 boolean、`type` 为 `click`/`null`、`image_base64` 为 string，否则抛「验证码数据格式不正确」。`success: false` 时抛 `message`。

| 键 | 说明 |
| --- | --- |
| `success` | 后端限流时为 `false` |
| `type` | `click`（或 `null`，老后端兼容） |
| `image_base64` | 数据 URI，`bg_width × bg_height` 的点选背景图 |
| `bg_width` / `bg_height` | 画布尺寸，前端据此设置 canvas 的 `width`/`height` 属性 |
| `char_count` | 需要点击的字符数，缺省按 4 |
| `hint` | 提示文字，如 `请依次点击：甲 → 乙 → 丙`。**字符坐标即答案，后端从不下发** |

### POST `checkUrl`

| 场景 | 表单字段 |
| --- | --- |
| 点选检验 | `click_points`（JSON 数组） |
| 二次验证 | `token`（由宿主表单提交，不经组件） |

点击点为 `{x, y, t}`：坐标是**图片像素**，`t` 为相对首次点击的毫秒数（首点为 0），后端据此拒绝机器式瞬时点击。CSS 像素到图片像素的换算由组件按画布实际显示宽度完成，宿主无需关心缩放与 DPR。点选检验由 `/admin/user/captchacheck` 完成。

响应规范化为 `{success, token, message}`。除标准形式外还兼容老后端：`code === 200`、`status === 'ok' | 'success'`，以及纯文本 `ok` / `"ok"` / `true` / `1`。`message` 缺失时依次退回 `error` 字段、`failText`。

### 验证模式

- **dual**：首次点选通过后后端签发一次性令牌，组件写入隐藏输入框；宿主表单提交时带 `token` 二次校验，**用过即毁**。
- **backend_only**：不签发令牌，验证状态存后端会话；宿主表单提交不带令牌，凭会话已验证状态**幂等放行**。

### 宿主表单复位契约

宿主表单提交失败时派发 `document` 级事件 `xq.form.error`，组件复位到未验证状态并换一轮新图。仅当表单内含本组件时才响应。触发元素统一为 `.xq-captcha`；弹窗内部选择器见下表。

## DOM 与样式契约

选择器集中在 [`src/captcha/core/selectors.ts`](src/captcha/core/selectors.ts)。除了公开的 `.xq-captcha`，弹窗内部一律使用 `xq-captcha-*` 命名空间。

| 用途 | 选择器 / 标记 |
| --- | --- |
| 触发元素（公开，唯一约定） | `.xq-captcha` |
| Bootstrap 模态框 | `#xq-captcha-modal`（`.modal.fade.xq-captcha-modal`） |
| 模态内容盒（抖动作用处） | `.xq-captcha-modal .modal-content` |
| 关闭按钮 | Bootstrap `.btn-close` + `data-bs-dismiss="modal"` |
| 图片画布 | `.xq-captcha-image` |
| 标记层 / 序号徽标 / 错误徽标 | `.xq-captcha-marker-layer` / `.xq-captcha-marker` / `.xq-captcha-marker--error` |
| 加载遮罩 | `.xq-captcha-loading`（内含 Bootstrap `.spinner-border`） |
| 提示条 | `.xq-captcha-hint`（Bootstrap `.alert.alert-primary`） |
| 错误 / 成功反馈条 | `.xq-captcha-message--error` / `.xq-captcha-message--ok`（Bootstrap `.alert`） |
| 刷新按钮 | `.xq-captcha-refresh`（Bootstrap `.btn.btn-sm.btn-outline-secondary`） |
| 点击扩散波纹 | `.xq-captcha-tap-wave` |
| 触发器呼吸点 | `.xq-captcha.xq-captcha-pulse` |

弹窗结构为 Bootstrap 标准三层：`.modal > .modal-dialog.modal-dialog-centered > .modal-content > .modal-header / .modal-body / .modal-footer`。遮罩由 Modal JS 动态插入的 `.modal-backdrop` 提供，组件不再自建遮罩或接管 `body.style.overflow`。

画布 `.xq-captcha-image` 以 CSS 宽度 100% / 高度 `auto` 显示，由 `width` / `height` 属性的固有比例推出尺寸；后端改画布尺寸时前端样式无需跟着改。标记层与画布同盒，徽标用**百分比**定位（`left/top = 坐标 ÷ 图片尺寸`），因此天然跟随缩放，不依赖画布像素坐标系。切换图片时画布用 `visibility` 隐藏而非 `hidden`，舞台高度不会塌，加载遮罩尺寸始终一致。

主题由 [`src/captcha/ui/theme.ts`](src/captcha/ui/theme.ts) 写入 `<html data-bs-theme>`：显式 `light` / `dark` 直接落盘；`auto` 用 `matchMedia('(prefers-color-scheme: dark)')` 解析成 `light` / `dark` 后再写入，并监听系统偏好变化。Bootstrap 内核只识别 `light` / `dark`，所以 `auto` 在 JS 层解析，组件不复制任何暗色令牌。所有颜色取自 `--bs-*` 设计令牌。

## 交互与无障碍

- **点选只用 `click`**：不监听 `touchstart`，移动端一次物理点击本就只触发一回，无需去重防抖。
- **失败反馈**：点错时把对应序号徽标切换为错误样式与 `!` 标记（DOM 实现，不绘制叉号），模态内容盒抖动并展示失败文案，短暂停留后自动换一张新图。
- **工具按钮**：关闭是原生 `<button class="btn-close">`，刷新是原生 `<button class="btn">`，键盘可用性由浏览器保证。
- **弹窗语义**：Bootstrap Modal 负责 `role="dialog"` / `aria-modal` / 焦点陷阱 / ESC 关闭；错误条 `role="alert"`、成功条与提示条 `role="status"`。
- **无 innerHTML**：弹窗全部用 DOM API 构建，文案走 `textContent`，后端下发的 `hint` / `message` 不会被当 HTML 解析。

## 演示页

`npm run html` 把演示页构建到 `html/index.html`，**不需要真实后端**：

- [`src/demo/mock-backend.ts`](src/demo/mock-backend.ts) 实现 `Transport`，在浏览器内按后端同一口径模拟生成与检验：点选容差 25 像素、120ms 网络延迟；dual 首次通过签发一次性令牌并立即作废答案，令牌用过即毁；`backend_only` 不签发令牌，凭会话已验证状态幂等放行。
- [`src/demo/mock-images.ts`](src/demo/mock-images.ts) 用 canvas 程序化生成渐变背景与汉字点选图，**不引入任何第三方素材**。

演示页的切换开关在 [`src/main.ts`](src/main.ts) 顶部：`const USE_REAL_BACKEND = false;`。改成 `true` 后，`transport` 由 `MockBackend` 换成 `FetchTransport`，组件请求与「二次验证 / 表单提交」一起走 `/admin/user/captchadata` 与 `/admin/user/captchacheck`；宿主若改了挂载路径，用 `dataUrl` / `checkUrl` 覆盖。

演示页只从 Bootstrap 引入全量 CSS 与 `collapse`（导航折叠），Modal 由组件自身导入，避免重复注册 `data-bs-dismiss` 委托监听。

页面控件：模式（dual/backend_only）、主题（auto/light/dark）两个下拉，重新初始化 / 刷新 / 重置 / 二次验证四个按钮，一个模拟登录表单，以及实时事件日志与状态 JSON 面板。页面底部「接入方式」同时给出模块引入与直接引入构建产物（`<link>` + `<script>`）两种示例。

> 演示后端只为跑通交互，**不是安全实现**：答案就在浏览器内存里。生产环境必须接真实后端。

## 目录结构

```
src/
├── captcha/                   组件本体（按职责拆分）
│   ├── click-captcha.ts       门面：编排各部件、对外 API 与回调
│   ├── index.ts               公共导出 + 页面级默认实例 + installGlobal
│   ├── core/                  options / types / selectors / state / dom / http / data-client / callbacks
│   ├── render/                canvas（图片画布与坐标换算）/ image-loader
│   ├── click/                 click-handler / marker-renderer（DOM 徽标）/ ripple
│   ├── ui/                    template（Bootstrap Modal 标记）/ modal（Modal JS 封装）
│   │                          trigger / theme / loading / message / hint / hidden-input
│   └── verify/                submitter / response（响应规范化与错误文案）
├── ts/index.ts                组件库入口（unbuild + esbuild 共用，挂全局实例）
├── demo/                      mock-backend / mock-images（仅演示页）
├── main.ts                    演示页入口
├── index.html                 演示页 HTML
├── partials/                  公共片段（xq-include）
├── public/                    xq-cp-dep 自动从 node_modules 拷贝的依赖（忽略提交）
├── vite.config.mjs            演示页构建配置（outDir ../html/）
└── styles/                    SCSS
    ├── index.scss             组件样式入口（不含演示页样式）
    ├── _trigger.scss          .xq-captcha 触发按钮与呼吸点
    ├── _modal.scss            模态框尺寸、footer 与失败抖动
    ├── _stage.scss            画布 / 徽标 / 加载 / 提示 / 反馈 / 波纹
    └── demo.scss              演示页布局
build.config.ts                unbuild 配置（ESM / CJS / d.ts，Bootstrap 列为 external）
dist/                          组件库产物：index.mjs / index.cjs / index.d.ts
                               js/xq-click-captcha[.min].js、css/xq-click-captcha[.min].css
html/                          演示页产物（npm run html，不发布）
```

## 共享核心 xq-captcha-core

请求传输、响应解析、回调、状态、选项合并与门面编排（含 `BaseCaptcha`）已抽到独立包 [`xq-captcha-core`](https://github.com/xqkeji/xq-captcha-core)，由 `xq-click-captcha` 与 `xq-slide-captcha` 共同依赖，避免两处重复维护。本包只保留点选特有的交互、标记、画布与弹窗 DOM。公开 API 与后端协议不变。

- **发布顺序**：先发 `xq-captcha-core`，再发本包；三者都用 OIDC Trusted Publishing，打 `v*` tag 触发。
- **本地开发**：`npm install` 后若 core 用 `file:` 链接，Vite 的 `xq-cp-dep` 插件无法对 symlink 建链（EPERM）。本地验证时把 `node_modules/xq-captcha-core` 换成真实目录拷贝即可；正式发布走 npm 版本依赖，无此问题。
- **回归测试**：core 自带 `npm test`（node:test，覆盖响应解析、选项合并、transport、数据校验，14 项）。

同页同时使用点选包与滑动包时，两个弹窗各有独立 id（`#xq-captcha-modal` / `#xq-slide-captcha-modal`），弹窗内查询限定在各自作用域，令牌输入框限定在各自表单，`data-bs-theme` 只写自己的弹窗，互不干扰。

## 构建说明

`npm run build` 依次执行 unbuild（ESM/CJS/d.ts，Bootstrap 与 `bootstrap/js/dist/modal` 列为 external）、esbuild（浏览器 IIFE JS，允许内联 Bootstrap Modal 与 xq-captcha-core）、uglify-js（压缩 JS）、sass（组件 CSS，基于 Bootstrap 设计令牌、只补充布局规则）、clean-css（压缩 CSS）与 xq-banner（版权头）。`npm run html` 单独用 Vite 构建演示页到 `html/`，两者互不覆盖。

ESM/CJS 使用 `import Modal from 'bootstrap/js/dist/modal'`，需要宿主解析 `bootstrap` 依赖（安装本包时会自动安装）；IIFE 产物已把 Modal 逻辑打进单个 JS 文件，`<script>` 引入即可用；样式仍按上文先加载 Bootstrap CSS。

新增演示页面：在 `src/` 下建 HTML，再到 `src/vite.config.mjs` 的 `rollupOptions.input` 增加一条入口。公共片段用 `<xq-include file="/partials/header.html"></xq-include>` 复用。`src/public/` 由 `vite-plugin-xq-cp-dep` 自动维护，不要手改。

## 素材与字体

本包仓库**不含任何二进制素材**：没有字体文件、图片文件或图标文件。

- 演示页的背景与点选图由 [`src/demo/mock-images.ts`](src/demo/mock-images.ts) 用 canvas 现场程序化绘制，仅供演示。
- 弹窗按钮使用 Bootstrap 原生外观：关闭按钮是 `.btn-close`，刷新按钮是 `.btn` + 文本箭头 `↻`，不使用任何第三方图标库。
- 生产环境的背景图与点选中文字体由后端模块 [`xqkeji/xq-app-admin`](https://github.com/xqkeji/xq-app-admin) 提供，授权说明见其 README「素材与字体」一节。
- 运行时依赖：`bootstrap`（MIT，Modal 组件与样式）、`xq-captcha-core`（AGPL-3.0-only，共享核心）与演示页入口用到的 `xq-util`（AGPL-3.0-only）；组件自身逻辑只用浏览器原生 API。

## 许可

AGPL-3.0-only，见 [LICENSE](LICENSE)。
