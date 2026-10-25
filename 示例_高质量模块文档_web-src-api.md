# 示例：高质量模块文档基准（web/src/api）

> 来源：dada-study-agent 仓库、`transcript-memory-map` worktree 的 `web/src/api/AGENTS.md`，经目录代码审查产出并作为格式基准。
> 用途：`目录代码审查L1_主编排.md` 第 8 节（`PROJECT_DOC` 结构）的**达标示例**。仅作参照，不含本仓库规则；原文件路径与内容保持原样，便于对照。
> 引用时请注意：下文所有路径均相对 dada-study-agent 仓库，不是本仓库路径。

---

# `web/src/api` — same-origin REST 客户端与形状校验层

Study 当前链路：`initializeChildChat` 解析 study 摘要；`study.ts` 负责 open/read/pause 与批次视图；`sendChildChat` 只提交 `{text}`，并校验自由聊天、固定学习命令及 active study 答案响应中的 `reply_text`、`batch`、`phase`、`next`、`sentence_done`、`attempt_count`、`total`。active study 答案只经 `/api/agent/child-chat` 提交；页面把服务端 transcript 反馈和服务端批次题面作为普通对话气泡展示，不使用题卡或 batch/next 答题端点，不做客户端判题或补造题面。句子题仅投影 `reference_zh + prompt`，不暴露参考答案字段。

## 1. 单元范围（C1）

- **负责**：把浏览器 `fetch` 封装成带固定失败语义的 same-origin 客户端；对服务端 JSON 做白名单形状校验与类型投影；提供跨页面复用的端点函数与响应类型。
- **不负责**：身份、角色、family、scope、permission、会话状态、题面、判分与学习档案状态——这些一律由服务端决定，本层只做透传与拒绝。
- **不负责**：页面状态、路由跳转、错误文案排版（由 `web/src/chat`、`web/src/child`、`web/src/parent` 的页面与 `FailureNotice` 承担）；页面若直接 `fetch` 同一端点，其响应解析不在本目录。
- **不负责**：鉴权。凭据只存在于浏览器自动附带的 same-origin httpOnly cookie；本层不读、不写、不回显任何 cookie/token。

## 2. 文件索引（C2）

| 文件 | 职责 | 对外入口（导出符号） | 主要依赖 | 主要消费者 |
| --- | --- | --- | --- | --- |
| `http.ts` | 只读 GET 公共层、固定失败文案、基础类型守卫 | `UNAVAILABLE`、`JsonResponse`、`getJson`、`isRecord`、`isStr`、`isStrOrNull`、`isBool`、`isNum` | 浏览器 `fetch` | 本目录 `auth.ts`、`users.ts`、`materials.ts`、`sessionLifecycle.ts`、`sessionTranscript.ts`、`childChat.ts` |
| `auth.ts` | 登出 | `logout` | `http.isRecord` | `chat/ProviderSidebar.tsx` |
| `sessionActivity.ts` | 服务端会话活跃刷新 | `touchSession` | 浏览器 `fetch` | `chat/ProviderWorkspace.tsx`、`child/ChildLayout.tsx` |
| `sessionLifecycle.ts` | `/api/capabilities` + `/api/session` 读取与形状校验 | `Principal`、`PrincipalRole`、`SessionLifecycle`、`parseSessionLifecycle`、`parseBootstrapPermissions`、`fetchSessionLifecycle` | `http`、`sessionTranscript` 的 `SessionStatus` | `App.tsx`；`chat/AgentChatPage.tsx`、`chat/ProviderSidebar.tsx`、`chat/ProviderWorkspace.tsx`、`chat/TtsTestPanel.tsx`、`chat/useChatPermissions.ts`（类型）；`parent/UserManagementPage.tsx`、`parent/OutlineManagementPage.tsx` |
| `sessionTranscript.ts` | `/api/session/transcript` 读取与形状校验 | `SessionStatus`、`MessageRole`、`SessionMeta`、`TranscriptMessage`、`SessionTranscriptResult`、`parseSessionTranscript`、`fetchSessionTranscript` | `http.getJson` | `chat/AgentChatPage.tsx`、`parent/useParentSession.ts`；类型另见 `child/useChildSession.test.ts` |
| `childChat.ts` | 孩子聊天初始化（含 study 摘要）、历史及主动材料预览载荷、单轮自由聊天与固定学习指令响应 | `ChildChatMessage`、`ChildChatMaterialUpdate`、`ChildChatSession`、`ChildChatHistory`、`ChildChatInitialization`、`initializeChildChat`、`parseChildChatHistory`、`fetchChildChat`、`sendChildChat` | `http`、`study.ts`、`materials.ts` 的材料类型 | `child/useChildSession.ts`、`child/ChildChatPage.tsx` |
| `study.ts` | 孩子隐藏学习会话 open/read/pause、批次视图与聊天响应题目形状校验 | `StudySession`、`StudyResult`、`BatchView`、`BatchNext`、`parseBatchQuestion`、`parseBatchNext`、`parseBatchView`、`parseStudySummary`、`openStudy`、`fetchStudy`、`pauseStudy` | `http` | `childChat.ts`、`child/ChildChatPage.tsx` |
| `voice.ts` | 共用语音合成与 capabilities 中 TTS 布尔字段白名单解析 | `synthesizeSpeech`、`TtsCapabilities`、`parseTtsCapabilities` | `http`、浏览器 `fetch` | `study.ts`；供聊天语音调用方复用 |
| `voice.test.ts` | 共用语音请求、4000 字边界、失败返回 null、capabilities 严格布尔校验 | 无（`vitest` 测试文件） | `voice.ts`、`vitest` stub global `fetch` | 仅测试运行 |
| `deviceWorkspace.ts` | 设备工作区与设备 transcript 读写 | `DeviceRole`、`DeviceWorkspace`、`DeviceTranscriptMessage`、`getDeviceWorkspace`、`getDeviceTranscript`、`appendDeviceTranscript` | 浏览器 `fetch`（显式 `credentials: 'same-origin'`） | `child/ChildLayout.tsx`、`chat/AgentChatPage.tsx` |
| `virtualDevices.ts` | 超管虚拟设备创建、进入、撤销、退出 | `VirtualDeviceResult`、`createVirtualDevice`、`enterOwnedVirtualDevice`、`enterVirtualDevice`、`revokeVirtualDevice`、`exitVirtualDeviceSession` | `http` | `chat/ProviderWorkspace.tsx`（`enterOwnedVirtualDevice`）、`child/ChildLayout.tsx`（`exitVirtualDeviceSession`） |
| `materials.ts` | 设备端学习资料预览/确认/列表/详情、当前 session 已绑定材料读取与家长监管纲要读删 | `MaterialUnitType`、`MaterialItem`、`MaterialGroup`、`MaterialTheme`、`MaterialPreview`、`MaterialSummary`、`MaterialDetail`、`previewChildMaterial`、`confirmChildMaterial`、`listChildMaterials`、`fetchChildSessionMaterials`、`getChildMaterial`；保留旧 `Outline*` 类型及遗留函数 | `http` | `child/ChildChatPage.tsx`、`child/MaterialCard.tsx`、`chat/AgentChatPage.tsx`；`parent/OutlineManagementPage.tsx`（旧 outline 列表/详情/删除） |
| `users.ts` | 家庭成员、密码、绑定选项、家庭 token | `FamilyUser`、`UsersResult`、`parseUsers`、`listUsers`、`BindOption`、`BindOptionsResult`、`parseBindOptions`、`listBindOptions`、`InitialCredentials`、`CreateUserResult`、`createUser`、`CreateOptionsResult`、`parseCreateOptions`、`readCreateOptions`、`getUserPassword`、`SetUserPasswordResult`、`setUserPassword`、`renameUser`、`updateUser`、`setUserStatus`、`deleteUser`、`deactivateUser`、`FamilyTokenResult`、`readFamilyToken`、`refreshFamilyToken`、`assignFamilyToken`、`assignFamilyTokenToOne` | `http` | `parent/UserManagementPage.tsx`；`parent/DeviceLockPage.tsx`（`listBindOptions`、`BindOption`） |
| `memories.ts` | 家庭成员记忆列表、乐观并发编辑、成员级清空 | `MemoryKind`、`MemoryEntry`、`MemoryListResult`、`listMemories`、`updateMemory`、`clearMemories` | `http`、same-origin `fetch` | `parent/MemoryManagementPage.tsx` |
| `childChat.test.ts`、`study.test.ts`、`materials.test.ts`、`negativeGuards.test.ts`、`sessionLifecycle.test.ts`、`sessionTranscript.test.ts`、`users.test.ts`、`memories.test.ts` | 端点请求合同、失败语义、形状守卫与白名单投影 | 无（`vitest` 测试文件） | 被测模块、`vitest` stub global `fetch` | 仅测试运行 |

本目录没有 barrel/index 文件，也没有独立配置文件；所有导入都指向具体模块文件。

Study/API 验证入口：`npm --prefix web run test -- src/child src/api`；构建入口：`npm --prefix web run build`。二者是前端自动化验证，不代替浏览器或 E2E 验收。

## 3. 改动定位（C3）

| 需求关键词 / 意图 | 该改的文件 | 具体符号 | 必须遵守的约束 |
| --- | --- | --- | --- |
| 新增只读端点客户端 | 已有语义最接近的模块，必要时新增文件 | `getJson` | 只发 same-origin GET；不得在 URL/header/body 携带身份、角色、scope、token |
| 新增写端点客户端 | `users.ts`、`materials.ts`、`virtualDevices.ts`、`childChat.ts` 中语义相近者 | `mutateJson`、`familyTokenRequest`、`submitUserMutation` 或模块内私有 helper | 成功必须同时校验 `ok === true` 与 `committed === true`；失败返回固定语义，不得伪造成功 |
| 统一失败文案 | `http.ts` | `UNAVAILABLE` | 所有失败分支复用该常量；页面据此渲染「暂时不可用」 |
| 新增/修改会话字段 | `sessionLifecycle.ts` | `parseSessionLifecycle`、`SessionLifecycle`、`parsePrincipal` | 缺失字段按 `null`；出现但类型错误时整体判失败；权限缺失只能收紧不能放宽 |
| 新增匿名首轮授权 | `sessionLifecycle.ts` | `fetchSessionLifecycle`、`parseBootstrapPermissions` | `/api/capabilities` 与 `/api/session` 并行读取；两者失败都不得扩大授权 |
| 新增 transcript 字段 | `sessionTranscript.ts` | `parseSessionTranscript`、`parseSessionMeta`、`parseMessage`、`TranscriptMessage` | 任何一条消息非法即整体失败，不返回部分消息 |
| 孩子聊天历史/turn | `childChat.ts` | `initializeChildChat`、`fetchChildChat`、`sendChildChat`、`parseChildChatHistory` | initialize 解析服务端摘要；历史消息可选 `action_summary` 与 `material_update`；材料载荷仅在完整合法且 import id 与 action 匹配时投影，transcript 主动预览的 `item_ref` 按 `materials.ts` 同样的正整数白名单投影；`sendChildChat` 只提交非空、≤4000 的 `{ text }`，学习响应字段逐项形状校验；不得发送 history、role、scope 或 system prompt |
| 孩子隐藏学习会话 | `study.ts`、`childChat.ts` | `openStudy`、`fetchStudy`、`pauseStudy`、`parseBatchView`、`parseBatchNext` | study 读取和控制使用固定 same-origin 端点；active study 答案走 `sendChildChat`；成功要求 `ok === true`，学习字段和批次题面形状错误整份落到固定失败；句子题只白名单投影 `reference_zh` 与 `prompt`，不保留参考答案 |
| 共用语音合成 / TTS 能力字段 | `voice.ts`、`voice.test.ts` | `synthesizeSpeech`、`TtsCapabilities`、`parseTtsCapabilities` | 合成仅 POST `/api/voice/synthesize`，body 仅 `{ text }`；空白或原文长度超过 4000 不请求；HTTP/JSON/网络/信封失败一律返回 `null`；只有 `tts_available`、`tts_auto_play` 均为布尔时才投影，缺失或非法返回 `null`，由调用方决定回落，不补默认值 |
| 设备工作区 / 设备 transcript | `deviceWorkspace.ts` | `getDeviceWorkspace`、`getDeviceTranscript`、`appendDeviceTranscript` | 只投影白名单字段（`id`/`role`/`family_id`/`device_id`/`expires_at`）；丢弃服务端附带的额外字段 |
| 设备端材料预览与确认 | `materials.ts` | `previewChildMaterial`、`confirmChildMaterial` | POST 仅接受 pending + `import_id`（保留 committed:true 内联兼容）；pending 是正常长状态，客户端不设失败期限，按 500ms 串行轮询到 ready/failed 或传输/形状错误，可由 AbortSignal 取消；ready 预览严格校验，失败不造卡。预览项可选投影正整数 `item_ref`（非法则整份失败）；确认仅提交 `{ import_id }`，或在提供有效非空排除项时增加去重后的 `exclude_item_refs`，需 `committed === true`、材料 id 字符串数组；缺少 `excluded_count` 按 0，非法值失败 |
| 设备端材料列表与详情 | `materials.ts` | `listChildMaterials`、`getChildMaterial` | 只读取当前设备 scope 的服务端材料摘要/详情；整批校验，失败返回 `null` |
| 当前 session 绑定材料恢复 | `materials.ts` | `fetchChildSessionMaterials` | `GET /api/child/materials/session` 只读；客户端不传 session/scope/role；整批校验，失败返回 `null` |
| 旧纲要预览与确认兼容函数 | `materials.ts` | `previewMaterial`、`confirmMaterial` | 函数保留为遗留客户端接口；其服务端旧 POST 路由已移除，不作为当前孩子端材料链路使用 |
| 纲要列表/详情/删除 | `materials.ts` | `listOutlines`、`listOutlinesResult`、`getOutline`、`deleteOutline` | 列表失败语义有两个入口：`listOutlines` 返回空数组，`listOutlinesResult` 返回 `null`；页面按是否能区分失败选择 |
| 家庭成员列表与增删改 | `users.ts` | `parseUsers`、`listUsers`、`createUser`、`updateUser`、`setUserStatus`、`deleteUser` | `GET /api/users` 必须带 `committed` 信封；任何一条成员非法即整批失败；成员 `role`/`status` 只透传服务端字符串 |
| 家庭成员学习记忆 | `memories.ts` | `listMemories`、`updateMemory`、`clearMemories` | 列表整批形状校验；编辑要求 `ok + committed + entry` 并提交 `expected_revision`；409 使用固定冲突文案；清空只接受已提交的非负整数数量；不提供单条删除。后端目标限认证家长本人或同家庭孩子，其他 parent 404；Web 页面路由为 `/settings/memories` |
| 需要一次性初始密码 | `users.ts` | `createUser`、`parseInitialCredentials`、`InitialCredentials` | 凭据只在内存中短暂存在；字段缺失或类型非法按「无凭据」处理，绝不臆造 |
| 新增成员入口授权 | `users.ts` | `readCreateOptions`、`parseCreateOptions` | 角色清单必须来自服务端；空列表或含非字符串即失败，不得本地补默认角色 |
| 密码编辑入口探测 | `users.ts` | `getUserPassword`、`setUserPassword` | 探测只认 `ok + committed` 信封；写失败不改变成员列表状态 |
| 绑定目标选择 | `users.ts` | `listBindOptions`、`parseBindOptions`、`BindOption` | 只渲染服务端返回的 `id`/`name`/`role`；不接受昵称或角色推断 |
| 家庭 token 读取/刷新/写回 | `users.ts` | `readFamilyToken`、`refreshFamilyToken`、`assignFamilyToken`、`assignFamilyTokenToOne` | token 只经请求体传输，绝不进入 URL/查询串/日志；刷新每次生成新值，与超管的幂等生成是两个端点 |
| 虚拟设备进入/退出 | `virtualDevices.ts` | `enterOwnedVirtualDevice`、`exitVirtualDeviceSession` | 设备 id 以服务端返回为准，绝不回显请求参数 |
| 超管虚拟设备管理 | `virtualDevices.ts` | `createVirtualDevice`、`enterVirtualDevice`、`revokeVirtualDevice` | 仅服务端 `is_super` 才有权限；客户端只校验 `committed` + 服务端 id |
| 登出 / 会话活跃 | `auth.ts`、`sessionActivity.ts` | `logout`、`touchSession` | 请求体为空；`logout` 仅在服务端确认 `ok + committed` 时返回 true，失败保留会话 |

## 4. 业务规则与不变量（条件区块：本层定义形状校验规则）

1. **身份只来自服务端**：请求 URL、header、body 不得出现 role、scope、scope_token、token、Authorization；违反后果是绕过服务端静态身份映射（根不变量 2/4）。
2. **成功信封**：只读端点要求 `ok === true`；产生持久化或动作提交的端点还要求 `committed === true`。缺一即按失败处理。
3. **固定失败语义**：任何 HTTP 非 ok、JSON 解析失败、网络异常、形状非法都落到 `暂时不可用`、`false`、`null` 或空数组，绝不返回半成品或伪造成功。
4. **整批校验**：列表类响应（成员、纲要、transcript、设备消息）中任何一条非法即整响应失败，不返回部分结果。
5. **缺字段收紧、错类型收紧**：可选字段缺失按 `null`/空处理；`principal.is_super` 与 `virtual_device` 出现非布尔值时整体判失败；`permissions` 缺失或非数组时按空数组处理（只收紧入口，不放宽授权）。
6. **白名单投影**：响应对象只拷贝显式列出的字段，服务端附带的额外字段（如 `scope_token`、内部列名）不得进入组件状态。
7. **动态 ID 编码**：路径参数必须 `encodeURIComponent`；凭据、token、密码只允许出现在 POST/PUT 请求体。
8. **设备端材料身份边界**：新材料请求不发送身份字段；确认请求体只含 `import_id` 和可选的去重 `exclude_item_refs`，family 与设备 scope 由服务端会话决定。排除引用只能是 1 起的整数且最多 200 个；客户端不发送重复或非法引用。
9. **状态词表**：`SessionStatus` 固定为 `active` / `finalized` / `expired`，`MessageRole` 固定为 `user` / `assistant`，非词表值一律判失败。`childChat.ts` 的 `ChildChatSession.status` 按服务端字符串透传、不做词表校验；改动该字段时需要同时决定是否收紧。

## 5. 端到端数据流（条件区块：本层位于请求链路）

- **家长入口与权限**：`App.tsx` / `chat/ProviderWorkspace.tsx` → `fetchSessionLifecycle()` → `GET /api/capabilities`（匿名首轮授权）+ `GET /api/session`（principal、permissions、session 元数据）→ 页面据服务端字段决定入口显隐与状态条；读取失败返回 `ok:false`，页面渲染固定失败。
- **家长聊天历史**：`chat/AgentChatPage.tsx` / `parent/useParentSession.ts` → `fetchSessionTranscript()` → `GET /api/session/transcript` → `messages` 按 `seq` 排序渲染。家长本轮发送由页面直接 `POST /api/agent/chat`，不经过本目录。
- **孩子入口**：`child/ChildLayout.tsx` → `getDeviceWorkspace()` + 直接读取 `GET /api/session`（`virtual_device`、`principal.is_super`、`principal.name`）；后者不经本目录，新增字段时须同步该调用点。
- **孩子聊天**：`child/ChildChatPage.tsx` 挂载先调用 `initializeChildChat()` → `POST /api/agent/child-chat/initialize`，成功确认 active session 后再调用 `fetchChildChat()` → `GET /api/agent/child-chat`；发送 → `sendChildChat()` → `POST /api/agent/child-chat`（前端只发 `{ text }`）→ 再次拉取历史渲染。`child/useChildSession.ts` 仍提供历史消息到流项目的映射；`chat/AgentChatPage.tsx` 的孩子分支走另一条链路：`getDeviceTranscript()` / `appendDeviceTranscript()` + 直接 `POST /api/agent/child-chat`。
- **材料**：`child/ChildChatPage.tsx`、`chat/AgentChatPage.tsx` 孩子分支 → `previewChildMaterial()` → `POST /api/child/materials/preview`（multipart，返回 pending 与 `import_id`）→ pending 是正常长状态，客户端按 500ms 串行轮询 `GET /api/child/materials/preview/{import_id}`，不设超时；仅 `ready` 且结果通过形状校验时成功，`failed`、任务丢失（如 404）、HTTP/形状错误失败，AbortSignal 取消时静默停止 → `confirmChildMaterial(importId, excludeItemRefs?)` → `POST /api/child/materials/confirm`（可选排除项）→ `getChildMaterial()` 读取详情。材料卡只在资料子 agent 成功并由服务端返回合法 ready 预览后出现。预览返回的 `item_ref` 只在服务端提供时投影，详情与 session 材料缺省不要求该字段。家长纲要管理仍由 `listOutlines()` / `getOutline()` / `deleteOutline()` 读取与删除旧 `material_outlines`；旧材料 preview/confirm POST 路由已移除。
- **成员与家庭 token**：`parent/UserManagementPage.tsx` → `listUsers()` / `readCreateOptions()` → 写操作（`createUser`、`updateUser`、`setUserStatus`、`deleteUser`、`setUserPassword`）→ 成功后重新 `listUsers()`；token 面板走 `refreshFamilyToken()` → `assignFamilyTokenToOne()` → 重新 `listUsers()`。
- **虚拟设备**：`chat/ProviderWorkspace.tsx` → `enterOwnedVirtualDevice()` → `POST /api/superadmin/virtual-devices/enter` → 跳转 `/child/chat`；退出由 `child/ChildLayout.tsx` 调 `POST /api/device/virtual-session/exit`。

## 6. 对外合同表（条件区块：跨模块类型/常量/函数）

| 导出符号 | 端点 | 形状与语义 | 消费者 | 变更影响面 |
| --- | --- | --- | --- | --- |
| `UNAVAILABLE` | — | 常量 `暂时不可用` | 本目录各客户端与测试 | 改文案会同时影响所有失败提示与测试断言 |
| `JsonResponse` | — | `{ ok, status, payload }` 类型 | 仅本目录类型声明 | 无外部消费者 |
| `getJson` | 任意只读路径 | GET + JSON 解析；异常折半为 `{ ok:false, status:0, payload:null }` | `users.ts`、`sessionLifecycle.ts`、`sessionTranscript.ts`、`childChat.ts` | 改动影响上述只读客户端（`materials.ts` 与 `deviceWorkspace.ts` 各自直接 `fetch`） |
| `isRecord` / `isStr` / `isStrOrNull` / `isBool` / `isNum` | — | 基础类型守卫（`isNum` 额外要求 `Number.isFinite`） | 本目录全部解析器 | 收紧会改变所有形状校验结果 |
| `synthesizeSpeech` / `synthesizeStudySpeech` | `POST /api/voice/synthesize` | 共用合成 / 保留签名的学习题面兼容入口；仅 HTTP ok、`ok === true`、非空字符串 `audio_base64` 返回音频，否则 `null`，不抛异常 | `study.ts`、题目 TTS；供聊天复用 | 可选语音失败不阻断文本路径 |
| `TtsCapabilities` / `parseTtsCapabilities` | 纯函数，解析 `GET /api/capabilities` 载荷 | 仅投影布尔 `tts_available` / `tts_auto_play`；任一缺失或非法返回 `null`，不补默认值 | 供语音调用方使用 | 回落行为由调用方决定 |
| `logout` | `POST /api/logout` | 无请求体；`ok && committed` → true | `chat/ProviderSidebar.tsx` | 登出按钮的会话保留/跳转行为 |
| `touchSession` | `POST /api/session/touch` | 要求 `ok`、`session_id`、`status === 'active'`、`last_active_at`、`finalized_at === null` | `chat/ProviderWorkspace.tsx`、`child/ChildLayout.tsx` | 活动节流与 `dada:session-activity` 事件触发 |
| `PrincipalRole` / `Principal` | `GET /api/session` | 角色 `parent`/`child`；`id`、`family_id` 必填，`username?`、`is_super` 布尔 | `Principal` 类型经 `SessionLifecycle` 间接消费 | 新增字段须同步服务端与 `parsePrincipal` |
| `SessionLifecycle` | `GET /api/session` + `/api/capabilities` | `ok`、`principal`、`permissions`、可选 session 元数据、`idle`、`virtual_device`、`message`、`bootstrap_permissions?` | `App.tsx`、`chat/*`（类型与 `useChatPermissions`）、`parent/*` | 类型增删影响 10 余个页面的入口显隐 |
| `parseSessionLifecycle` | 纯函数 | 校验 `/api/session` 载荷；principal 非法即失败 | 仅模块内 + 测试 | 测试直接断言失败态结构 |
| `parseBootstrapPermissions` | 纯函数 | 从 capabilities 载荷取字符串数组，否则 `[]` | 本模块内无调用点 | 改动只影响未来调用方 |
| `fetchSessionLifecycle` | `GET /api/capabilities` + `GET /api/session` | 并行读取；无条件携带 `bootstrap_permissions` | `App.tsx`、`chat/AgentChatPage.tsx`、`chat/ProviderSidebar.tsx`、`chat/ProviderWorkspace.tsx`、`chat/TtsTestPanel.tsx`、`parent/UserManagementPage.tsx`、`parent/OutlineManagementPage.tsx` | 影响启动门、设置页入口权限、会话状态条 |
| `SessionStatus` / `MessageRole` | `GET /api/session*` | 状态与角色的固定词表 | `sessionLifecycle.ts`（类型）、页面类型 | 词表变更需同步服务端状态机 |
| `SessionMeta` / `TranscriptMessage` / `SessionTranscriptResult` | `GET /api/session/transcript` | 会话元数据、消息（`seq`/`role`/`text`/`created_at`/`action_summary?`）、结果信封 | `chat/AgentChatPage.tsx`、`parent/useParentSession.ts` | 影响家长聊天历史渲染与 `seq` 排序 |
| `parseSessionTranscript` | 纯函数 | 校验 transcript；任一消息非法即失败 | 仅模块内 + 测试 | 失败态结构被测试断言 |
| `fetchSessionTranscript` | `GET /api/session/transcript` | 401 时返回带 `unauthenticated: true` 的失败结果；其它失败不带该标记 | `chat/AgentChatPage.tsx`、`parent/useParentSession.ts` | 现有调用方只判断 `ok`，新增消费者若区分「未识别身份」需自行读取该字段 |
| `ChildChatMessage` / `ChildChatMaterialUpdate` / `ChildChatSession` / `ChildChatHistory` | `GET /api/agent/child-chat` | 消息 `seq`/`role`/`text`/`action_summary?`/`material_update?`；主动预览为 `{kind:'preview',import_id,status:'ready',preview}`；会话 `session_id`/`status`/`last_active_at`/`finalized_at` | `child/useChildSession.ts`（消息类型）、`child/ChildChatPage.tsx` | 影响孩子历史消息及材料卡显示 |
| `parseChildChatHistory` | 纯函数 | 校验基础 transcript 并按 `seq` 升序；完整合法且 `action_summary === child_material_preview:{import_id}` 的 assistant 预览载荷才保留，否则忽略卡片字段并保留消息文本 | 仅模块内 + `childChat.test.ts` | malformed 可选卡不导致文本历史失败 |
| `initializeChildChat` | `POST /api/agent/child-chat/initialize` | 成功需 HTTP ok、`ok === true`、`session.session_id` 为字符串且 `session.status === 'active'`；失败返回 `false` | `child/ChildChatPage.tsx` | 孩子端挂载前建立服务端孩子聊天 session |
| `fetchChildChat` | `GET /api/agent/child-chat` | `ok:false` 时返回固定失败结果 | `child/ChildChatPage.tsx`、`child/useChildSession.ts` | 孩子历史读回与失败提示 |
| `sendChildChat` | `POST /api/agent/child-chat` | 只发 `{ text }`（非空、≤4000），前端不发送 `outline_id`；返回白名单字段 `replyText`、`study`、`batch`、`phase`、`next`、`sentenceDone`、`attemptCount`、`total`；不会投影参考答案字段 | `child/ChildChatPage.tsx` | 自由聊天、固定命令和 active study 答案；服务端拒绝或可选字段形状非法均固定失败 |
| `DeviceRole` / `DeviceWorkspace` / `DeviceTranscriptMessage` | `GET /api/device/workspace`、`GET /api/device/transcript` | 工作区 `id`/`role`/`family_id`/`device_id`/`expires_at`；消息 `id`/`role`/`text`/`created_at` | `chat/AgentChatPage.tsx`（仅 `DeviceRole`、`DeviceTranscriptMessage`；`DeviceWorkspace` 只在模块内作返回类型） | 设备端身份展示与历史渲染 |
| `getDeviceWorkspace` | `GET /api/device/workspace` | 白名单投影；`expires_at` 必须为有限正数 | `child/ChildLayout.tsx`、`chat/AgentChatPage.tsx` | 决定设备上下文是否被认作已授权 |
| `getDeviceTranscript` | `GET /api/device/transcript` | 逐条校验 `id`/`created_at`/`content.role`/`content.text` | `chat/AgentChatPage.tsx` | 设备聊天历史渲染与刷新 |
| `appendDeviceTranscript` | `POST /api/device/transcript` | `{ content }` 直传；`ok && committed` → true | `chat/AgentChatPage.tsx` | 设备 transcript 写入；失败不回填 |
| `VirtualDeviceResult` | — | `{ ok, deviceId, message }` | 模块内与调用方返回类型 | 超管进入流程的错误展示 |
| `createVirtualDevice` | `POST /api/superadmin/virtual-devices` | 需 `committed` 与字符串 `id` | 本目录内无调用点 | 若接入需同时处理 `id` 与 `device_id` 命名差异 |
| `enterOwnedVirtualDevice` | `POST /api/superadmin/virtual-devices/enter` | 需 `committed` 与字符串 `device_id` | `chat/ProviderWorkspace.tsx` | 超管「进入孩子端」入口 |
| `enterVirtualDevice` | `POST /api/superadmin/virtual-devices/{deviceId}/session` | 需 `committed` 与字符串 `device_id`；不回显请求 id | 本目录内无调用点 | 按设备 id 进入的备用路径 |
| `revokeVirtualDevice` | `DELETE /api/superadmin/virtual-devices/{deviceId}` | `ok && committed` → true | 本目录内无调用点 | 撤销虚拟设备 |
| `exitVirtualDeviceSession` | `POST /api/device/virtual-session/exit` | `ok && committed` → true | `child/ChildLayout.tsx` | 虚拟设备返回家长端 |
| `MaterialUnitType` / `MaterialItem` / `MaterialGroup` / `MaterialTheme` / `MaterialPreview` / `MaterialSummary` / `MaterialDetail` | `GET/POST /api/child/materials/*` | 设备端分类材料形状；`MaterialItem.item_ref?` 仅在响应提供时白名单投影，出现时必须为 >=1 的整数；缺省不影响预览、详情与 session 材料 | `child/ChildChatPage.tsx`、`child/MaterialCard.tsx`、`chat/AgentChatPage.tsx` | 新材料卡片渲染与列表字段 |
| `previewChildMaterial` | `POST /api/child/materials/preview` + `GET /api/child/materials/preview/{import_id}` | 签名 `previewChildMaterial(file, { signal? })`；multipart `image`；POST 只接受 pending + `import_id`，兼容合法 `committed:true` 内联预览；pending 是正常长状态，按 500ms 间隔串行轮询且不设客户端失败期限；ready 校验 `result`，failed、任务丢失、HTTP/形状错误固定失败，abort 返回 `aborted:true` 并停止 | `child/ChildChatPage.tsx`、`chat/AgentChatPage.tsx` | 设备端上传预览与失败提示 |
| `confirmChildMaterial` | `POST /api/child/materials/confirm` | `confirmChildMaterial(importId, excludeItemRefs?)`；仅在非空时发送去重 `exclude_item_refs`（整数 1..200）；需 `ok === true`、`committed === true` 与字符串数组 `material_ids`（允许空数组，表示本次无新材料入库）；缺少 `excluded_count` 投影为 0，出现时须为非负整数 | `child/ChildChatPage.tsx`、`chat/AgentChatPage.tsx` | 新资料落库、可选排除条目与后续详情读取 |
| `listChildMaterials` / `getChildMaterial` | `GET /api/child/materials/outlines[/{id}]` | 列表整批校验；详情按材料 id 读取，失败返回 `null` | `child/ChildChatPage.tsx`、`chat/AgentChatPage.tsx` | 设备端已确认材料列表与详情 |
| `previewMaterial` / `confirmMaterial` | 已退役的 `POST /api/materials/preview`、`POST /api/materials/confirm` | 遗留客户端函数保留；服务端不再提供这两个 POST 路由 | 无当前孩子端消费者 | 不用于新材料链路 |
| `getOutline` | `GET /api/materials/outlines/{id}` | 家长监管旧 outline 详情，成功返回详情，否则 `null` | `parent/OutlineManagementPage.tsx` | 旧纲要详情展示 |
| `listOutlines` | `GET /api/materials/outlines` | 失败返回 `[]`，不区分空列表与失败 | `parent/OutlineManagementPage.tsx` | 失败会被渲染成「暂无纲要」 |
| `listOutlinesResult` | `GET /api/materials/outlines` | 失败返回 `null`，可区分失败 | `materials.ts` 的 `listOutlines`、`parent/OutlineManagementPage.tsx`（经 `listOutlines`）、`materials.test.ts` | 列表读取的可区分失败语义 |
| `deleteOutline` | `DELETE /api/materials/outlines/{id}` | 需 `committed` 与 `deleted_id` 字符串；不要求与请求 id 相同 | `parent/OutlineManagementPage.tsx`、`materials.test.ts` | 家长端只按 `ok` 处理；孩子端当前不调用 |
| `FamilyUser` / `UsersResult` | `GET /api/users` | 成员 `id`/`name`/`role`/`status`/`created_at`/`has_family_token`/`family_token` | `parent/UserManagementPage.tsx` | 成员表格与 token 行渲染 |
| `parseUsers` | 纯函数 | 需 `ok + committed + users[]`；任一条非法即整批失败 | 仅模块内 + 测试 | 成员列表形状收紧会影响所有行 |
| `listUsers` | `GET /api/users` | 失败返回固定失败结果 | `parent/UserManagementPage.tsx` | 成员列表刷新 |
| `BindOption` / `BindOptionsResult` | `GET /api/devices/bind-options` | 选项 `id`/`name`/`role` | `parent/DeviceLockPage.tsx` | 绑定账号选择器 |
| `parseBindOptions` / `listBindOptions` | `GET /api/devices/bind-options` | 需 `committed` 信封；任一条非法即失败 | `listBindOptions` 由 `parent/DeviceLockPage.tsx` 使用 | 绑定目标必须来自服务端清单 |
| `InitialCredentials` / `CreateUserResult` | `POST /api/users` | 一次性 `username`/`password`，缺失按 `null` | `parent/UserManagementPage.tsx` | 新建家长的凭据展示与清除 |
| `createUser` | `POST /api/users` | `{ name, role }`；需 `committed`；凭据从响应顶层读取 | `parent/UserManagementPage.tsx` | 新建成员与一次性凭据 |
| `CreateOptionsResult` / `parseCreateOptions` / `readCreateOptions` | `GET /api/users/create-options` | 需 `committed` 与非空字符串角色数组 | `parent/UserManagementPage.tsx` | 新增成员弹窗是否可打开 |
| `getUserPassword` | `GET /api/users/{id}/password` | `ok && committed` → true；忽略响应中的其他字段 | `parent/UserManagementPage.tsx` | 密码编辑入口展开 |
| `SetUserPasswordResult` / `setUserPassword` | `PUT /api/users/{id}/password` | `{ password }`；需 `committed`；`username` 缺失或非法按 `null` | `parent/UserManagementPage.tsx` | 密码保存与登录名回显 |
| `renameUser` | `PATCH /api/users/{id}` | `{ name }`，`ok && committed` → true | 本目录内无调用点 | 与 `updateUser` 共享同一端点 |
| `updateUser` | `PATCH /api/users/{id}` | `{ name?, status? }`，需 `committed` | `parent/UserManagementPage.tsx` | 成员改名与状态切换 |
| `setUserStatus` | `PATCH /api/users/{id}` | 委托 `updateUser`，只传 `status` | `parent/UserManagementPage.tsx` | 启用/停用按钮 |
| `deleteUser` / `deactivateUser` | `DELETE /api/users/{id}` | `ok && committed` → true；`deactivateUser` 是 `deleteUser` 的别名 | `deleteUser` 由 `parent/UserManagementPage.tsx` 使用 | 成员移除（服务端为软删除） |
| `FamilyTokenResult` | — | `{ ok, token }` | 模块内与调用方 | token 面板状态 |
| `readFamilyToken` | `GET /api/family/score-token` | 需 `committed`；`token` 可为 `null` | 本目录内无调用点 | 读取家庭 token |
| `refreshFamilyToken` | `POST /api/family/score-token/refresh` | 每次返回新值；需 `committed` | `parent/UserManagementPage.tsx` | 打开 token 面板即轮换 |
| `assignFamilyToken` | `PUT /api/family/score-token/assignments` | `{ user_ids, token? }`；返回服务端已应用成员 id 列表 | 模块内（被 `assignFamilyTokenToOne` 复用） | 批量写入/解绑 |
| `assignFamilyTokenToOne` | `PUT /api/family/score-token/assignments` | 单成员 + 显式 token（空串=解绑）；仅在返回包含该成员时成功 | `parent/UserManagementPage.tsx` | 成员行 token 保存 |

## 7. 已知限制与约定（导航用）

- 本目录不提供 `/api/child-switch/users`、`POST /api/family/score-token`（超管幂等生成）的客户端；需要时新增函数，不要假定现有函数已覆盖。
- `GET /api/session` 的 `principal.name`、`temp` 字段不经过本目录，由 `web/src/child/ChildLayout.tsx` 直接读取。
- 家长本轮聊天发送、孩子 `POST /api/agent/child-chat` 在 `web/src/chat/AgentChatPage.tsx`、`web/src/parent/useParentSession.ts` 内直接 `fetch`，不经过 `sendChildChat`。
- 已退休的 `/child/entry`、`/child/review`、`/child/dialogue` 在本目录没有任何客户端或类型；不要把它们重新接回。

## 8. 验证方式（C4）

依赖安装完成后在本目录所属仓库执行：

```powershell
npm --prefix web run test -- src/api src/child
npm --prefix web run build
npm --prefix web run test:e2e -- e2e/study.spec.ts e2e/material-upload.spec.ts
```

- Vitest 配置为 `jsdom`、只包含 `src/**/*.test.{ts,tsx}`、各测试自行 stub `globalThis.fetch`，不发真实网络请求。
- 关键断言覆盖点：请求路径与方法、请求体不含身份/角色/scope/token、`ok`/`committed` 信封门槛、malformed 载荷落到固定失败、列表整批拒绝、白名单投影丢弃额外字段、家庭 token 只出现在请求体。
- 若依赖未安装，测试与构建会直接失败；此时只能做静态核对，不得用放宽断言或伪成功替代真实结果。

## 词/短语跨批幂等（当前实现）

materials.ts 的预览响应项不含 status；预览解析允许缺省并省略该字段，若出现则只允许 pending/active/duplicated，详情/会话绑定材料仍要求合法 status。MaterialItem.status 为可选字段，预览不补造状态；MaterialPreview.duplicates 为 null 或 {caption,items:[{english,meaning_zh,unit_type}]}，经白名单投影与形状校验。parseMaterialPreview 接受 materials=[]；confirmChildMaterial 在 ok+committed 下接受 material_ids=[] 并返回成功。空数组不能替代 ok/committed 成功门禁；前端不自行判重或推断复习结果。验证：`npm --prefix web run test -- src/api/materials.test.ts`。

已确认详情/会话项 status 由服务端数据库真实值提供，客户端仍要求 pending/active/duplicated；回归覆盖 active 的详情与会话响应，缺失或非法 status 仍整批拒绝。

## Transcript 主动材料预览卡

`GET /api/agent/child-chat` 的 transcript 消息字段名为 `action_summary` 与 `material_update`。服务端主动材料消息使用 `action_summary=child_material_preview:{import_id}`，并附 `material_update={kind:'preview',import_id,status:'ready',preview}`；`preview` 沿用 `MaterialPreview` 字段合同。

`parseChildChatHistory` 对可选材料卡执行完整字段校验：外层必须是 ready preview，action summary 必须与外层 `import_id` 精确匹配，内层 preview 必须完整且内外 `import_id` 相等；材料分组、条目、主题和重复项均按 `materials.ts` 的预览形状校验。失败状态、缺字段、错形状或关联 id 不符时只丢弃 `material_update`，保留合法的 `seq/role/text`，让文本 transcript 与 Composer 保持可用。该投影不生成 import id，也不解析文本内容。

验证：`childChat.test.ts` 覆盖合法主动预览载荷及缺失/非法载荷下文本保留；页面使用服务端返回的 `import_id` 调既有 `confirmChildMaterial`。

## 学习对话化（2026-10-23）

`BatchView` 严格解析服务端批次状态与当前题面：词/短语题只投影 `{item_ref, context_en, prompt}`，句子题只投影 `{item_ref, reference_zh, prompt}`；即使载荷含额外英文参考答案字段，也不会进入返回对象。`sendChildChat` 对服务端聊天响应可选的 `reply_text`、`batch`、`phase`、`next`、`sentence_done`、`attempt_count`、`total` 执行形状校验；出现非法值时按固定「暂时不可用」失败，未提供的可选字段不补造。`ChildChatPage` 的 active study 答案只调用 `POST /api/agent/child-chat` 并只发送 `{text}`；成功后重新读取已提交 transcript，题面从服务端 `batch.current` 或 `next.question` 投影为普通 tutor 气泡，批次进度保持纯文本。旧 batch/next 答题客户端已删除；客户端不评分、不合成题面或反馈。验证：`npm --prefix web run test -- src/child src/api`、`npm --prefix web run build`。
