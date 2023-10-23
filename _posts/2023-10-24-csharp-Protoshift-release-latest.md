---
layout: post
title: "csharp-Protoshift v1.0.1 release"
date: 2023-10-23
tags: [csharp-Protoshift]
comments: true
author: YYHEggEgg
---

[csharp-Protoshift v1.0.1](https://github.com/YYHEggEgg/csharp-Protoshift/releases/tag/v1.0.1) has been released!

# csharp-Protoshift

csharp-Protoshift is an advanced, manageable compatibility layer for a certain anime game.

## Current Features

- Basic Protoshift functionality. It can enhance the compatibility of certain secondary game servers.
- Simple proxy server management commands.
- Automatic compilation and execution of Windy (lua) scripts. The image below shows the `welcome-to-csharp-Protoshift.lua` configuration that is enabled by default in `config_example.json`. You can disable it by setting `#/WindyConfig/OnlineExecWindys/[0]/OnlineExecMode` to `Disabled`.

  ![Windy Preview](https://github.com/YYHEggEgg/csharp-Protoshift/blob/main/csharp-Protoshift/Images/windy_welcome-to-csharp-Protoshift.jpg)

- Protobuf / `query_cur_region` / Ec2b and other utility commands.

For information about building and usage, please view our [README](https://github.com/YYHEggEgg/csharp-Protoshift/blob/main/README.md).
