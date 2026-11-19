# 鉴权与调用约定

> 接口参数真源：`openapi/<proto>/<domain>.yaml`（本仓 openapi 目录）。

## 总入口：GuidRequest 通用透传

所有协议接口（企微 PC / 个微 / Fila 手机版）经同一个开放平台端点透传调用：

```json
{
  "app_key": "your_app_key",
  "app_secret": "your_app_secret",
  "path": "/msg/send_text",
  "data": {
    "guid": "目标设备 guid",
    "content": "hello"
  }
}
```

- `app_key` / `app_secret`：开通后在开放平台应用（open_app）里获取，随 body 传递（不是 header）。
- `path`：协议接口路径，即 `openapi/<proto>/` 里的 path。
- `data`：协议接口的请求体。**`data.guid` 必填**，其余字段见对应接口的 OpenAPI schema。
- 响应：透传协议侧原始响应（信封结构见 errors.md）。

## 设备 guid 从哪来

- 每个协议实例（一台登录了企微/微信/手机版的设备会话）对应一个 guid。
- agent 自助链路：`CreateInstance` 创建实例 → 实例返回 guid → 登录（见 login.md）→ 之后所有调用都用这个 guid。
- 实例有过期时间（`expired_at`），**过期后所有透传调用统一返回错误**（"坐席已过期"），续期走商务/后台。

## 权限模式（调用方需知的限制）

实例的 ApiMode 决定 path 放行范围：

| 模式 | 行为 |
|---|---|
| Full | 按 app↔guid 授权表 scopes 白名单放行（`scopes[0]=="*"` 为全通配） |
| Saas | 前缀黑名单：`/open` `/session` `/arch` `/finder` `/private` 拒绝，`/openim` 豁免 |
| Sph | 15 个路径枚举白名单 |

另有频控（复用线上口径）：部分路径 1 次/分钟/设备；滑动窗口默认 100 次/分钟/app（视频号类 500）。超频返回限流错误。

## 协议判别

调用不需要声明协议——由 guid 所属实例自动路由：

- 企微 PC 实例 → 企微 PC 协议接口面
- 个微实例 → 个微协议接口面
- Fila 手机版（client_type=4000）→ Fila 协议接口面（未知路径返回 `command not supported`）

## 三协议接口目录

| 协议 | 目录 | 接口数 |
|---|---|---|
| 企微 PC（wework_pc） | `openapi/wework_pc/` | 184 |
| 个微（wechat） | `openapi/wechat/` | 199 |
| Fila 手机版（fila） | `openapi/fila/` | 168 |

OpenAPI 文档说明：

- `x-params-source`: 参数来源标记（apifox-schema / call-sites / apifox-example / fila-swagger）。
- 字段标 `x-from-call-sites: true`：官方文档缺失、从实际调用补录的字段（类型未知按 string 占位，实参以调用惯例为准）。
- `x-undocumented-params: true`：接口暂无参数文档（当前为 0 个）。
