# 聚合聊天 · Agent 开放平台对接文档

[聚合聊天（juhebot）](https://juhebot.com) 开放平台 API 文档。**企业微信 PC、微信个人号、Fila 企微手机端**三协议统一成一个 HTTP API 面，供 AI Agent / 自动化程序 / 开发者自助对接：建实例 → 扫码登录 → 收消息回调 → 发消息 → 对账，全程无需人工后台操作。

## 基础信息

| 项 | 值 |
|---|---|
| API Base URL | `https://chat-admin.juhebot.com` |
| 鉴权 | `app_key` + `app_secret`（请求体携带） |
| 获取凭证 | 联系商务开通后发放（见文末） |
| 协议格式 | 全部 `POST`，`Content-Type: application/json` |
| 消息接收 | HTTP 回调推送（带 HMAC-SHA256 签名，可校验） |

所有端点与示例路径均为相对路径，实际请求 = `Base URL + 路径`，例如创建实例：

```bash
curl -X POST https://chat-admin.juhebot.com/open/CreateInstance \
  -H "Content-Type: application/json" \
  -d '{"app_key":"your_app_key","app_secret":"your_app_secret","channel":1,"expired_at":1767225600}'
```

## 快速开始（七步闭环）

1. **建实例** `POST /open/CreateInstance`（三选一通道：1=企微 PC / 2=微信个人号 / 3=Fila 手机端）
2. **取登录码** `POST /open/GetLoginQrcode`（返回二维码图/链接，Fila 返回 auth_code）
3. **轮询登录** `POST /open/CheckLoginQrcode`（状态归一：waiting → scanned → success）
4. **配回调** `POST /open/SaveCallback`（保存前自动做连通性验证）
5. **验推送** `POST /open/TestCallback`（发送带签名的测试推送， HMAC 签名可复算）
6. **发消息** `POST /open/GuidRequest` 透传协议接口（如 `path: /msg/send_text`）
7. **对账** `POST /open/GetMsgList`

完整示例（含每步请求/响应 JSON）见 **[manual/quickstart.md](manual/quickstart.md)**。

## 手册目录

| 手册 | 内容 |
|---|---|
| [auth.md](manual/auth.md) | 鉴权与调用约定（GuidRequest 形态 / 权限模式 / 频控） |
| [quickstart.md](manual/quickstart.md) | 最小闭环（建实例 → 取码 → 配回调 → 发消息 → 对账） |
| [login.md](manual/login.md) | 三协议登录状态机（统一枚举 + 各协议原始形态） |
| [callback.md](manual/callback.md) | 回调对接（配置 / payload 结构 / 签名校验 / 重试熔断） |
| [errors.md](manual/errors.md) | 错误码判型（信封键 / 平台与协议侧 / 三零判型） |
| [pitfalls.md](manual/pitfalls.md) | 坑手册（file_type 矩阵 / 产物格式 / 键名大小写 / 二维码形态） |
| [recipes.md](manual/recipes.md) | 接口实现配方（参数怎么填：值来源 / 转换 / 常量 / 上传发送链路衔接） |
| [manual/README.md](manual/README.md) | 手册索引与三协议接口总表 |

## OpenAPI 接口面（协议接口全量定义）

`openapi/` 目录按协议按域拆分的 OpenAPI 3.1 文档，是 `GuidRequest` 透传接口的全量清单（共 552 个）：

| 协议 | 接口数 | 目录 |
|---|---|---|
| 企微 PC 协议 | 184 | [openapi/wework_pc/](openapi/wework_pc/) |
| 微信个人号协议 | 199 | [openapi/wechat/](openapi/wechat/) |
| Fila 企微手机版 | 168 | [openapi/fila/](openapi/fila/) |

> OpenAPI 中的路径即 `GuidRequest` 的 `path` 参数取值；接口面之外的路径会被服务端拒绝。

## 适用场景

- AI Agent / LLM 工具集成（Claude、GPT 等工具调用多账号企微/微信收发消息）
- SCRM / 客服系统对接（多实例消息汇聚、自动回复）
- 自动化运营（群发、朋友圈、素材上传下载、账号管理）

## 开通方式

开放平台按 app 维度开通：联系商务付款后创建 `open_app` 发放 `app_key` / `app_secret`，之后实例创建、登录、回调配置、消息收发全部由你的程序/Agent 自助完成。

- 官网：[juhebot.com](https://juhebot.com)
- 技术支持：开通后提供对接群支持

## 关键词

企业微信 API · 微信个人号 API · 企微机器人 · 微信机器人 · 消息回调 · 聚合聊天 · 开放平台 · AI Agent 对接 · 多协议消息网关 · WeWork API · WeChat Bot API
