---
name: Loopback 回调 host 归一化修复
overview: 记录 Next.js 把 redirect_uri 取值里的 127.0.0.1 改写成 localhost、导致 loopback 授权请求被拒的原因，以及通过 pnpm 补丁在框架层修复的方案、验证证据与维护约束。
isProject: false
---

# Loopback 回调 host 归一化修复

> 日期：2026-09-20
> 前置方案：[08-轻量SSO票据方案.plan.md](08-轻量SSO票据方案.plan.md)、[09-SSO外部服务接入文档.plan.md](09-SSO外部服务接入文档.plan.md)
> 关联产物：`patches/next.patch`（经 `pnpm.patchedDependencies.next` 生效）

## 一、现象

外部服务（meta-mystia-manager）以 `http://127.0.0.1:{port}/sso/callback` 请求 `GET /api/v1/sso/authorize`，线上稳定返回 `400 {"message":"invalid-object-structure"}`；同一份代码的其他取值为：

| 取值                                   | 结果                                             |
| -------------------------------------- | ------------------------------------------------ |
| `http://127.0.0.1:{port}/sso/callback` | 400 `invalid-object-structure`                   |
| `http://127.1:{port}/sso/callback`     | 303                                              |
| `http://[::1]:{port}/sso/callback`     | 303                                              |
| `https://example.com/callback`         | 400 `invalid-redirect-uri`（正常走到白名单校验） |

两条关键证据：

- 服务器上直接执行仓库源码：`checkSsoRedirectUriFormat('http://127.0.0.1:…')` 返回 `true`，产物里的白名单也确实含 `127.0.0.1`。
- 应用侧日志打印的 `request.nextUrl.searchParams.get('redirect_uri')` 是 `http://localhost:…`，而 curl 打印的请求行里是 `http%3a%2f%2f127.0.0.1%3a…`：**请求值在进入业务代码前就被改了**。

## 二、原因

`next/dist/server/web/next-url.js` 的 `parseURL()` 会对整条 URL 字符串做一次 localhost 归一化：

```js
const REGEX_LOCALHOST_HOSTNAME =
	/(?!^https?:\/\/)(127(?:\.(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}|\[::1\]|localhost)/;
function parseURL(url, base) {
	return new URL(
		String(url).replace(REGEX_LOCALHOST_HOSTNAME, 'localhost'),
		base && String(base).replace(REGEX_LOCALHOST_HOSTNAME, 'localhost')
	);
}
```

要点：

1. 正则本意是归一化 host，却作用在**整串**上；`replace` 没有 `g` 标志，因此**只替换第一个匹配**，谁先出现谁被改。
2. `NextRequest` 构造时固定走 `new NextURL(...)`，所以 `request.nextUrl.searchParams` 拿到的永远是改写后的值。
3. 归一化对象里 `127.0.0.1`（编码后仍是明文点分十进制）、`[::1]`（编码后是 `%5B%3A%3A1%5D`，正则匹配不到）命运不同，这解释了为什么只有 `127.0.0.1` 出问题。
4. 改写结果取决于基准 URL 里的 host（`attachRequestMeta` 用 `${protocol}://${fetchHostname}:${port}${req.url}` 拼 base，`protocol` 取 `x-forwarded-proto`）：
    - base host 是 `localhost` 时，host 自己吃掉唯一一次替换，query 里的 `127.0.0.1` 侥幸保留（`next dev` 默认就是这种情形，所以"dev 能过"）；
    - base host 是 `0.0.0.0`（standalone 生产入口 `server.js` 的默认 `process.env.HOSTNAME || '0.0.0.0'`）时，第一个匹配落在 query 上，`redirect_uri` 被改成 `localhost`，而 `localhost` 不在白名单里 → 400。

`skipMiddlewareUrlNormalize` / `__NEXT_NO_MIDDLEWARE_URL_NORMALIZE` 只影响 middleware 与 `request.url`，不改变 `nextUrl.searchParams`，因此没有配置项可以关掉这个行为。

## 三、结论：修框架行为，不放宽白名单

把 `localhost` 加进 `checkSsoRedirectUriFormat` / `clients.ts` 的回环白名单虽然能让请求通过，但方向是错的：

- 白名单表达的是"该 client 允许用哪些回调地址"，让业务去适配框架的归一化产物，等于把实现细节写进信任边界；
- 改写值会一路外溢：取消授权回跳也会变成 `localhost:{port}`，客户端监听 `127.0.0.1` 时就多出 IPv6/主机名解析的不确定性；
- 真正的缺陷是 Next 把 query/hash 也当成了 host 来归一化。

## 四、实现

`patches/next.patch` 中新增（原有 hunk 未改，仅 `index` 行哈希随 pnpm 重新生成）：

| 文件                                                  | 说明                                                                                                                                                                                                        |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dist/server/web/next-url.js`                         | 把替换限制在第一个 `?`/`#` 之前，host 归一化行为不变                                                                                                                                                        |
| `dist/esm/server/web/next-url.js`                     | 同上（Turbopack 与 ESM 路径）                                                                                                                                                                               |
| `dist/compiled/next-server/app-route.runtime.prod.js` | 同一变换的压缩版。webpack 生产构建的 `route.js` 直接 `require` 这份 runtime bundle（已从 `pnpm build` 产物 `.next/server/app/api/v1/sso/authorize/route.js` 的 require 列表确认），只补 dist 源码在生产无效 |

补丁形态：

```js
function replaceLocalhostHostname(value) {
	const queryIndex = value.search(/[?#]/u);
	return queryIndex === -1
		? value.replace(REGEX_LOCALHOST_HOSTNAME, 'localhost')
		: `${value.slice(0, queryIndex).replace(REGEX_LOCALHOST_HOSTNAME, 'localhost')}${value.slice(queryIndex)}`;
}
```

> 注意：`pnpm patch-commit` 在 pnpm 10 会生成 `patches/next@15.5.25.patch` 并追加版本化条目 `"next@15.5.25"`，与本仓库既有的 `"next": "patches/next.patch"` 冲突（会报 `ERR_PNPM_UNUSED_PATCH`）。正确做法是把生成物内容合并回 `patches/next.patch`、移除多余条目后再 `pnpm install`。

## 五、验证（2026-09-20）

| 检查                                                                                                          | 结果                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm install` / `pnpm install --frozen-lockfile`                                                             | 通过，`pnpm-lock.yaml` 中 next 的 patch hash 同步更新                                                                                                                                           |
| 模块级：`0.0.0.0` / `127.0.0.1` / `izakaya.cc` 三种 base                                                      | `searchParams.get('redirect_uri')` 一律保留 `http://127.0.0.1:9931/sso/callback`                                                                                                                |
| 回归：host 归一化本身                                                                                         | `new NextURL('http://127.0.0.1:3000/x?a=1').host` 仍为 `localhost:3000`                                                                                                                         |
| 生产端到端：`next build` + `next start -H 0.0.0.0 -p 3123`，带 `Host: izakaya.cc`、`X-Forwarded-Proto: https` | `redirect_uri=http://127.0.0.1:…` → `404 feature-disabled`（已通过格式校验，本地未启用 SSO 功能）；对照 `redirect_uri=http://localhost:…` → 仍 `400 invalid-object-structure`，白名单语义未放宽 |

## 六、维护约束与后续

- 补丁文件因此从 4.2 KB 增至约 256 KB：`app-route.runtime.prod.js` 是单行压缩产物，改一个字符整行进出 diff。
- 只覆盖 webpack 生产 runtime；若改用 `next build --turbopack` 或启用 `__NEXT_EXPERIMENTAL_REACT`，需为 `app-route-turbo*.runtime.prod.js` / `app-route-experimental.runtime.prod.js` 补同构 hunk（dev 走 Turbopack 时用的是 dist ESM，已覆盖）。
- 升级 Next 版本时须重建补丁（`patches/next.patch` 本身就是按版本维护）。
- 生效链路：`pnpm install` → `pnpm build` → 重启（compiled runtime 在构建期被打进 `.next`，仅重装依赖不会生效）。
- 建议向上游提 issue/PR：`REGEX_LOCALHOST_HOSTNAME` 应只作用于 origin，不应改写 query/hash 取值。
