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
3. 生成空的技能台词翻译，每个技能给出两句台词翻译

## 推理技能效果

1. 分析用户给出的技能描述，将复杂技能分拆为多个简单效果
2. 分析各效果的类型，将效果分为触发、主动、转化、状态等

## 实现各个效果

1. 通过 addEffect(...) 注册效果。将触发类效果注册在特定事件，将其他效果注册在对应字段

## 示例

输入：

权计：锁定技，出牌阶段开始时，或当你受到1点伤害后，你摸一张牌，然后将一张手牌置于武将牌上，称为“权”。每有一张“权”，你的出牌阶段使用【杀】次数和手牌上限便+1。出牌阶段各限一次，你可以移去一张“权”并选择一项：1.令一名角色摸X张牌；2.对至多X名角色各造成1点雷电伤害，有角色因此进入濒死状态时，你可以选择另一名体力值最少的角色，弃置其场上一张牌或令其回复2点体力（X为“权”数且至少为1）。

输出：

local quanji = fk.CreateSkill {
  name = "quanji",
  tags = { Skill.Compulsory },
  derived_piles = "quan_pile",
}

Fk:loadTranslationTable{
  ["quanji"] = "权计",
  [":quanji"] = "锁定技，出牌阶段开始时，或当你受到1点伤害后，你摸一张牌，然后将一张手牌置于武将牌上，称为“权”。每有一张“权”，你的出牌阶段使用【杀】次数和手牌上限便+1。出牌阶段各限一次，你可以移去一张“权”并选择一项：1.令一名角色摸X张牌；2.对至多X名角色各造成1点雷电伤害，有角色因此进入濒死状态时，你可以选择另一名体力值最少的角色，弃置其场上一张牌或令其回复2点体力（X为“权”数且至少为1）。",

  ["quan_pile"] = "权",
  ["#quanji-ask"] = "权计：请将一张手牌置为“权”",
  ["#quanji_draw"] = "权计：移去一张“权”，令一名角色摸%arg张牌",
  ["#quanji_damage"] = "权计：移去一张“权”，对至多%arg名角色造成雷电伤害",
  ["#quanji-recover"] = "权计：你可以选择除 %dest 以外一名角色，弃置其场上一张牌或令其回复2点体力",
  ["quanji_discard"] = "弃置%dest场上一张牌",
  ["quanji_recover"] = "令%dest回复2点体力",

  ["$quanji1"] = "",
  ["$quanji2"] = "",
}

quanji:addEffect("active", {
  anim_type = "offensive",
  expand_pile = "quan_pile",
  prompt = function(self, player, selected_cards, selected_targets)
    local n = math.max(#player:getPile("quan_pile") - 1, 1)
    if self.interaction.data == "draw_card" then
      return "#quanji_draw:::"..n
    else
      return "#quanji_damage:::"..n
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
    if player:getMark("quanji_draw-phase") == 0 then
      table.insert(choices, "draw_card")
    end
    if player:getMark("quanji_damage-phase") == 0 then
      table.insert(choices, "Damage")
    end
    return UI.ComboBox { choices = choices, all_choices = { "draw_card", "Damage" } }
  end,
  can_use = function(self, player)
    return #player:getPile("quan_pile") > 0 and
      (player:getMark("quanji_draw-phase") == 0 or player:getMark("quanji_damage-phase") == 0)
  end,
  card_filter = function(self, player, to_select, selected)
    return #selected == 0 and table.contains(player:getPile("quan_pile"), to_select)
  end,
  target_filter = function(self, player, to_select, selected)
    if self.interaction.data == "draw_card" then
      return #selected == 0
    else
      return #selected < math.max(#player:getPile("quan_pile") - 1, 1)
    end
  end,
  on_use = function(self, room, effect)
    local player = effect.from
    room:moveCardTo(effect.cards, Card.DiscardPile, nil, fk.ReasonPutIntoDiscardPile, quanji.name, nil, true, player)
    if player.dead then return end
    if self.interaction.data == "draw_card" then
      room:setPlayerMark(player, "quanji_draw-phase", 1)
      local target = effect.tos[1]
      if not target.dead then
        target:drawCards(math.max(#player:getPile("quan_pile"), 1), quanji.name)
      end
    else
      room:setPlayerMark(player, "quanji_damage-phase", 1)
      local tos = table.simpleClone(effect.tos)
      room:sortByAction(tos)
      for _, p in ipairs(tos) do
        if not p.dead then
          room:damage{
            from = player,
            to = p,
            damage = 1,
            damageType = fk.ThunderDamage,
            skillName = quanji.name,
          }
        end
      end
    end
  end,
})

local spec = {
  on_use = function(self, event, target, player, data)
    local room = player.room
    player:drawCards(1, quanji.name)
    if player:isKongcheng() or player.dead then return end
    local card = room:askToCards(player, {
      skill_name = quanji.name,
      include_equip = false,
      min_num = 1,
      max_num = 1,
      prompt = "#quanji-ask",
      cancelable = false,
    })
    player:addToPile("quan_pile", card, true, quanji.name)
  end,
}

quanji:addEffect(fk.EventPhaseStart, {
  anim_type = "drawcard",
  can_trigger = function(self, event, target, player, data)
    return target == player and player:hasSkill(quanji.name) and
      player.phase == Player.Play
  end,
  on_use = spec.on_use,
})

quanji:addEffect(fk.Damaged, {
  anim_type = "masochism",
  trigger_times = function(self, event, target, player, data)
    return data.damage
  end,
  can_trigger = function(self, event, target, player, data)
    return target == player and player:hasSkill(quanji.name)
  end,
  on_use = spec.on_use,
})

quanji:addEffect("targetmod", {
  residue_func = function (self, player, skill, scope, card, to)
    if player:hasSkill(quanji.name) and card and card.trueName == "slash" then
      return #player:getPile("quan_pile")
    end
  end,
})

quanji:addEffect("maxcards", {
  correct_func = function(self, player)
    if player:hasSkill(quanji.name) then
      return #player:getPile("quan_pile")
    end
  end,
})

quanji:addEffect(fk.EnterDying, {
  anim_type = "support",
  is_delay_effect = true,
  can_trigger = function(self, event, target, player, data)
    if data.damage and data.damage.skillName == quanji.name and not player.dead then
      local skill_event = player.room.logic:getCurrentEvent():findParent(GameEvent.SkillEffect)
      if skill_event and skill_event.data.skill.name == quanji.name and skill_event.data.who == player then
        local targets = table.filter(player.room:getOtherPlayers(target, false), function(p)
          return (#p:getCardIds("ej") > 0 or p:isWounded()) and
            table.every(player.room:getOtherPlayers(target, false), function(q)
              return q.hp >= p.hp
            end)
        end)
        if #targets > 0 then
          return true
        end
      end
    end
  end,
  on_cost = function(self, event, target, player, data)
    local room = player.room
    local targets = table.filter(player.room:getOtherPlayers(target, false), function(p)
      return (#p:getCardIds("ej") > 0 or p:isWounded()) and
        table.every(room:getOtherPlayers(target, false), function(q)
          return q.hp >= p.hp
        end)
    end)
    local to = room:askToChoosePlayers(player, {
      targets = targets,
      min_num = 1,
      max_num = 1,
      prompt = "#quanji-recover::" .. target.id,
      skill_name = quanji.name,
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
    local choices = {}
    if #to:getCardIds("ej") > 0 then
      table.insert(choices, "quanji_discard::"..to.id)
    end
    if to:isWounded() then
      table.insert(choices, "quanji_recover::"..to.id)
    end
    local choice = room:askToChoice(player, {
      choices = choices,
      skill_name = quanji.name,
    })
    if choice:startsWith("quanji_discard") then
      local id = room:askToChooseCard(player, {
        target = to,
        flag = "ej",
        skill_name = quanji.name,
      })
      room:throwCard(id, quanji.name, to, player)
    else
      room:recover{
        who = to,
        num = 2,
        recoverBy = player,
        skillName = quanji.name,
      }
    end
  end,
})

return quanji
