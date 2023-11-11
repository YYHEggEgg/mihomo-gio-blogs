---
layout: post
title: "csharp-Protoshift v1.0.2 release"
date: 2023-11-10
tags: [csharp-Protoshift]
comments: true
author: YYHEggEgg
---

[csharp-Protoshift v1.0.2](https://github.com/YYHEggEgg/csharp-Protoshift/releases/tag/v1.0.2.1) has been released!

csharp-Protoshift is an advanced, manageable compatibility layer for a certain anime game.

## Update - v1.0.2

- Support initiated JIT compiling for Protoshift Handlers. For information, please view [PR #36](https://github.com/YYHEggEgg/csharp-Protoshift/pull/36).
- Fixed the issue whereby `HandlerGenerator` being unable to fetch and restore Protos due to the Git `safe.directory` configuration in certain environments.
- Fixed the issue whereby `HandlerGenerator` being unable to fetch and restore Protos when there are spaces in the project directory.
- Fixed the issue whereby `csharp-Protoshift` being unable to invoke `luac` to compile Windy lua scripts when there are spaces in the project directory.
- Fixed the issue whereby `HandlerGenerator` displaying abnormal exit prompt text when the external application invocation fails.
- Fixed the issue whereby `HandlerGenerator` being unable to correctly invoke Powershell in Windows to execute post-generation scripts when there are spaces in the project directory.
- Fixed the issue where `csharp-Protoshift-Replay` could not start due to the inability to find the resource folder.
- Fixed the problem where the command line option `--orderby-packet-speed` of `ProtoshiftBenchmark` did not actually take effect.
- Added support for proactive JIT compilation to `csharp-Protoshift-Replay`.
- Fixed the issue where `ProtoshiftBenchmark` and `csharp-Protoshift-Replay` prompted error due to syntax errors during compilation.
- Added `-f, --source-file` and `--fully-replay-packet-time` command line options to `csharp-Protoshift-Replay`.
- Fixed the running issues of the "run-benchmark" series scripts.
- Added more built-in scripts. For more information, please refer to [Wiki - Development - Built-in Scripts](https://github.com/YYHEggEgg/csharp-Protoshift/wiki/EN_Development#built-in-scripts).

## Current Features

- Basic Protoshift functionality. It can enhance the compatibility of certain secondary game servers.
- Simple proxy server management commands.
- Automatic compilation and execution of Windy (lua) scripts. The image below shows the `welcome-to-csharp-Protoshift.lua` configuration that is enabled by default in `config_example.json`. You can disable it by setting `#/WindyConfig/OnlineExecWindyMode` to `Disabled`.

  ![Windy Preview](https://github.com/YYHEggEgg/csharp-Protoshift/blob/v1.0.2.1/csharp-Protoshift/Images/windy_welcome-to-csharp-Protoshift.jpg)

- Protobuf / `query_cur_region` / Ec2b and other utility commands.

For more information about using, please view our [README](https://github.com/YYHEggEgg/csharp-Protoshift/blob/v1.0.2.1/README.md).

<div align="center">
  <a href="https://discord.gg/NcAjuCSFvZ">
    <img alt="Discord - miHomo Software" src="https://img.shields.io/discord/1144970607616860171?label=Discord&logo=discord&style=for-the-badge">
  </a>
</div>
