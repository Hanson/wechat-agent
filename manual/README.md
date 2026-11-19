# 聚合聊天协议 · Agent 对接文档

> 接口表由文档管线自动维护、与线上服务核对一致；对接指南见本目录各手册。

| 协议 | 接口面 | 零参数待补 | OpenAPI |
|---|---|---|---|
| 企微 PC 协议 | 184 | 0 | `openapi/wework_pc/` |
| 个微协议 | 199 | 0 | `openapi/wechat/` |
| Fila 企微手机版 | 168 | 0 | `openapi/fila/` |

| 手册 | 内容 |
|---|---|
| [auth.md](auth.md) | 鉴权与调用约定（GuidRequest 形态/权限模式/频控） |
| [quickstart.md](quickstart.md) | 最小闭环（建实例→取码→配回调→发消息→对账） |
| [login.md](login.md) | 三协议登录状态机（统一枚举 + 各协议原始形态） |
| [callback.md](callback.md) | 回调对接（配置/payload 结构/签名校验/重试熔断） |
| [errors.md](errors.md) | 错误码判型（信封键/平台与协议侧/三零判型） |
| [pitfalls.md](pitfalls.md) | 坑手册（file_type 矩阵/产物格式/键名大小写/二维码形态） |
| [recipes.md](recipes.md) | 接口实现配方（参数怎么填：值来源/转换/常量/上传发送链路衔接） |
