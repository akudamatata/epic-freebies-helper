# 维护日志

这个文件用于记录项目内的重要修复、行为调整、兼容性变更和文档规则更新。

## 维护要求

- 以后任何 AI 或人工在这个仓库里做了实际改动，只要会影响运行逻辑、错误处理、文档说明、排障方式或用户使用预期，都需要在本文件追加一条记录。
- 每条记录至少写清楚：日期、问题现象、根因判断、改动文件、结果。
- 不要覆盖旧记录，只追加。
- 如果只是纯格式化、无行为变化的小改动，可以不记；但只要修了 bug、改了流程、改了提示文案、补了排障规则，就需要记。

---

## 2026-04-27

## 2026-04-28

### 默认 GitHub Actions 调度改为每周一次

- 现象：
  - 之前 workflow 默认是每天运行一次。
  - 对大多数普通用户来说，这个频率偏高，不符合 Epic 周免按周刷新的节奏，也会额外消耗 GitHub Actions 分钟数。
- 根因判断：
  - 默认 schedule 更适合放在 Epic 周免刷新之后，并按周执行。
  - 同时需要在文档里明确告诉用户：如果他们想改成别的时间，应直接修改 workflow 里的 cron。
- 改动文件：
  - `.github/workflows/epic-gamer.yml`
  - `.github/workflows/README.md`
  - `.github/workflows/README.en.md`
  - `README.md`
  - `README.en.md`
  - `docs/maintenance-log.md`
- 处理结果：
  - 默认 GitHub Actions 调度改为 `20 15 * * 4`，对应 `UTC 周四 15:20` / `北京时间周四 23:20`。
  - 将“为什么默认按周跑”“如何自己改 cron”写入中英文 GitHub Actions 文档。
  - 在中英文 README 的功能概览中同步说明默认是每周四运行一次，并支持自行调整。

### Device not supported 弹窗再次导致领取失败

- 现象：
  - 商品页点击 `Get` 后没有进入 checkout，日志连续出现 `Purchase button ... click returned without visible progress`
  - 最终保存 `purchase_debug/click_no_effect-*.txt`
  - 调试文本里已经能看到 `Device not supported`、`Cancel`、`Continue`
- 根因判断：
  - 代码虽然已有 `Device not supported` 弹窗处理逻辑，但它只会在“点击已经被判断为有进展”之后才执行。
  - 本次页面在弹出 `Device not supported` 后，没有被 `_has_purchase_progress()` 识别成有效进展，导致流程在点击阶段就提前判失败，后续弹窗处理根本没有接上。
- 改动文件：
  - `app/services/epic_games_service.py`
  - `README.md`
  - `README.en.md`
  - `AGENTS.md`
  - `docs/maintenance-log.md`
- 处理结果：
  - 新增 `_is_device_not_supported_visible()`，把 `Device not supported` + `Cancel` + `Continue` 组合识别为明确的阻塞弹窗状态。
  - `_has_purchase_progress()` 现在会把该弹窗视为“点击后的有效进展”，从而继续进入现有的弹窗消除逻辑，而不是过早返回 `click_no_effect`。
  - `_handle_device_not_supported_modal()` 改为复用同一套可见性判断，避免前后条件不一致。

### 增加维护日志要求

- 现象：
  - 项目修复较多，历史经验容易散落在 issue、对话和零散文档里，后续 AI 或维护者很难快速知道哪些坑已经修过、为什么这么改。
- 根因判断：
  - 仓库里缺少一个明确的“变更必须登记”入口，也没有在项目指令里要求后续改动同步记录。
- 改动文件：
  - `AGENTS.md`
  - `docs/maintenance-log.md`
- 处理结果：
  - 新增本文件作为统一维护日志。
  - 在 `AGENTS.md` 中增加要求：后续所有会影响行为、排障或用户预期的改动，都必须同步写入本文件。

### Epic 隐私政策确认页导致登录后流程异常

- 现象：
  - 登录验证码通过后，日志显示 `Login success` / `Right account validation success`
  - 但后续在读取 `//egs-navigation` 时超时
  - 当前 URL 落在 `/id/login/correction/privacy-policy`
- 根因判断：
  - 这是 Epic 账号状态问题，不是 `GLM`、`Gemini` 或 `AiHubMix` 接口问题。
  - 部分账号登录后需要先手动确认一次隐私政策页面，脚本之前把它误当成普通商城页超时。
- 改动文件：
  - `app/services/epic_authorization_service.py`
  - `app/services/epic_games_service.py`
  - `README.md`
  - `README.en.md`
  - `.github/workflows/README.md`
  - `.github/workflows/README.en.md`
- 处理结果：
  - 识别 `privacy-policy correction` 特殊页面并给出明确错误提示。
  - 文档中补充说明：用户需要先用正常浏览器手动登录 Epic，完成该确认页后再重跑 workflow。

### checkout 多选题与 challenge router 的 GLM 返回格式兼容问题

- 现象：
  - `Device not supported` 修复后，流程已经能进入 checkout。
  - 但 checkout 安全校验阶段又出现新的结构化解析失败：
    - `ImageAreaSelectChallenge` 缺少 `challenge_prompt` / `points`
    - challenge router 有时只返回 `image_drag_multi`
  - 最终日志表现为 `instant_checkout_unconfirmed` 或 checkout solve loop 反复失败。
- 根因判断：
  - `llm_adapter` 之前主要补了 drag/drop 的若干变体，但还没有覆盖 area-select 返回的矩形框数组。
  - 同时 challenge router 返回的 `image_drag_multi` 是别名，当前路由识别只收 `image_drag_multiple`，导致 fallback 解析失败。
- 改动文件：
  - `app/extensions/llm_adapter.py`
  - `docs/maintenance-log.md`
- 处理结果：
  - 新增 area-select 矩形框归一化，把 `[[x_min, y_min, x_max, y_max], ...]` 和 `[{x_min, ...}, ...]` 转成 schema 需要的 `points`。
  - 给缺失的 `challenge_prompt` / `inferred_rule` 自动补默认值。
  - 为 `image_drag_multi` 增加别名映射，统一归一化成 `image_drag_multiple`。
