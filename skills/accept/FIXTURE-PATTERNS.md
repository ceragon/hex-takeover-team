# Fixture Patterns — evaluate_script 食谱库

所有食谱基于 `midBattleV1` 夹具（tick 920，倒计时 01:28，蓝方 71 格 / 红方 69 格 / 水域 4 格）。

每个食谱是一段可直接传给 `evaluate_script` 的 JavaScript 函数。假设 `window.__HT_VERIFY__` 和 `window.__APP_STATE__` 已挂载。

---

## 1. load-fixture — 加载夹具 + 关闭标题屏

初始化整个验收环境。每次 `/accept` 开头必跑。

```js
() => {
  const state = window.__HT_VERIFY__.loadFixture('mid-battle-v1')
  const app = window.__APP_STATE__
  app.clearOverlays()
  app.phase = 'playing'
  return { ok: true, tick: state.summary.tick, gold: state.summary.gold }
}
```

---

## 2. move-unit — 移动单位到指定格

把某个单位瞬移到目标 hexId。不触发渲染重绘（适用于已存在的单位）。

```js
(unitId, targetHexId) => {
  const world = window.__APP_STATE__.world
  const unit = world.systems.barracks.units.find(u => u.id === unitId)
  if (!unit) return { ok: false, reason: `unit ${unitId} not found` }
  unit.hexId = targetHexId
  unit.moveProgress = 0
  return { ok: true, unit: unitId, nowAt: targetHexId }
}
```

用法：`evaluate_script({ function: "...", args: [3, "5,4"] })`

---

## 3. modify-hp — 修改单位 HP

设置指定单位的当前 HP（可用于模拟残血、濒死状态）。

```js
(unitId, newHp) => {
  const world = window.__APP_STATE__.world
  const unit = world.systems.barracks.units.find(u => u.id === unitId)
  if (!unit) return { ok: false, reason: `unit ${unitId} not found` }
  unit.hp = newHp
  return { ok: true, unit: unitId, hp: newHp, maxHp: unit.maxHp }
}
```

---

## 4. spawn-unit — 注入新单位（需 reload）

向兵营单位列表注入新单位。注意：运行时 push 不触发渲染。
必须 reload 页面后重新 loadFixture 再注入，或者在 loadFixture 之后立即注入（冻结状态下渲染器下一帧会拾取）。

```js
(id, key, tier, owner, hexId) => {
  const world = window.__APP_STATE__.world
  const stats = {
    skeleton: { hp: 120, atk: 15, speed: 0.3, range: 1.0, flying: false, trait: 'melee_tanky' },
    wolf: { hp: 80, atk: 20, speed: 0.6, range: 1.0, flying: false, trait: 'melee_fast' },
    warrior: { hp: 240, atk: 20, speed: 0.4, range: 1.0, flying: false, trait: 'melee_heavy' },
    archer: { hp: 120, atk: 35, speed: 0.5, range: 3.0, retreatThreshold: 1.5, flying: false, trait: 'keep_distance' },
    cavalry: { hp: 200, atk: 25, speed: 0.6, range: 2.0, flying: false, trait: 'ground_siege' },
    dragon: { hp: 200, atk: 40, speed: 0.3, range: 3.0, flying: true, trait: 'dragon_aoe_skill' },
  }
  const s = stats[key]
  if (!s) return { ok: false, reason: `unknown key: ${key}` }
  const unit = {
    id, key, tier, owner, hexId,
    hp: s.hp, maxHp: s.hp, atk: s.atk, speed: s.speed,
    range: s.range, retreatThreshold: s.retreatThreshold,
    flying: s.flying, trait: s.trait,
    moveProgress: 0, attackCd: 0, attackIntervalTicks: 15,
  }
  world.systems.barracks.units.push(unit)
  return { ok: true, spawned: id, at: hexId }
}
```

用法：`evaluate_script({ function: "...", args: [99, "dragon", "gold", "player", "4,4"] })`

如果渲染器没拾取，执行 reload 序列：
1. `navigate_page({ type: 'reload' })`
2. 等待 `__HT_VERIFY__` 可用
3. 重新 `load-fixture`
4. 重新执行本食谱

---

## 5. fast-forward — 快进 N 个 tick

推进游戏逻辑 N 个 tick（每 tick = 100ms 游戏时间）。返回推进后的状态快照。

```js
(n) => {
  return window.__HT_VERIFY__.tick(n)
}
```

常用场景：
- 快进 4 tick 观察死亡动画过渡（dying = 4）
- 快进 15 tick 观察一个完整攻击 CD
- 快进 60 tick（6 秒）观察产金/产兵一轮

---

## 6. set-gold — 设置金币

直接修改玩家或敌方金币数。

```js
(amount, who) => {
  const world = window.__APP_STATE__.world
  const player = who || 'player'
  world.systems.resource.gold[player] = amount
  return { ok: true, player, gold: amount }
}
```

用法：`evaluate_script({ function: "...", args: [500, "player"] })`

---

## 7. open-tile — 开启地块

模拟玩家点击开启一个 openable 格子。使用 VerifyAPI 的 clickAndNext 保证原子性。

```js
(q, r) => {
  return window.__HT_VERIFY__.clickAndNext(q, r)
}
```

注意：需要该格子处于 openable 状态且玩家金币足够。先用 `getState()` 查看 `clickable` 列表找可用格子。

---

## 8. get-state — 获取当前状态快照

不推进 tick，只读取当前格式化状态。

```js
() => {
  return window.__HT_VERIFY__.getState()
}
```

返回结构：`{ summary: { tick, timer, phase, gold, playerTiles, enemyTiles, ... }, board, clickable, unclickable }`

---

## 9. set-building-hp — 修改建筑 HP

设置指定格子上建筑的当前 HP（可用于模拟箭塔/兵营残血）。

```js
(hexId, newHp) => {
  const world = window.__APP_STATE__.world
  const tile = world.tiles[hexId]
  if (!tile) return { ok: false, reason: `tile ${hexId} not found` }
  if (!tile.building) return { ok: false, reason: `no building at ${hexId}` }
  tile.building.hp = newHp
  return { ok: true, hexId, buildingType: tile.building.type, hp: newHp }
}
```

---

## 组合模式

测试一个 AC 通常需要组合多个食谱：

- **观察死亡动画**：`modify-hp(unitId, 1)` → `fast-forward(1)` 触发击杀 → `fast-forward(4)` 观察 dying 过渡 → `take_screenshot`
- **验证开地块花费**：`set-gold(25)` → `open-tile(q, r)` → `get-state()` 确认金币扣减
- **验证编队渲染**：`spawn-unit` 多个同格 → 如果渲染没更新则 reload 序列 → `take_screenshot`
- **验证倒计时**：`get-state()` 记录 timer → `fast-forward(60)` → `get-state()` 确认 timer 减少 6
