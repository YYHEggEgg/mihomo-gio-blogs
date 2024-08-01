---
layout: post
title: "ZZZ Server Emu Wish system Design Brief"
date: 2024-08-01
tags: [ZZZ, PS, Design]
comments: true
author: YYHEggEgg
---

This article mainly describes the specific logic of the wish system I implemented for the ZZZ Server Emulator [JaneDoe-ZS](https://git.xeondev.com/NewEriduPubSec/JaneDoe-ZS.git) and its connection with the configuration file/save.

The design goal of this wish system is to support all possible policies in the mihoyo gacha through a single configuration file.

It is recommended to refer to the following materials for reading:

- Configuration file [gacha.jsonc](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/assets/GachaConfig/gacha.jsonc)
- Archive implementation [bin.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_proto/out/bin.rs#L122) (protobuf source file is not publicly available due to regulations)

The specific implementation is a lot of a mess, but if you want it, the link is also set up:

- CsReq / ScRsp docking code [gacha.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_gameserver/src/handlers/gacha.rs)
- Main logic implementation [gacha_model.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_gameserver/src/logic/gacha/gacha_model.rs)
- Code model of the configuration file [gacha_config.rs](https://git.xeondev.com/YYHEggEgg/JaneDoe-ZS/src/branch/master/nap_data/src/gacha/gacha_config.rs)

## Outline of the idea

First of all, regarding the general design direction of the Wish system, we refer to the theory in the video of @一棵平衡树 on Bilibili:

- The pulling system is a finite state automaton that only needs to output the results of a pull.
- First, determine the rarity: through the basic probability mechanism and the pity mechanism to determine what rarity's item the player gets.
- Next, determine the category in which the item is obtained by using the hard pity mechanism, the "divine casting set track" mechanism and the smoothing mechanism (a mechanism that prevents the player from keeping weapons out without characters or vice versa).
- Finally, by equalizing the probabilities within the same category, it is determined what item the player ends up getting.

Obviously, the following parameters would actually need to be provided if all the functionality in this description is to be realized:

- The player's pity progress at each star level, e.g. 8 pulls already without an A, 70 pulls already without an S;
- The player's previous acquisitions of items of a certain category. For example, the last soft pity was skewed, or it was already full of fixed tracks.

However, during the design process, we actually observed the following phenomena:

- Pity needs to allow for inheritance across pools, including the rotation of character pools with the dual character pool mechanic utilizing this feature.
- Wish and other pull mechanisms do not change in the long term and will work for multiple pools.
- Weapon pools have a large and small pity mechanism, but can only decide if they come out as a standard or event-exclusive, so implementing a "God Casting Fixed Track" mechanism needs to be considered as well.
- Some Wish pools have introduced specific preferential policies, including discount on pulls, early guarantee, and free agent selection.

Therefore, the final design solution was to separate the global and pool configurations. The global configuration is mainly stored in a separate table, with strings as indexes, and the pool configuration only needs to reference strings.

## How the gacha pool itself is configured

The configuration of the gacha pool itself, in addition to the necessary IDs, pity inheritance group (`sharing_guarantee_info_category`) and other basic parameters, mainly needs to define what items can be obtained.

Let's take a look at a sample configuration of a single gacha pool:

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

First, the cross-pool shared pity can be defined through `sharing_guarantee_info_category`, and the pity progress of the same configuration will be shared. The direct fields are mainly some values commonly used in the game, and have little to do with the design, but are quite related to the client.

Let's look at the values inside `gacha_items`. The full information of each star level is defined, and the items are distinguished by category (`category`), and each category also has its own policy. The values ending with `tag` mainly refer to various models mentioned in the following "Global Configuration".

```jsonc
[
  {
    // In this exclusive frequency band, the basic probability of obtaining an S-level agent through frequency adjustment is 0.600%,
    // When an S-level agent is obtained through frequency adjustment, there is a 50.000% chance that it will be the exclusive S-level agent of this period.
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

The most important distinguishing attribute in the item list is the rarity (`rarity`), followed by which probability model is used (only one can be selected). Under each rarity, the items are further divided into categories, and each category needs to carry `category_tag` (the dictionary key in the above code, such as `Standard:Agent`), but it needs to match the various `included_category_tags` defined in the category guarantee policy.

For the item configuration of each rarity, you can simply reference the category guarantee policy through `category_guarantee_policy_tags`, but you need to pay attention to:

- The intersection of all the categories referenced by the guarantee policy and the categories defined in this rarity must be non-empty;
- Only a single category guarantee policy with a fixed track can be used (`chooseable` is `true`);
- If a category guarantee policy with a fixed track is used, then the category referenced by this policy must be a subset of the intersection of the categories referenced by other policies and the categories defined in this rarity.

The above description is very obscure and difficult to understand, but in short, it is because:

- Players cannot be unable to not draw any items in the extreme case where all the guarantees are triggered;
- Players cannot be unable to draw items that do not include the ones they have chosen in the extreme case mentioned above.

In addition, in the configuration of each category, `is_promotional_items` is used to display which items are currently being promoted to the client; these parameters do not affect the actual drawing process, and are only used to display to the client.

## Global Wish configuration

Let's scroll to the bottom of the file and can see the global model configuration:

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
  // Define additional properties for wish display information.
  "common_properties": {
    // ...
  }
}
```

### Probability model

In `probability_model_map`, the global definition of the wish probability model. For each star level, each mechanism needs to be defined separately, but the ProbabilityModel itself does not define the applicable star level, only carries the Tag used for reference.

Let's take a look at a typical mihoyo gacha (character pool) S-level item model:

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

It's clear at a glance. The initial probability is $0.6\%$ and does not change; from the 73rd pull onwards, the probability increases by $6\%$ per pull. It should be noted that the `start_chance_percent` here specifies the probability percentage of this pull, that is, the probability of the 74th pull is $6.6\%$.

### Extra items policy

The following part lists a set of Stardust / Starburst return models: ~~(I can't remember the strange names outside of Genshin)~~

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

It is referenced in the configuration of the "rarity" level under the Wish pool. There was originally a consideration of doing a "agent profile" type of conversion logic, but later it was thought unnecessary, this should be handled uniformly by the code that adds characters.

Of course, this also reflects that this design is probably not appropriate, because if you get the W-Engine, you should return a fixed value, and if you get an agent, you should throw it to another place to handle the conversion; this reasoning should be that the configuration should be done at the "category" level. It's too cumbersome to go up one level.

### `Discount` discount model

This kind is really the most complicated. Anyway, each policy now only has one implementation, all put here:

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

Let's explain according to the actual case: The newcomer discount of the ZZZ standard wish pool is "20% off for the first 50 pulls", "The first 50 pulls must obtain an S-level agent" and "After 300 pulls, you can choose an S-level agent from the 'standard wish pool'".

The main difficulty is that "The first 50 pulls must obtain an S-level agent", which is a bit difficult to understand, mainly because of the impact of corner cases:

- If a player is lucky and pulls an S-level agent before the 50th pull, then sorry, this guarantee is gone; anyway, that's how it's implemented in the official server.
- But if he is lucky, then you can't take his guarantee by a ball, right? So what comes out must still be an agent.

From this we find that splitting it into two policies is reasonable, and they need to be completely separated from each other:

- The first 50 pulls must obtain an S-level agent (`advanced_guarantee_map` defines 50 as the highest guarantee);
- The first S-level agent must be an agent (`must_gain_item_map` defines that an S-level agent must be obtained).

---

`ten_pull_discount_map` has nothing to talk about, let's go back and talk about the agent selection policy of `free_select_map`.

Agent selection was originally a very obvious thing, but here we expanded its function a bit: `milestones` is an array. It defines the number of pulls required to obtain the qualification for selection, that is, if it is defined as `[300, 200]`, then it can support the user to obtain the second selection qualification when pulling the 500th time.

The main storage method is two parts, one part is how many times the user has obtained selection (under this policy); the other part is how many pulls the user has made under the condition of this policy, and the value is reduced after the exchange of this part. That is, although the `milestones` here is `[300]`, if it is changed to `[300, 200]` in the future, the pulls over 300 that the player has obtained will theoretically be retained and calculated together with the new selection condition.

### Guarantee policy model

The model currently in use is directly as follows:

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

The `CategoryGuaranteePolicy` itself can be applied to multiple star levels and multiple Wish pools, because when the actual drawing guarantee mechanism is triggered, only the ones that are actually available in the Wish pool will be selected. `promotional-items` is the description of the "big guarantee" mechanism. The category Tag it contains has both agents and W-Engines, and when it is applied to the Wish pool, if one of them is defined, it will take effect. `trigger_on_failure_times: 1` means that it will guarantee an S-level agent after a 50/50 failure.

Note that `chooseable-up-bangboo` has `trigger_on_failure_times` set to 0. After the publish of ZZZ, the Bangboo pool adopted a non-failure design, so this guarantee policy will definitely be triggered. Of course, it is stipulated that `chooseable: true` can be selected for promotion.

In addition, this model was originally compatible with the "divine casting set track" mechanism, and `clear_status_on_target_changed` means that when the UP is changed, the fixed track value is cleared.

### Constant values of additional properties

Here are some properties that are specific to the client and have to be set:

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

- The three Tags are defined due to the display requirements of the client;
- Note that in ZZZ, the rarity of S-level items is 4, not 5. Following this, B specifies a two-star item.

## Current Wish status saving

Wish status is stored by a dictionary whose key is the pool's pity inheritance category (`sharing_guarantee_info_category`), and all pools in this group share the pity information.

The pity information mainly contains the Pity of the player at each rarity and the triggering process of each pity (in layman's terms, how many times it was skewed). The reason for the Pity designation is that it is the progress when the player **makes his or her next Wish pull**. For example, if a player has never pulled in a certain pool, his Pity should be considered 1.

The advantage of this is that the definition of Pity changes from a future state to the present when the player actually makes a Wish pull.

- When a player pulls, the Pity value is taken directly from the player's save to find the corresponding probability;
- After the pull is complete, write to the draw record store before changing the player's Wish state, reflecting the state of the card as it is being drawn.

## Wish record saving

In the server_only Proto of srv-bins, the `GachaRecordBin` that records card pulls is actually the equivalent of recording all the information about the entire Wish pool. However, since we believe that the configuration file for this card pulling implementation is quite maintainable, only the `gacha_schedule_id` will be referenced in the Wish record, and nothing more will be recorded.

The record of a single pull a player has ever made is primarily the following information:

- The `gacha_schedule_id` of the target Wish pool;
- The time at which the Wish pull was performed;
- The final item obtained;
- The status of the Wish at the time of the pull, which is a complete copy.

## Details (or TODO List)

- During the second test of ZZZ, the (unreleased) weapon pool had a "search priority" mechanism similar to the "divine casting set track," but it could not be made to work at that time.
- This system does not implement the function of making a pool disappear after a specified number of pulls.
- Some of the validation logic in this article (such as the aforementioned validation of the set relationship of items contained in categories) may not be implemented in the code.
