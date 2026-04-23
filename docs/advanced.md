# 开发者进阶文档

这份文档面向想继续改项目、提 PR、或者自己二次开发的读者。

语言版本：

- 简体中文（当前页）
- [English](advanced.en.md)

如果你只是想配置并使用项目，请先看：

- [README](../README.md)
- [2026-04-22 开发纪要](development-log-2026-04-22.md)
- [README (English)](../README.en.md)
- [Development Log (English)](development-log-2026-04-22.en.md)

---

## 项目结构

| 文件 | 作用 |
| --- | --- |
| [`app/deploy.py`](../app/deploy.py) | 运行入口，负责浏览器启动、登录、领取和调度 |
| [`app/services/epic_authorization_service.py`](../app/services/epic_authorization_service.py) | 登录、登录结果监听和登录后验证 |
| [`app/services/epic_games_service.py`](../app/services/epic_games_service.py) | 周免发现、商品页进入、加购、结账、checkout 验证处理 |
| [`app/settings.py`](../app/settings.py) | 环境变量、模型路由和默认值 |
| [`app/extensions/llm_adapter.py`](../app/extensions/llm_adapter.py) | Gemini / AiHubMix / GLM 兼容适配 |
| [`.github/workflows/epic-gamer.yml`](../.github/workflows/epic-gamer.yml) | GitHub Actions 工作流入口 |

---

## 本地开发

```bash
uv sync
uv run black . -C -l 100
uv run ruff check --fix
```

说明：

1. 这个仓库当前不建议补跑测试。
2. 改动验证码链路时，优先保留日志和截图。
3. 改动 checkout 流程时，优先保证“未确认成功就不要报成功”。

---

## 这次适配里实际踩过的坑

下面这些不是猜测，都是这次实际遇到并修过的问题。

### 1. GLM 不是简单改个 Base URL 就能用

`hcaptcha-challenger` 内部调用的是 `google-genai` 风格的多模态接口。  
所以接 GLM 时，不能只把 `GEMINI_BASE_URL` 改成智谱地址。

真正要做的是保留上层调用方式，再在适配层里把图片、消息和结构化输出转换成 GLM 能接受的格式。

---

### 2. 不同验证码阶段，题型真的会变

登录阶段和 checkout 阶段的题型不一定一样。

| 阶段 | 题型 |
| --- | --- |
| 登录 | `image_drag_single` |
| checkout | `image_label_multi_select` |

如果适配层只对拖拽题做兼容，结账时就会死在第二道验证上。

---

### 3. GLM 的输出格式并不稳定

这次遇到过的返回形式包括：

| 返回形式 | 说明 |
| --- | --- |
| `Source Position: (...)` | 文本坐标 |
| `{"source": [...], "target": [...]}` | 结构化拖拽坐标 |
| `{"answer":"..."}` | 被包在 `answer` 里的字符串 |
| `image_label_multi_select` | 只返回题型名 |
| 半结构化 JSON | 字段不完整或坏掉的响应 |

所以 [`llm_adapter.py`](../app/extensions/llm_adapter.py) 现在做了很多“解包和转 schema”的兼容。

---

### 4. Epic checkout 不只会弹 hCaptcha

这次已经确认过，结账过程中可能出现下面这些状态：

| 场景 | 是否遇到过 |
| --- | --- |
| `Device not supported` | 是 |
| `One more step` | 是 |
| 额外的 checkout iframe | 是 |
| 页面仍停留在 `Place Order` | 是 |

因此 [`epic_games_service.py`](../app/services/epic_games_service.py) 现在做了这些事：

1. 检查设备不支持弹窗并尝试继续。
2. 识别 checkout 安全校验。
3. 在 `Place Order` 后循环观察真实结果。
4. 没确认成功就不误报成功。

---

### 5. “已拥有”判断不能扫整页文本

曾经误把页面里的版权文本 `owned by ...` 当成了“已拥有”。

后来修正成：

1. 优先看按钮和 checkout 状态。
2. 只识别高精度成功文案。

---

### 6. Artifact 非常重要

真正把问题定位清楚，靠的不只是控制台日志，还包括：

| 文件 | 作用 |
| --- | --- |
| `epic-runtime-<run_id>` 解压后的 `purchase_debug/*.png` | 看页面实际长什么样 |
| `epic-runtime-<run_id>` 解压后的 `purchase_debug/*.txt` | 看页面文本和 frame 内容 |
| `epic-logs-<run_id>` 解压后的日志文件 | 看完整执行链路 |

如果没有这些 artifact，很多 checkout 问题只能靠猜。

---

## 维护建议

如果你继续维护这个项目，优先关注下面几类变化：

1. Epic 登录页是否更换验证码类型。
2. 商品页按钮文案是否变化。
3. checkout iframe 和 `Place Order` 行为是否变化。
4. GLM / Gemini 的响应格式是否变化。
5. GitHub Actions 运行环境是否变化。
