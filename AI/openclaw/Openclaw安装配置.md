## 安装依赖
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl build-essential -y
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
sudo apt-get install python3-pip
sudo apt-get install nodejs jq curl
```
 安装 nvm
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

激活 nvm (如果当前终端未生效，关闭终端重开，或运行下面这行)
export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"

安装 Node.js 24.x
```bash
nvm install 24.14.0
nvm use 24.14.0
nvm alias default 24
```

验证
```
node -v
npm -v
```
## 安装openclaw
```
npm install -g openclaw

openclaw onboard --install-daemon
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
openclaw gateway restart
```
clawhub安装skills问题：
```bash
$ clawhub install weather
✖ Rate limit exceeded
Error: Rate limit exceeded
```
需要登录clawhub：
```bash
clawhub login --token clh_0tH9uB7DXvSK008hMbQ6TUgDTa0UBimnBoyH9yv6nbQ
```

## 安装tavily-search
```
clawhub install tavily-search
```
修改~/.openclaw/skills/tavily-search/SKILL.md,替换如下内容：
```markdown
---
name: tavily-search
description: Search the web in real-time using Tavily Search API, optimized for LLM consumption.
requires:
  env:
    - TAVILY_API_KEY
  bins:
    - curl
    - jq
---

# Tavily Web Search Skill

When the user asks to search the web, find current information, or look up recent events, use the Tavily Search API.

## Basic Search

Write the request JSON to a temp file, then execute with curl:

\`\`\`bash
cat > /tmp/tavily_request.json << 'REQEOF'
{
  "query": "$QUERY",
  "search_depth": "basic",
  "max_results": 5,
  "include_answer": true
}
REQEOF

bash -c 'curl -s -X POST "https://api.tavily.com/search" \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer ${TAVILY_API_KEY}" \
  -d @/tmp/tavily_request.json' | jq '.answer, .results[] | {title, url, content}'
\`\`\`

## Advanced Search (for deep research questions)

Set `"search_depth": "advanced"` for comprehensive results. Note: advanced search uses 2 API credits per request.

## Parameters Guide

- **search_depth**: "basic" (fast, 1 credit) or "advanced" (thorough, 2 credits)
- **max_results**: Number of results (default 5, max 20)
- **include_answer**: Get an AI-generated summary answer
- **include_domains**: Restrict to specific domains, e.g. ["arxiv.org"]
- **exclude_domains**: Exclude specific domains
- **topic**: "general" (default) or "news"
- **days**: For news topic, limit to recent N days

## Response Format

The API returns JSON with:
- `answer`: AI-generated summary answer to the query
- `results`: Array of search results with `title`, `url`, `content`, `score`

Always present the results clearly with source URLs for attribution.

```
重启
```
openclaw gateway restart
```
## 安装飞书
```
python3 -m venv venv
source venv/bin/activate
pip install lark-oapi -U
```
install.py:
```python
import lark_oapi as lark
## P2ImMessageReceiveV1 为接收消息 v2.0；CustomizedEvent 内的 message 为接收消息 v1.0。
def do_p2_im_message_receive_v1(data: lark.im.v1.P2ImMessageReceiveV1) -> None:
    print(f'[ do_p2_im_message_receive_v1 access ], data: {lark.JSON.marshal(data, indent=4)}')
def do_message_event(data: lark.CustomizedEvent) -> None:
    print(f'[ do_customized_event access ], type: message, data: {lark.JSON.marshal(data, indent=4)}')
event_handler = lark.EventDispatcherHandler.builder("", "") \
    .register_p2_im_message_receive_v1(do_p2_im_message_receive_v1) \
    .register_p1_customized_event("out_approval", do_message_event) \
    .build()
def main():
    cli = lark.ws.Client("cli_a93a79ce2af8dccb", "OLlSphUKOgwuuyHZtlaNjhi80m3hSjKK",
                         event_handler=event_handler,
                         log_level=lark.LogLevel.DEBUG)
    cli.start()
if __name__ == "__main__":
    main()
```
启动长链接（在配置完Openclaw端feishu后即可Ctrl+C结束掉）：
```
$ python3 install.py
```
配置飞书：
登录[飞书开放平台](https://open.feishu.cn/?lang=zh-CN)
进入开发者后台：
![[Openclaw安装配置-3.png]]
![[Openclaw安装配置-2.png]]
创建企业自建应用：
![[Openclaw安装配置-4.png]]
![[Openclaw安装配置-5.png]]
填入名称描述和选择图标后创建即可创建我们的应用：
 ![[Openclaw安装配置-6.png]
 点击进入TestMyOpenclawBot，然后就能找到我们的应用凭证的App ID和App Secret:
 ![[Openclaw安装配置-7.png]]

## Openclaw端配置：
 ```
 openclaw plugins install @openclaw/feishu
 ```
添加通道：
```
openclaw channels add
```
我们选中飞书:
 ![[Openclaw安装配置-9.png]]
输入我们刚刚的应用凭证的App Secret和App ID：
 ![[Openclaw安装配置-12.png]]
一路下一步，再到Select a channel时直接退出即可：
 ![[Openclaw安装配置-13.png]]
最后，修改配置文件 ~/.openclaw/openclaw.json，加入如下配置：
![[Openclaw安装配置-14.png]]
```json
"plugins": {
    "entries": {
      "whatsapp": {
        "enabled": true
      },
      "feishu": {
        "enabled": true
      }
    },
    "installs": {
      "feishu": {
        "source": "npm",
        "spec": "@openclaw/feishu",
        "installPath": "/home/finalreality/.openclaw/extensions/feishu",
        "version": "2026.3.7",
        "resolvedName": "@openclaw/feishu",
        "resolvedVersion": "2026.3.7",
        "resolvedSpec": "@openclaw/feishu@2026.3.7",
        "integrity": "sha512-CHPcL+WHYKYR2HJKRYsRtlXx/wbQRy5axltjjH9qXkR8ghxygDmOHZREjxyFEbjFJ3wnIuvgjLE7JYTg3nPpDA==",
        "shasum": "c4b31dbe2ff0bc7034334873482ad18ac60a0767",
        "resolvedAt": "2026-03-11T01:51:53.935Z",
        "installedAt": "2026-03-11T01:51:56.824Z"
      }
    }
  }
```
重启
```
openclaw gateway restart
```
这时，我们就可以结束长连接。
登录飞书，在飞书端直接输入对话会出现：
 ![[Openclaw安装配置-1.png]]
我们在openclaw端审批下：
```
openclaw pairing approve feishu T9ZJTQSM
```
tavily-search找不到API_KEY问题：
编辑.openclaw/workspace/.env文件，写入如下配置：
```
TAVILY_API_KEY=tvly-dev-XXXXXX-XXXzqLBbZdjWwyCMiI6qbtjkzwrsvKsXgGbopvpXXX
```

## 解决无法执行命令与操作问题
要恢复OpenClaw的全部功能，需通过终端命令修改配置参数，将权限配置文件（.openclaw/openclaw.json）中的相关设置调整为完全开放模式。请依次执行以下命令：
```bash
#1. 设置工具配置文件为全权模式：  
openclaw config set tools.profile full
#2. 设置会话可见性为全部：  
openclaw config set tools.sessions.visibility all
#3. 重启网关以使配置生效：  
openclaw gateway restart
```