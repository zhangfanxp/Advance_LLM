# 问题澄清与建议助手

一个基于 Gradio 的本地 AI 咨询应用。它先使用百炼 `qwen3.8-flash` 将用户的问题拆解为单选题或多选题，收集必要背景并检查信息完整性；信息充分后，再由 `qwen3.7-flash` 结合原问题和全部问答，流式生成个性化建议。用户还可以在初步答复后继续提问，获得更细化的选择题和建议。

## 功能

- 根据原始问题生成选择题，支持单选、多选及可选的个性化文字补充。
- 自动检查背景信息是否足够；不足时继续追问并保留已提交答案。
- 达到 5 轮追问时提醒用户，但不强制停止。
- 收集完整后流式生成最终建议，并支持继续提问、反复细化。
- 对话仅保存在当前 Gradio 会话内存中，不写入数据库或本地历史文件。

## 环境要求

- macOS
- Python 3.10 或更高版本
- [uv](https://docs.astral.sh/uv/) 包和虚拟环境管理器
- 有权限调用下列模型和兼容接口的阿里云百炼 API Key：
  - 问题拆解与完整性判断：`qwen3.8-flash`
  - 最终答复：`qwen3.7-flash`
  - API 地址：`https://llm-0nr9mznstg4wrz4h.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`

## macOS 本地部署

在终端中执行以下步骤。若已经在项目目录中，可跳过克隆步骤。

### 1. 安装 uv 并获取项目

如果尚未安装 uv，请按 [uv 官方安装说明](https://docs.astral.sh/uv/getting-started/installation/) 安装，然后重新打开终端。

```bash
git clone https://github.com/zhangfanxp/Advance_LLM.git
cd Advance_LLM
```

### 2. 创建并启用虚拟环境

在项目根目录创建虚拟环境。建议使用 Python 3.11：

```bash
uv python install 3.11
uv venv --python 3.11
source .venv/bin/activate
```

启用后，终端提示符通常会显示 `(.venv)`。之后在该终端使用 `python` 和 `uv pip` 即会使用项目虚拟环境。

### 3. 安装依赖

```bash
uv pip install -r requirements.txt
```

### 4. 配置百炼 API Key

复制示例环境文件并编辑：

```bash
cp .env.example .env
open -e .env
```

将文件中的占位符替换为百炼 API Key：

```dotenv
DASHSCOPE_API_KEY=你的百炼APIKey
```

不要在等号两侧添加空格或将 Key 加入引号。`.env` 已加入 `.gitignore`，不会作为项目配置上传。不要把 API Key 提交到 GitHub 或发送到聊天中；如果曾经泄露，请先在百炼控制台撤销并重新生成。

### 5. 启动应用

```bash
python app.py
```

终端会显示本地访问地址，通常是 <http://127.0.0.1:7860>。在浏览器中打开该地址即可使用。停止应用时，在运行服务的终端按 `Control-C`；退出虚拟环境可运行 `deactivate`。

## 使用 Cloudflare Tunnel 绑定自定义域名

以下示例将 `agent.frank1998.uk` 指向本地 Gradio 服务 `http://127.0.0.1:7860`。需要已安装并登录 `cloudflared`、在 Cloudflare 管理的域名，以及已创建的 Tunnel。本文对应的本机 Tunnel 名称是 `next-js`；如果你的 Tunnel 名不同，请先运行 `cloudflared tunnel list` 查看，并在下方命令中替换。

### 1. 确认 Tunnel 和凭据

```bash
cloudflared --version
cloudflared tunnel list
ls -l ~/.cloudflared/config.yml
```

Tunnel 配置文件通常包含 Tunnel 名称、凭据文件和 ingress 规则。不要把凭据文件提交到 GitHub 或分享给他人。

### 2. 添加域名到 ingress 规则

打开 Tunnel 配置：

```bash
nano ~/.cloudflared/config.yml
```

在 `ingress:` 下、最后的兜底规则（通常是 `- service: http_status:404`）之前添加：

```yaml
  - hostname: agent.frank1998.uk
    service: http://127.0.0.1:7860
```

确认缩进与文件中其他 ingress 规则一致，并且兜底规则仍位于最后。保存 nano 文件：按 `Control-O`、回车，再按 `Control-X`。

验证配置：

```bash
cloudflared tunnel --config ~/.cloudflared/config.yml ingress validate
cloudflared tunnel --config ~/.cloudflared/config.yml ingress rule https://agent.frank1998.uk
```

第二条命令应显示该域名命中 `http://127.0.0.1:7860`。

### 3. 为域名创建 DNS 路由

对当前配置的 Tunnel 执行一次：

```bash
cloudflared tunnel route dns next-js agent.frank1998.uk
```

如果已创建过该 DNS 路由，不需要重复创建。可以检查 DNS 解析：

```bash
dig +short agent.frank1998.uk
```

### 4. 分别启动应用和 Tunnel

先在一个终端窗口启动应用：

```bash
cd /项目所在路径/Advance_LLM
source .venv/bin/activate
python app.py
```

再打开另一个终端窗口，运行 Tunnel：

```bash
cloudflared tunnel --config ~/.cloudflared/config.yml run
```

保持两个终端窗口和这台 Mac 持续运行。应用进程或 Tunnel 停止、Mac 休眠或断网时，域名将无法访问。然后检查：

```bash
curl -I http://127.0.0.1:7860
curl -I https://agent.frank1998.uk
```

第二条命令正常时应返回 `HTTP/2 200`。要停止服务，在对应终端按 `Control-C`。

**公开访问和费用提示：**以上配置不启用登录或访问控制，知道域名的任何人都可以使用应用，并可能消耗百炼 API 额度。若需要限制访问，请在 Cloudflare Zero Trust 中为该域名配置 Cloudflare Access，再对外分享域名。

## 操作方法

1. 在“你想咨询什么？”输入问题，例如“如何教育好一个孩子？”，然后点击“提交问题”。
2. 对模型给出的每道题选择答案。单选题选一个选项，多选题可选多个选项；遇到没有合适选项时，可填写“其他补充（可选）”。每道题至少选择一个选项或填写补充。
3. 点击“提交回答并检查”。如果模型判断背景不足，会显示新一批选择题，已提交的问答会保留；单轮最多 8 个问题。选项可能包含“不知道/不确定”或“不适用”。
4. 追问达到 5 轮时页面会显示提示，但仍会继续检查和追问。
5. 信息充分后，`qwen3.7-flash` 会结合原问题和全部问答流式生成答复。
6. 答复完成后，可以点击“继续提问，获取更细化建议”生成新的细化问题并更新建议。该操作可以重复进行。
7. 点击“开始新问题”可清空当前内容并开始新的咨询。

## 数据与隐私

问答状态只保存在当前 Gradio 会话内存中，不会写入数据库或本地历史文件；刷新页面、结束会话或重启服务后不会保留。使用过程中，原始问题和回答会发送到配置的百炼 API。应用只展示最终答复，不展示模型思考过程。

## 常见问题

- **提示需要设置 `DASHSCOPE_API_KEY`**：确认项目根目录存在 `.env`，变量名和值正确，并且当前终端从项目根目录启动应用。
- **模型调用失败**：确认网络、API Key、模型权限和接口地址；并确认百炼账号能够调用配置的 `qwen3.8-flash` 与 `qwen3.7-flash`。
- **找不到 `uv` 命令**：安装 uv 并重新打开终端。
- **依赖或 Python 版本不匹配**：确认已启用 `.venv`，且虚拟环境使用 Python 3.10 或更高版本。
