---
name: lunarltk-game-skill
description: "用于编写新月杀的技能。当用户要求为新月杀编写技能时，使用该文件"
---

# 新月杀：技能编写说明

## 概述

新月杀是桌游《三国杀》的一种实现，用户在设计好三国杀技能后，可以按照技能描述编写出相应代码实现该技能

## 步骤

1. 创建技能骨架：分析技能名字，找出合适的内部名，创建技能骨架
2. 配置翻译表：分析技能描述，为技能骨架赋值，配置翻译表
3. 推理技能效果：分析技能描述，分析成新月杀的技能实现细节
4. 实现各个效果：根据需求查找每种类型的字段、需要用的API

## 创建技能骨架

1. 通过 fk.CreateSkill(...) 创建技能
2. 分析技能名字，使用汉语拼音命名技能，为技能骨架的name赋值

## 配置翻译表

1. 通过 Fk:loadTranslationTable{...} 配置翻译表
2. 分析用户给出的技能效果描述，根据技能标签，为技能骨架的tags赋值
3. 生成技能台词翻译，每个技能给出两句空白台词

## 推理技能效果

1. 分析用户给出的技能描述，将复杂技能分拆为多个简单效果
2. 分析各效果的类型，将效果分为三类：触发、主动、状态
3. 触发类效果，表示在特定“时机”执行一定动作；主动类效果，包括主动效果("active")和转化效果("viewas")，表示可以主动执行一定动作或转化卡牌；状态类效果，包括距离效果("distance")、禁止效果("prohibit")、攻击范围效果("atkrange")、卡牌增益效果("targetmod")等，表示对角色持续产生影响

## 实现各个效果

1. 通过 addEffect(...) 注册效果。将触发类效果注册在特定“时机”，将其他效果注册在对应字段。有多种效果时，先注册主动效果和转化效果，再注册其余效果
  触发类效果：读取 .\packages\freekill-core\ltk\core\events 目录下的 .lua 文件，获得触发类效果对应的时机
  主动效果：注册为"active"
  转化效果：注册为"viewas"
  距离效果：注册为"distance"
  禁止效果：注册为"prohibit"
  攻击范围效果：注册为"atkrange"
  手牌上限效果：注册为"maxcards"
  卡牌增益效果：注册为"targetmod"
  filter效果：注册为"filter"
2. 标记(mark)拥有后缀， -phase / -turn / -round 分别表示持续到 阶段/回合/轮次结束

## 示例

1. 触发类效果

输入：

摸牌：出牌阶段开始时，或当你受到1点伤害后，你可以令一名其他角色摸一张牌，然后你将一张手牌置于武将牌上，称为“权”。

输出：

local mopai = fk.CreateSkill {
  name = "mopai",
  derived_piles = "quan_pile",
}

Fk:loadTranslationTable{
  ["mopai"] = "摸牌",
  [":mopai"] = "每回合限一次，出牌阶段开始时，或当你受到伤害后，你可以令一名其他角色摸一张牌，然后你将一张手牌置于武将牌上，称为“权”。",

  ["quan_pile"] = "权",
  ["#mopai-choose"] = "摸牌：你可以令一名其他角色摸一张牌，然后你将一张手牌置为“权”",
  ["#mopai-ask"] = "摸牌：请将一张手牌置为“权”",

  ["$mopai1"] = "",
  ["$mopai2"] = "",
}

local spec = {
  on_cost = function(self, event, target, player, data)
    local room = player.room
    local to = room:askToChoosePlayers(player, {
      targets = room:getOtherPlayers(player, false),
      min_num = 1,
      max_num = 1,
      prompt = "#mopai-choose",
      skill_name = mopai.name,
      cancelable = true,
    })
    if #to > 0 then
      event:setCostData(self, { tos = to })
      return true
    end
  end,
  on_use = function(self, event, target, player, data)
    local room = player.room
    local to = event:getCostData(self).tos[1]
    to:drawCards(1, mopai.name)
    if player:isKongcheng() or player.dead then return end
    local card = room:askToCards(player, {
      skill_name = mopai.name,
      include_equip = false,
      min_num = 1,
      max_num = 1,
      prompt = "#mopai-ask",
      cancelable = false,
    })
    player:addToPile("quan_pile", card, true, mopai.name)
  end,
}

mopai:addEffect(fk.EventPhaseStart, {
  anim_type = "drawcard",
  can_trigger = function(self, event, target, player, data)
    return target == player and player:hasSkill(mopai.name) and
      player.phase == Player.Play and
      player:usedSkillTimes(zhanshen.name, Player.HistoryTurn) == 0 and
      #player.room:getOtherPlayers(player, false) > 0 
  end,
  on_cost = spec.on_cost,
  on_use = spec.on_use,
})

mopai:addEffect(fk.Damaged, {
  anim_type = "masochism",
  can_trigger = function(self, event, target, player, data)
    return target == player and player:hasSkill(mopai.name) and
      player:usedSkillTimes(zhanshen.name, Player.HistoryTurn) == 0 and
      #player.room:getOtherPlayers(player, false) > 0
  end,
  on_cost = spec.on_cost,
  on_use = spec.on_use,
})

return mopai

2. 主动效果

输入：

主动：出牌阶段各限一次，你可以移去一张“权”并选择一项：1.令一名角色摸X张牌；2.对至多X名角色各造成1点雷电伤害（X为“权”数且至少为1）。

输出：

local zhudong = fk.CreateSkill {
  name = "zhudong",
}

Fk:loadTranslationTable{
  ["zhudong"] = "主动",
  [":zhudong"] = "出牌阶段各限一次，你可以移去一张“权”并选择一项：1.令一名角色摸X张牌；2.对至多X名角色各造成1点雷电伤害（X为“权”数且至少为1）。",

  ["#zhudong_draw"] = "主动：移去一张“权”，令一名角色摸%arg张牌",
  ["#zhudong_damage"] = "主动：移去一张“权”，对至多%arg名角色造成1点雷电伤害",

  ["$zhudong1"] = "",
  ["$zhudong2"] = "",
}

zhudong:addEffect("active", {
  anim_type = "offensive",
  expand_pile = "quan_pile",
  prompt = function(self, player, selected_cards, selected_targets)
    local n = math.max(#player:getPile("quan_pile") - 1, 1)
    if self.interaction.data == "draw_card" then
      return "#zhudong_draw:::"..n
    else
      return "#zhudong_damage:::"..n
    end
  end,
  card_num = 1,
  min_target_num = 1,
  max_target_num = function(self, player)
    if self.interaction.data == "draw_card" then
      return 1
    else
      return math.max(#player:getPile("quan_pile") - 1, 1)
    end
  end,
  interaction = function(self, player)
    local choices = {}
    if player:getMark("zhudong_draw-phase") == 0 then
      table.insert(choices, "draw_card")
    end
    if player:getMark("zhudong_damage-phase") == 0 then
      table.insert(choices, "Damage")
    end
    return UI.ComboBox { choices = choices, all_choices = { "draw_card", "Damage" } }
  end,
  can_use = function(self, player)
    return #player:getPile("quan_pile") > 0 and
      (player:getMark("zhudong_draw-phase") == 0 or player:getMark("zhudong_damage-phase") == 0)
  end,
  card_filter = function(self, player, to_select, selected)
    return #selected == 0 and table.contains(player:getPile("quan_pile"), to_select)
  end,
  target_filter = function(self, player, to_select, selected, selected_cards)
    if self.interaction.data == "draw_card" then
      return #selected == 0
    else
      return #selected < math.max(#player:getPile("quan_pile") - 1, 1)
    end
  end,
  on_use = function(self, room, effect)
    local player = effect.from
    room:moveCardTo(effect.cards, Card.DiscardPile, nil, fk.ReasonPutIntoDiscardPile, zhudong.name, nil, true, player)
    if player.dead then return end
    if self.interaction.data == "draw_card" then
      room:setPlayerMark(player, "zhudong_draw-phase", 1)
      local target = effect.tos[1]
      if not target.dead then
        target:drawCards(math.max(#player:getPile("quan_pile"), 1), zhudong.name)
      end
    else
      room:setPlayerMark(player, "zhudong_damage-phase", 1)
      local tos = table.simpleClone(effect.tos)
      room:sortByAction(tos)
      for _, p in ipairs(tos) do
        if not p.dead then
          room:damage{
            from = player,
            to = p,
            damage = 1,
            damageType = fk.ThunderDamage,
            skillName = zhudong.name,
          }
        end
      end
    end
  end,
})

return zhudong

3. 距离效果

输入：

骑马：锁定技，其他角色计算与你的距离+X（X为你的黑色“权”数）。

输出：

local qima = fk.CreateSkill {
  name = "qima",
  tags = { Skill.Compulsory },
}

Fk:loadTranslationTable{
  ["qima"] = "骑马",
  [":qima"] = "锁定技，手牌数大于你的其他角色计算与你的距离+X（X为你的黑色“权”数）。",

  ["$qima1"] = "",
  ["$qima2"] = "",
}

qima:addEffect("distance", {
  correct_func = function(self, from, to)
    if to:hasSkill(qima.name) and from:getHandcardNum() > to:getHandcardNum() then
      return #table.filter(to:getPile("quan_pile"), function (id)
        return Fk:getCardById(id).color == Card.Black
      end)
    end
  end,
})

return qima

4. 禁止效果

输入：

防守：若你的手牌中没有基本牌，你不能成为【杀】的目标。

输出：

local fangshou = fk.CreateSkill {
  name = "fangshou",
}

Fk:loadTranslationTable{
  ["fangshou"] = "防守",
  [":fangshou"] = "若你的手牌中没有基本牌，你不能成为【杀】或【决斗】的目标。",

  ["$fangshou1"] = "",
  ["$fangshou2"] = "",
}

fangshou:addEffect("prohibit", {
  is_prohibited = function(self, from, to, card)
    if to:hasSkill(fangshou.name) and card and
      not table.find(to:getCardIds("h"), function (id)
        return Fk:getCardById(id).type == Card.TypeBasic
      end) then
      return table.contains({ "slash", "duel" }, card.trueName)
    end
  end,
})

5. 攻击范围效果

输入：

剑围：若你的装备区没有武器牌，你的攻击范围+1；体力值大于你的角色视为在你的攻击范围内；你视为不在手牌数大于你的角色的攻击范围内。

输出：

local jianwei = fk.CreateSkill {
  name = "jianwei",
}

Fk:loadTranslationTable{
  ["jianwei"] = "剑围",
  [":jianwei"] = "若你的装备区没有武器牌，你的攻击范围+1；体力值大于你的角色视为在你的攻击范围内；你视为不在体力上限大于你的角色的攻击范围内。",

  ["$jianwei1"] = "",
  ["$jianwei2"] = "",
}

jianwei:addEffect("atkrange", {
  correct_func = function(self, from, to)
    if from:hasSkill(zhenqiao.name) and #from:getEquipments(Card.SubtypeWeapon) == 0 then
      return 1
    end
  end,
  within_func = function (self, from, to)
    return from:hasSkill(jianwei.name) and to.hp > from.hp
  end,
  without_func = function (self, from, to)
    return to:hasSkill(jianwei.name) and from.maxHp > to.maxHp
  end,
})

6. 卡牌增益效果

输入：

天义：出牌阶段限一次，你可以与一名角色拼点：若你赢，本回合你可以多使用一张【杀】、使用【杀】无距离限制。

输出：

Fk:loadTranslationTable{
  ["tianyi"] = "天义",
  [":tianyi"] = "出牌阶段限一次，你可以与一名角色拼点：若你赢，本回合你可以多使用一张【杀】、使用【杀】无距离限制。",

  ["#tianyi"] = "天义：与一名角色拼点，若赢，你本回合使用【杀】获得增益",

  ["$tianyi1"] = "",
  ["$tianyi2"] = "",
}

tianyi:addEffect("active", {
  anim_type = "offensive",
  prompt = "#tianyi",
  max_phase_use_time = 1,
  card_num = 0,
  target_num = 1,
  can_use = function(self, player)
    return not player:isKongcheng() and player:usedSkillTimes(tianyi.name, Player.HistoryPhase) == 0
  end,
  card_filter = Util.FalseFunc,
  target_filter = function(self, player, to_select, selected, selected_cards)
    return #selected == 0 and to_select ~= player and player:canPindian(to_select)
  end,
  on_use = function(self, room, effect)
    local player = effect.from
    local target = effect.tos[1]
    local pindian = player:pindian({ target }, tianyi.name)
    if player.dead then return end
    if pindian.results[target].winner == player then
      room:addPlayerMark(player, "tianyi_win-turn", 1)
    else
      room:addPlayerMark(player, "tianyi_lose-turn", 1)
    end
  end,
})

tianyi:addEffect("targetmod", {
  residue_func = function(self, player, skill, scope, card)
    if player:getMark("tianyi_win-turn") > 0 and card and card.trueName == "slash" and scope == Player.HistoryPhase then
      return 1
    end
  end,
  bypass_distances =  function(self, player, skill, card, to)
    return player:getMark("tianyi_win-turn") > 0 and card and card.trueName == "slash"
  end,
})

7. filter效果

输入：

武神：锁定技，你手牌中的红桃牌视为黑桃【杀】。

输出：

Fk:loadTranslationTable{
  ["wushen"] = "武神",
  [":wushen"] = "锁定技，你手牌中的<font color='red'>♥</font>牌视为♠【杀】",

  ["$wushen1"] = "",
  ["$wushen2"] = "",
}

wushen:addEffect("filter", {
  anim_type = "offensive",
  card_filter = function(self, to_select, player)
    return player:hasSkill(wushen.name) and to_select.suit == Card.Heart and
    table.contains(player:getCardIds("h"), to_select.id)
  end,
  view_as = function(self, player, card)
    return Fk:cloneCard("slash", Card.Spade, card.number)
  end,
})

return fangshou
