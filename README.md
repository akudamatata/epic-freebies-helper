# Epic Awesome Gamer

一个基于 Playwright + `hcaptcha-challenger` 的 Epic 周免自动领取项目，当前仓库已经针对 GitHub Actions 做了适配，也保留了 Docker 部署方式。

这个分支版本和原版相比，重点变化有两类：

- 针对 GitHub Actions 做了自动化运行适配。
- 在原有 Gemini/AiHubMix 配置之外，新增了 GLM 接入。

## 仓库梳理

项目入口和职责大致如下：

- [`app/deploy.py`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/app/deploy.py) 是运行入口，负责浏览器启动、登录、领取和调度。
- [`app/services/epic_authorization_service.py`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/app/services/epic_authorization_service.py) 负责 Epic 登录和登录后的额外交互。
- [`app/services/epic_games_service.py`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/app/services/epic_games_service.py) 负责抓取周免数据、判断是否已入库、加购与下单。
- [`app/settings.py`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/app/settings.py) 负责环境变量、模型配置和 LLM 兼容补丁加载。
- [`app/extensions/llm_adapter.py`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/app/extensions/llm_adapter.py) 是这次新增的适配层，统一处理 Gemini/AiHubMix 和 GLM。
- [`.github/workflows/epic-gamer.yml`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/.github/workflows/epic-gamer.yml) 是 GitHub Actions 定时运行入口。
- [`docker/docker-compose.yaml`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/docker/docker-compose.yaml) 是 Docker 部署入口。

运行链路是：

1. 读取环境变量和模型配置。
2. 启动 Camoufox/Playwright 浏览器。
3. 登录 Epic。
4. 拉取 Epic 周免接口，过滤已领取内容。
5. 进入商品页处理加购、瞬时结账和 hCaptcha。
6. hCaptcha 由 `hcaptcha-challenger` 调用多模态模型识别。

## GLM 支持说明

### 为什么不能只改一个 Base URL

当前 `hcaptcha-challenger` 内部直接调用的是 `google-genai` 的文件上传和 `generate_content` 多模态接口，不是简单的 HTTP 文本请求。

这意味着：

- Gemini/AiHubMix 这条链路可以继续靠兼容补丁工作。
- GLM 不能只把 `GEMINI_BASE_URL` 换成智谱地址，否则请求体格式不兼容。

这次新增的 [`app/extensions/llm_adapter.py`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/app/extensions/llm_adapter.py) 做的事是：

- `LLM_PROVIDER=gemini` 时，继续走原来的 Gemini/AiHubMix 兼容补丁。
- `LLM_PROVIDER=glm` 时，把 `google-genai` 的调用转成智谱 OpenAI-compatible `chat/completions` 请求。
- 把图片输入转成 Base64 `image_url`，让 `hcaptcha-challenger` 继续按原来的方式工作。

### 推荐的 GLM 配置

这个项目的验证码识别依赖视觉模型，所以推荐：

- `GLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4`
- `GLM_MODEL=glm-4.5v`

不建议直接照搬“Coding Plan”里的文本/编程模型配置，因为这里的核心任务是图像理解，不是代码生成。你给的智谱文档可作为接口风格参考：

- [智谱 Coding Plan Quick Start](https://docs.bigmodel.cn/cn/coding-plan/quick-start)

## GitHub Actions 部署

这是最适合这个仓库的方式，原因很直接：

- 免费。
- 不需要自己挂机。
- 当前仓库已经提供现成工作流。

### 1. Fork 后设为私有仓库

建议使用私有仓库运行，避免把账号相关操作暴露在公开仓库里。

### 2. 配置 Secrets

进入仓库：

`Settings` -> `Secrets and variables` -> `Actions`

至少需要配置：

| Secret | 必填 | 说明 |
| --- | --- | --- |
| `EPIC_EMAIL` | 是 | Epic 邮箱，必须关闭 2FA |
| `EPIC_PASSWORD` | 是 | Epic 密码，必须关闭 2FA |

如果你使用 Gemini/AiHubMix，再加：

| Secret | 必填 | 说明 |
| --- | --- | --- |
| `GEMINI_API_KEY` | 是 | Gemini 或 AiHubMix API Key |
| `GEMINI_BASE_URL` | 否 | 默认是 `https://aihubmix.com` |
| `GEMINI_MODEL` | 否 | 默认是 `gemini-2.5-pro` |
| `LLM_PROVIDER` | 否 | 推荐显式设为 `gemini` |

如果你使用 GLM，再加：

| Secret | 必填 | 说明 |
| --- | --- | --- |
| `GLM_API_KEY` | 是 | 智谱 API Key |
| `GLM_BASE_URL` | 否 | 默认是 `https://open.bigmodel.cn/api/paas/v4` |
| `GLM_MODEL` | 否 | 推荐 `glm-4.5v` |
| `LLM_PROVIDER` | 建议 | 设为 `glm`，避免和 Gemini 配置混用时歧义 |

如果你不填 `LLM_PROVIDER`，程序会在存在 `GLM_API_KEY` 时自动切到 `glm`。

如果你想细分不同验证码任务使用的模型，也可以额外配置这些可选 Secrets：

- `CHALLENGE_CLASSIFIER_MODEL`
- `IMAGE_CLASSIFIER_MODEL`
- `SPATIAL_POINT_REASONER_MODEL`
- `SPATIAL_PATH_REASONER_MODEL`

### 3. 启用工作流权限

在仓库设置里确认：

- `Actions` -> `General` -> `Workflow permissions`
- 选择 `Read and write permissions`

### 4. 手动运行一次

进入 `Actions` 页面，选择 `Epic Awesome Gamer (Scheduled)`，点击 `Run workflow` 先跑一次。

当前工作流做的事情是：

- 安装 Python 3.12 和 `uv`
- 安装依赖
- 拉取 Camoufox 浏览器资源
- 在虚拟显示环境里执行 `uv run app/deploy.py`

## Docker 部署

如果你更想在 VPS/NAS 上自己跑，也可以用 Docker。

修改 [`docker/docker-compose.yaml`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/docker/docker-compose.yaml) 或 [`docker/.env`](/Users/ronchy2000/Documents/Developer/Workshop/epic-awesome-gamer/docker/.env)。

Gemini/AiHubMix 示例：

```yaml
environment:
  - LLM_PROVIDER=gemini
  - GEMINI_API_KEY=sk-xxxx
  - GEMINI_BASE_URL=https://aihubmix.com
  - GEMINI_MODEL=gemini-2.5-pro
```

GLM 示例：

```yaml
environment:
  - LLM_PROVIDER=glm
  - GLM_API_KEY=your_glm_key
  - GLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4
  - GLM_MODEL=glm-4.5v
```

如果你希望不同任务用不同模型，也可以继续覆盖这些变量：

- `CHALLENGE_CLASSIFIER_MODEL`
- `IMAGE_CLASSIFIER_MODEL`
- `SPATIAL_POINT_REASONER_MODEL`
- `SPATIAL_PATH_REASONER_MODEL`

未单独配置时，程序会优先复用 `GLM_MODEL` 或 `GEMINI_MODEL`。

## 本地开发

```bash
uv sync
uv run black . -C -l 100
uv run ruff check --fix
```

注意：

- 仓库说明明确写了“测试不允许执行”，所以不要补跑测试。
- 首次运行前要准备好浏览器依赖和对应 API Key。

## 常见问题

### 1. GitHub Actions 登录超时

GitHub 共享 IP 有时会被 Epic 风控。出现登录超时，通常换个时间重新运行即可。

### 2. GLM 返回模型或接口错误

先检查三项：

- `LLM_PROVIDER=glm`
- `GLM_BASE_URL=https://open.bigmodel.cn/api/paas/v4`
- `GLM_MODEL=glm-4.5v`

如果用了纯文本模型，验证码识别大概率会失败，因为这里必须处理图片。

### 3. 必须关闭 2FA 吗

是。这个项目运行在无头自动化环境里，无法稳定处理短信或邮箱二次验证。

## 免责声明

- 本项目仅用于学习和技术研究。
- 自动化操作可能违反 Epic 服务条款，使用风险自担。
