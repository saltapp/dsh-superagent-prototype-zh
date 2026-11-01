# 008 实操手册

本手册演示如何扩展 dsh-superagent。它使用 [001 领域模型](001-domain-zh.md) 与 [002 授权](002-authorization-zh.md) 中的术语，以及 [007 架构](007-architecture-zh.md) 中的组件。

## 新增一个来源

本配方实现一个名为 `webhook` 的来源 kind。它通过一个虚构的 CLI `hookq` 读取一个假想的消息队列，并把消息转成触发。其他所有接口都属于 dsh-superagent。

实现包含四项职责：翻译事件、定义接收与授权规则、提供回复与上下文、维护一个轮询循环。下面的编号步骤按顺序搭建这些部件。

示例假定如下 CLI 契约：

- `hookq pull --after <n> --json` 打印位置 `n` 之后的消息，最旧的在前：`{"me": "<the agent's id on the queue>", "items": […], "next": <position>}`。每个条目含 `id`、`conversation`、`thread`（开启其线程的那条消息的 id，在该消息本身上不存在）、`sender`（含 `id` 与 `bot`）、`mentions_me`、`text`、`at` 和 `url`。
- `hookq reply --conversation <id> --to <message id> --key <key> --text <text>` 在一条消息下方回复，每个 key 只回一次。若对话已被删除，它会在标准错误上以 `{"error": {"code": 410, "message": "…"}}` 失败。
- `hookq thread --conversation <id> --root <message id> --json`，配合 `--after <cursor>` 可读后续分页，打印线程的一页，最旧的在前：`{"items": […], "next": "<cursor>"}`，最后一页不含 `next`。

请把该 kind 及其插件入口放在 `src/sources/webhook.ts`，参照 `src/sources/github.ts`。如果该来源变得复杂，就像飞书来源那样，把翻译、渠道与轮询代码拆到独立模块中。

### 1. 把事件映射到领域

在写代码之前先选定稳定的映射。触发 id、线程 id 与渠道地址都会被持久化，因此之后再改动它们，可能让既有记录失去联系。

| 概念 | 在这个 webhook 队列中 |
| --- | --- |
| 触发 | 一条消息 |
| 线程 | 一条消息及其在同一对话中的回复：`<conversation>:<id of the first message>` |
| 操作者 | 发送者在该队列上的 id，与许可名单中的写法一致 |
| 渠道地址 | 回复所落的对话与消息 |
| 接收范围 | 一个对话，在准入表中有自己的一行 |

### 2. 配置：schema、字段、事实与健康

该模块导入后续步骤所用到的东西：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { setTimeout as delay } from 'node:timers/promises'
import { z } from 'zod'
import { key, threadKey, type OutboundMessage, type Trigger } from '../core/index.ts'
import { CliError } from '../infra/commands.ts'
import { KV } from '../kv.ts'
import { health, sourceContext, type SourceContext } from './host.ts'
import type { SourceKind } from './kind.ts'
```

```ts
const ConfigSchema = z.object({
  conversations: z.array(z.object({
    id: z.string().regex(/^[a-z0-9-]+$/),
    listen: z.enum(['mentions', 'all']),
  }).strict()).default([]),
  pollSeconds: z.number().int().min(10).max(3600).default(30),
}).strict()
export type WebhookConfig = z.infer<typeof ConfigSchema>

export const webhookKind: SourceKind<WebhookConfig> = {
  kind: 'webhook',
  title: { zh: 'Webhook 队列', en: 'Webhook queue' },
  description: { zh: '接收 webhook 队列里下列对话的消息。', en: 'Receives the messages of these conversations from the webhook queue.' },
  schema: ConfigSchema,
  fields: [
    { key: 'conversations', type: 'rows', label: { zh: '接收的对话', en: 'Conversations' }, columns: [
      { key: 'id', type: 'text', label: { zh: '对话', en: 'Conversation' }, pattern: '[a-z0-9-]+' },
      { key: 'listen', type: 'select', label: { zh: '接收', en: 'Receive' }, options: [
        { value: 'mentions', label: { zh: '仅 @', en: '@ only' } }, { value: 'all', label: { zh: '全部消息', en: 'All' } }] },
    ] },
    { key: 'pollSeconds', type: 'number', advanced: true, min: 10, max: 3600,
      label: { zh: '轮询间隔 / 秒', en: 'Polling interval / seconds' } },
  ],
  health: ['webhook'],
  // find, scope, channel, threadContext, policy and environment follow in the next steps.
}
```

- `kind` 是稳定的来源 id。它会出现在触发的 `source`、线程键前缀、`source:webhook` 设置区块和来源策略中。
- `schema` 校验已存储的配置并提供默认值。在运营者保存某个实例之前，注册表会依据这些默认值创建一个已启用的实例。请让默认值什么都不接收；本示例默认没有任何对话。
- `fields` 无需自定义面板代码即可生成来源设置表单。`rows` 字段编辑以第一列为键的条目；`advanced` 会隐藏不常用的设置。如果该来源能把 id 解析成显示名称，请实现 `refreshLabels`，并把这些名称保存在 `KV.sources.labels(kind)` 下。带 `choose` 与 `refresh` 的列随后就可以提供它们。
- 可选的 `facts(deployment)` 展示由 Setup 派生的取值，例如某个 CLI 背后的账号。Setup 目前只检查 `SETUP_FIELDS` 中列出的 GitHub 与飞书字段，因此本示例不提供 facts。
- `health` 声明该来源的健康记录，初始为 `starting`。注册会认领每个名称，并拒绝与 `runtime`、`reply`、`worker` 或 effect kind 等记录冲突。请使用该 kind 的名称，并把 `sync:<id>` 留给策略同步。一条 `error` 会让状态脚本以退出码 2 结束，并让每小时巡检通知运营者，因此只有可能需要人工介入时才报告它。

### 3. 把消息翻译成触发

在转换之前先校验 CLI 的响应：

```ts
/** One message as `hookq pull` and `hookq thread` print it. */
const MessageSchema = z.looseObject({
  id: z.string().min(1), conversation: z.string().min(1), thread: z.string().optional(),
  sender: z.looseObject({ id: z.string(), bot: z.boolean().default(false) }), mentions_me: z.boolean().default(false),
  text: z.string(), at: z.string(), url: z.string().optional(),
})
type QueueMessage = z.infer<typeof MessageSchema>
/** The answer of `hookq pull`, whose items are read one by one, and of `hookq thread`. */
const PageSchema = z.looseObject({ me: z.string(), items: z.array(z.unknown()), next: z.number().int() })
const ThreadPageSchema = z.looseObject({ items: z.array(MessageSchema), next: z.string().optional() })
```

```ts
/** The Trigger of one queue message; the agent's own messages produce none. */
export function webhookTrigger(message: QueueMessage, me: string): Trigger | undefined {
  if (message.sender.id === me) return undefined
  const thread = `${message.conversation}:${message.thread ?? message.id}`
  return {
    id: key('webhook', message.id),
    source: 'webhook',
    threads: [{ source: 'webhook', id: thread }],
    actor: { source: 'webhook', id: message.sender.id },
    human: !message.sender.bot,
    ...(message.mentions_me ? { addressed: 'mention' as const } : {}),
    at: message.at,
    text: message.text,
    channel: { source: 'webhook', thread: `webhook:${thread}`,
      address: { conversation: message.conversation, message: message.id, ...(message.url ? { url: message.url } : {}) } },
    ...(message.url ? { url: message.url } : {}),
    meta: { conversation: message.conversation, messageId: message.id, root: message.thread ?? message.id },
  }
}
```

- **稳定的 id。** 通过 `key` 由队列 id 推导触发 id。`Engine.receive` 对已存储的 id 返回 `duplicate`，而重复接受只会创建一个工单。因此重新读取消息是安全的。若要把编辑当作新事件，就把编辑版本纳入 id，正如 GitHub 用评论的 `updated_at` 所做的那样。
- **线程。** 第一个线程是主线程，拥有该评估。额外的线程只增添关联上下文。id 由来源选择；它的完整键是 `webhook:<id>`。
- **发送者。** 在 `actor` 中使用来源原生的 id。对机器人和 CI 设置 `human: false`。名单内的机器人只能成为推断候选。空的 actor id（例如 CI 的情况）会使用"不在名单且未指向"的准入单元格。只有当消息确实指向智能体时，才把 `addressed` 设为 `mention` 或 `direct`；这会让名单内人士的消息获得明确强度。绝不为智能体自己的消息生成触发。
- **回复目标与元数据。** `channel` 指明回复去哪里；`url` 链接到来源消息。`channel.thread` 为有序投递而把消息分组：工单较晚的消息会排在它较早的消息之后，直到某条较早的消息连续失败三次；其他工单不必等待。把来源专属字段留在 `meta` 里，例如接收对话与线程根。`meta` 和 `text` 都是不可信的数据，不是授权。

智能体可以通过 `orchestrator_read trigger/<id>` 与 `thread/<key>` 读取新的 kind。这些会暴露消息文本、去向与邻近触发。新的 kind 不会自动获得内置技能、原始服务记录的资源，或 pull request 绑定。

文本搜索无需额外代码即可工作。当人们使用来源专属的引用（例如消息 id、URL 或 `#12`）时，实现 `find(query)`。返回这些引用所指向的线程键与搜索词：

```ts
  find: query => ({ threads: [], terms: [...new Set(query.match(/\bm[0-9]+\b/g) ?? [])] }),
```

这里，查询中的队列消息 id（例如 `m42`）会在触发中搜索它。

### 4. 来源接收什么：scope

```ts
  scope({ meta }, config) {
    const conversation = config.conversations.find(item => item.id === meta.conversation)
    if (!conversation) return { listened: false, reason: 'conversation_not_listed' }
    return { listened: true, observed: conversation.listen === 'all', row: conversation.id }
  },
```

Engine 会通过 `SourceRegistry` 为每个到达的触发调用 `scope`。超出范围的消息会连同理由被存储为已过滤，且不会创建线程。请把消息交给 `receive`，而不是在轮询循环里静默丢弃，这样面板才能解释过滤的原因。

`observed: true` 表示该来源在那个范围内接收未被指向的消息，从而允许对它们套用准入表的决定。`row` 标识对应的准入行。

重新校验会带着授权中存储的元数据调用 `scope`。请让它只依赖 `meta` 与配置。恢复只要求 `listened`，不要求 `observed`。没有 `scope` 时，kind 会接收它产生的每一个触发，但不观察任何触发。

### 5. 回复：渠道

```ts
  channel: {
    allowed: (address, config) => config.conversations.some(item => item.id === address.conversation),
    send: sendWebhook,
    link: address => ({ url: address.url ?? '',
      label: { zh: `Webhook 对话 ${address.conversation}`, en: `Webhook conversation ${address.conversation}` } }),
  },
```

```ts
export async function sendWebhook(host: Pick<SourceContext, 'commands'>, message: OutboundMessage): Promise<'sent' | 'gone'> {
  const { conversation, message: to } = message.channel.address
  if (!conversation || !to) throw new Error('Missing webhook reply target')
  try {
    await host.commands.run('hookq', ['reply', '--conversation', conversation, '--to', to, '--key', message.id,
      '--text', message.text])
    return 'sent'
  } catch (error) {
    if (error instanceof CliError && error.code === 410) return 'gone'
    throw error
  }
}
```

- **幂等发送。** Outbox 调用 `SourceRegistry.send`，后者调用 `channel.send`。来源插件未加载期间，消息会等待。一次发送可能在远端服务已经收到之后才失败，因此重试绝不能造成消息重复。请用 `message.id` 作为幂等键：`hookq` 支持 `--key`；GitHub 会存一个隐藏的评论标记；飞书接受幂等键。
- **永久失败与可重试失败。** 成功时返回 `sent`，只有投递永久不可能时才返回 `gone`，例如示例中的 410。`gone` 的消息会作为已丢弃保留，绝不再入队；它会出现在 `GET sources` 与来源页面上。其他失败请抛出，以便 Outbox 在 5 秒后重试，翻倍至 300 秒。`CliError.code` 是 CLI 输出所报告的代码。
- **允许的目标。** `allowed(address, config)` 只检查已保存的地址与当前配置。把所有必需的事实放进地址里；例如飞书会记录该群是否为私信。已不再被监听的目标会被丢弃，而来源被禁用时消息保持等待。没有 `allowed` 时，只要来源启用，所有地址都被允许。
- **链接。** `link(address, links)` 返回面板经 `SourceRegistry.link` 显示的 URL 与标签。

### 6. 供研判使用的线程上下文

```ts
  threadContext: {
    messageId: trigger => String(trigger.meta.messageId ?? ''),
    // A queue thread is flat: its root, and the message a reply answers, are on its first page, as on GitHub.
    read: async (commands, trigger, _part, cursor) => {
      const page = ThreadPageSchema.parse(JSON.parse(await commands.run('hookq', ['thread', '--conversation',
        String(trigger.meta.conversation), '--root', String(trigger.meta.root), '--json', ...(cursor ? ['--after', cursor] : [])])))
      const messages = page.items.map(item => ({ id: item.id, from: item.sender.id, at: item.at, text: item.text,
        ...(item.sender.bot ? { bot: true as const } : {}), ...(item.url ? { url: item.url } : {}) }))
      return { messages, ...(page.next ? { next: page.next } : {}), body: { messages } }
    },
  },
```

主动研判需要触发周边的对话。`orchestrator_eval` 会读取一个初始窗口；`orchestrator_eval_read` 会展开分页、根消息或父消息。实现可选的 `threadContext` 接口来提供这些读取：

- `read(commands, trigger, part, cursor, config)` 返回一个 `ContextPart`。它的 `messages` 使用 `ContextMessage`：`{id, from, at, text}`，外加适用的 `bot`、`deleted`、`reply_to`、`url`、`path` 和 `line` 字段。最后一页省略 `next`。`body` 是展示给研判会话的内容，这里就是同一批消息。可选的 `status` 可以描述特殊结果，例如飞书的 `chat_neighbours`。`part` 是 `thread`、`root` 或 `parent`。
- `messageId(trigger)` 在其中找到触发自己的那条消息。
- `images(text)` 与 `image(…)` 供消息携带图片的 kind 使用（如飞书）；本示例省略了它们。

没有 `threadContext` 的 kind 不会给研判提供线程上下文；触发自身的文本仍然可读。

如果消息是对更早消息的回复，请加上 `repliesTo(trigger)`。它返回对应的触发 id，让后续发言通知和 worker 输入在已存储时能显示被回复的那条消息。例如，若 `hookq` 提供 `reply_to`，而翻译把它存为 `meta.replyTo`：

```ts
  repliesTo: trigger => typeof trigger.meta.replyTo === 'string' ? key('webhook', trigger.meta.replyTo) : undefined,
```

### 7. 谁可以请求：策略中的来源部分

```ts
  policy: {
    schema: z.object({ allowed: z.array(z.object({ id: z.string().min(1), name: z.string().trim().max(100).optional() }).strict())
      .default([]) }).strict(),
    fields: [{ key: 'allowed', type: 'rows', label: { zh: '能发起工作的发送者', en: 'Senders who may originate work' }, columns: [
      { key: 'id', type: 'text', label: { zh: '发送者', en: 'Sender' } },
      { key: 'name', type: 'text', label: { zh: '名字', en: 'Name' }, optional: true }] }],
    explicit: { zh: '名单里的人在消息里 @ 智能体。', en: 'Someone on the list mentions the agent in a message.' },
    rows: stored => ConfigSchema.parse(stored).conversations.map(conversation => {
      const all = conversation.listen === 'all'
      return { id: conversation.id, label: conversation.id, note: all ? { zh: '全部消息', en: 'All' } : { zh: '仅 @', en: '@ only' },
        cells: { listedUnaddressed: all, unlistedAddressed: true, unlistedUnaddressed: all } }
    }),
  },
```

- `schema` 校验来源策略，包括 `allowed` 发送者 id。id 比较不区分大小写。准入表单独校验，并在各 kind 之间共用。这个策略部分保存在 `policy.sources.webhook`，并计入策略版本。
- `fields` 在 **评估 → 逐个来源** 下生成表单，以 `allowed` 开头。`explicit` 说明如何指向智能体。可选的 `unnamed: {person, bot}` 为不透明 id（例如飞书的 `ou_…`）提供显示标签；本示例省略它，因为队列 id 本身就是可读名称。
- `rows(config)` 列出准入范围及其可用单元格。每个行 id 必须与 `scope` 返回的 `row` 一致。未指向消息的单元格只在来源确实观察这些消息的地方可用。没有 `rows` 的 kind 使用默认单元格。
- 可选的 `bots` 列出有资格成为推断候选的机器人，条目形状与 `allowed` 相同。没有它，任何机器人都不被准入。
- 可选的 `sync: {label, ready(part), run(host, part, signal)}` 会在启动时以及点击 **立即同步** 时刷新成员关系。飞书使用群成员关系。失败会出现在 `sync:<id>` 下，并在 1 分钟后重试，翻倍至 1 小时，直到成功。

### 8. 给执行器的变量

```ts
  environment: {
    variables: [{ key: 'DSH_SUPERAGENT_WEBHOOK_CONVERSATION', description: {
      zh: '接下工单的 webhook 对话；工单不是从这个来源来的时为空',
      en: 'The webhook conversation that started the ticket; empty when the ticket did not start on this Source' } }],
    values: ({ trigger }) => ({
      DSH_SUPERAGENT_WEBHOOK_CONVERSATION: trigger?.source === 'webhook' ? String(trigger.meta.conversation ?? '') : '' }),
  },
```

执行设置会在 prepare 命令旁显示已声明的 `variables`，即使该来源已禁用。在启用状态下，`values` 会依据部署、配置与发起它的触发，提供工单专属的取值。

prepare 命令读取环境变量；worker 在其指令中读取这些值。请使用 `DSH_SUPERAGENT_<KIND>_…` 前缀，对来自其他来源的工单返回空值，并且绝不包含凭据。

### 9. 插件入口及其循环

```ts
export const name = 'dsh-superagent-webhook'
export const inject = ['orchestrator']
export function apply(ctx: Context): void {
  ctx.effect(() => ctx.orchestrator.sources.register(webhookKind), 'webhook: source kind')
  ctx.effect(() => {
    const source = sourceContext(ctx.orchestrator, webhookKind)
    const controller = new AbortController()
    // DSH ends its process on an unhandled rejection, so a failure that escapes the loop becomes the Source's health.
    const running = watch(source, controller.signal)
      .catch((error: unknown) => { source.report('webhook', { state: 'error', detail: String(error) }) })
    return async () => { controller.abort(); await running }
  })
}
```

该插件注入 `orchestrator`。它的第一个 effect 注册 kind，并在销毁时注销。第二个用 `SourceContext` 运行轮询循环。销毁会中止该循环并等待其结束，应用停止或重启期间也一样。

```ts
async function watch(ctx: SourceContext, signal: AbortSignal): Promise<void> {
  await ctx.ready
  // Setup restarts dsh-superagent once it resolves the accounts; until then the Source reads nothing.
  if (!ctx.configured()) { ctx.report('webhook', health('paused', 'setupPending')); return }
  let failures = 0
  while (!signal.aborted) {
    const config = ctx.config<WebhookConfig>('webhook')
    if (!config) ctx.report('webhook', health('paused', 'sourceOff'))
    else {
      const at = new Date().toISOString()
      try {
        const { read, unreadable } = await pollQueue(ctx, signal)
        failures = 0
        ctx.report('webhook', { state: 'ready', detail: `${read} messages, ${unreadable} unreadable`, at })
      } catch (error) {
        failures += 1
        // One network failure is retried on the next round; three in a row need a person.
        if (!signal.aborted && failures >= 3) {
          ctx.report('webhook', health('error', 'pollFailures', { count: String(failures), error: String(error) }))
        }
      }
    }
    await delay((config?.pollSeconds ?? 30) * 1000, undefined, { signal }).catch(() => {})
  }
}

/** Read the queue after the cursor, take in each message, then move the cursor past them. */
export async function pollQueue(ctx: SourceContext, signal: AbortSignal): Promise<{ read: number; unreadable: number }> {
  const after = ctx.state.get(KV.webhook.cursor) ?? 0
  const output = await ctx.commands.withSignal(signal).run('hookq', ['pull', '--after', String(after), '--json'])
  const page = PageSchema.parse(JSON.parse(output))
  let unreadable = 0
  for (const item of page.items) {
    if (signal.aborted) throw new Error('The webhook Source stopped during the read')
    const message = MessageSchema.safeParse(item)
    if (!message.success) { unreadable++; continue }
    const trigger = webhookTrigger(message.data, page.me)
    if (!trigger) continue
    await ctx.receive(trigger)
    const conversation = { label: { zh: '对话', en: 'Conversation' }, value: message.data.conversation }
    for (const ref of trigger.threads) ctx.threads.describe(threadKey(ref), { facts: { conversation } })
  }
  ctx.state.set(KV.webhook.cursor, page.next)
  return { read: page.items.length, unreadable }
}
```


- **Setup 与开关。** 等待 `ctx.ready`，并在 `ctx.configured()` 为假期间不读取任何内容。每轮读取 `ctx.config(kind)`：来源关闭时它为 undefined。
- **游标。** 用 `KV`（`src/kv.ts`）中的带类型条目把读取位置存进 `ctx.state`，例如 `webhook: {cursor: record<number>('webhook-cursor')}`。`tests/store.spec.ts` 要求键字面量只出现在那个注册表里。外部包可以用导出的 `record` 与 `family` 定义键。把值设为 `undefined` 会移除记录；`ctx.atomic` 会把多次写入归入一个事务。

  只有在覆盖到的每个触发都已存储之后，才推进游标。被中断的一轮会让游标保持不变；稳定的 id 让重放变得安全。对于时间戳游标，`src/sources/poll.ts` 中的 `cursorAt(started)` 会减去五秒，以容纳远端的迟到写入。
- **线程元数据。** 收到触发之后，调用 `ctx.threads.describe(key, {label, url, facts})` 填充其显示信息。`ctx.threads.patch(key, meta)` 委托给 `threads.patchMeta`，设置由来源拥有的键，或在值为 `undefined` 时移除它们。两者都会忽略不存在的线程，包括被过滤触发所在的线程。
- **单条错误。** 统计格式异常的条目并继续，而不是让整批失败。`src/infra/commands.ts` 中的 `validItems(item, list, tally)` 支持这种做法。对于独立轮询的线程，请使用 `src/sources/poll.ts` 中的 `pollThreads`；它把每次失败记录为 `pollError`。它的 `polled` 谓词会保留活跃订阅，以及从最后一次触发与该工单结束两者中较晚者之后的两周窗口。
- **健康。** 只通过 `ctx.report` 写入已声明的记录。`health(state, code, values)` 提供固定消息，例如 `setupPending`、`sourceOff` 和 `pollFailures`。在报告 `error` 之前先统计连续失败次数。
- **中止。** 把 signal 传给 `ctx.commands.withSignal(signal)` 和 `delay`，并在条目之间检查它，这样一次停止或重启会立刻结束循环。

本示例不发送由来源发起的通知。需要它们的来源会调用 `ctx.notify`；Outbox 会像其他回复那样投递它们。

本示例也不把线程绑定到工单完成。只有当来源的线程生命周期决定工作是否完成时（GitHub 的 pull request 就是这样），才使用 `ctx.tickets.bindThread` 与 `threadClosed`。工单何时结束仍由 Engine 决定。

### 10. 注册插件

把该模块构建为 `tsdown.config.ts` 中自己的入口：

```ts
entry: { index: 'src/index.ts', 'github-watch': 'src/sources/github.ts', 'feishu-watch': 'src/sources/feishu.ts',
  'webhook-watch': 'src/sources/webhook.ts' },
```

然后在 `package.json` 中导出构建产物：在 `exports` 里加入 `"./webhook-watch": "./lib/webhook-watch.js"`，并把该插件加入本 bundle 的 `cordis.patch.yml` 的 `insert` 列表，先设为禁用，就像飞书那样：

```yaml
    - id: dsh-superagent-webhook
      name: dsh-superagent/webhook-watch
      disabled: true
```

在某个 profile 的 `cordis.patch.yml` 中用 `id: dsh-superagent-webhook` 与 `disabled: false` 启用该插件。构建并重启。新的来源会出现在来源设置中，以及 **评估 → 逐个来源** 下。

把它的模块与测试加入 [007 架构](007-architecture-zh.md) 的模块表和测试表。`tests/docs.spec.ts` 会检查这两个索引是否完整。

### 11. 测试

按照 `tests/sources.spec.ts` 的模式创建 `tests/webhook.spec.ts`：在临时目录中放一个真实的 Store，一个带 `HealthBoard` 的 `SourceRegistry`，一个使用替身端口的 Engine，以及由 `sourceContextOf` 得到的 `SourceContext`。

通过监视 `Commands.run` 来伪造 CLI 调用。也要监视 `withSignal`，因为它通常会返回一个新的 `Commands` 实例。

```ts
function host(overrides: Partial<SourceContextDeps> = {}) {
  const store = new Store(join(mkdtempSync(join(tmpdir(), 'webhook-')), 'db.sqlite'))
  store.saveSettings(SettingsSchema.parse({
    sources: { webhook: { enabled: true, config: { conversations: [{ id: 'ops', listen: 'mentions' }] } } },
    policy: { sources: { webhook: { allowed: [{ id: 'alice' }] } } },
  }))
  // The registry claims the kind's health entries on the board, where the test reads them.
  const board = new HealthBoard()
  const sources = new SourceRegistry(() => store.settings(), board)
  sources.register(webhookKind)
  const engine = new Engine({ repo: store, scope: sources, policy: () => settingsPolicy(store.settings(), sources),
    assessor: { request: vi.fn() }, executor: { start: vi.fn(), deliver: vi.fn(), stop: vi.fn() },
    outbox: { send: vi.fn(), status: () => undefined, withdraw: vi.fn() } })
  const commands = new Commands(() => testDeployment(), {}, new AbortController().signal)
  const receive = vi.fn((trigger: Trigger) => { const result = engine.receive(trigger); engine.evaluateDue(); return result })
  // The Source's own notices go to the outbox, which keeps their receipts and sends nothing here.
  const deps: SourceContextDeps = { store, engine, outbox: { send: message => { store.receipts.add(message) } }, commands, sources,
    deployment: testDeployment(), ready: Promise.resolve(), configured: () => true, receive, ...overrides }
  return { ...deps, store, engine, commands, receive, health: board.all, ctx: sourceContextOf(deps, webhookKind) }
}
/** The queue as `hookq` answers it. */
function queue(h: ReturnType<typeof host>, answer: () => unknown) {
  vi.spyOn(h.commands, 'withSignal').mockReturnValue(h.commands)
  return vi.spyOn(h.commands, 'run').mockImplementation(async () => JSON.stringify(answer()))
}

it('takes in what the queue holds after the cursor, once, and moves the cursor after it', async () => {
  const h = host()
  queue(h, () => ({ me: 'agent', items: [message('m1'), { id: 'broken' }, message('m2', { conversation: 'elsewhere' })], next: 3 }))
  expect(await pollQueue(h.ctx, new AbortController().signal)).toEqual({ read: 3, unreadable: 1 })
  expect(await h.receive.mock.results[0]!.value).toBe('stored')
  expect(await h.receive.mock.results[1]!.value).toBe('filtered')
  expect(h.store.get(KV.webhook.cursor)).toBe(3)
  // An allowed sender who mentions the agent makes an explicit request.
  expect(h.engine.candidate(key('webhook', 'm1'))?.strength).toBe('explicit')
  await pollQueue(h.ctx, new AbortController().signal)
  expect(await h.receive.mock.results[2]!.value).toBe('duplicate')
  h.store.close()
})
```

`message(id, fields)` 用于构造队列条目。至少应覆盖：

- 翻译：稳定的 id、线程、操作者、`human` 与 `addressed`，以及智能体自己的消息不产生触发；
- 一轮轮询：触发只存一次，范围外的一条为 `filtered`，游标在它们之后推进，名单内发送者的提及被判为 `explicit`；
- 被中途打断的一轮：游标保持不变；
- `send`：成功返回 `sent`，仅 410 返回 `gone`，其他任何失败都抛出，CLI 的错误由 `commandError` 构造；
- 线程上下文：某一页的消息与下一页的游标，按 `tests/registry.spec.ts` 通过 `SourceRegistry.threadContext` 读取某个 kind 的线程的方式来验证；
- 插件：`apply` 注册 kind，并在 Setup 等待期间报告 `setupPending`；它的销毁函数注销 kind；逃离循环的失败会变成来源的健康状态，而不是未处理的拒绝；
- 策略中该来源的部分及其准入行，按 `tests/registry.spec.ts` 与 `tests/policy.spec.ts` 检查飞书来源的方式来验证。

### 检查清单

1. 映射触发、线程、操作者、渠道地址与接收范围。
2. 编写 kind：`schema`、`fields`、`health`，以及若 Setup 会派生的话，`facts`。
3. 把事件翻译成带稳定 id 的触发，并丢弃智能体自己的消息。
4. 在 `scope` 中仅依据 `meta` 决定该来源接收什么。
5. 给它一个渠道，其 `send` 对每条消息只发布一次，且只有对话永不可能收到时才回答 `gone`。
6. 若它能够读取线程，就给研判提供它的线程上下文。
7. 给它策略中属于自己的部分，含 `allowed` 与准入行。
8. 注册它的执行环境变量。
9. 编写插件入口和一个循环：等待 Setup、遵循开关、在 `receive` 之后推进游标、按条目保留错误、报告健康并在中止时停止；在 `KV` 中命名这些游标。
10. 添加构建入口、导出与 `cordis.patch.yml` 插入，并在 007 的表格中列出该模块与其测试。
11. 像 `tests/sources.spec.ts` 测试其他来源那样测试它。
