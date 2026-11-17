# PixAI Proxy

把 PixAI 的绘图能力封装成 **OpenAI 兼容接口**，附带账号池调度、免费额度自动领取、可选注册机。

镜像公开，源码私有。

```bash
docker pull ghcr.io/shawlei/pixai-proxy:latest
```

## 特性

- **OpenAI 兼容**：`POST /v1/images/generations`、`GET /v1/models`，任何支持自定义 `base_url` 的客户端都能接
- **账号池**：多账号轮转，自动巡检，区分「凭据失效」与「账号封禁」
- **免费额度**：每日领取 + 任务上报，账号越多额度越多
- **模型库**：25+ 模型，支持按名字调用、按模型加载尺寸预设与推荐 LoRA
- **图库**：生成结果自动落盘，支持预览、下载、批量删除
- **注册机**：mihomo 节点轮询 + 邮箱池，自动开 NSFW 后入池
- **定时任务**：按小时点自动签到与做任务
- **管理面板**：完整 WebUI，移动端适配

## 部署

### 1. 拉镜像

```bash
docker pull ghcr.io/shawlei/pixai-proxy:latest
```

### 2. 准备目录

```bash
mkdir -p pixai/data pixai/mihomo
cd pixai
```

### 3. 写 compose 文件

保存为 `docker-compose.yml`：

```yaml
services:
  pixai-proxy:
    image: ghcr.io/shawlei/pixai-proxy:latest
    container_name: pixai-proxy
    restart: unless-stopped
    ports:
      - "8099:8099"
    volumes:
      - ./data:/data
      - ./mihomo:/root/.config/mihomo
    environment:
      ADMIN_PASSWORD: ${ADMIN_PASSWORD:-}
      DB_PATH: /data/app.db
      IMG_DIR: /data/images
      TZ: Asia/Shanghai
      TMPDIR: /tmp
      MIHOMO_API: http://mihomo:9090
      MIHOMO_PROXY: http://mihomo:7890
    depends_on:
      - mihomo
    shm_size: "1gb"

  mihomo:
    image: metacubex/mihomo:latest
    container_name: pixai-mihomo
    restart: unless-stopped
    volumes:
      - ./mihomo:/root/.config/mihomo
    ports:
      - "127.0.0.1:7890:7890"
      - "127.0.0.1:9090:9090"
```

> 不用注册机的话可以整段删掉 `mihomo`，同时移除 `MIHOMO_*` 环境变量和 `depends_on`。

### 4. 设置管理密码

```bash
echo "ADMIN_PASSWORD=你的强密码" > .env
docker compose up -d
```

不设置也行，启动时会随机生成并打在日志里：

```bash
docker logs pixai-proxy 2>&1 | grep ADMIN_PASSWORD
```

### 5. 打开面板

浏览器访问 `http://<你的服务器IP>:8099`，用上一步的密码登录。

## 首次配置

1. **加账号**：账号池 → 服务器浏览器登录 / 粘贴 cURL / 一键抓取代码
2. **领额度**：概览 → 「💰 领免费额度」
3. **生成 Key**：对外 Key → 新建，拿到 `sk-pixai-xxx`
4. **测试**：生图测试页随便画一张

## 调用方式

```
base_url : http://<你的服务器IP>:8099/v1
api_key  : sk-pixai-xxxx
model    : tsubaki-3-flash 等（见 /v1/models）
```

Python：

```python
from openai import OpenAI

c = OpenAI(base_url="http://<你的IP>:8099/v1", api_key="sk-pixai-xxx")
r = c.images.generate(
    model="tsubaki-3-flash",
    prompt="1girl, masterpiece, best quality",
    size="1024x1024",
)
print(r.data[0].url)
```

curl：

```bash
curl -X POST http://<你的IP>:8099/v1/images/generations \
  -H "Authorization: Bearer sk-pixai-xxx" \
  -H "Content-Type: application/json" \
  -d '{"prompt":"1girl, masterpiece","size":"1024x1024"}'
```

### 高级参数（非标准 OpenAI 字段）

```json
{
  "prompt": "klee (genshin impact), blonde hair, red eyes",
  "model": "tsubaki-3-flash",
  "size": "1104x1824",
  "loras": {"2044126774982672394": 0.7},
  "style_key": "classic-japanese",
  "inference_profile": "flash",
  "prompt_helper": true,
  "creativity": "medium"
}
```

- `loras`：`{versionId: 权重}`，从模型库页面查
- `style_key`：画风，见模型库
- `inference_profile`：质量档位，flash 快、quality 慢但精细
- `prompt_helper`：提示词助手，开启后中文会被翻译增强，建议保持 `true`

模型和 LoRA 默认透传：不传就用 PixAI 平台默认，传了才生效。

## 数据说明

- 所有账号、密钥、订阅都存在容器的 `./data/app.db` 里
- 镜像本身不含任何账号数据，首次启动是空池
- 不设置 `ADMIN_PASSWORD` 会随机生成，不会用固定默认值

## 系统要求

- Docker 20.10+
- 内存 ≥ 2GB（注册机要跑无头浏览器）
- 磁盘 ≥ 6GB（镜像约 2.7GB，含 Chromium）

## 更新

```bash
docker pull ghcr.io/shawlei/pixai-proxy:latest
docker compose up -d --force-recreate
```

`--force-recreate` 必须加，光 `restart` 不会换镜像。`./data` 卷会保留全部数据。

## 常见问题

**生图报 `账号池为空`**
池里没有可用账号，或者全部在冷却。去账号池页面添加，或等自动巡检恢复。

**画出来的人物不对**
PixAI 用 danbooru 标签训练，冷门角色模型不认识。用 `角色名 (作品名)` 格式，或加对应角色的 LoRA。

**注册机一直失败**
先检查代理节点：注册机页面看节点列表和延迟。节点全挂时走直连容易触发 429。

**怎么加更多免费额度**
每个账号每天能领 1 万左右，号越多额度越多。注册机可以批量造号，注意控制频率避免 429。

## License

仅供个人学习研究使用。
