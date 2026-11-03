# 进社区目录：awesome-dsh-plugin

市场标签页（dshmarket 等）只认这份精选目录，**在应用里贴任何地址都搜不到**；想让别人在市场里搜到自己的插件，唯一路径是往 `awesome-dsh-plugin` 提 PR。

目录的远端数据源之一：`https://cdn.jsdelivr.net/npm/dsh-plugin-catalog/index.json`（format 1）。条目长这样：

```json
{
  "name": "<包名>",
  "title": "<显示名>",
  "description": { "zh": "...", "en": "..." },
  "homepage": "https://github.com/<owner>/<repo>",
  "repository": "https://github.com/<owner>/<repo>",
  "metadata": { "category": "ui", "keywords": [], "labels": [], "sortKeys": [] },
  "versions": [{ "version": "1.0.0", "publishTimestamp": "...", "source": { "type": "npm" }, "engines": {} }]
}
```

> 现实提醒：目录里绝大多数条目的 `source.type` 是 `npm`，但真机验证过，**带 `tarball` 的条目照样能从市场装成功**。没有 npm 账号不必因此放弃收录。

## 条目文件

路径：`data/plugins/<owner>__<repo>.yml`（**一个插件一个文件**，两个下划线连接 owner 与 repo）

```yaml
url: https://github.com/P-PPJ/dsh-chat-search
name: P-PPJ/dsh-chat-search
category: ui
description:
  en: 'Ctrl+F search inside the open conversation, with live match counting.'
  zh: '在当前对话内用 Ctrl+F 查找关键词，命中数量实时增长。'
icon: https://raw.githubusercontent.com/P-PPJ/dsh-chat-search/main/assets/icon-180.png
tarball: https://github.com/P-PPJ/dsh-chat-search/releases/download/v1.0.1/dsh-chat-search-1.0.1.tgz
```

逐字段的硬规则：

| 字段 | 规则 | 不过会怎样 |
| --- | --- | --- |
| `url` | 必须与仓库真实地址一致（含大小写） | 检查脚本报错 |
| `name` | 必须 `owner/repo` | 同上 |
| `category` | 枚举内取值（见下） | 同上 |
| `description.en` | **必填**、以句号结尾、含 `: ` 要加引号、内容属实 | YAML 解析失败或被维护者打回 |
| `description.zh` | 必填（中文本地化）/等长可信 | |
| `icon` | **GitHub 托管的位图**（PNG），SVG 不收 | 图标不显示 |
| `tarball` | **GitHub Release 上的 https `.tgz`**；`latest/download/` 时文件名不能带版本号，钉 tag 时带版本号正常 | 安装失败或被打回 |

`category` 枚举：

```
agi ui usage theme model identity session memory tools wsl browser vision voice
docs skill workflow git notify dev security remote market fun
```

## 仓库侧的硬性条件

- `package.json` 里必须有 **`dsh.bundle`**（只写 `dsh.client` 会被拒）
- 仓库要有**真实代码**（不接受空壳、不接受聚合包）
- 仓库创建**满 1 天**：`MIN_AGE_DAYS = 1`，`ageDays = (Date.now() - new Date(created_at)) / 86400000`
- 加 topic：`dsh-plugin`
- 可选但强烈建议：`screenshots.json` 放仓库根，列 1–8 个相对路径（市场详情页按这个顺序展示）

```json
["screenshots/panel-zh.png", "screenshots/panel-zh-list.png"]
```

**年龄不够不要重提。** `regate.yml` 每 6 小时按同一句措辞重跑，「days old」判定会自动转绿。重提只会得到一个新的红灯。

## 提 PR 的完整操作（无 gh CLI，纯 API）

```powershell
$env:https_proxy = 'http://127.0.0.1:7890'      # 只有需要时
$c   = "protocol=https`nhost=github.com`n`n" | git credential fill
$tok = ($c | Select-String '^password=').ToString().Replace('password=','')
$H   = @{ Authorization = "token $tok" }

# 1) fork（已 fork 则跳过）
Invoke-RestMethod -Method Post -Headers $H `
  -Uri 'https://api.github.com/repos/awesome-dsh-plugin/awesome-dsh-plugin/forks'

# 2) 在 fork 上建分支（从上游 main 的最新 sha）
$up   = Invoke-RestMethod -Headers $H 'https://api.github.com/repos/awesome-dsh-plugin/awesome-dsh-plugin/commits/main'
$sha  = $up.sha
Invoke-RestMethod -Method Post -Headers $H -ContentType 'application/json' `
  -Uri 'https://api.github.com/repos/<you>/awesome-dsh-plugin/git/refs' `
  -Body (@{ ref = "refs/heads/add-<pkg>"; sha = $sha } | ConvertTo-Json)

# 3) 写文件（middle 路径前先确认父目录存在；对不存在的分支写文件会 404，不是 422）
$yaml   = Get-Content -Raw '.\entry.yml'
$bytes  = [Text.Encoding]::UTF8.GetBytes($yaml)
$b64    = [Convert]::ToBase64String($bytes)
$bodyObj = @{ message = 'Add <pkg>'; content = $b64; branch = 'add-<pkg>' }
$body = [Text.Encoding]::UTF8.GetBytes(($bodyObj | ConvertTo-Json -Compress))
Invoke-RestMethod -Method Put -Headers $H -ContentType 'application/json; charset=utf-8' `
  -Uri 'https://api.github.com/repos/<you>/awesome-dsh-plugin/contents/data/plugins/<owner>__<repo>.yml' -Body $body

# 4) 开 PR
$pr = @{ title = 'Add <owner>/<repo>: <一句话>'; head = "<you>:add-<pkg>"; base = 'main' }
... -Uri 'https://api.github.com/repos/awesome-dsh-plugin/awesome-dsh-plugin/pulls'
```

PowerShell 5.1 的坑：中文 JSON **必须**先转 UTF-8 字节再发，否则 422；`[IO.File]::ReadAllBytes('相对路径')` 按进程 cwd 解析，一律用绝对路径。

## 改 owner / 改仓库名之后

目录里存的是**绝对地址**，改名字不会自动跟随。要发**一个同时含删除与新增的 PR**：

1. 删 `data/plugins/<old-owner>__<repo>.yml`
2. 加 `data/plugins/<new-owner>__<repo>.yml`
3. 新文件里 `url` / `name` / `icon` / `tarball` 全部换新 owner

GitHub 会把这个 PR 识别成 `renamed`（PR 页面标题前缀会显示）。只加不删，市场里同一个插件会出现两条。

> GitHub 对旧 owner 的 301 跳转**只在旧名字没人重新注册时有效**，不要把它当长期方案。

## 提交后

- 首次贡献者的 fork PR 可能要等维护者批准才会跑检查（显示 no checks 属正常）
- 维护者会人读仓库与 README；「描述夸大」是最常见的打回原因
- 合并后目录有同步延迟，市场里搜不到先等一会儿再刷
- 自己的记录：PR 链接、检查状态、合并时间
