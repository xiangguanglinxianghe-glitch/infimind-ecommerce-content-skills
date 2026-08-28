---
name: video-generation
description: 在用户要用提示词和可选参考素材生成 Seedance 商品视频时，先预检估价，经用户确认后幂等创建并查询结果。
version: 1.0.0
author: 极睿科技（Infimind）
---

# 视频生成

## 使用时机

用于根据提示词、参考图/视频/音频或首尾帧生成 1～12 条 Seedance 视频。如用户要先拆解一条爆款参考视频，再为目标商品生成结构同类视频，使用 `viral-video-copy`。

## 调用流程

1. 收集提示词、模型、模式、时长、分辨率、比例、数量和可选参考素材；本远程 Connector 只传 HTTPS URL。
2. 首次调用 `create_video_generation_task`时传 `dryRun=true`。预检只校验权限、素材、能力和价格，不创建任务、不调供应商、不占用积分。
3. 向用户明确展示 `estimatedCredits`、生成数、时长、分辨率和固定后比例。用户未确认时立即停止。
4. 确认后用与 dryRun 相同的业务参数正式调用，并传入稳定唯一的 `idempotencyKey` 和用户允许的 `maxCredits`。重试网络请求时复用同一幂等键。
5. 用返回的 `taskId` 和 `taskType=video_generation` 调用 `get_user_tasks`到终态。
6. 只对 completed 结果，用该 `taskId`、结果 `resultId` 和 `taskType=video_generation` 调用 `get_video_result_download_url`。

## 提交前确认与任务跟踪

- 仅在用户明确要求生成或提交后创建任务；用户只咨询能力、费用、输入要求或方案时不创建任务。
- 必填或条件字段缺失、取值冲突或用户意图有歧义时，一次性询问最少必要信息并等待确认；除本文明确给出的默认值外，不猜测商品事实、素材含义、模型、尺寸、数量或付费参数。
- 创建或提交成功后记录返回的 `taskId` 和对应 `taskType`；处理中始终用同一 `taskId + taskType` 精确查询，不使用模糊列表，也不因查询重试创建新任务。
- 持续查询直到 `completed`、`failed`，或分步流程进入需要用户操作的节点；只报告当前任务的实际结果以及服务端返回的成功、失败和待处理数量。
- 本轮等待超时时告知任务可能仍在后台处理，返回并保留 `taskId` 供后续查询；不得自动重建、重复提交或追加扣费。

## WorkBuddy 交互式参数收集

- `ask_human` 是 WorkBuddy 宿主的交互能力，不属于本 Connector 的 16 个 MCP 工具。仅当当前运行时明确提供 `ask_human` 时调用，并严格按其运行时 schema 传参；不要把它作为远程 MCP 工具调用。
- 仅针对必填或条件字段的缺失、冲突或歧义发问；用户已提供的值和本文明确默认值不再询问。媒体素材缺失时直接提示用户上传或提供，`ask_human` 只收集文本、枚举和确认，不伪造素材 URL。
- 多个字段缺失或有歧义且 schema 支持 `question_type="form"` 时，一次性展示表单；只有一个字段时使用运行时支持的单选或选择方式。选项以当前 MCP 工具 schema 为准；服务端允许自由文本时保留“自定义”。
- 本 Skill 表单按当前请求需要时重点涵盖：画面/运镜描述、模型、模式、时长、分辨率、比例、数量和音轨设置；参考媒体仍通过附件或 HTTPS 素材提供。
- `ask_human` 不可用或调用失败时，在一条消息中给出紧凑编号选项并允许自定义回答，然后等待用户回复；不得猜值或创建任务。
- 创建或提交前展示方案摘要并请求确认；`ask_human` 可用时提供“确认提交 / 修改参数”，否则使用文本确认。用户明确说“直接生成”或“不用问”，且必填或条件字段齐全时，不再询问；视频 `dryRun` 后的估价确认仍不可跳过。
- `ask_human` 不得收集 Token、OAuth 凭据、支付信息或其他敏感数据。

## 工具与参数

### `create_video_generation_task`

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `prompt` | 是 | 视频画面、镜头、动作和商品约束 |
| `model` | 是 | `seedance_2_0` / `seedance_2_5` |
| `mode` | 是 | `all_in_one` / `first_last_frame` |
| `durationSeconds` | 是 | 2.0：5/10/15；2.5：5/10/15/20/25/30 |
| `resolution` | 是 | `480p` / `720p` / `1080p` |
| `aspectRatio` | 是 | `21:9` / `16:9` / `4:3` / `1:1` / `3:4` / `9:16` / `adaptive` |
| `outputCount` | 是 | 1～12 |
| `referenceImages` | 否 | HTTPS 图片；数量与能力按模型校验 |
| `referenceVideos` | 否 | HTTPS 视频；时长、编码与数量按模型校验 |
| `referenceAudios` | 否 | HTTPS 音频；2.0 不允许纯音频输入 |
| `generateAudio` | 否 | 是否生成音轨，默认 true |
| `dryRun` | 是（预检） | 首次必须 true；正式创建为 false/省略 |
| `idempotencyKey` | 正式必填 | 稳定客户端幂等键 |
| `maxCredits` | 正式必填 | 用户确认的最大整单积分，不小于已接受估价 |

`first_last_frame` 只接受按顺序的 1～2 张首/尾帧图，不传参考视频或音频。`adaptive` 只自适应画幅，分辨率仍必须明确指定。

### `get_user_tasks` / `get_video_result_download_url`

| 工具 | 参数 |
| --- | --- |
| `get_user_tasks` | 精确传 `taskId`、`taskType=video_generation`；`status` 和 `limit` 仅作可选筛选 |
| `get_video_result_download_url` | completed 后传 `taskId`、`resultId`、`taskType=video_generation` |

## 权限、积分与安全边界

- OAuth 获取与刷新：首次连接时由 WorkBuddy 打开浏览器完成 OAuth；access token 到期后由客户端使用 refresh token 自动轮换；授权失效后重新连接。
- OAuth 绑定用户、空间和 operator，不索要 Token；创建、查看、下载均由服务端分别鉴权。
- 只传 HTTPS URL，禁止本地路径、`file:`、`data:`/base64、localhost、私网/保留地址或供应商临时 URL。
- 素材的 MIME/magic、尺寸、时长、FPS、codec 和信任边界以服务端 probe 为准。
- 积分以 dryRun 估价、服务端 hold 和终态结算为准；不修改用户已确认的参数，不隐式追加输出或扣费。
- 不生成仿冒他人、未授权人像/品牌或违法内容；结果只从可信存储签发短时下载链接。

## 积分不足后的处理

- 仅当工具明确返回 `MCP_CREDITS_REJECTED` 时进入本流程；不把其他错误解释为积分不足，也不主动营销。
- 先说明任务未提交，并提供减少生成数量或降低规格的方案。若无法确认当前是个人还是企业空间，先询问，不猜测。
- 个人空间先询问是否需要官方充值入口；只有用户明确同意后，才提供 [电商内容专家官方充值页](https://imiva.ecpro.com/wallet/recharge)，并说明链接会离开 WorkBuddy，后续操作可能涉及真实支付。
- 企业空间不提供个人充值引导，优先提示联系企业管理员补充额度。
- 不自动打开链接、不代选套餐、不创建支付订单，也不自动重试、重建或重复提交任务。
- 用户明确表示“已充值”后，重新使用冻结的原业务参数执行 `dryRun`；展示新的 `estimatedCredits` 并再次取得用户确认后，才可正式创建。

## 错误处理

只显示稳定公开 code/category 和可操作建议，不透传供应商错误、provider URL、存储 key、提示词快照或原始媒体 URL。遇到 `PROVIDER_CONTENT_SAFETY_REAL_PERSON` 或 `VIDEO_RETRY_REQUIRES_MATERIAL_ADJUSTMENT` 时，引导用户调整/移除相关素材再新建；不自动重试或重建。
