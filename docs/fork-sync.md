# 本地 fork 与上游同步

本仓库是 [omdsh-dev/dsh-genui](https://github.com/omdsh-dev/dsh-genui) 的 fork，本地有自用补丁。
每次动手改之前，先把原作者的最新版本合进来；改完把产物一起重建，否则 DSH 里跑的还是旧代码。

## 远端约定

```text
origin    git@github.com:colsonSung/dsh-genui.git    # 我们的 fork：日常提交推这里
upstream  https://github.com/omdsh-dev/dsh-genui.git  # 原作者仓库：只 fetch，从不 push
```

- `main` 跟踪 `origin/main`；fork 的 `main` 可以随时给上游开 PR，上游合并后再同步回来。
- 只往 `origin` 推。`upstream` 只读，任何情况下不 push。

## 每次改之前：同步上游

```sh
cd ~/Documents/deepseek-harness/default-workspace/dsh-genui

git status --porcelain                                  # 必须干净；有改动先提交
git fetch upstream
git rev-list --left-right --count upstream/main...HEAD   # 左=落后几个提交，右=本地补丁几个提交
git merge upstream/main                                 # 用 merge，不用 rebase
```

**为什么是 merge 而不是 rebase**：`main` 已经推到 fork，rebase 会改写已发布的提交；
merge 让「上游提交」和「本地补丁」各自保持原样，也方便以后单独把补丁提成 PR。

冲突时先看下面的「本地补丁清单」：我们要保留的是规则，不是某一段旧代码。
上游重写了同一块布局，就按新结构把规则重放一遍，不要整块覆盖。

## 改完：重建产物 + 验证

DSH 加载的是 `lib/`（构建产物），不是 `src/`。只改源码不重建，页面里什么都不会变。

```sh
pnpm install       # 仅当 pnpm-lock.yaml / package.json 有变化
pnpm run build     # 重建 lib/（客户端 bundle 与 lib/assets/*）
pnpm run test      # 单元测试；全绿再往下走
```

让运行中的 DSH 用上新构建，二选一：

1. 页面刷新（`Cmd + Shift + R`）；
2. 插件管理里把 `include:genui` 关掉再打开 —— 宿主会热替换客户端模块，不必重启 App。

## 合并后要复核的三件事

1. `package.json` 的 `engines.dsh` 是否仍覆盖本机宿主（2026-11-09 时为 `0.2.2-rc.1`）。
2. `cordis.patch.yml` 的插件行 id 是否仍为 `genui`（插件管理按 `include:genui` 寻址）。
3. 系统提示里的组件清单有 3400 字符硬预算，`tests/plugin-genui.spec.ts` 会拦；
   上游新增组件时先确认还放得下。

## 本地补丁清单

合并前先看这张表，冲突优先出现在这里。

| 文件 | 本地改动 | 合并提示 |
| --- | --- | --- |
| `src/client/GenuiBlock.module.css` | `stat` 指标行：`.row > .stat` 均分宽度；`@container (max-width: 760px)` 两列 + 图号 34px→28px + 换行处去掉列分隔线；`.statValue` 的 `overflow-wrap` 由 `anywhere` 改为 `break-word` | 上游若重写 stat 布局，把这三条规则按新结构重放；不要整块覆盖上游代码 |
| `locale/en.json`、`locale/zh.json` | 插件显示名与描述（`meta.title` = Gen-UI） | 上游若补了自己的 locale，保留我们的 title，描述按需合并 |
| `package.json` | `exports` 增加 `./locale/*.json`；`files` 增加 `locale/*.json` | 上游若已导出 locale，去重保留一份 |
| `scripts/verify-pack.mjs` | 打包门禁支持通配导出键 | 上游若自己支持了通配，删掉我们的实现 |
| `CHANGELOG.md` | `[Unreleased]` 下的本地条目 | 与上游条目并列保留，按时间倒序 |
| `AGENTS.md`、`docs/fork-sync.md`、`README.zh-CN.md` | 协作守则与同步流程 | 冲突时以本文件为准 |

## 版本基线

- 上次合并的上游基线：`c2624ce`（0.16.0，2026-11-03）

每次合并后更新这一行：

```sh
git log -1 upstream/main --format='%h %ad %s' --date=short
```
