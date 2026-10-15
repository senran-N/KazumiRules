# KazumiRules

Kazumi（fork: senran-N/Kazumi）动画规则仓库。规则市场（应用内「规则市场」页）从这里拉取目录与规则文件。

## 目录结构

```
index.json          # 目录清单：规则条目数组
{name}.json         # 规则文件：一条完整规则，文件名与条目 name 一致
```

## index.json 条目格式

```json
{
  "name": "示例站点",
  "version": "1.0.0",
  "useNativePlayer": true,
  "author": "作者",
  "lastUpdate": 1710000000,
  "antiCrawlerEnabled": false
}
```

| 字段 | 说明 |
| --- | --- |
| `name` | 规则名，必须与规则文件内的 `name` 一致（比较忽略大小写），同时是下载文件名 `{name}.json` |
| `version` | 规则版本，应用按此判断是否需要更新 |
| `useNativePlayer` | 固定 `true`，播放地址由应用内置播放器装载 |
| `author` | 作者署名 |
| `lastUpdate` | 最近更新时间的 UNIX 秒级时间戳 |

## 规则文件格式（anime / api 1）

```json
{
  "type": "anime",
  "api": "1",
  "id": "com.example.newsource",
  "name": "示例站点",
  "version": "1.0.0",
  "baseURL": "https://example.com",
  "defaults": {
    "header:User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
  },
  "search": { "...": "搜索阶段，必填" },
  "detail": { "...": "详情阶段，可选" },
  "roads":  { "...": "线路与剧集阶段，必填" },
  "play":   { "...": "播放地址阶段，必填" },
  "verification": { "...": "反爬验证声明，可选" }
}
```

### 顶层字段

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `type` | 是 | 固定 `anime` |
| `api` | 是 | 固定 `"1"`（字符串），规则格式版本 |
| `id` | 是 | 全局唯一标识，推荐反向域名风格（如 `com.example.newsource`） |
| `name` | 是 | 规则名，与 `index.json` 条目一致 |
| `version` | 是 | 语义化版本，更新规则时递增 |
| `baseURL` | 是 | 站点根地址，阶段内相对 URL 以此为基准解析 |
| `defaults` | 否 | 默认变量，`header:xxx` 键会作为请求头随所有阶段请求下发 |
| `verification` | 否 | 反爬验证声明（验证码类型、触发检测等），声明后应用会引导用户完成验证 |

### 阶段（stage）

`search` / `detail` / `roads` / `play` 共用同一结构：

```json
{
  "mode": "xpath",
  "request": {
    "method": "GET",
    "url": "https://example.com/search/@keyword",
    "headers": { "Referer": "https://example.com/" },
    "query": {},
    "bodyType": "none",
    "body": null
  },
  "listPath": "//ul[@class='list']/li",
  "fields": {
    "id":    { "path": "./a/@href" },
    "title": { "path": "./a/@title" },
    "url":   { "path": "./a/@href" }
  },
  "nextPath": "//a[text()='下一页']/@href"
}
```

- `mode`：`xpath`（HTML 选择器）、`api`（JSON 路径）、`embeddedJson`（页面内嵌 JSON）、`webview`（需 WebView 渲染的页面）。
- `request.url` 中的变量：`@keyword` 为搜索关键词；`@url` 表示沿用上一阶段产出的条目地址。
- `listPath`：定位条目列表；`fields` 从每个条目内提取字段。
- 阶段产出契约：
  - `search` 与 `roads` 条目必须含 `id`、`title`、`url`；
  - `roads` 条目额外用 `road` 字段声明所属线路，同一 `road` 值的条目聚合为一条线路；
  - `play` 条目必须含 `url`（可直接装载的媒体地址），可用 `referer` 与 `header:xxx` 附加请求头。
- `id` 必须在全站范围稳定：观看进度（含 WebDAV 同步）以它为键。

### 更新一个规则

1. 修改 `{name}.json`，递增 `version`；
2. 同步更新 `index.json` 中对应条目的 `version` 与 `lastUpdate`。

应用按 `index.json` 版本对比本地已装版本提示更新；下载后按条目 `name` 校验规则文件内的 `name`，不一致会拒绝安装。

## 贡献流程

1. 在 Kazumi 应用内「动画规则 → 导入或新建」编写规则，用「开始测试」走完 搜索 → 线路 → 播放 验证链；
2. 「导出 JSON 文件」得到规则文件，提交 PR：新增/修改 `{name}.json` 并同步 `index.json` 条目；
3. 合并后市场在 5 分钟缓存窗口过后即可看到新版本。
