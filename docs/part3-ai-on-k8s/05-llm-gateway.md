# 3.5 LLM 网关与成本治理 [实用]

> 来源：[LiteLLM](https://docs.litellm.ai/) · [Envoy AI Gateway](https://github.com/envoyproxy/ai-gateway) · [OTel GenAI 语义约定](https://opentelemetry.io/docs/specs/semconv/gen-ai/) · 验证环境：待验证

## 概念：为什么需要

团队从「各自调 OpenAI key」长大后会撞三堵墙：

- **key 无法统一管控**：每个开发者一把 key，泄露了不知道、吊销了要挨个通知（[已必备]）
- **token 成本看不见**：月底账单吓人，但说不清是哪个团队、哪个功能烧的（[已必备]）
- **换模型要改业务代码**：厂商涨价 / 断供只能被动接受，切换成本高（[已必备]；[潜在] 多模型并存成为常态——不同业务用不同模型，统一管控成为刚需）

场景实例：大模型 API 涨价被迫切换供应商——没有网关时这是一次全量代码改造，有网关时只是一次配置变更。网关层一次解决三件事。

- **从哪来**：业务直连厂商 SDK → key 共享打 cookie → 出现统一代理 → 演化为 K8s 原生网关
- **是什么**：
  - **LiteLLM**：统一 OpenAI 兼容端点的代理——多 provider 路由、虚拟 key 池、按 key 预算限流，事实标准
  - **Envoy AI Gateway**：K8s 原生 GenAI 网关（CNCF 沙箱）——provider 路由、故障 fallback、token 级限流，与 Gateway API 体系接轨
- **往哪去**：网关成为 LLM 时代的「接入层」，配合推理侧的 Inference Extension（模型路由扩展，Gateway API 体系里按模型路由流量的标准）形成完整流量链路

**引导反思**：换模型应该是运营决策而不是代码变更——网关把「技术绑定」变成「运营选择」，这是 AI 应用规模化的前提。

## 动手（验证环境：待验证）

LiteLLM 最小配置：一个虚拟 key（网关签发的代理 key，不直接暴露厂商 key）+ 月度预算 + provider fallback（主后端不可用时自动切换备选后端）。

```yaml
# config.yaml：一个虚拟 key + 月度预算 + provider fallback
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
  - model_name: gpt-4o        # 同名双后端 = 免费获得 fallback
    litellm_params:
      model: anthropic/claude-sonnet
      api_key: os.environ/ANTHROPIC_API_KEY
litellm_settings:
  budget_duration: 30d
general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
```

业务代码从此只认 `http://litellm-gateway.internal/v1`，OpenAI SDK 直接指向它：

```bash
# 验证网关（待验证）
curl http://litellm-gateway.internal/v1/chat/completions \
  -H "Authorization: Bearer $VIRTUAL_KEY" \
  -d '{"model": "gpt-4o", "messages": [{"role": "user", "content": "hi"}]}'
```

**成本观测闭环**：网关转发时按 OTel GenAI 语义约定打点（`gen_ai.usage.input_tokens` / `output_tokens` 等标准字段——AI 调用观测的统一字段口径，SPEC §7 规定全书统一）→ 进 Prometheus → Grafana 按团队 / feature 两个维度做 showback（成本分摊展示，先看见再治理）。

## 实用技巧

- 虚拟 key 按团队 / 项目分配，一人一把或一服务一把：泄露了只吊销一把，审计也有粒度
- 预算先粗后细：先按团队设月度预算，再细化到 feature；`budget_duration: 30d` 是月度滚动窗口
- 同名双后端是 fallback 的最简实现：主 provider 限流 / 故障时自动切到备选
- 网关是缓存、降级、审计的天然落点：这些能力都发生在网关层，业务代码不用动
- 与 3.4 的 GPU 成本合并：token 是「用量账单」，GPU 是「算力账单」，合起来才是完整 AI 成本视图

## CKAD 考点对照

无（Part 3 不在 CKAD 范围内）。

## 考察问题

- 为什么业务代码不应该 import 厂商 SDK 的专有类型？（线索：防腐层——业务代码与外部依赖之间加一层抽象，隔离变化；换模型是运营决策不是代码变更）
- 网关挂了会怎样？怎么让网关自己不成为单点？（线索：多副本、无状态、健康检查）
- 虚拟 key 泄露了怎么办？如何做到「吊销一把 key 不影响其他团队」？（线索：key 池与预算的绑定关系）

## 经验之谈

- 观点（不署名）：「先统一网关，再谈优化」——厂商切换、缓存、降级、审计都发生在网关层，晚建不如早建（社区通行经验，具体引用待补）

## 架构师视角

- 解决什么问题：key 管控、成本可见、模型可切换——把「换模型」从代码变更变成运营决策
- 何时用：团队化之后是必选项；多模型并存、成本要分摊、审计要留痕
- 何时不用：单人原型 / 单一模型无所谓——先直连，别过度设计
- 权衡：多一跳延迟（通常 < 10ms 量级）换管控力；网关本身要运维，别让它成为新的单点
- 固定三问：
  - 平台提供了什么能力边界？——平台是否已提供统一网关（LiteLLM / Envoy AI Gateway / 商业网关），还是需要业务侧自建
  - 业务接入点在哪？——OpenAI 兼容端点 URL + 虚拟 key；业务代码只改 base_url
  - 需要和基础设施团队对齐什么？——预算额度、模型白名单、审计日志留存、网关高可用

## 核心收获表

| 能力维度 | 具体收获 |
|---|---|
| key 治理 | 虚拟 key 池：按团队 / 服务分配、独立预算、可单独吊销 |
| 成本可见 | token 成本按团队 / feature 维度出报表，月底账单不再是一笔糊涂账 |
| 模型可切换 | 换模型只改网关配置，业务代码零改动 |
| 观测口径 | 按 OTel GenAI semconv 打点，与 3.4 / 4.2 / lab3 同一字段口径 |

## 推荐开源项目表

| 项目 | 链接 | 研读重点 |
|---|---|---|
| LiteLLM | https://docs.litellm.ai/ | 虚拟 key、预算、fallback 配置 |
| Envoy AI Gateway | https://github.com/envoyproxy/ai-gateway | K8s 原生网关、Gateway API 接轨 [进阶] |
| OTel GenAI 语义约定 | https://opentelemetry.io/docs/specs/semconv/gen-ai/ | gen_ai.usage.* 字段口径 |

## 常见问题排查

| 高频报错 / 现象 | 排查路径 |
|---|---|
| 401 Invalid API key | 虚拟 key 未创建 / 已吊销 → 用 master key 重新生成；检查环境变量注入 |
| 429 Too Many Requests | 预算 / 限流触发 → 查该 key 的用量与 budget 设置；区分「预算超了」与「厂商限流」 |
| fallback 不生效 | 检查同名 model 双后端配置；确认主 provider 是「报错」还是「超时」——超时需配超时阈值 |
| token 计数对不上 | 核对打点字段口径（input / output tokens 是否含缓存命中）；与厂商账单对照 |
| 延迟比直连高 | 网关一跳通常 < 10ms 量级；检查是否触发重试 / fallback 路径 |