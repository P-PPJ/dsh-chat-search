---
name: publishing-dsh-plugin
description: Use when turning a locally working DeepSeek Harness (DSH) plugin into something other people can actually install — packaging the Cordis bundle, pushing it to GitHub, writing its README, getting it listed in the plugin market (awesome-dsh-plugin), and debugging why the install fails on someone else's machine (missing Git, ECONNRESET, package-name collisions, mirror sources that do not apply).
---

# 发布一个 DSH 插件，并让别人真的装上

一句话：**插件能注册 ≠ 别人能装。** 中间有六道关卡，任何一道没交代清楚，最后都会变成用户嘴里那句「装不上，你这插件是坏的」。

| 关卡 | 失败时的表现 | 实际归属 |
| --- | --- | --- |
| 1 注册 | 装上了，快捷键毫无反应 | 你的代码 |
| 2 打包 | 装上了，但缺 locale / 缺 patch | 你的 `package.json` |
| 3 仓库 | 链接 404、推不上去、release 传不上 | 仓库 |
| 4 README | 对方不知道该往框里填什么 | 文档 |
| 5 目录 | 插件市场里搜不到 | 收录流程 |
| 6 安装链路 | `ERR_PNPM_*`、`ECONNRESET`、找不到 `git` | 对方环境 |

**先把 1、2 做到「本地能装能跑」，再碰 3**。顺序反过来，你会花一整天发一个装上没反应的包。

---

## 关卡 1：做成一个能被注册的 bundle

最小可用结构（`client.js` 是主体，`index.js` 可以只是个空壳 Host 半）：

```
<plugin-dir>/
  package.json
  cordis.patch.yml      ← 让 Harness 把这个包挂进插件树
  client.js             ← Client 半，__ModuleLoader__.load 工厂
  index.js              ← Host 半，可以是 export function apply() {}
  icon.svg
  locale/zh.json  locale/en.json
  README.md
```

`package.json` 里决定成败的几个字段：

```jsonc
{
  "name": "dsh-chat-search",              // 这个名字必须和下面三处字面量一致
  "version": "1.0.0",
  "type": "module",
  "exports": {
    ".": "./index.js",
    "./client": "./client.js",
    "./locale/*.json": "./locale/*.json",
    "./icon": "./icon.svg"
  },
  "files": ["index.js", "client.js", "cordis.patch.yml", "icon.svg", "locale"],
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": {
      "platform": "web",
      "immediately": true,
      "inject": ["@deepseek-ai/dsh-client-ui-conversation"]
    }
  }
}
```

`cordis.patch.yml`：

```yaml
- insert:
    - id: dsh-chat-search            # 与包名一致
      name: dsh-chat-search          # 与包名一致
```

**三个必须一致的字面量**（这里最容易埋雷，装上了却不生效就是因为这个）：

1. `package.json` 的 `name`
2. `cordis.patch.yml` 里的 `id` / `name`
3. `client.js` 里 `window.__ModuleLoader__.load({ id: '<这里>' })`

Client 半的最小骨架：

```js
window.__ModuleLoader__.load({
  id: 'dsh-chat-search',
  factory(require) {
    const React = require('react');
    return {
      inject: ['slots'],
      apply(ctx) {
        const dispose = ctx.slots.inject('shell.overlay', () =>
          ctx.slots.register(
            { name: 'shell.overlay', id: 'dsh-chat-search', order: 60 },
            Component,
          ),
        );
        ctx.effect(() => dispose);
      },
    };
  },
});
```

UI 插件的几条硬约束（踩过就懂）：

- `shell.overlay` 是 **click-through** 层：层本身 `pointer-events:none`，panel 自己开 `pointer-events:auto`，否则整个界面点不动。
- 别引用 `styles`、`host` 这类自由变量，只用 `apply(ctx)` 拿到的 `ctx` 和 `require('react')`；样式用组件内 `<style>`，卸载自动消失。
- 颜色只用主题 token（`--dsw-alias-*` / `--dsw-*`），别写死 `#fff`，否则深浅色主题里会瞎。
- 抢快捷键（如 `Ctrl+F`）要在 **capture 阶段** `addEventListener('keydown', h, true)` 并 `preventDefault()`，冒泡阶段会被浏览器原生查找先吃掉。
- 定位对话内容用 **当前** 的 DOM 契约：`[data-conversation-region="chat"]`、`[data-conversation-scroll]`、`[data-conversation-session]`。旧的 `data-chat-anchor-key` / `data-chat-node-key` / `data-chat-turn` 在 0.2.x 已经不存在了——照老插件抄一定失效。

**本地验证三件套**（缺一不可）：

```bash
node --check client.js && node --check index.js
npm pack --dry-run          # 看 files 白名单真的把运行文件带上了
```

再写个校验脚本，把下列能机检的都机检，别靠眼看：

- `exports` 里每个目标文件存在
- `files` 列出的项目真实存在
- patch 的 `id`/`name` == 包名
- `client.js` 的 `load({id})` == 包名、注入了 `slots`、出现了 `shell.overlay`
- locale 的 `meta.title/description` 不是模板占位文案
- `icon` 能显示且 < 256 KiB

最后**真的装一次**：把包目录 `plugin_manager install_bundle`（target 填绝对路径）进当前 profile，它会被 link 成 junction，改源码即生效；然后 `cordis_inspect_query`（client / Slots / listSubTree，root=`shell.overlay`）确认自己的 `id` 出现在 occupants 里且 `active: true`。

> 装不上时先看**是不是别的依赖在连坐**：profile 里任何一个依赖的下载失败都会让整棵依赖树装不动。救急可以在 profile 的 `.npmrc` 写 `offline=true` 装完再删掉——不要留在那里。

---

## 关卡 2：打包内容自检

```bash
npm pack --silent
tar -tzf dsh-chat-search-1.0.0.tgz | sort
```

要求：`client.js`、`index.js`、`cordis.patch.yml`、`icon.svg`、`locale/*.json`、`README*.md` 一个不少；不要带 `node_modules`、不要带 `.env`。

**零依赖、零 install script** 是最大的加分项：这样用户永远不会撞上 `pnpm` 的 `allowBuilds` 拦截提示，安装永远只是「下载 + 解包」。

---

## 关卡 3：推到 GitHub

没有 `gh` CLI 也能全流程做完，只需要一个 token：

```powershell
# Git Credential Manager 里已有的 token
$c = "protocol=https`nhost=github.com`n`n" | git credential fill
$tok = ($c | Select-String '^password=').ToString().Replace('password=','')
```

然后是老三样：`git init` → commit → `git remote add origin https://github.com/<owner>/<repo>.git` → push。

打 tag 并上传预构建包（**手上有 .tgz 才叫「预构建」，市场只会用这个**）：

```powershell
# 建 release
$rel = curl.exe -sS -X POST -H "Authorization: token $tok" `
  -H 'Content-Type: application/json' -d '{"tag_name":"v1.0.0","name":"v1.0.0"}' `
  "https://api.github.com/repos/<owner>/<repo>/releases"
# 传资产（Content-Type 很关键，否则资产状态会一直 uploaded 失败）
$up = "https://uploads.github.com/repos/<owner>/<repo>/releases/<id>/assets?name=pkg-1.0.0.tgz"
curl.exe -X POST -H "Authorization: token $tok" -H 'Content-Type: application/gzip' `
  --data-binary "@pkg-1.0.0.tgz" $up
```

PowerShell 5.1 的两个坑：

- 发中文 JSON 会 422：body 必须 `[Text.Encoding]::UTF8.GetBytes($json)` 并带 `charset=utf-8`。
- `[IO.File]::ReadAllBytes('相对路径')` 以**进程 cwd** 解析，全部用绝对路径。

仓库还要设置：`dsh-plugin` topic、`license`（市场会显示）、`homepage`。

---

## 关卡 4：README 怎么写

结构照抄这个顺序，缺一节就会有人来问：

1. **一句话定位 + 截图**（截图放最上面，别放文末）
2. **为什么还需要一个**：和现有方案对比表，写清你解决了什么别人没解决的
3. **功能**：分条列，别写形容词
4. **快捷键表**：按键 → 作用
5. **安装**：按「需要什么前置条件」分组，见下
6. **它怎么工作**：一段机制说明，让人判断能不能信
7. **限制**：**必须诚实写**。收录审核打回的第一大原因就是夸大
8. **文件结构 / 开发 / 许可证**

安装段是重灾区。不要只丢一条命令，要分三类并标注前置条件：

```markdown
**不需要 Git 的装法**（推荐）
<release .tgz 直链> / <CDN 直链> / <本地 .tgz 路径>

**需要本机装过 Git 的装法**
https://github.com/<owner>/<repo>     ← pnpm 会调 git ls-remote 去解析

**放进 profile**（本地开发）
```

再配一张「报错 → 真正原因 → 怎么办」的对照表，把用户会撞的每一条错误原文都抄进去。这张表能省掉你未来 80% 的答疑。

中英双语：英文版不是可选项——社区目录要求 `description.en` 必填，维护者也会读英文 README。

模板见 `references/readme-template.md`。

---

## 关卡 5：进插件市场（awesome-dsh-plugin）

市场的规则很硬：**它只列精选目录里的条目，用户贴什么地址都搜不到**。想让别人在市场里搜到，唯一路径是提 PR 进 `awesome-dsh-plugin`。

一个插件一个文件：`data/plugins/<owner>__<repo>.yml`

```yaml
url: https://github.com/<owner>/<repo>
name: <owner>/<repo>
category: ui            # 枚举：agi ui usage theme model identity session memory tools wsl browser vision voice docs skill workflow git notify dev security remote market fun
description:
  en: 'Real description, must end with a period.'
  zh: '真实的中文描述。'
icon: https://raw.githubusercontent.com/<owner>/<repo>/main/assets/icon-180.png
tarball: https://github.com/<owner>/<repo>/releases/download/v1.0.0/pkg-1.0.0.tgz
```

硬性要求（逐条都真的会卡）：

- `package.json` 里必须有 **`dsh.bundle`**；只声明 `dsh.client` 会被拒
- `url` 必须与仓库真实地址一致，`name` 必须是 `owner/repo`
- `description.en` 必填、以句号结尾、内容属实
- `icon` 必须是 **GitHub 托管的位图**（PNG），SVG 不收
- `tarball` 必须是 **GitHub Release 上的 https .tgz**
- 仓库要加 `dsh-plugin` topic，**创建满 1 天**；年龄不够**不要重提**，`regate.yml` 每 6 小时自动重跑，会自己转绿
- `screenshots.json` 放仓库根（1–8 个相对路径），市场详情页按这个顺序展示
- 一个 PR 最多 3 条；首次贡献者的 fork PR 可能要等维护者批准才会跑检查
- 只收真实插件，不收聚合包

PR 机制：fork → 建分支 → 加文件 → 提 PR。用 contents API 而不是本地 clone 也行（见 `references/registry-entry.md`）。

**改仓库名/改用户名后**：目录里存的还是旧地址。要「删旧文件 + 加新文件」放在同一个 PR，GitHub 会把它识别成一个 rename。只加不删，市场里会出现同一个插件两条。

---

## 关卡 6：让别人真的装上

这一关最反直觉：**大多数「装不上」跟你的包无关。**

安装器接受三种输入：**包名** / **Git 地址** / **本地路径或压缩包**。它们的失败原因完全不同：

| 填法 | 真实前置条件 | 典型报错 |
| --- | --- | --- |
| 包名 | 包已发到 npm；「国内镜像源」只对**这一种**生效 | 404 |
| `https://github.com/o/r` | 本机**必须装了 Git**（Windows 不自带） | `ERR_PNPM_GIT_RESOLVE_FAILED` + `'git' 不是内部或外部命令` |
| `.tgz` 直链 | 本机能连上那个主机 | `ECONNRESET` / `ETIMEDOUT` |
| 本地文件路径 | 无（连网络都不用） | — |

三个必须记住的事实：

1. **安装源的「国内镜像」只改写 npm 包名的请求**。填 GitHub 地址或 `.tgz` 直链时它完全不参与——这是应用自己的规则，不是配置问题。
2. **GitHub Release 直链会 302 到 `objects.githubusercontent.com`**，国内常被重置（实测 19.5 秒后 `ECONNRESET`，tar 只有 183 KB）。救急办法是给地址套加速前缀：`<加速域名>/` + 完整 GitHub 地址，规则就这一条。
3. **jsDelivr 可以当免费 CDN 分发 tarball**：`https://cdn.jsdelivr.net/gh/<owner>/<repo>@<ref>/<path>.tgz`。`@main` 有缓存会滞后，钉 commit 更稳。

加速前缀实测（同一条 183 KB tarball，无代理）：`ghfast.top` 4.0s、`gh-proxy.com` 4.2s、`ghproxy.net` 9.6s，`pnpm add` 4.6 秒装出正确版本，校验和与官方一致。

**最终兜底永远是本地文件**：把 `.tgz` 直接传给对方，让他在框里填绝对路径。零网络、1.3 秒装完。写文档时把这条放在显眼处——它救过的人比任何镜像都多。

**验证方式**：不要只看 manifest，真的跑一遍安装：

```powershell
# Harness 自带的 pnpm（不在 PATH 里）
$pnpm = 'D:\DeepSeek Harness\resources\runtime\primary-runtime\dependencies\pnpm\bin\pnpm.mjs'
node $pnpm add '<被测地址>' --reporter=append-only
# 然后核对装上来的版本与文件
```

还要防**包名冲突**：社区里可能已经有同名包。同名会互相覆盖依赖、抢同一个 slot（比如两个插件都劫持 `Ctrl+F`）。发布前先在市场搜一遍自己的包名。

---

## 发布后

- **改用户名/仓库名**：GitHub 对旧 owner 做 301 跳转，旧链接暂时不会立刻坏，**但跳转只在旧名字没人注册时有效**。所以：仓库内所有地址、git remote、目录条目都要换；目录条目走「删旧 + 加新」的 PR。
- **升版本**：改 `version` → 重新 `npm pack` → 新 tag → 传新资产 → 更新目录条目的 `tarball`。旧的 `dist/*.tgz` 别删，已经装过的人的依赖可能还指着它。
- **验证闭环**：每次都从「陌生机器 + 空目录」的角度跑一遍安装命令，而不是在你自己的 profile 里点重试。

## 交付前的检查清单

- [ ] 三处字面量一致（`name` / patch `id+name` / `load({id})`）
- [ ] `node --check` 全过，`npm pack --dry-run` 里运行文件齐全
- [ ] 零依赖、零 install script
- [ ] 本地真实装一次，`shell.overlay` 的 occupants 里能看到自己且 `active`
- [ ] README 有截图、对比表、快捷键表、**按前置条件分组的安装段**、报错对照表、**诚实的限制**
- [ ] Release 上有 `.tgz` 资产（`state=uploaded`）
- [ ] 仓库有 `dsh-plugin` topic、license、homepage
- [ ] `screenshots.json`、`assets/` 位图图标就位
- [ ] 目录条目：`url`/`name`/`tarball`/`icon` 与仓库一致
- [ ] 用**陌生网络**验证过至少两条安装路径（含本地文件兜底）
- [ ] 文档里写清了「哪种填法需要 Git / 需要能连 GitHub」

## 参考

- `references/readme-template.md` — 可直接套用的 README 骨架
- `references/registry-entry.md` — 目录条目字段与 PR 的完整操作
- `references/install-troubleshooting.md` — 报错 → 真因 → 解法，含实测数字
- 想改插件代码本身，用 `cordis-plugin-development` 技能（本技能只管「怎么发出去、别人怎么装」）
