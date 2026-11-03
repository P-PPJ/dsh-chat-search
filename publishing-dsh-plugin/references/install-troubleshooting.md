# 安装报错对照：报错原文 → 真因 → 解法

**先记住一件事**：安装器接受三类输入——**包名** / **Git 地址** / **本地路径或压缩包**。三类失败原因完全不同，先把用户填的那一行归类，再看表。

```
填的是 "dsh-chat-search"                 → 走 npm registry（安装源镜像只对这一种生效）
填的是 "https://github.com/o/r" 或 "git+" → 走 git ls-remote / git clone（本机必须有 git）
填的是 "https://…/x.tgz"                 → 直接 HTTP 下载（只要求能连上那台主机）
填的是 "D:\path\x.tgz" 或目录            → 纯本地（零网络，永远成功）
```

---

## 一、`git` 相关

### `ERR_PNPM_GIT_RESOLVE_FAILED`

```
[ERR_PNPM_GIT_RESOLVE_FAILED] Failed to resolve git dependency "https://github.com/o/r":
git ls-remote failed: 'git' 不是内部或外部命令，也不是可运行的程序或批处理文件。
```

**真因**：填了 GitHub 仓库地址，这台机器没有 Git。Windows 默认不带 Git，很多人从没装过——这跟网络、代理、墙**完全无关**。

**先验证再解释**：同一台机器 `curl https://api.github.com/repos/o/r` 如果能 200，就证明网络通、问题只在本机。实测案例：报这个错的机器 `curl` 返回 `200` / 3.14 秒。

**解法**（按成本排序）：

1. 改填 `.tgz` 直链或本地文件路径（最快，零安装）
2. `winget install Git.Git`（需要管理员权限，装完重开终端/重启应用）
3. 让插件发到 npm，改填包名

---

## 二、下载相关

### `ECONNRESET` / `ETIMEDOUT` / `无法连接 github.com`

**真因**：GitHub Release 资产会 302 到 `objects.githubusercontent.com`，该域名在大陆经常被重置。实测：一条只有 **183 KB** 的 tarball，无代理 **19.5 秒后连接被重置**；同一时刻带代理 1.5 秒 302 完成。

**解法 A：套加速前缀**（规则只有一条：前缀 + 完整 GitHub 地址）

```
https://ghfast.top/https://github.com/<owner>/<repo>/releases/download/v1.0.0/<pkg>.tgz
```

实测（无代理，同一条 183883 B 文件）：

| 前缀 | 结果 |
| --- | --- |
| `https://ghfast.top/` | 200，4.0s |
| `https://gh-proxy.com/` | 200，4.2s |
| `https://ghproxy.net/` | 200，9.6s |
| `https://hub.gitmirror.com/` | DNS 失败 |
| `https://ghproxy.cc/` | 证书过期 |
| `https://gh.llkk.cc/` | 超时 |

用 `pnpm add https://ghfast.top/https://github.com/...tgz` 实测 4.6 秒装成功，sha256 与本地一致。

**解法 B：jsDelivr 当 CDN**

```
https://cdn.jsdelivr.net/gh/<owner>/<repo>@<ref>/dist/<pkg>.tgz
```

所以仓库里最好真的提交一份 `dist/*.tgz`（`.gitignore` 里要放行 `!dist/*.tgz`）。`@main` 走缓存、快但会滞后；钉 commit 更稳。注意它**不是稳定线路**：同一台机器连测 3 次全部 30 秒超时也出现过，所以别把它写成唯一推荐。

**解法 C：本地文件兜底（永远有效）**

把 `.tgz` 直接发给对方，让他在框里填**绝对路径**。实测 `bare path` 与 `file:path` 两种写法都 1.3 秒装好。这条要写在 README 最显眼的位置。

### `ERR_PNPM_FETCH_404` / `404 Not Found`

**真因**：包名没发到 npm（或拼错），或插件还没被目录收录。注意 `registry.npmmirror.com` 对不存在的包同样返回 404——**404 不等于「镜像没同步」**，先 `npm view <pkg> version` 确认公网上到底有没有。

---

## 三、pnpm 相关

### 卡在 `allowBuilds` / 依赖的安装脚本被拦

```
有依赖的安装脚本被 pnpm 拦下，请在 profile 的 pnpm-workspace.yaml 的 allowBuilds 中放行后重试
```

**真因**：这是**被安装的依赖**带 postinstall 之类的脚本，不是你的插件有问题（前提是你零依赖、零 install script）。请用户在 profile 的 `pnpm-workspace.yaml` 里放行后重试。

### 装一个插件却把整个 profile 卡死

**真因**：pnpm 会重解整棵依赖树，profile 里**任何一个**旧依赖连不上都会让这次安装失败。实测：profile 里一条指向已失效 GitHub Release 的旧依赖，让一次毫不相关的安装直接报 `network`。

**解法**：临时在 profile 的 `.npmrc` 写 `offline=true` 装完即删。**不要留在那里**——它会让后续所有安装都只能吃缓存。

---

## 四、装上了但没反应

按这个顺序查，命中率从高到低：

1. **没刷新页面**。Client 半是懒加载的，装完必须刷新 Web UI。
2. **三处字面量不一致**：`package.json` 的 `name`、`cordis.patch.yml` 的 `id`/`name`、`client.js` 里 `__ModuleLoader__.load({ id })`。不一致时包会装进 `node_modules` 但不注册。
3. **没进 `dsh.profile.bundles`**。只写进 `dependencies` 只是「装了这个包」，不会加载。
4. **被同名/同类插件抢了快捷键**。两个插件都监听 `Ctrl+F`，先注册的那个赢（或者两个都弹）。卸载/禁用旧的，或让自己的监听走 capture 阶段。
5. **布局属性已过时**。插件用 `data-chat-anchor-key` 之类老属性定位，在新版本里这些属性根本不存在，于是「打开了但什么都没发生」。

---

## 五、诊断命令

```powershell
git --version                      # 有没有 git
node --version                     # Harness 需要 node >= 22
curl.exe -sS -o NUL -w "%{http_code} %{time_total}\n" https://api.github.com/repos/<o>/<r>
curl.exe -sSL -o NUL -w "%{http_code} %{time_total} %{size_download}\n" '<tarball 地址>'

# Harness 自带的 pnpm（不在 PATH）
$pnpm = 'D:\DeepSeek Harness\resources\runtime\primary-runtime\dependencies\pnpm\bin\pnpm.mjs'
node $pnpm add '<被测地址>' --reporter=append-only
```

**验收标准**：不要在你自己的 profile 里点重试，而是建一个空目录、照 README 的一条命令跑一遍，看它能不能装出正确的 `version` 与文件清单。你写的每一条安装路径都应该被这样验过一次。

---

## 六、正面清单（发布前把这些消掉，比事后答疑便宜得多）

- 零依赖、零 install script → 永远不会撞 `allowBuilds`
- Release 上有 `.tgz` 资产（`state=uploaded`）→ 有一条不需要 git 的路径
- 仓库里有 `dist/*.tgz` → jsDelivr 可以兜一层
- README 写了「哪种填法需要 git / 需要能连 GitHub」→ 用户自己就能对号入座
- 把 `.tgz` 也直接交给对方 → 零网络兜底
