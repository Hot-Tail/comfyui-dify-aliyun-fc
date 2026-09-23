# ComfyUI + Dify 自动化图像生成工作流（阿里云函数计算版）

## 从部署到调度，实现电商批量生图与自动化内容生产

## 这是什么

这是一个在 **阿里云函数计算 FC 3.0** 上部署 **ComfyUI** 并完成文生图测试，最终**导出 API 工作流 JSON，供 Dify 调度**的完整操作记录。内容涵盖选型、创建 NAS、部署应用、切换 SD1.5 模型、导出 API 工作流、与 Dify 集成、成本控制以及踩坑与解决方案。

## 核心优势：为什么不用现成生图平台？

> 阅读提醒：本章节是本文档的核心价值说明，也是选择自建 ComfyUI + Dify 调度方案的关键原因，请重点阅读。

### 1. 数据私有，不泄露给第三方

三方生图平台（如 Midjourney、DALL·E、文心一言等）要求用户将提示词、参考图、生成结果上传至其服务器，数据归属和隐私合规难以控制。自建 ComfyUI 部署在你自己的阿里云账号下，**所有图片、模型、工作流数据都在你的 VPC 内**，不经过任何第三方平台。对于电商客户而言，商品图、模特图属于核心商业资产，数据不出域是刚需。

### 2. 工作流可定制、可复用、可沉淀为数字资产

三方平台只提供“输入提示词 → 输出图片”的黑盒，无法控制采样器、LoRA、ControlNet、IPAdapter 等细节。ComfyUI 是**节点式白盒工作流引擎**，你可以自由组合模型、控制构图、批量处理。更关键的是，工作流可以**导出为 JSON 文件**，这是可以版本控制、可以分享、可以嵌入到其他应用中的**可编程数字资产**。一次搭建，长期复用，边际成本趋近于零。

### 3. 与 Dify 集成，实现自动化调度

三方平台需要人工逐张操作，无法嵌入业务流程。本方案将 ComfyUI 的 API 工作流 JSON 接入 Dify，实现“用户输入提示词 → Dify 调度 ComfyUI → 自动返回图片”的完整链路。**Dify 作为调度中心，ComfyUI 作为图像生成引擎**，两者配合可以支撑电商批量生图、社交媒体内容生产、自动化设计流程等业务场景。

### 4. 成本可控，按需付费

三方平台多采用订阅制（如 Midjourney 约 10-30 美元/月），且有调用次数或生成速度限制。本方案采用阿里云函数计算 **Serverless 按量付费**：

- 无请求时自动缩容到 0，**不产生计算费用**。
- 单张图生成成本约 **0.02-0.2 元**。
- NAS 容量型存储月费约 **1.4 元**。
- 整体月成本可控制在 **2 元以内**（测试阶段）。

对于低频或批量场景，成本远低于订阅制平台。

### 5. 无平台锁定，技术栈完全自主

三方平台的模型、接口、定价策略随时可能变化，用户无法控制。自建方案基于开源 ComfyUI 和阿里云基础设施，**技术栈完全自主**，模型可替换、工作流可修改、部署位置可迁移，不受任何单一平台策略影响。

### 6. 支持批量与自动化，适合电商等场景

三方平台通常不支持批量任务队列和 API 自动化。自建 ComfyUI + Dify 调度方案天然支持：

- **电商批量生图**：为商品目录批量生成统一风格主图、场景图。
- **社交媒体内容生产**：批量生成配图、海报、封面图。
- **自动化设计流程**：将图像生成嵌入现有业务系统，无需人工逐张操作。

**一句话总结**：现成平台是“租用别人的黑盒工具”，自建 ComfyUI + Dify 是“拥有自己的自动化生产线”。短期看前者省事，长期看后者更可控、更省钱、更能沉淀资产。

---

## 解决了什么问题

- **本地无 NVIDIA 显卡无法运行 ComfyUI**：通过阿里云函数计算（Serverless GPU）按需调用，无需自购显卡。
- **云 GPU 成本高、闲置浪费**：利用函数计算弹性伸缩，无请求时缩容到 0，仅按实际出图时长计费；配合 NAS 容量型存储，月成本控制在 2 元以内。
- **临时域名 1 天失效**：记录临时域名失效的应对方法，建议绑定自定义域名。
- **Flux 模型加载慢**：切换为 SD1.5 小模型，加载时间大幅缩短。
- **手动出图无法满足业务需求**：导出 ComfyUI API 工作流 JSON，接入 Dify 实现自动化调度，完成从“人工操作”到“系统自动化”的升级。

## 操作流：ComfyUI + Dify 协同工作流

### 整体架构

```
用户输入提示词
      │
      ▼
┌─────────────────┐
│   Dify 应用      │  ← 用户界面 / 对话入口
│  （调度中心）     │
└────────┬────────┘
         │ POST /prompt（携带工作流 JSON）
         ▼
┌─────────────────┐
│  ComfyUI 服务    │  ← 阿里云函数计算 FC
│  （图像生成）     │
└────────┬────────┘
         │ 返回 prompt_id
         ▼
   Dify 轮询 /history/<prompt_id>
         │
         ▼
   获取生成结果 → 展示给用户
```

**调用链路**（三步异步流程）：

1. **提交任务**：Dify 向 ComfyUI 服务发送 POST 请求，请求体中包含导出的 ComfyUI 工作流 JSON。ComfyUI 接收任务后，立即返回一个唯一的 `prompt_id`。
2. **轮询状态**：Dify 使用 `prompt_id`，通过 GET 请求循环查询任务的执行状态。
3. **获取结果**：判断任务成功后，再次发起 GET 请求获取最终结果，将生成图片的访问链接以 Markdown 格式回复用户。

### 商业目的

这套操作流的核心目的是**将 ComfyUI 的图像生成能力，通过 Dify 封装成自动化、可批量调用的服务**，典型应用场景包括：

- **电商批量生图**：为商品目录批量生成统一风格的主图、场景图。有现成的 ComfyUI 工作流（如 Ohneis Product Workflow）专门用于批量处理产品照片，支持多 SKU、批量一致性处理。
- **社交媒体内容生产**：批量生成配图、海报、封面图。
- **自动化设计流程**：将图像生成嵌入现有业务系统，无需人工逐张操作。

### Dify 侧配置要点

- **HTTP 请求节点**：POST 到 ComfyUI 的 `/prompt` 地址，Body 为导出的 API JSON，将提示词等动态参数替换为 Dify 变量。
- **轮询节点**：使用 `prompt_id` 循环 GET `/history/<prompt_id>`，设置最大轮询次数（如 30 次，间隔 5 秒），防止死循环。
- **结果展示**：将生成文件的访问链接拼接为 Markdown 图片格式，直接回复用户。

## 怎么用

按照以下步骤，从零开始在阿里云函数计算部署 ComfyUI，并导出 JSON 接入 Dify。环境准备 → 创建 NAS → 部署应用 → 切换模型 → 出图测试 → 导出 API → Dify 集成 → 成本控制。如遇问题，可查阅「踩坑记录」章节。

---

## 环境准备

- **阿里云账号**：已实名认证，余额 ≥ 100 元。
- **函数计算 FC 3.0**：已开通。
- **NAS 文件存储**：用于持久化模型和输出文件。
- **VPC 与交换机**：与函数计算同地域、同可用区，确保内网互通。
- **Dify 环境**：已部署（本地或云端均可），用于调度 ComfyUI。

## 部署步骤

### 1. 创建 NAS（容量型）

- 在 NAS 控制台创建 **通用型 NAS**，存储规格选 **容量型**（0.35 元/GB/月）。
- 地域选择 **华东1（杭州）**，可用区与后续函数计算保持一致。
- 记录 NAS 文件系统 ID、挂载点域名。

### 2. 部署 ComfyUI 应用

- 进入函数计算控制台 → 应用中心，搜索 **ComfyUI**，选择官方模板。
- 配置应用：
  - 应用名称：`comfyui-<随机后缀>`
  - 地域：华东1（杭州）
  - GPU 规格：16GB 显存（Tesla 系列）
  - 内存：32GB，vCPU：8 核
- 在 **存储** 中挂载 NAS：
  - 远端目录：`/comfyui`
  - 函数本地目录：`/mnt/comfyui`
- 点击 **创建应用**，等待 3-5 分钟部署完成。

### 3. 访问 ComfyUI

- 部署成功后，在应用详情页找到 **访问域名**。
- 首次打开可能提示“域名生效中”，等待 1-10 分钟或重新部署。
- 进入 ComfyUI 界面，默认工作流可能包含 Flux 模型，但加载较慢。

### 4. 切换 SD1.5 模型

- 在 ComfyUI 中，找到 `Load Checkpoint` 节点。
- 点击 `ckpt_name` 下拉，若只有 Flux，需通过 **模型广场** 下载 SD1.5：
  - 进入 Function AI 控制台 → 项目 → 模型广场。
  - 筛选 Checkpoints，选择 `sd-v1-5-inpainting.ckpt`，下载到 `models/checkpoints/`。
- 下载完成后，在 `Load Checkpoint` 中选择该模型。

### 5. 出图测试

- 在 `CLIP Text Encode (Prompt)` 节点输入提示词，例如：
  ```
  a cute cat sitting on a cloud, fluffy plush toy style, soft lighting
  ```
- 点击 **Queue Prompt**，等待出图。
- 首次加载模型可能需要 30-60 秒，之后出图约 20 秒。

### 6. 导出 API 工作流 JSON

- 在 ComfyUI 菜单中点击 **Workflow → Export (API)**。
- 浏览器会下载一个 JSON 文件，保存备用。
- 该 JSON 文件即为 Dify 调用的“说明书”，包含工作流全部节点信息。

### 7. 接入 Dify

- 在 Dify 工作流中添加 **HTTP 请求节点**：
  - 方法：POST
  - URL：`http://<ComfyUI访问域名>/prompt`
  - Body：粘贴导出的 API JSON，将提示词字段替换为 Dify 变量（如 `{{positive_prompt}}`）
- 添加 **代码节点** 实现轮询：使用返回的 `prompt_id` 循环 GET `/history/<prompt_id>`，设置最大次数与间隔，超时返回错误。
- 获取结果后，将图片文件名与访问地址拼接，以 Markdown 格式回复用户。

## 成本控制

- **函数计算**：无请求时自动缩容到 0，不产生计算费用。单张图成本约 0.02-0.2 元。
- **NAS 存储**：容量型 0.35 元/GB/月，SD1.5 模型约 4GB，月费约 1.4 元。
- **费用预警**：在阿里云费用中心设置预算预警（如 20 元/月）和高额消费预警（如 1 元/天）。

## 踩坑记录

| 问题 | 现象 | 原因 | 解决方法 |
|------|------|------|----------|
| 找不到 ComfyUI 模板 | 应用中心搜索无结果 | 控制台版本或入口不对 | 使用 FC 3.0 应用中心直达链接 |
| 临时域名失效 | 访问域名提示“温馨提示” | `*.devsapp.net` 仅 1 天有效 | 重新部署获取新域名，或绑定自定义域名 |
| 预热脚本 curl 缺失 | `curl: command not found` | 容器内未安装 curl | 改用 Python `urllib` 发送请求 |
| GPU 配额超额 | `Function's reserved GPU usage exceeded` | 设置最小实例数=1 触发预留 | 删除弹性策略，缩容到 0 |
| Flux 模型加载太慢 | 首次出图等待 3 分钟以上 | 模型 23GB | 换用 SD1.5 `sd-v1-5-inpainting.ckpt` |
| KSampler 缺 negative | 报错 `Required input is missing: negative` | 工作流缺少负向节点 | 复制一个 `CLIP Text Encode` 连到 negative |
| NAS 自动创建多出一个 | 出现两个 NAS 文件系统 | 自动配置时系统新建 | 删除闲置 NAS，保留系统创建的容量型 |
| Dify 应用 URL 仍指向旧端口 | 发布后访问报错 | `.env` 中 URL 未更新 | 修改 `APP_WEB_URL` 为 `http://127.0.0.1:8080` |

## 长期使用建议

1. 使用自定义域名替代临时域名，避免 1 天失效。
2. 保持函数计算最小实例数为 0，不用时不产生计算费用。
3. 模型文件放在 NAS 容量型存储，按需下载。
4. 定期检查费用账单，设置预警。
5. 导出 API 工作流 JSON，接入 Dify 实现自动化调度。

## 许可

本仓库的部署脚本和文档采用 **MIT License** 发布。

**ComfyUI** 采用 **GPL-3.0** 许可。使用 ComfyUI 构建的商业产品，如果修改了 ComfyUI 本身，需按 GPL-3.0 开源修改部分；若仅通过 API 调用 ComfyUI 生成内容，商业收益来自独立开发的代码，通常不受 GPL 传染。

**SD1.5 模型** 采用 **CreativeML Open RAIL-M** 许可，允许商业使用和作为服务分发。但需在服务条款中实施安全过滤器（NSFW 检测、仇恨言论拦截等）。

**使用硅基流动 Embedding 需自行注册并申请 API Key，本仓库不包含任何个人密钥。**

MIT License

Copyright (c) 2026 Author

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```


**结论：文档可直接上传 GitHub，无个人隐私或敏感信息。**
