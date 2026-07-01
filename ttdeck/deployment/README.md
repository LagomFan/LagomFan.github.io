# TTDeck 部署教程

这里放 TTDeck Companion Server 的用户部署教程。Companion Server 是用户自己部署的只读服务，TTDeck iPhone app 只连接用户填写的服务器地址；开发者服务器不参与车辆数据流。

## 两种教程形式

- [手动配置教程](manual-setup.md)：给自己动手配置的用户，覆盖 `en-US`、`en-GB`、`zh-Hans`、`zh-Hant`、`ja`、`ko`、`de`、`fr`、`es`、`it`。
- [AI 配置文档](ai-setup-guide.md)：英文 Markdown。给用户复制链接或全文发送给 AI，让 AI 按用户自己的 NAS、VPS、Docker、反向代理环境一步步协助配置。

## 服务器地址怎么选

TTDeck 不强制公网。服务器地址只需要满足一个条件：iPhone 能访问到 Companion Server。

- 家庭 NAS / 局域网：可以使用 `http://192.168.1.20:4020` 或类似局域网地址。离开该网络后 App 无法刷新，除非用户自己配置了远程访问。
- 公网 HTTPS：适合需要外出访问的用户。建议使用自有域名和 HTTPS，例如 `https://ttdeck.example.com`。
- 本机测试：`http://127.0.0.1:4020` 只适合同一台机器或模拟器测试，不适合作为 iPhone 真机地址。

## 发布链接说明

当前公开托管目标：

- 手动教程入口（浏览器）：`https://lagomfan.github.io/ttdeck/deployment/manual-setup.md`
- 手动教程入口（Markdown 原文）：`https://raw.githubusercontent.com/LagomFan/LagomFan.github.io/main/ttdeck/deployment/manual-setup.md`
- AI 配置文档（浏览器）：`https://lagomfan.github.io/ttdeck/deployment/ai-setup-guide.md`
- AI 配置文档（Markdown 原文，推荐发给 AI）：`https://raw.githubusercontent.com/LagomFan/LagomFan.github.io/main/ttdeck/deployment/ai-setup-guide.md`
