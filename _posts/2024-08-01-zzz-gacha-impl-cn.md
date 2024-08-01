---
layout: post
title: "ZZZ Server Emu 抽卡系统 设计简要"
date: 2024-08-01
tags: [ZZZ, PS, Design]
comments: true
author: YYHEggEgg
---

本文主要讲述本人为 ZZZ Server Emulator [JaneDoe-ZS](https://git.xeondev.com/NewEriduPubSec/JaneDoe-ZS.git) 实现的抽卡系统的具体逻辑以及其与配置文件/存档间的联系。

此抽卡系统的设计目标是通过单个配置文件，支持所有米池可能出现的策略。

建议参考以下资料阅读：

- 配置文件 [gacha.jsonc](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/assets/GachaConfig/gacha.jsonc)
- 存档实现 [bin.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_proto/out/bin.rs#L122) (protobuf 源文件因规定不作公开)

具体实现就有点屎山代码了，想看的话传送门也摆上：

- CsReq / ScRsp 对接代码 [gacha.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_gameserver/src/handlers/gacha.rs)
- 抽取主要逻辑实现 [gacha_model.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_gameserver/src/logic/gacha/gacha_model.rs)
- 配置文件的代码模型 [gacha_config.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_data/src/gacha/gacha_config.rs)

## 思想概要

首先，关于抽卡系统的总设计大方向，参考了 b 站@一棵平衡树 视频中的理论：

- 抽卡系统是一个有限状态自动机，只需输出抽卡结果。
- 首先确定星级：通过基础概率机制和保底机制的作用来确定玩家获得的是什么星级的物品。
- 其次确定类别：通过大保底机制、「神铸定轨」机制与平稳机制 (一种防止玩家一直出武器不出角色或反过来的机制) 确定获得物品的类别。
- 最后，通过在同一类别中均分概率，决定玩家最终获得什么物品。

显然，如果要实现这段描述中的所有功能，其实只需要提供以下参数：

- 玩家在各个星级的保底进度，例如已经 8 抽没有出 A，已经 70 抽没有出 S；
- 玩家之前获得某类别物品的情况。例如上一个小保底歪了，或者已经满定轨了。

但是在设计的过程中我们其实观察到以下现象：

- 保底需要允许跨池继承，包括角色池的轮替与双角色池机制都利用这一功能。
- 保底等抽卡机制长期不会变化，并且会对多个卡池均生效。
- 武器池具有大小保底机制，但只能决定出的是常驻还是限定，因此实现「神铸定轨」机制也需要考虑。
- 一些卡池引入了特定的优惠策略，包括抽取打折、提前保底、代理人自选等。

因此，最终采取了全局与卡池配置分离的设计方案。全局配置主要通过单独的表存放，以字符串作为索引，卡池配置只需要引用字符串即可。

## 卡池本身配置

卡池本身的配置除了必要的 ID、保底继承组别 (`sharing_guarantee_info_category`) 等基本参数以外，主要还是要定义可以抽到哪些物品。

来看单个卡池的配置样例：

```jsonc
{
  "gacha_schedule_id": 2001001,
  "gacha_parent_schedule_id": 2001,
  "comment": "1.0, First Half, Allen / 艾莲, Agent",
  "gacha_type": 2,
  "cost_item_id": 111,
  "start_time": "2024-07-04T06:00:00+08:00",
  "end_time": "2024-07-24T11:59:59+08:00",
  "sharing_guarantee_info_category": "Character Event Wish",
  "gacha_items": [
    // ...
  ]
}
```

首先，跨池共享保底可通过 `sharing_guarantee_info_category` 定义，该配置相同的卡池将会共享保底进度。直属的字段主要是游戏里例行使用的一些值，与设计并无太大联系，跟客户端相关性倒是比较大。

我们来看 `gacha_items` 里面的值。里面定义每个星级的全部信息，其中的物品按照类别 (`category`) 来进行区分，每个类别也都有自己的策略。各个 `tag` 结尾的值主要引用下文「全局配置」中会提到的各种模型。

```jsonc
[
  {
    // 在本期独家频段中，调频获取S级代理人的基础概率为 0.600%，
    // 当调频获取到S级代理人时，有 50.000% 的概率为本期限定S级代理人。
    "rarity": 4,
    "extra_items_policy_tags": ["S-item"],
    "probability_model_tag": "get-S-90-AgentPool",
    "category_guarantee_policy_tags": ["promotional-items"],
    "categories": {
      "Standard:Agent": {
        "item_ids": [
          1021, // 猫又
          1101, // 珂蕾妲
          1041, // 「11号」
          1141, // 莱卡恩
          1181, // 格莉丝
          1211 // 丽娜
        ],
        "category_weight": 50
      },
      "Event-Exclusive:Agent": {
        "is_promotional_items": true,
        "item_ids": [
          1191 // 艾莲
        ],
        "category_weight": 50
      }
    }
  }
  // ...
]
```

物品列表中最重要的区分属性是稀有度 (`rarity`)，其次是使用了哪种概率模型 (只能选择一种)。每种稀有度下面再细分产生物品类别，每个类别都需要携带 `category_tag` (上示代码中的字典键，如 `Standard:Agent`)，但需要与类别保底策略中定义的各个 `included_category_tags` 相符。

对于每个稀有度的物品配置，在 `category_guarantee_policy_tags` 引用类别保底策略即可，但需要注意：

- 所有策略引用类别与本稀有度中定义的类别，其交集必须非空；
- 只能使用单个定轨用类别保底策略 (`chooseable` 为 `true`);
- 如果使用了定轨用类别保底策略，则在其他策略引用类别与本稀有度中定义的类别的交集之上，该定轨用类别保底策略引用的类别必须是其子集。

上面的描述非常晦涩难懂，但总之是因为：

- 不可以使玩家在所有保底均触发的极端情况下，导致抽不到任何物品；
- 不可以使玩家在上述极端情况下，被允许抽到的物品不包含自己定轨选择的。

此外，在每个类别的配置中，`is_promotional_items` 用于向客户端显示哪些物品正在进行概率 UP；这些参数均不影响实际的抽卡过程，仅用于向客户端展示。

## 全局抽卡配置

我们翻到文件下方，可以看到全局的模型配置：

```jsonc
{
  // ...
  "probability_model_map": {
    // ...
  },
  "extra_items_policy_map": {
    // ...
  },
  "discount_policies": {
    // ...
  },
  "category_guarantee_policy_map": {
    // ...
  },
  // 定义供祈愿显示信息使用的额外属性。
  "common_properties": {
    // ...
  }
}
```

### 概率模型

`probability_model_map` 中是全局定义的抽卡概率模型。对于每种星级的每种机制需要分别定义，但 ProbabilityModel 本身不定义适用的星级，只携带用于被引用的 Tag。

来看一个典型的米池 (角色池) S 级物品模型：

```jsonc
{
  "probability_model_map": {
    "get-S-90-AgentPool": {
      "points": [
        {
          "start_pity": 1,
          "start_chance_percent": 0.6
        },
        {
          "start_pity": 73,
          "start_chance_percent": 0.6,
          "increment_percent": 6
        }
      ]
    }
  }
}
```

一目了然。初始概率为 $0.6\%$ 而不变；从 73 抽往后，每抽概率上升 $6\%$. 需要注意的是这里的 `start_chance_percent` 规定的是这一抽的概率百分比，也就是第 74 抽出的概率才是 $6.6\%$.

### 物品返还模型

下面这部分列出了一组星尘/星辉返还模型: ~~(原以外的那些奇怪的名字都记不住)~~

```jsonc
{
  "extra_items_policy_map": {
    "S-item": {
      "id": 115, // 信号余波
      "count": 40
    },
    "A-item": {
      "id": 115, // 信号余波
      "count": 8
    },
    "B-item": {
      "id": 117, // 信号残响
      "count": 20
    }
  }
}
```

它在卡池配置下「稀有度」一级的配置中被引用。本来有想过做「代理人档案」一类的转化逻辑，后来想想没必要，这应该是要供添加角色的代码那边统一处理的。

当然这也体现出这样设计大概欠妥当，因为如果获得音擎就应该返回固定值，反之获得了代理人就应该扔到其他地方处理转换；这么推理那么应该把配置做到「类别」一级。嫌太繁琐索性就高一级了。

### `Discount` 优惠模型

这一类真的是最复杂的。反正每种策略现在都只有一个实现，都放这算了:

```jsonc
{
  "discount_policies": {
    "ten_pull_discount_map": {
      "5x-10-poll-discount-8": {
        "use_limit": 5,
        "discounted_prize": 8
      }
    },
    "must_gain_item_map": {
      "first-S-Agent": {
        "use_limit": 1,
        "rarity": 4,
        "category_tag": "Standard:Agent"
      }
    },
    "advanced_guarantee_map": {
      "50-poll-S": {
        "use_limit": 1,
        "rarity": 4,
        "guarantee_pity": 50
      }
    },
    "free_select_map": {
      "standard-banner-300-S": {
        // ...
        "rarity": 4,
        "category_tags": ["Standard:Agent"],
        "milestones": [300]
      }
    }
  }
}
```

我们不妨按照实际案例来解释：绝区零常驻池的新人优惠是「前 50 抽 8 折」、「首个 50 抽必出 S 级代理人」和「进行 300 抽后可以自选一个『常驻频段』S 级代理人」。

主要困难的是「首个 50 抽必出 S 级代理人」，理解起来有点难度，主要包含 corner case 的影响:

- 如果一个人在 50 抽保底前欧气爆棚，提前出了，那么不好意思，这个保底没有了；反正官服这么实现的。
- 但如果他欧了，那么你总不能让他出个球然后还吞他保底吧？所以出的还得是一个代理人。

由此我们发现，将它拆分成两个策略才是合理的，彼此之间需要完全分离：

- 首个 50 抽必出 S (`advanced_guarantee_map` 中定义 50 发为最高保底);
- 首个 S 必定是代理人 (`must_gain_item_map` 中定义必须获取 `Standard:Agent`).

---

`ten_pull_discount_map` 没有什么聊头，我们回过头来谈谈 `free_select_map` 的这个代理人自选策略。

代理人自选本来也是很显然的一件事情，但是这边拓展了一下它的功能: `milestones` 是一个数组。其定义获取自选需要的抽数，也就是说，如果为其定义 `[300, 200]`，那么它可以支持用户抽第 500 抽时获得第二次自选的资格。

主要存储方式是两部分，一部分是用户获取过多少次自选 (在本策略下)；一部分是用户在本策略启用的条件下抽过多少发，进行了兑换这个值就减去所需的抽数。也就是说，虽然这里的 `milestones` 为 `[300]`，如果未来将其改为 `[300, 200]`，那么玩家之前超过 300 的抽数理论上会保留下来一起计算新的自选条件。

### 保底策略模型

同样直接看现在使用的模型：

```jsonc
{
  "category_guarantee_policy_map": {
    "promotional-items": {
      "included_category_tags": [
        "Event-Exclusive:Agent",
        "Event-Exclusive:W-Engine"
      ],
      "trigger_on_failure_times": 1,
      "clear_status_on_target_changed": false,
      "chooseable": false
    },
    "chooseable-up-bangboo": {
      "included_category_tags": ["Standard:Bangboo", "Event-Exclusive:Bangboo"],
      "trigger_on_failure_times": 0,
      "clear_status_on_target_changed": false,
      "chooseable": true
    }
  }
}
```

`CategoryGuaranteePolicy` 本身可以适用于多种星级与多种卡池，因为在实际抽卡的保底机制触发时只会选择卡池中实际可用的那些。`promotional-items` 就是「大保底」机制的描述。它包含的类别 Tag 同时有代理人和音擎，将它运用到卡池的时候如果哪个有定义，哪个就会生效。`trigger_on_failure_times: 1` 就是歪一次触发必出的意思。

注意到 `chooseable-up-bangboo` 的 `trigger_on_failure_times` 为 0. 绝区零公测后邦布池采取了不歪的设计，这里保证保底策略一定触发。当然它是规定了 `chooseable: true` 的，可选 UP.

另外，这个模型本来是可以兼容「神铸定轨」机制的，`clear_status_on_target_changed` 就是更换 UP 清空定轨值的意思。

### 额外属性常量值

这里定义了特定于客户端而不得不设定的一些属性：

```jsonc
{
  "common_properties": {
    "up_item_category_tag": "promotional-items",
    "s_item_rarity": 4,
    "a_item_rarity": 3,
    "ten_pull_discount_tag": "5x-10-poll-discount-8",
    "newcomer_advanced_s_tag": "50-poll-S"
  }
}
```

- 三个 Tag 都是由于客户端的显示需求而定义的；
- 注意到在 ZZZ 中，S 物品的稀有度是 4 而不是 5。以此向下类推，B 是二星物品。

## 当前抽卡状态存储

抽卡状态由一个字典存储，它的键就是卡池的保底继承组别 (`sharing_guarantee_info_category`)，本组的卡池均共享保底信息。

保底信息中主要包含每个稀有度下玩家的 Pity、各个保底策略的触发进程 (通俗来说就是歪了几次) 和优惠策略的已使用次数。这里使用 Pity 称呼的原因是，它是玩家**进行下一次抽卡时的保底进度**。例如如果玩家在某个卡池中从未抽取过，则他的 Pity 应视为 1.

这样做的好处是在玩家实际进行一次抽卡时，其 Pity 的定义便从将来的状态变成了现在。

- 在玩家进行抽取时，直接取出 Pity 值便可查找相应的概率；
- 抽取完毕后，在更改玩家抽卡状态前写入抽卡记录的存储，可以反映抽卡进行时的各项状态。

## 抽卡记录存储

在 srv-bins 的 server_only Proto 中，对记录抽卡的 `GachaRecordBin` 实际上相当于记录了整个卡池的所有信息。但由于我们认为本抽卡实现的配置文件具有相当好的可维护性，因此抽卡记录中只会引用 `gacha_schedule_id`，而不会进行更多记录。

一个玩家曾进行过的单次抽卡记录主要就是以下信息：

- 目标卡池的 `gacha_schedule_id`；
- 进行抽卡的时间；
- 最终获取的物品 (包含额外物品)；
- 抽卡进行时的抽卡状态，是一份完整的复制。

## 细节记录 (又或者是 TODO List)

- 绝区零在二测时 (未开放的) 武器池具有与「神铸定轨」类似的「调频校准」机制，但当时无法令其工作。
- 这套系统没有实现指定抽数用完就让一个卡池原地消失的功能。
- 本文里的一些验证逻辑 (例如前述关于类别所含物品集合关系的验证)，在代码里可能没有实现。
