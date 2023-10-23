---
layout: post
title: "csharp-Protoshift 常见问题 FAQs"
date: 2023-10-23
tags: [Server, csharp-Protoshift]
comments: true
author: YYHEggEgg
---

这里是您可能遇到的一些常见问题合集。它将会持续进行更新。本位最后更新于 `2023/10/23`。

这里并不包含高级开发的说明。有关这些信息请参阅 csharp-Protoshift Wiki [Development](https://github.com/YYHEggEgg/csharp-Protoshift/wiki/CN_Development) 页面。

## 无法启动服务器：`PropertyRequired: #/NetConfig.RemoteAddress`

抓到一个没有仔细看 README 的！

打开 `csharp-Protoshift/config.json`，便可以看到注释行：

```json
{
// ...
  "NetConfig": {
    "BindAddress": "0.0.0.0",
    "BindPort": 22102,
    // Set your real server ip here
    // "RemoteAddress": {
    //   "IpAddress": "127.0.0.1",
    //   "AddressPort": 20041
    // }
  },
// ...
}
```

取消注释并以您的真实服务器地址替换，就可以完成 `config.json` 的最小配置。

## 我如何以仅工具模式启动程序？

直接运行 `./run --utils-only` 即可。

如果正在使用 `dotnet run`，可使用 `dotnet run -- --utils-only`。

## 我能否停止 `packet.log` / `player.stat.log` 的记录？

您只需要在 `config.json` 中更改相应的 `Enable` 值即可指定是否创建日志。

```json
{
  "$schema": "resources/config-schemas/config_schema_latest.json",
  "EnableFullPacketLog": true,
  "EnablePlayerStatLog": true,
  // ...
}
```

如果您想要在 `latest.packet.log` 中取消特定包的记录，您可以通过更改 `#/PacketLogExcluding` 来实现。

如果您想要在 `latest.player.stat.log` 中取消特定 Category 的记录，目前没有选项可以进行这种控制。您也可以 [创建 Issue](https://github.com/YYHEggEgg/csharp-Protoshift/issues/new/choose) 来呼吁添加对于此功能的支持。

有关这两种日志的具体功能与格式，请参阅 csharp-Protoshift Wiki: [Development - `latest.packet.log` 注解](https://github.com/YYHEggEgg/csharp-Protoshift/wiki/CN_Development#latestpacketlog-注解) / [Development - `latest.player.stat.log` 注解](https://github.com/YYHEggEgg/csharp-Protoshift/wiki/CN_Development#latestplayerstatlog-注解)。
