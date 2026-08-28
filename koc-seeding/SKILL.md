---
name: koc-seeding
description: 在用户要根据商品图和卖点生成 KOC 种草拼图时，默认直接创建，或按用户明确要求先准备候选方案再提交。
version: 1.0.0
author: 极睿科技（Infimind）
---

# KOC 种草拼图

## 使用时机

用于根据 1～10 张商品图、商品名和可选卖点/人群/场景生成一张 KOC 种草拼图。如用户需要封面加多张内容图及笔记文案，使用 `image-text-commerce`。

## 调用流程

### 默认直接创建

1. 收集商品图和商品名，可选收集类目、卖点、目标人群、场景和人物要求。
2. 确认版式和生成参数，仅传 HTTPS 素材 URL，说明当前 OAuth 空间和计费边界。
3. 调用 `create_koc_collage_task`。

### 仅在用户明确要求查看/选择候选方案时

1. 用同样输入调用 `prepare_koc_collage_task`，展示返回候选，不自行代替用户选择。
2. 用户选择 1～3 个方案后，用 prepare 返回的任务 ID 和 index 调用 `submit_koc_collage_task`。

两种流程最后都用创建/prepare 返回的 `taskId` 与 `taskType=koc_grid` 调用 `get_user_tasks`。超时或失败时不得自动重建任务。

## 提交前确认与任务跟踪

- 仅在用户明确要求生成或提交后创建任务；用户只咨询能力、费用、输入要求或方案时不创建任务。
- 必填或条件字段缺失、取值冲突或用户意图有歧义时，一次性询问最少必要信息并等待确认；除本文明确给出的默认值外，不猜测商品事实、素材含义、模型、尺寸、数量或付费参数。
- 创建或提交成功后记录返回的 `taskId` 和对应 `taskType`；处理中始终用同一 `taskId + taskType` 精确查询，不使用模糊列表，也不因查询重试创建新任务。
- 持续查询直到 `completed`、`failed`，或分步流程进入需要用户操作的节点；只报告当前任务的实际结果以及服务端返回的成功、失败和待处理数量。
- 本轮等待超时时告知任务可能仍在后台处理，返回并保留 `taskId` 供后续查询；不得自动重建、重复提交或追加扣费。

## 工具与参数

### `create_koc_collage_task` / `prepare_koc_collage_task`

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `images` | 是 | 1～10 张商品图 HTTPS URL |
| `productName` | 是 | 商品名 |
| `productCategory` | 否 | 商品类目 |
| `sellingPoints` | 否 | 卖点、材质、功效和价格等已确认信息 |
| `targetAudience` | 否 | 目标人群 |
| `usageScenario` | 否 | 使用场景 |
| `modelDescription` | 否 | 人物描述；提供 `aiModelId` 时建议不同时填 |
| `aiModelId` | 否 | 已授权 AI 模特 ID |
| `collageLayout` | 否 | `grid_5x2` / `grid_3x3` / `grid_2x2` |
| `promptCandidateCount` | 仅 prepare | 4 / 6 / 10，默认 6 |
| `aspectRatio` | 否 | 不填时由版式推导；`auto` 仅 GPT Image 2 支持 |
| `resolution` | 否 | `1k` / `2k` / `3k` / `4k`，服务端按模型校验 |
| `quality` | 否 | `low` / `medium` / `high` |
| `model` | 否 | 当前公开图像模型 |

### `submit_koc_collage_task`

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `taskId` | 是 | `prepare_koc_collage_task` 返回的任务 ID |
| `selectedPlanIndexes` | 是 | 用户选择的 1～3 个候选 index |

### `get_user_tasks`

| 参数 | 用法 |
| --- | --- |
| `taskId` | 直接创建或 prepare 返回的任务 ID |
| `taskType` | 固定为 `koc_grid` |
| `status` | 可选状态筛选 |
| `limit` | 精确查询时取最小值 |

## 权限、积分与安全边界

- OAuth 获取与刷新：首次连接时由 WorkBuddy 打开浏览器完成 OAuth；access token 到期后由客户端使用 refresh token 自动轮换；授权失效后重新连接。
- OAuth 绑定用户、空间和 operator，不索要 Token；只能创建、提交和查询有权限的任务。
- 仅传 HTTPS URL，不传本地路径、base64、localhost、私网/保留地址或存储内部 URL。
- 工具会在服务端重验素材、AI 模特、空间、能力和积分；客户端不替代业务事实。
- 不编造商品功效、价格、认证或人物身份，不代用户选择候选方案。

## 积分不足后的处理

- 仅当工具明确返回 `MCP_CREDITS_REJECTED` 时进入本流程；不把其他错误解释为积分不足，也不主动营销。
- 先说明任务未提交，并提供减少生成数量或降低规格的方案。若无法确认当前是个人还是企业空间，先询问，不猜测。
- 个人空间先询问是否需要官方充值入口；只有用户明确同意后，才提供 [电商内容专家官方充值页](https://imiva.ecpro.com/wallet/recharge)，并说明链接会离开 WorkBuddy，后续操作可能涉及真实支付。
- 企业空间不提供个人充值引导，优先提示联系企业管理员补充额度。
- 不自动打开链接、不代选套餐、不创建支付订单，也不自动重试、重建或重复提交任务。
- 用户明确表示“已充值”后，再次展示当前参数并取得用户确认，才可调用创建或提交工具；由服务端在该次调用中重验素材、权限和积分。现有工具不支持无副作用余额预检，不得提前声称“预检已通过”。

## 错误处理

仅返回稳定公开错误码和修正建议，不透传供应商错误、原始候选负载、存储 URL 或内部堆栈。权限、余额、素材或内容安全失败需先修正；不自动重试 submit 或创建。
