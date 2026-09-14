# 部署说明：jinhuazhang.top/publicpolicy/

目标：静态站 + 同源 API（问答 / 共创）部署到 `https://jinhuazhang.top/publicpolicy/`。

## 1. 目录与进程

| 组件 | 建议位置 | 说明 |
|---|---|---|
| 静态文件 | 例如 `/var/www/publicpolicy/` | `web/` 下 html/css/js/data/assets |
| API 服务 | systemd 跑 `web/server/app.py`，或反代到内部端口 | 勿把密钥写进静态目录 |
| 内容仓库 | `hengdaoye50/public-policy-textbook` | Issue/PR 目标 |

同步静态资源示例（在开发机 `web/`）：

```bash
# 仅示例：按你的发布方式调整
rsync -av --exclude 'server/inbox' --exclude 'server/logs' \
  ./ user@host:/var/www/publicpolicy/
```

`data/` 在改正文后需重新 `build_data.py` 再发布。

## 2. 服务端环境变量（勿入库）

```bash
export PP_MOCK=0
export PORT=8080
export PP_ALLOW_ORIGINS=https://jinhuazhang.top
export FIREWORKS_API_KEY=fw_...
export LLM_BASE=https://api.fireworks.ai/inference/v1
export LLM_MODEL=accounts/fireworks/models/deepseek-v4p1-flash
export GITHUB_REPO=hengdaoye50/public-policy-textbook
export GITHUB_TOKEN=ghp_...   # fine-grained: 仅 issues + pull_requests
# 可选加固
export PP_ACCESS_TOKEN=...          # 前端 js/config.js accessToken 同步
# export TURNSTILE_SECRET=...
```

systemd 示例：

```ini
[Service]
WorkingDirectory=/opt/publicpolicy
Environment=PP_MOCK=0
Environment=PORT=8080
Environment=PP_ALLOW_ORIGINS=https://jinhuazhang.top
EnvironmentFile=/etc/publicpolicy.env
ExecStart=/usr/bin/python3 server/app.py
Restart=always
```

`/etc/publicpolicy.env` 权限 `600`，存放密钥。

## 3. Nginx 反代（子路径 `/publicpolicy/`）

要点：

1. `/publicpolicy/` → 静态根  
2. `/publicpolicy/api/` → 反代到 `127.0.0.1:8080`  
3. 限流与上传体积限制在网关再加一层  

示例：

```nginx
# 在 jinhuazhang.top 的 server {} 内
limit_req_zone $binary_remote_zone pp_api:10m rate=10r/m;

server {
    # ... 原有站点配置 ...

    # 静态教材站
    location /publicpolicy/ {
        alias /var/www/publicpolicy/;
        index index.html;
        try_files $uri $uri/ /publicpolicy/index.html;
    }

    # API：转发到 Python 服务
    location /publicpolicy/api/ {
        limit_req zone=pp_api burst=5 nodelay;
        client_max_body_size 12m;

        proxy_pass http://127.0.0.1:8080/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # 仅在前置层覆盖 XFF；Python 侧会取第一跳
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

说明：

- Python `SimpleHTTPRequestHandler` 以站点根为 `web/`；若静态与 API 分目录，可只用反代把 `/publicpolicy/api/*` 打到服务，静态仍由 Nginx 提供。  
- 若 API 与静态同机同目录，也可让 `app.py` 同时伺服静态（当前实现），则可改为 `proxy_pass http://127.0.0.1:8080/publicpolicy/;` 整段反代，更简单。  
- **推荐整段反代**（静态+API 同一进程或同一目录树），少踩 alias 路径问题。

整段反代示例：

```nginx
location /publicpolicy/ {
    limit_req zone=pp_api burst=8 nodelay;
    client_max_body_size 12m;
    proxy_pass http://127.0.0.1:8080/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 120s;
}
```

注意：`proxy_pass http://127.0.0.1:8080/;` 会把 `/publicpolicy/xxx` 转成 `/xxx`，与当前以 `web/` 为根的 app.py 匹配。

## 4. 前端生产配置

`web/js/config.js`：

```js
mock: false,
qaEndpoint: "api/ask",
contributeEndpoint: "api/contribute",
accessToken: "",  // 若启用 PP_ACCESS_TOKEN 再填
```

页面均用相对路径，挂到 `/publicpolicy/` 即可。

## 5. 验收清单

- [ ] `https://jinhuazhang.top/publicpolicy/index.html` 可开  
- [ ] 目录 / 章节 / 图谱 / 编写与文献 正常  
- [ ] 问答有 Fireworks 回答（非 mock 文案）  
- [ ] 共创能创建 GitHub Issue  
- [ ] 恶意 Origin / 超频请求被拒  
- [ ] 密钥未出现在前端与 Git 历史  

## 6. 安全提醒

- Token 仅服务端；GitHub 用 fine-grained 最小权限  
- 生产 CORS 勿用 `*`  
- 网关限流 + 应用内限流双保险  
- 审计日志：`server/logs/audit.jsonl`  
