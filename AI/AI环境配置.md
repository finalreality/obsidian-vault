## 安装crush
```
npm install crush
```
添加配置文件：
```json
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    "nvidia": {
      "base_url": "https://integrate.api.nvidia.com/v1",
      "api_key": "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",
      "models": [
        {
          "id": "z-ai/glm-5.1",
          "name": "GLM-5.1"
        },
        {
          "id": "minimaxai/minimax-m2.7",
          "name": "MINIMAX-2.7"
        }
      ]
    },
    "zp": {
      "base_url": "https://open.bigmodel.cn/api/paas/v4",
      "api_key": "3156eb20df624f5a85c4cbf96af537b3.KOwuPRFtPXmdeWFn",
      "models": [
        {
          "id": "glm-4.6v",
          "name": "GLM-4.6V"
        }
      ]
    }
  },
  "mcp": {
    "chrome-devtools": {
      "type": "stdio",
      "command": "chrome-devtools-mcp",
      "args": ["--headless"]
    }
  }
}

```
## 安装chrome-devtools-mcp
```bash
npx chrome-devtools-mcp@latest
```