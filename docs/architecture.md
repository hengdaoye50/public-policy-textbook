# 部署与共创架构说明

## 站点

| 项 | 说明 |
|---|---|
| 线上路径 | `https://jinhuazhang.top/publicpolicy/` |
| 静态资源 | `index.html` / `book.html` / `chapter.html` / `about.html` + `css/` `js/` `data/` `assets/` |
| 数据生成 | 源 Markdown → `build_data.py` → `data/*.json` |

前端**不保存**任何模型或 GitHub Token。

## 共创链路

```
用户（网站表单 / GitHub）
        │
        ▼
服务端代理（jinhuazhang.top 或同域 API）
  · 校验附件类型与大小
  · 模型整理为标题 + 正文 + 标签
        │
        ├─► GitHub Issue（勘误、建议、案例线索）
        └─► GitHub Pull Request（可直接合并的文稿补丁）
```

目标仓库：本仓库 `hengdaoye50/public-policy-textbook`。

## 问答链路

```
用户提问（网站首页）
        │
        ▼
服务端 /api/ask
  · 限流与审计
  · 调用模型 API（密钥仅存服务端）
  · 以本仓库正文为知识库（RAG 或全文摘要）
  · 输出控制在约 1000 字
```

## 建议的环境变量（仅服务端）

- `GITHUB_TOKEN`：具备 issues / pull_requests 权限的 fine-grained token  
- `LLM_API_KEY` / `LLM_BASE_URL` / `LLM_MODEL`  
- `ALLOWED_ORIGIN=https://jinhuazhang.top`

## 安全要点

1. Token 不进前端仓库、不进浏览器、不进日志。  
2. `/api/*` 做鉴权或至少限流 + 来源校验。  
3. 上传附件在服务端再做一次类型与大小检查。  
4. Issue/PR 创建使用最小权限 token，并记录审计日志。
