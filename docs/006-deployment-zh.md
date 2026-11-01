# 006 部署

本指南用于安装、配置和运营 dsh-superagent，涵盖账号、启动检查、升级与故障排查。它建立在 [001 领域模型](001-domain-zh.md) 到 [005 面板](005-panel-zh.md) 之上。

## 运行哪些东西

- **一个专用的 DeepSeek Harness profile**，本指南中称为 `superagent`。请从 Web profile 模板创建它。它会加载本 bundle，并在 `127.0.0.1` 上提供自己的面板。请把它与你日常工作的 profile 分开：本插件会改动进程级的 `PATH` 和 git 设置，而且它的进程需要独立地重启与升级。
- **一个智能体 GitHub 账号。** 智能体以这个账号发表评论、推送分支、创建 pull request，所用的是一个只以该账号登录的 GitHub CLI。
- **可选：** 一个飞书机器人，用于处理聊天中的请求。

插件配置只保存状态目录和机器上限。其余一切都在面板里设置：来源用于配置仓库与账号，策略用于许可名单，执行用于 worker 目录与命令。

### 权限与隔离

本 bundle 为执行器定义了 `dsh-superagent-worker`，为主 Session 定义了 `dsh-superagent-router`。它们会出现在 DeepSeek Harness 的预设列表里，但只供 dsh-superagent 使用。请不要在个人会话中选用它们，也不要禁用它们的配置行。应用需要它们来创建和恢复自己的会话。

在 Web 界面中打开一个空的受管会话时，可能会选中该 profile 的默认预设。dsh-superagent 会在第一轮之前恢复其角色专属的预设。

两个预设都以 `danger-full-access` 运行，身份是运营者的操作系统用户，并会读取已订阅线程中的消息。账号封装默认负责选择智能体的凭据，但不隔离进程。worker 可以通过路径执行程序，也可以读取运营者的主目录。若需要隔离，请使用一个只包含智能体凭据的专用操作系统账号。

## 前置条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 tmux
- GitHub CLI（`gh`）2.48 或更高版本。来源的 CLI 字段必须指向 `gh` 或 `lark-cli` 的封装脚本；宿主依赖它们的子命令与输出格式。
- github.com 上的一个仓库；不支持 GitHub Enterprise Server。为了给每个工单创建检出，请保留一份本地克隆，供 prepare 命令添加 worktree。如果该仓库使用了 Git LFS，请安装 `git-lfs`。
- 为 DeepSeek Harness 配置好的模型服务商。除非在评估或执行设置中指定了其他模型，dsh-superagent 使用该 profile 的默认模型。
- 若要使用飞书：`lark-cli`（已在 1.0.90 上验证）以及一个带机器人的飞书或 Lark 应用
## 账号

### 智能体 GitHub 账号

1. 为智能体创建一个 GitHub 用户账号，并授予它对仓库的写权限。GitHub App 不能充当智能体，因为 dsh-superagent 需要读取该账号的通知。
2. 给它一个只使用自身凭据的 GitHub CLI：

   ```sh
   mkdir -p ~/.local/bin
   cat > ~/.local/bin/gh-agent <<'EOF'
   #!/bin/sh
   GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
   EOF
   chmod +x ~/.local/bin/gh-agent
   gh-agent auth login
   gh-agent api user --jq .login
   ```

   以智能体账号登录；最后一条命令会打印它的登录名。dsh-superagent 也以同样的方式读取该登录名。

### 飞书机器人（可选）

1. 创建一个带机器人能力的自建应用。授予它接收消息事件、读写群消息、读取群成员的权限，然后发布应用版本。
2. 给机器人一份独立的 `lark-cli` 配置，限定为机器人身份，并加一个封装脚本：

   ```sh
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
   cat > ~/.local/bin/lark-bot <<'EOF'
   #!/bin/sh
   export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
   exec lark-cli "$@"
   EOF
   chmod +x ~/.local/bin/lark-bot
   ```

   第一条命令从标准输入读取应用密钥。请使用独立的配置目录：`lark-cli` 对每个 app id 只允许一份 profile，而你的个人配置里可能已经有该应用的用户登录。

   宿主会拒绝任何能以用户身份行动的 CLI。它会从机器人配置中读取 app id、飞书/Lark 品牌和机器人的 `open_id`。

   两份配置共用同一个钥匙串条目来保存应用密钥。若只想移除机器人配置，请删除它的目录；不要运行 `lark-cli config remove`，那会影响共用的密钥。

## 安装

在一台尚无 `superagent` profile 的机器上，执行下面这段一次即可：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上面钉住的版本是公开版本；其他版本必须满足 `package.json` 中的 `engines.dsh`。

第四条命令会依据 DeepSeek Harness Web 模板创建 `~/.dsh/profiles/superagent`。第五条安装本 bundle。设置 `DSH_HOME` 可以改变默认的 `~/.dsh` 目录。

创建 profile 的命令只应在全新安装时执行。如果该 profile 已存在，这条命令会替换它，并删掉它的 `cordis.patch.yml`，其中包含机器配置。升级时不要使用它。

## 机器配置

把 [examples/profile.cordis.patch.yml](../examples/profile.cordis.patch.yml) 复制为 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换掉该文件，并替换其中每个 `<placeholder>`。若要使用飞书，还需在那里启用 `dsh-superagent-feishu` 条目。`webserver` 条目用于设置面板端口。

| 键 | 必填 | 含义 |
| --- | --- | --- |
| `stateDir` | 是 | 绝对路径：数据库、锁文件、`worker-bin` 中的命令 shim、智能体的 git 配置，以及默认情况下 `work` 中的工单目录。升级前请备份 |
| `promptMaxChars`、`admissionContextMaxPages`、`admissionContextTimeoutMs` | 否 | 可保存的提示词配置的最大规模，以及一次研判在多少时间内读取多少页线程 |

首次启动之后，请在 **设置 → 部署** 或 DeepSeek Harness 的插件管理器中修改机器上限。改动无需重启即生效。`stateDir` 无法在那里修改。无效取值或未知配置键会阻止启动；日志会指明是哪个键。

## 启动与停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 会在 tmux 会话 `dsh-superagent`（其他 profile 则为 `dsh-<profile>`）中运行 `dsh --profile superagent`。它最多等待一分钟，直到运行时就绪或报告某项配置要求。首次运行时，它会打印缺失的配置信息。它会拒绝重复实例，包括在本脚本之外启动的实例。

`stop` 与 `restart` 会请求优雅关闭，并为执行器最多等待五分钟。它们绝不会强制杀掉进程，也不会碰其他 tmux 会话。

`status` 会打印进程 id、健康状况和配置问题。若使用其他 profile 或端口，请用 `--profile` 和 `--url`，或设置 `DSH_SUPERAGENT_PROFILE` 与 `DSH_SUPERAGENT_URL`。

`status`、`start` 和 `restart` 会以以下四种退出码之一结束：

| 退出码 | 含义 |
| --- | --- |
| 0 | 服务正常运行 |
| 1 | 它没有运行、重复运行、没有响应，或其运行时报告了错误；重启可能解决 |
| 2 | 它在运行，但运行时之外的某个部件报告了错误，例如某个来源或策略同步（`sync:<source id>`）；重启无法解决 |
| 3 | 它在运行并等待配置；重启无法解决 |

仍在启动中的来源算作正常运行。配置过程无法检查的飞书机器人也算作正常运行。

`status` 会在 `errors` 下返回失败的部件及其详情，在 `setup` 下返回配置要求。`start` 和 `restart` 会把这些问题按行打印；如果服务始终起不来，它们会以退出码 1 结束。只有主动执行的 `stop` 会暂停自动恢复。

每个 profile 都会在 `.artifacts/` 下保存日志与控制文件：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中 pid 属于该脚本。只保留最新的 20 份。
- 控制文件：`control/profiles/<profile>/`，包含停止标记、锁文件与进程 id。

打开日志中打印的 URL，然后在侧边栏中选择 **Superagent**。

### 每小时巡检

请在你日常使用的那个 DeepSeek Harness profile 中创建这项巡检，而不是在 `superagent` 里。这样即使智能体服务已停止，巡检仍能继续运行。在那个 profile 中让智能体按下面的提示词创建一个每小时任务，并把检出路径填进去：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment.md, fix what you can, and tell me what was wrong and what you did.
```

`--respect-stop` 会保住那次主动停止。只有 `stop` 写入该标记；`start` 会清除它。`restart` 不写标记，因此巡检可以恢复那些关闭时超时的重启。

## 设置

在轮询或工作开始之前，配置必须成功。总览会列出缺失的要求，并链接到对应设置。设置变更会重新执行配置检查。如果你在应用之外修好了某些东西（例如 CLI 认证），请点击 **重新检查**。

1. **来源。** 设置 GitHub 仓库（`owner/name`）和智能体 CLI，例如 `/home/me/.local/bin/gh-agent`。请求通常通过智能体账号的通知到达。提及该账号的新 Issue 或新评论会在下次轮询时到达；通过编辑既有 Issue 正文来添加提及则不会。请改为发一条新评论。若要使用飞书，请设置机器人 CLI、群列表、每个群的"接收全部"或"仅接收 @"模式，以及是否接收私信。
2. **策略。** 两条示例规则可以编辑或删除。在 **逐个来源** 下，添加可以请求工作的 GitHub 账号和飞书用户；在某个名单里有人之前，任何请求都不会变成工单。使用飞书时，可以选择一个群来同步该名单，时机是启动时以及点击 **立即同步** 时。
3. **执行。** 目录根与 prepare 命令。若要为每个工单给出一份仓库检出：

   ```sh
   git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
   git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
   ```

   请把个人使用的 CLI（例如已登录你自己账号的那个）列为禁用命令。
4. **提示词。** 设置团队段落：GitHub 上的披露声明行、评审机器人、提交作者以及自动合并条件。

### 管理 API

面板使用的是一套本地 HTTP API，智能体也可以调用它。`SETUP.md` 用它保存设置，服务脚本则读取 `GET state`。

把请求发到 `http://127.0.0.1:<port>/orchestrator/api/`，并带上 `x-orchestrator-request: 1`。Host 必须是 `127.0.0.1:<port>` 或 `localhost:<port>`。缺少请求头，或 `Origin` 来自其他站点，都会收到 403。

API 的保存拥有与面板相同的权限。与通过主 Session 工具所做的保存不同，它们可以修改执行环境和配置中的账号字段。

智能体可以依赖以下路由与字段。`src/app/api.ts` 中的其他路由服务于面板，会随之变动。

| 请求 | 响应 |
| --- | --- |
| `GET state` | 每个页面都会读取的内容：`setup`，含 `problem` 与 `feishuProblem`，各自为 `null` 或一个配置问题（含 `place`、`code`、`values` 和 `detail`），以及配置解析出账号后的 `accounts`；`health`，每个部件一条记录（见下文），含 `state`（`starting`、`ready`、`paused` 或 `error`）、`detail` 和 `at`；`load`，即执行器 Session 的 `running`、工单的 `queued` 与 `maxWorkers`；`paused`；以及 `masterId` |
| `GET settings` | `blocks`：`policy`、`eval` 和 `executor`，各自带 `value` 与 `revision`；`sources`：来源实例，每个带 `id`、`kind`、`installed`、其区块的 `value`（形如 `{"enabled": …, "config": {…}}`）和 `revision`，以及该种类的表单和它的配置 `problem`（若有）；`policyVersion`；还有 `environment`，即执行环境的 `root` 与 `variables` |
| `GET sources` | 每个来源的状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、各部件 `health`、配置 `problem`、最近一天的 `stats`、当天被 `filtered` 的触发，以及当天被 `dropped` 的消息（最新的丢弃在前），每条含 `channel`、`preview`、它原本所属的 `ticket`（`id` 与 `title`）、存储中能查到时的 `thread`，以及 `dropped.at`、`dropped.reason`、`channelGone` 或 `channelIgnored` |
| `PUT settings/<block>`，请求体为 `{"value": …, "revision": n}` | 保存一个区块：`saved`、该区块新的 `revision`、`policy_changed` 与 `restarting`。`value` 是整个区块，形式与 `GET settings` 中的一致；缺失的键取默认值 |
| `POST executor/pause`，请求体为 `{"paused": true}` 或 `false` | 暂停或恢复处理：`paused` |
| `POST setup/check` | 通过重启 dsh-superagent 重新执行配置：`restarting: true`。若配置中没有需要修的内容，会被拒绝 |
| `POST policy-sync/<source id>` | 立刻同步该来源的许可名单：`policy`，即已保存的来源部分 |
| `GET prompts`、`PUT prompts`，请求体为 `{"config", "baseRevision"}` | 读取提示词配置：`current`（含 `revision` 与 `config`）、`defaults`、`sections`（团队段落，按会话读取的顺序）、`skillName`（团队技能名称的模式）、`builtin`（团队技能不能使用的名称）以及 `skillPrefix`（会话加载技能名时使用的前缀）；或保存它：返回新的 `revision` |
| `POST tickets/<id>/retry`、`POST tickets/<id>/stop` | 按面板的做法重试或停止一个工单 |

健康记录按职责分组：

| 记录 | 汇报内容 |
| --- | --- |
| `runtime` | 应用自身 |
| `models` | 正在等待模型预算重试的会话 |
| `evals` | 最近一小时内、没有更晚判断也未被忽略的 `unavailable` 评估 |
| `assessment`、`launch`、`delivery`、`relay`、`notice`、`reply` | [003 执行](003-execution-zh.md) 中描述的已存储工作；`delivery` 统计等待中的工作，`reply` 涵盖渠道消息 |
| `worker` | 执行器 Session 的事件处理；首次失败之后才会出现 |
| `github`、`feishu`、`history` | 来源轮询、事件与历史回溯 |
| `sync:<source id>` | 许可名单同步 |

一次拒绝会带一个状态码和 `error`，即其英文文本：

- 400：无效取值（`issues`，每个带 `path`）、账号检查失败（`setup`）、固定的拒绝（`code` 与 `values`），或请求体不是 JSON、或过大。
- 404：对象不存在。
- 409：修订号过期。
- 500：其他一切情况。

各区块的形态：

- `source:github`、`source:feishu`：`{"enabled": true, "config": {…}}`。GitHub 的 config 含 `repository` 和 `cli`，以及默认关闭的开关 `observeIssues`、`observePulls`、`observeComments`、`observeCi`——它们会接收仓库中每个新 Issue、新 pull request、新评论和默认分支的 CI；飞书的则含 `cli`、`chats`（形如 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`，或 `"all"`）、`directMessages`，以及取值为 `"none"` 或 `"mentions"` 的 `otherChats`。
- `policy`：`{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`。`bots` 列出那些消息可以成为推断候选的机器人。`admission` 的键是群 id，飞书另可用 `other` 或 `direct`，GitHub 则用 `issues`、`pulls`、`comments` 或 `ci`；未设置的单元格取默认值（见 [002 授权](002-authorization-zh.md)）。
- `eval`：`mode`、`timeoutSeconds` 和 `model`。
- `executor`：`model`、`maxWorkers`、`directory`、`prepare` 和 `blockedCommands`。处理暂停是一项独立设置 `paused`，由 `POST executor/pause` 设置。

修改仓库或 CLI 时，会先执行对应的配置检查。检查失败返回 400，详情在 `setup` 中。账号变更成功，或在配置待完成期间进行的任何保存，都会返回 `restarting: true`。应用重启期间，API 暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

## 配置检查

配置检查使用 DeepSeek Harness 的子进程服务，与 worker 的 shell 相同。它会移除名称中含 KEY、PASSWORD、SECRET 或 TOKEN 的环境变量。

配置检查会验证 GitHub 仓库和智能体的推送权限，写入 worker 环境，并确认 git 对 github.com 只使用智能体的凭据。飞书配置失败只会禁用飞书来源。

配置过程写入的进程环境：

- **命令 shim**，位于 `<stateDir>/worker-bin`，放在 `PATH` 最前：`gh` 会运行智能体的 CLI；`lark-cli` 会运行机器人的 CLI；被禁用的命令会失败。shim 在运行目标程序之前，会先把自己的目录从 `PATH` 中移除。
- **git 配置**，位于 `<stateDir>/worker-gitconfig`：替换运营者的全局与系统 git 配置。它选用智能体 CLI 作为 github.com 的凭据助手，把 GitHub 的 SSH 地址重写为 HTTPS，并定义 Git LFS 过滤器。个人的 git 设置（例如 HTTP 代理）不会生效。这里不设置提交身份，git 从不提示输入凭据，且 `GH_TOKEN` 与 `GITHUB_TOKEN` 会被移除。

`worker-bin` 或 `worker-gitconfig` 中若有非 dsh-superagent 写入的文件，配置检查会中止。若工单目录位于某个在其自身配置里添加了凭据的 git 仓库内，该工单会失败。

## 试一试

用一个在许可名单上的 GitHub 账号，开一个 Issue 或发一条评论，内容包含 `@<agent login> <request>`。在一个轮询间隔内（默认一分钟），总览应当出现一个工单。智能体会确认该请求，在自己的会话和目录中工作，并在原始对话中交付结果。若要使用飞书，请在已配置的群里 @ 该机器人。

## 升级

1. 等到没有执行器正处于一轮之中。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 运行 `git pull && pnpm install && pnpm build`。
4. 运行 `scripts/dsh-superagent restart`，并检查它的状态。

重启会结束运行中的派工，并带着恢复说明启动替代的派工。团队文本和被覆盖的提示词部分会保留；没有覆盖的部分跟随新的默认值。升级后请检查被标记为 **默认内容已更新** 的部分。

被较新版本迁移过的数据库无法由旧版本打开。因此回滚需要升级前的备份。升级 DeepSeek Harness 本身之后，请重启正在运行的实例，以便面板加载匹配的产物。

### 智能体预设

worker 与 router 预设会在 5 秒后把 shell 命令转入后台。会话若没有对应预设就无法加载。当它们的会话仍在使用时，请保留 `cordis.patch.yml` 中的 `dsh-superagent-worker` 与 `dsh-superagent-router` 行；否则恢复会以 `agent-preset/not-found` 失败。

## 故障排查

配置问题会以面板的语言显示在总览上，并附上修复链接；日志和 API 中显示的是下面的英文原文。

| 现象 | 原因与处理 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片上设置智能体的 CLI |
| `The agent GitHub CLI was not found.` | 设置封装脚本的绝对路径，或一个位于 `PATH` 上的命令名 |
| `The agent GitHub CLI cannot read its account` | 用 `auth login` 为封装脚本登录 |
| `@… cannot read …` 或 `@… cannot push to …` | 给智能体账号授予对仓库的写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片上设置仓库，`owner/name` |
| 工单失败并提示 `The prepare command failed: …` | 输出会指明原因，例如克隆路径错误；修好命令后重试该工单 |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高版本 |
| 工单失败并提示 `The git configuration of the repository in … sets …` | 从该克隆的 `.git/config` 中移除被指名的凭据或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 把被指名的文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或者另选一个 `stateDir` |
| `The bot lark-cli allows user identities` | 让飞书来源指向一份仅机器人可用的配置，做法见上文"飞书机器人" |
| `The bot lark-cli does not work.` 或 `The bot identity of app … is …` | 检查配置中的 app id 与密钥 |
| `The Feishu bot is not a member of the chat.` | 在群设置中把机器人加入该群；下次尝试即可成功 |
| `status` 退出码为 2 | 某个部件报告了错误，重启无法解决：`errors` 会按管理 API 中列出的那样，逐个列出部件及其详情；每个部件恢复后会自动清除自己的记录 |
| `status` 退出码为 3 | 配置在等待：`setup` 说明缺什么，总览会链接到相应的设置项 |
| `status` 退出码为 1 | 服务没有运行、重复运行、没有响应，或其运行时报告了错误；输出与最新日志会指出是哪种 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …` 或 `… could not run: …` | 某个 CLI 失败但没有打印原因；以运营者的用户身份手动运行被指名的程序，即可看到它自己的消息 |
| 面板提示 dsh-superagent 没有响应 | 它正在重启，或因配置问题没能启动；DeepSeek Harness 的日志会指明原因 |
| 日志中出现 `dsh-superagent configuration has keys it does not use` | 移除被指名的键；提示会说明每个取值现在该放在哪里 |
| 日志提示插件与正在运行的 dsh 不兼容 | 安装一个处于 `engines.dsh` 范围内的 DeepSeek Harness 版本 |
| 升级 DeepSeek Harness 后，面板显示 "Failed to load plugins" | 重启正在运行的实例 |
| 提及之后没有任何反应 | 检查作者是否在该来源的许可名单上、来源是否已开启、处理是否被暂停。在 GitHub 上，通过编辑 Issue 正文添加的提及不会被接收；请在评论中 @ 智能体 |
