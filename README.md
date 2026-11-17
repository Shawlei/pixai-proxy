# PixAI Proxy

OpenAI 兼容的 AI 绘图网关。

## 部署

```bash
docker pull ghcr.io/shawlei/pixai-proxy:latest
```

`docker-compose.yml`：

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

```bash
echo "ADMIN_PASSWORD=你的密码" > .env
docker compose up -d
```

面板 `http://IP:8099`。不设密码会随机生成，见 `docker logs pixai-proxy`。

## 调用

```
base_url  http://IP:8099/v1
api_key   面板「对外 Key」生成
model     /v1/models 里选
```

## 更新

```bash
docker pull ghcr.io/shawlei/pixai-proxy:latest
docker compose up -d --force-recreate
```

数据在 `./data`，不丢。
