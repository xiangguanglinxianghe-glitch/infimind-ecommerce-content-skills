---
name: product-main-image
description: 在用户要用商品素材生成一组电商主图时，创建商品主图任务并精确查询结果。
version: 1.0.0
author: 极睿科技（Infimind）
---

# 商品主图

## 使用时机

用于根据商品图、参考图或风格描述，策划并生成 5～10 张电商主图。单张精修用 `smart-refine`；长详情页用 `detail-page`；复制某张参考图的视觉结构用 `visual-migration`。

## 调用流程

1. 收集 1～5 张商品图、商品类目，并至少提供参考图或生成风格之一。
2. 确认语言、卖点、场景、数量及可选的图位/文案模式；仅传 HTTPS 素材 URL。
3. 告知将使用 OAuth 授权空间并按服务端规则计费，然后调用 `create_product_main_image_task`。
4. 保留返回的任务 ID，用 `get_user_tasks(taskId, taskType=product_main_image)` 轮询到终态。
5. 超时只停止当前查询，不得自动重建任务。

## 提交前确认与任务跟踪

- 仅在用户明确要求生成或提交后创建任务；用户只咨询能力、费用、输入要求或方案时不创建任务。
- 必填或条件字段缺失、取值冲突或用户意图有歧义时，一次性询问最少必要信息并等待确认；除本文明确给出的默认值外，不猜测商品事实、素材含义、模型、尺寸、数量或付费参数。
- 创建或提交成功后记录返回的 `taskId` 和对应 `taskType`；处理中始终用同一 `taskId + taskType` 精确查询，不使用模糊列表，也不因查询重试创建新任务。
- 持续查询直到 `completed`、`failed`，或分步流程进入需要用户操作的节点；只报告当前任务的实际结果以及服务端返回的成功、失败和待处理数量。
- 本轮等待超时时告知任务可能仍在后台处理，返回并保留 `taskId` 供后续查询；不得自动重建、重复提交或追加扣费。

## 工具与参数

### `create_product_main_image_task`

| 参数 | 必填 | 说明 |
| --- | --- | --- |
| `productImages` | 是 | 1～5 张商品图 HTTPS URL |
| `referenceImages` | 条件 | 最多 5 张参考图；与 `generationStyle` 至少提供一项 |
| `productCategory` | 是 | 用户可识别的商品类目 |
| `generationStyle` | 条件 | 生成风格；与 `referenceImages` 至少提供一项 |
| `language` | 否 | 图内文案语言，默认简体中文 |
| `coreSellingPoints` | 否 | 商品核心卖点 |
| `usageScenario` | 否 | 使用/展示场景 |
| `imageCount` | 否 | 5～10，默认 5 |
| `positionTypes` | 否 | 按输出位置排列，最多 10 项 |
| `positionCopyModes` | 否 | 与图位对应的 `auto` / `with_copy` / `without_copy` |
| `aiModelId` | 否 | 已授权 AI 模特 ID |
| `aspectRatio` | 否 | 输出比例；`auto` 仅 GPT Image 2 支持 |
| `resolution` | 否 | 公开分辨率，由模型能力再校验 |
| `quality` | 否 | `low` / `medium` / `high` |
| `model` | 否 | 当前公开图像模型 |

### `get_user_tasks`

| 参数 | 用法 |
| --- | --- |
| `taskId` | 创建返回的任务 ID |
| `taskType` | 固定为 `product_main_image` |
| `status` | 可选状态筛选 |
| `limit` | 精确查询时取最小值 |

## 权限、积分与安全边界

- OAuth 获取与刷新：首次连接时由 WorkBuddy 打开浏览器完成 OAuth；access token 到期后由客户端使用 refresh token 自动轮换；授权失效后重新连接。
- OAuth 决定用户、空间和 operator，不索要 Token；他人或其他空间的任务不可查看。
- 本远程 Connector 只传 HTTPS URL，不传本地路径、base64、localhost、私网或临时内部 URL。
- 输入数量、MIME/magic、模型能力、素材权限和积分均以服务端复核为准。
- 不虚构商品功效、价格、认证或品牌关系；不将参考图视为可复制商标或人物身份的授权。

## 积分不足后的处理

- 仅当工具明确返回 `MCP_CREDITS_REJECTED` 时进入本流程；不把其他错误解释为积分不足，也不主动营销。
- 先说明任务未提交，并提供减少生成数量或降低规格的方案。若无法确认当前是个人还是企业空间，先询问，不猜测。
- 个人空间先询问是否需要官方充值入口；只有用户明确同意后，才提供 [电商内容专家官方充值页](https://imiva.ecpro.com/wallet/recharge)，并说明链接会离开 WorkBuddy，后续操作可能涉及真实支付。
- 企业空间不提供个人充值引导，优先提示联系企业管理员补充额度。
- 不自动打开链接、不代选套餐、不创建支付订单，也不自动重试、重建或重复提交任务。
- 用户明确表示“已充值”后，再次展示当前参数并取得用户确认，才可调用创建或提交工具；由服务端在该次调用中重验素材、权限和积分。现有工具不支持无副作用余额预检，不得提前声称“预检已通过”。

## 错误处理

只显示稳定公开错误码和可操作建议，不透传供应商错误、任务内部报文或存储路径。素材、权限、余额或内容安全错误要先让用户修正；不自动重试会再次扣费的创建。
