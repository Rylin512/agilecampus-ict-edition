# 实现约定

本文记录 AgileCampus 中「光看代码容易踩错」的口径与不变量。面向修改本仓库的开发者；用户使用说明见 [README](../README.md)。

## 目录

- [时区列的读写口径](#时区列的读写口径)
- [工作台与全域常驻栏](#工作台与全域常驻栏)
- [助手会话的两种 scope](#助手会话的两种-scope)
- [右侧助手面板](#右侧助手面板)
- [助手的页面跳转](#助手的页面跳转)
- [助手的建团与入团](#助手的建团与入团)
- [助手的多动作与读权限](#助手的多动作与读权限)
- [助手的子任务与附件](#助手的子任务与附件)
- [助手的发送、转发、建群与邀请](#助手的发送转发建群与邀请)
- [助手的团队资源占用与标签](#助手的团队资源占用与标签)
- [助手的「上文」](#助手的上文)
- [审批流与系统消息](#审批流与系统消息)
- [云资源 dry-run](#云资源-dry-run)
- [数据库与时间列](#数据库与时间列)

## 时区列的读写口径

站内所有时间列都是 `timestamp without time zone` —— 存的是**钟面读数**，本身不带偏移，光看数据无从知道那串 `18:19` 属于哪个时区。PG 自己写时间（`now()` / `defaultNow()`）时落的是**库会话时区**的钟面，所以应用的读写也都按会话时区换算：`src/db/schema.ts` 的 `dbTimestamp` / `dbTimestampNow`（连上库时用 `SHOW timezone` 探明一次，见 `src/db/index.ts`）。

这样**库会话时区配成什么都不错**：`docker-compose.prod.yml` 与 `Dockerfile` 都不设 `TZ`，两个容器同为 UTC，照旧；运维把库配在 +08 也一样对。

- **千万别对已运行过的库就地改时区**：老行按旧时区落盘、新行按新时区落盘，同一列里混进两种含义 —— 那是改坏数据，不是修 bug。
- 新增时间列**必须走 `dbTimestamp` / `dbTimestampNow`**，别直接用 drizzle 自带的 `timestamp()`：后者把无时区值一律当 UTC 读（`new Date(值 + "+0000")`），库会话不是 UTC 时整站差一个偏移。物理类型两者完全相同（都是 `timestamp`），故 `db:push` 不会产生 ALTER；`tests/db-timestamp.test.ts` 把这两条都围住了。
- `src/lib/time.ts` 是纯函数层（`tests/time.test.ts` 覆盖跨时区/夏令时/日界）。注意 `node:22-alpine` 官方镜像是 full-ICU；若产物退化成 small-ICU，时区换算会**静默**退化为 UTC。

## 工作台与全域常驻栏

### 工作台自定义（`users.dashboard_layout`）

工作台布局存 `users.dashboard_layout`（jsonb，可空）：`NULL` = 从未自定义（默认布局），`{ v: 1, items: [{ id, type, x, y, w, h, config? }] }` = 用户布局，`items: []` = 主动把卡片删空。`id` 是**实例**主键（第一张 `note`，之后 `note-2`、`note-3`…），`type` 是卡片种类；老文档没有 `type` 时按 `id` 推断（`note-2` → `note`）。列刻意不标类型（照 `messages.tool_calls` 先例），读写在 `src/lib/dashboard-layout-store.ts` 收口，进出都过一次 `normalizeLayoutDoc`：坏数据回落默认、未知/重复实例 id 丢弃、尺寸与坐标夹取、重叠卡片向下推开（不自动回填空隙，允许留白）。

- **纯函数引擎** `src/lib/dashboard-layout.ts` 不碰 IO、不引 zod，客户端拖拽预览与服务端落库共用同一套规则；zod 只在 server action `src/app/(app)/dashboard/actions.ts` 校验信封，身份只取 session，不接受客户端 userId。
- **拖拽语义**：拖到哪格占哪格，重叠的卡片向下让位；位置按放下时算，可以自由留白（删卡也不牵动其他卡片）。缩放不低于目录里的最小尺寸、不超出 12 列。拖动时被拖的卡钉在起点、用 `transform` 连续跟手（网格吸附只决定其他卡片怎么让位），缩放时宽高按指针连续变化；松手后由 FLIP 过渡（180ms）滑入吸附位，其他卡片被挤开、归位同样走这条过渡。卡片内的滚动区域隐藏滚动条（仍可滚动）。
- **可重复卡片是实例化的**：便签 / 时钟 / 倒计时每次「添加卡片」都追加一个新实例（配置按实例各存一份，倒计时与专注计时互不串味）；其余卡片单实例，重复添加不会产生第二张。专注计时是纯客户端状态，按实例 id 存 `localStorage`，刷新后继续走。
- **卡片目录**：看板、四张单项统计、最新记录、我的项目、即将到期、我的团队、消息、快捷入口、项目里程碑、团队资源占用、最近附件、待办审批、专注计时，以及可重复的便签 / 时钟 / 倒计时。原「任务概览（四合一）」已移除（旧的 `task_summary` 文档项在 `normalizeLayoutDoc` 里被丢弃）。
- **时钟设置**：可开农历（`Intl.DateTimeFormat` 的中国农历日历 + 中文日名），世界时钟最多 3 个时区（`config.zones`，服务端做去重与截断，超过上限的丢弃）。
- **保存不 revalidate**：编辑态下整页数据回流会用服务端旧数据覆盖本地未保存布局。请求串行 + latest-wins，飞行期间的新改动合并成待发；失败保留本地布局，工具栏可重试。
- **窄屏只读**：视口 `<1024px` 或正文容器 `<720px`（打开右侧助手抽屉）时按 `y,x` 堆叠，编辑入口隐藏，编辑中也会自动退出。卡片数据由服务端一次性加载，加卡不需要再请求。

### 全域常驻栏（`users.rail_layout`）

「全域常驻栏」是每个登录页面左侧的一条常驻卡片栏（`src/app/(app)/rail/`），放工作台卡片：

- 卡片在任何页面都可正常交互；栏头有**自己的**「编辑」按钮，与工作台编辑互不影响。
- 布局存在 `users.rail_layout`，缩放随布局一起存；默认收起，收起 / 展开按账号存 `users.rail_collapsed`，服务端首屏就按上次的选择渲染。
- 可与助手抽屉同时展开；`<1024px` 由 CSS 整条隐藏（手机端不做浮层）。

## 助手会话的两种 scope

`conversations` 一表两态，由 `CHECK` 钉住形状：`kind='project'` ⇒ `project_id` 非空且 `owner_id` 为空（全员共享）；`kind='personal'` ⇒ 反之（严格按人隔离）。`/api/chat` 与 `/api/chat/commit` 的 `scope` 缺省为 `project`（老客户端不受影响）。

`personal` 时 `projectId` 可选，含义是**焦点项目**：它只影响这一轮——提示词末尾多一段「当前项目」快照，工具面多放出 `list_project_activity` / `list_comments` / `list_chat_history` / `create_subtask` / `delete_subtask` / `delete_attachment` 六件（个人基础工具 36 件 → 42 件）——**不改变会话归属**，落库仍是本人的个人会话（请求体可带 `conversationId` 指定；缺省落到最近一条，没有才新建）；权限在编排入口先按 `getProjectForUser` 校（无权 403，且不会先落一条空会话）。不带 `projectId` 时没有隐含项目，涉及项目的工具必须由模型在草案里显式给 `projectId`，**落库时逐条重校 `getProjectForUser`**（越权即 403，属预期而非 bug）。个人回合的响应会附 `options`（按草案里的 projectId 给成员/里程碑下拉，无权项目跳过），供草案卡渲染选择项。

- **`actorId` 只来自 session**：两处路由的请求体里没有身份字段，草案里夹带 `userId` 在 lib 层被丢弃。
- 个人会话**每人可以有多条**：`owner_id` 上是普通索引（`conversations_owner_idx`），不再是唯一约束。并发安全改由 `pg_advisory_xact_lock(hashtextextended(actorId, 0))` 保证（`resolvePersonalConversation` / `startPersonalConversation` / 清空记录都在同一把锁下）——**不要**为了防并发再加回唯一索引。
- `conversations.last_message_at`（可空）记录最近一次落消息的时间，会话列表按 `coalesce(last_message_at, created_at) DESC` 排序；老数据为空时回落到创建时间。
- `messages.turn_id`（可空）把同一轮的提问与回答绑在一起：删任意一条 = 删整轮。老数据没有 turn_id，删除时在 SQL 里按**原始** `created_at` 相等配对 —— 读出来的 `Date` 已被截到毫秒，绝不能拿它反查（见 `lib/time.ts` 的说明）。
- 个人会话 REST：`POST /api/chat/conversations`（新建/复用空会话）、`GET|PATCH|DELETE /api/chat/conversations/[id]`（取消息/改名/删除）、`DELETE /api/chat/messages/[id]`（成对删一轮）。全部只认 session 身份，别人的资源一律 404。
- 个人上下文存在 `personal_contexts`；`visibility='shared'` 的条目会进**项目助手**的快照，也会进右侧助手在项目页时的「当前项目」段（`shared` 是默认值：写之前请知道队友看得到，`private` 才只在本人快照里）。
- 快照对上下文有字符预算与降级链（任务明细 → 里程碑明细 → 上下文），改 `snapshot.ts` 时别打乱这个顺序。

## 右侧助手面板

- 入口是顶栏的「AI 助手」按钮（`src/components/assistant-shell.tsx`）：桌面端（≥1024px）把正文挤窄一列，移动端覆盖 + 遮罩；Escape、遮罩点击与面板右上角 × 都能收起。打开状态不跨刷新持久化。
- 面板**常驻挂载**，收起只是移出视口 + `inert`：`AgentChat` 的 `useState(initialMessages)` 只播种一次，开合与切换页面都不会重放或丢掉消息。切页时面板状态也保留。
- 面板必须是 `<header>` 的**兄弟节点**：顶栏带 `backdrop-blur`，会成为 `fixed` 后代的包含块，放进 header 里定位就错了。
- 焦点项目从 `pathname` 推（`src/lib/project-path.ts`，只认 `/projects/<uuid>`，大小写 uuid 都认）。焦点不是会话归属：切换项目不会开新会话；草案卡的「继续处理后续动作」按**出卡时**那个项目续做（每条消息各自记着焦点）。
- 面板顶部是会话列表（`src/components/assistant-conversations.tsx`）：切换、新建、行内改名、删除。列表走**正常文档流**（抽屉带 `transform`、宇宙主题还有 `backdrop-filter`，absolute/fixed 弹层会被裁切）。「新对话」会复用最近一条空会话，连点不会堆空行。
- 「上次看的是哪条」记在 cookie `ac_assistant_conversation`（客户端写、layout 读），服务端校验归属后才用；没有或无效就回落到最近一条。整页刷新后仍停在原会话。
- `AgentChat` 用 `viewKey` 重新播种：只在**显式**切换/新建/删除当前会话/重置时递增。发送时后端替新用户懒建会话**不递增** —— 否则刚发出去的消息连同输入框会一起被 remount 掉。
- 每条消息旁的删除按钮删掉**同一轮**的提问与回答（`messages.turn_id`；老数据按原始 `created_at` 配对）。删到空会话后 `last_message_at` 归 NULL，列表退回按创建时间排序。删除**不撤销**已确认落库的改动。
- **助手正文按 Markdown 渲染**（`src/components/markdown.tsx`）：react-markdown + GFM，支持粗体、列表、任务清单、代码块与表格；默认**不渲染原始 HTML**，链接一律新窗口打开，远端图片只给可点开的链接；表格与代码块在窄面板里横向滚动。用户自己的提问仍按纯文本显示，免得 `*`、`#` 被吃掉。单个换行由 remark-breaks 转成 `<br>`，模型整段平铺时不会粘成一行。
- **「继续处理后续动作」不展示提示词**：发出去的是给模型看的自包含提示词（原请求复述 + 续做规矩），面板里只显示一行「继续上一轮请求」。识别靠 `CONTINUATION_PREFIX` 开头标记，刷新、切会话后同样生效（不给消息表加字段）；再续一次时从提示词里取回最初那句人话，不会把提示词套娃进「原请求」。
- 每条会话一次取最近 200 条消息（`listPersonalConversationMessages`），列表最多 100 条 —— 都不做分页/「加载更早」。
- 助手设置（记忆条数 + 个人上下文 + **删除全部聊天记录**）在「AI 设置」页（`/settings`），面板标题行的「设置」直达。重置只删本人的 personal 会话（项目共享会话不动）；面板靠 `window` 事件 `ac:assistant-reset` 立即清空。

## 助手的页面跳转

用户说「打开工作台」「跳到赤壁演习的关系图」「带我看看东吴实验室的资源占用」时，助手用 `open_page` 替他把右侧面板所在的页面换过去 —— 它**只换页面，不读数据**，也不产生确认卡（只读、立即生效）。

- **目标是白名单枚举**（`NAV_TARGETS`）：`dashboard` / `projects` / `teams` / `messages` / `settings`，项目四页（`project` / `project_graph` / `project_flow` / `project_timeline`），`task`，团队四页（`team_members` / `team_projects` / `team_resources` / `team_labels`）。**路径由服务端拼**（`src/lib/agent/navigation.ts`），模型只报目标与 id，不能自由发挥 URL。
- **id 与权限当场校验**：项目页走 `getProjectForUser`、团队页走 `getTeamForMember`、任务页走 `getTaskDetail`（按任务实际所属项目）；无权一律 403，页面不会被打开。缺 id 报可操作的错，提示先 `list_tasks` / `get_task_detail` 取回。
- **返回形态**：工具返回 `{ __navigate: true, label, href }`；编排器把各步的跳转意图按 `href` 去重后随响应下发 `navigations`。面板**只跳最后一个**（连开两页没有意义），并在该条回答下留「打开…」链接，方便回头再点。
- 面板挂在 `(app)/layout.tsx` 里、跨路由不卸载，所以跳转不会丢失当前对话或未确认的草案卡。
- 工具只在个人 scope 出现：项目助手没有「换页」的语义。

## 助手的建团与入团

- `create_team({ name })`：拟「创建团队」草案，确认后经 `createTeam` 落库，建团者自动成为 admin。
- `join_team({ inviteCode })`：拟「加入团队」草案。**邀请码只能来自用户这句话**（快照里没有邀请码，提示词也禁止猜测）；拟稿时先按码查出团队名，卡片显示「加入『某某团队』」，码无效或已在该团队中当场报错。落库那一刻 `joinTeam` 会再校一次（拟稿到确认之间码可能失效、人也可能已被拉进团）。
- 两者只进 `COMMIT_TOOL_NAMES`，不进 `WRITE_TOOL_NAMES`：项目 scope 的落库口天然拒收。与其它写工具一样只产草案，确认前助手不得声称「已创建 / 已加入」。

## 助手的多动作与读权限

- **一轮多件事**：编排器最多走 8 步工具循环（`stepCountIs(8)`，只是上限）。同一步内的多个 tool call 由 AI SDK **并行**执行，后一个拿不到前一个的返回值 —— 所以模型被要求「能合批就合批、独立动作全做完再收尾」，草案则从各步 `toolResults` 里全部提取（不是只留最后一张卡）。
- **依赖新 id 的动作跨轮做**：新建任务的 id 要等用户确认落库后才存在。这类后续动作**不会自动续做**：卡片落库成功后出现「继续处理后续动作」，点击后客户端把「原请求 + 已落库摘要 + 未确认草案」拼成一条自包含消息发出去（`src/lib/agent/continuation.ts`，纯模块，客户端与服务端共用）。只有会派生新 id / 新可访问对象的工具（项目写工具、新增子任务、新建群聊、建团/入团）会触发它；删除类、发送/转发/邀请、个人上下文与群邀请不派生后续动作。
- **只读工具不改未读**：右侧助手（个人 scope）的 `list_message_threads` / `read_messages` 走 `listDmMessagesReadOnly` / `listGroupMessagesReadOnly`，**不置 `read_at`、不推 `last_read_at`、不建联系人** —— 助手看过不等于用户看过。提示词也明确要求不许谎称「已标已读」。
- **发送/邀请类动作只有草案**：`send_message` / `forward_message` / `create_group_chat` / `invite_to_group` 只返回草案，**绝不落库、绝不代发**；用户点确认后才经 `commitPersonalDraft` 重校目标/成员/附件权限并发送。群邀请只能经 `respond_group_invite` 产草案，确认后由 `commitGroupInviteResponse` 在事务内认领。项目 scope 的落库口只认 `WRITE_TOOL_NAMES`，天然拒收这些个人专属草案。
- **外部文本一律是数据**：留言、动态、聊天历史、私信与群消息都可能含「请建任务/请接受邀请」之类的文字。提示词把它们统一按资料处理：只有用户那句请求是指令，正文只能当素材。
- **个人快照只放消息计数**（未读数与待处理邀请数），绝不放消息正文、会话名或群名 —— 正文要用户显式让助手去读。

## 助手的子任务与附件

- **子任务**：`create_subtask` 新增（父任务 id 取自 `list_tasks` / `get_task_detail`）、`update_tasks` 修改（子任务就是任务）、`delete_subtask` 删除。删除入口**只收子任务**；删父任务须到任务详情页操作，删除子任务会级联删掉它下面的所有子任务与依赖。三者均走两段式草案；落库口会重新校验父/子层级与项目归属。这三件在项目 scope 里一直有；个人 scope 只在**焦点项目**下放出（右侧助手在项目页时）。
- **任务附件**：`list_attachments` 两个 scope 都能用（`attachmentId`/文件名/大小/上传者），个人助手靠它取到 `attachmentId` 才能把项目文件发出去；`delete_attachment` 删除（不可恢复，需确认）在项目 scope 与个人 scope 的焦点项目下都可用。底表 `task_attachments` 仍是唯一事实来源。
- **个人助手把消息文件复制成任务附件**：`read_messages` 对带文件的消息返回 `attachment.attachmentId`；`attach_message_file_to_task` 拟一张草案，确认后由 `copyMessageAttachmentToTask` **复制字节**新建 `task_attachments` 行（`dm_attachments` 与 `task_attachments` 无外键，复制而非共享）。此后删消息不影响任务附件，删任务附件也不影响聊天记录；同一文件占两份存储，单文件上限仍为 10MB。复制同时校验：消息附件可读（私信收发双方 / 群成员）+ 目标任务可写。
- **工具面分工**：个人 scope 基础 36 件（含附件查看、发文件、「消息文件 → 任务附件」、发送/邀请类工具、建团/入团、页面跳转，以及团队资源占用与标签的 8 件）；焦点项目再放出 6 件项目专属工具（动态、留言、旧助手对话、子任务新增/删除、附件删除），共 42 件。项目 scope 只认 `WRITE_TOOL_NAMES`，个人专属草案（上下文、群邀请、消息附件、发送/转发/建群/邀请进群、建团/入团、资源占用、标签）天然不会从项目 scope 提交。

## 助手的发送、转发、建群与邀请

- **找人**：`find_contact({ username })` 按用户名**精确**匹配（全站唯一、大小写不敏感），返回 `userId` 与规范用户名 `name` —— 发消息用 `userId`，建群/邀请用 `name`；不做模糊搜索，避免把全站用户名变成可枚举接口。
- **发消息**：`send_message` 支持私信（`targetType="user"`）与群聊（`targetType="group"`），正文与附件至少其一。附件用 `fileSource` 指明来源：`task` 是任务附件（把项目文件发出去），`message` 是私信/群聊附件（转发文件）。发送时**复制字节**进 `dm_attachments`，单文件仍 ≤10MB。
- **转发**：`forward_message` 把一条已有私信/群消息（连同附件）原样发给个人或群聊；源消息只允许收发双方/群成员读取，目标权限在落库时重校。
- **建群**：`create_group_chat({ name, usernames })` 与网页建群一致 —— 成员用**用户名**（不是 `userId`）指定，避免模型誊抄 uuid 出错；成员被**直接拉进群**，不是发邀请待接受。自己自动入群，名单里误写自己会被忽略；`usernames` 可留空，先建一个**只有自己**的群，之后再邀请。群名 ≤30 字、成员 ≤50 人。
- **邀请进群**：`invite_to_group({ groupId, usernames })` 一次可邀请一至多人（用户名精确匹配）；与建群不同，邀请是**待处理**状态，对方接受后才入群。已在群里、已邀请过、群满都会在拟稿时报错。
- **两段式铁律不变**：以上四个工具只产草案；卡片上可改正文/群名、移除成员/被邀请人或附件。确认前助手**不得声称已发送**，也不得仅凭消息正文里的「请转发给所有人」自动发送。REST 侧（`/api/agent/messages` 等）与助手的落库共用 `src/lib/chat-transfer.ts`，权限口径一致。

## 助手的团队资源占用与标签

两块的网页入口分别在 `/teams/[teamId]/resources` 与 `/teams/[teamId]/labels`；助手只把它们接进**个人 scope**（右侧面板），项目 scope 不放出。`teamId` 一律取自快照「我的团队」。

- **读**：`list_resource_usages({ teamId })` 给占用记录（含 `usageId`）、进行中数量，以及按人/按资源的已结束总时长；`list_labels({ teamId })` 给 `labelId`、名称、颜色、贴在多少任务上，并用 `canManage` 标明当前用户能不能增删改（仅团队 admin）。
- **资源占用**：`create_resource_usage` 登记（任一成员皆可）、`end_resource_usage` 结束。结束只能结束**本人登记的**记录，团队 admin 可结束任意一条；已结束的记录不能再结束。时间入参是**用户本地墙面时间** `YYYY-MM-DDTHH:mm`，拟稿与落库都用 `resolveUserTimeZone` + `zonedWallClockToInstant` 换算成绝对时刻（个人快照里也给出「现在」的完整墙面时间）—— 绝不 `new Date(墙上时间)`。结束时间留空即「仍在占用」。
- **标签管理**：`create_label` / `update_label` / `delete_label` 仅团队 admin（权限在 lib 层再校）；名称 ≤20 字、同团队不区分大小写去重。删除会从所有任务上撕下该标签，卡片会显示会影响多少任务。
- **给任务贴/撕标签**：`edit_task_labels({ taskId, add, remove })` 用**增量**语义，卡片勾选的是**最终全量集合**；落库走 `setTaskLabels` 的全量替换，并由 lib 重校任务写权限与标签的团队归属。跨团队标签、添加已有的、移除没有的、最终没变化都会当场报错。
- **两段式不变**：六件写工具只产草案；确认前助手不得声称已登记/已结束/已建/已删/已贴。

## 助手的「上文」

每轮的请求体是「最近 N 条聊天记录 + 本轮这句话」（`conversation.ts` 的 `recentMessagesForModel`），N 是**用户级**偏好 `users.ai_history_limit`（0 = 不看历史，默认 10，上限 50）—— **两个 scope 共用同一个数**，右侧面板与项目 scope 读的是同一条。常量与钳制在 `src/lib/agent/history-limit.ts`（纯模块，界面要引它显示上下限，**别把 `db` 拖进去**），存取在 `src/lib/agent/memory.ts`。

- 取的是**最近** N 条：先按 `created_at` 倒序取再反转。正序取 N 条那是「最早的头几条」。
- 同一轮的问与答是**同一条 INSERT** 落的，`now()` 在一条语句里取值相同，`created_at` 逐字相等 —— 故排序列里必须补一个「答在下、问在上」，否则同一轮的两句谁先谁后全看 PG 心情。
- 反转后开头若是「答」就丢掉：上下文不能以答话打头（有些模型也不接受）。
- 单条消息的长度**不做截断**（条数上限就是唯一的闸）—— 往输入框里贴长文会直接进提示词。

## 审批流与系统消息

审批流：成员发起、**拥有对应审核权限**的成员审批（任一通过即生效）。审批人资格在 `src/lib/team-permissions.ts` 收口：admin 天然拥有全部四项权限，其余成员由团队管理员在 `/teams/[teamId]/permissions` 按人勾选。

四类申请：

| 类型 | 发起 | 落库时机 |
|---|---|---|
| `task_done` | 学生把任务标成完成（含待办直跳）时由 `lib/task.ts` 拦截转申请 | 任务保持未完成直至通过；同任务同时最多一条 pending（partial unique index 兜底） |
| `resource_usage` | 无资源审核权的成员登记占用转为申请 | **通过时才建** `resource_usages` 行；有权者登记沿用直登路径 |
| `cloud_resource` | 云实例创建/释放在需要审批时转入 | 创建与释放共用同一条申请链 |
| `project_acceptance` | 成员为项目申请结题验收 | 项目保持 active 直至通过；有验收权者可在项目页直接结题；结题前任务必须全部完成（申请时与通过时双重校验） |

- 恢复进行中（`reopenProject`）只把项目置回 active，任务保持已完成。
- 通知走站内**系统消息**：申请时提醒有对应权限的人，决定后回执申请人（拒绝原因随回执带给申请人）。系统消息写入失败不阻断主流程。
- 工作台「待办审批」卡片只列我在对应团队有审核权的申请；admin 所在团队四类全列。

## 云资源 dry-run

`/cloud` 提供 RDS / Redis 的开通、审批、状态轮询与释放演示。默认 `CLOUD_DRY_RUN=true`，不会调用真实华为云接口、不会产生费用；真实模式在第二阶段接入。

- 团队权限页新增「开通/释放云资源」权限。
- 普通成员提交进入审批，有权限成员可直接创建。
- 开通向导包含区域、可用区、VPC、子网、安全组与规格白名单。
- 实例详情展示模拟内网地址、端口、操作记录。
- 外部 API：`/api/cloud/catalog`、`/api/cloud/instances`、`/api/cloud/instances/{id}`。
- 轮询端点：`POST /api/cron/cloud-jobs`（`CRON_SECRET` 保护）。
- 每团队实例数受 `CLOUD_MAX_INSTANCES_PER_TEAM` 限制（默认 2）。

## 数据库与时间列

- 表结构由 `src/db/schema.ts` 定义，直接用 `drizzle-kit push` 同步；仓库没有迁移文件，生产 compose 的 `migrate` 服务会在应用启动前自动推送。
- 时间列一律使用 `dbTimestamp` / `dbTimestampNow`，不要直接用 drizzle 的 `timestamp()`；原因见[时区列的读写口径](#时区列的读写口径)。
- 附件字节存于数据库或存储层，`task_attachments` 与 `dm_attachments` 无外键；消息附件复制为任务附件时是**复制字节**，不是共享记录。
- 账号注销是匿名化：擦除 PII、保留 `users.id` 作为墓碑，避免他人项目里的附件与动态因外键失效。

## 相关文档

- [Agent 写入 API](agent-api.md)
- [技术债与待办](BACKLOG.md)
- [README](../README.md)
