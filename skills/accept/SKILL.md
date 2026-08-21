---
name: accept
description: >
  Verify a GitHub issue's Acceptance Criteria by driving the game through
  Chrome DevTools MCP and producing player-narrative feedback. Use when the
  user says "/accept", "accept this issue", "verify acceptance criteria",
  or names an issue number for acceptance testing.
argument-hint: "<issue-number>"
---

# /accept — 挑剔玩家验收

## Goal

Given an issue number, verify each Acceptance Criteria item by driving the running game via Chrome DevTools MCP, then post a GitHub comment with AC verdicts + player-voice observations.

## Constraints

- Use Chrome DevTools MCP tools (`evaluate_script`, `take_screenshot`, `navigate_page`, `wait_for`, `list_console_messages`) for game interaction.
- Use `gh` CLI for all GitHub operations (read issue, post comment).
- Use `midBattleV1` fixture as the base scenario (via `window.__HT_VERIFY__.loadFixture('mid-battle-v1')`).
- Do NOT modify source code, fix bugs, or run test suites.
- Do NOT start the dev server — detect it or report BLOCKED.
- Leave the dev server running after finishing.
- Screenshots go to `/tmp/accept-<issue>-*.png` (temporary, not in repo).

## Execution Model

This skill is LLM-autonomous orchestration. You are given a goal and constraints, not a pipeline. You decide:
- Which AC items to test first
- What fixture patches are needed per AC
- How many screenshots to take
- When to fast-forward ticks vs inspect state directly

Read the AC list, plan your approach, construct scenarios, drive the game, observe, and write the comment. Refer to `FIXTURE-PATTERNS.md` for reusable `evaluate_script` recipes. Adopt the persona from `PERSONA.md` for all output text.

## Dev Server Port Detection

1. Run `lsof -ti:5173` — if a PID is returned, use `http://localhost:5173`
2. Else run `lsof -ti:3000` — if a PID is returned, use `http://localhost:3000`
3. Else report BLOCKED: "Dev server not reachable on 5173 or 3000."

## Game Initialization Sequence

1. `navigate_page` to the detected dev server URL
2. Poll `evaluate_script` until `window.__HT_VERIFY__` is defined (max 10s, 1s intervals)
3. `evaluate_script` → `window.__HT_VERIFY__.loadFixture('mid-battle-v1')`
4. `evaluate_script` → dismiss title screen:
   ```js
   () => {
     const app = window.__APP_STATE__
     app.clearOverlays()
     app.phase = 'playing'
     return 'ok'
   }
   ```
5. Now the game is in a frozen mid-battle state, ready for patching and observation.

## Key Technical Facts

- Runtime `units.push()` does NOT trigger rendering. To add new units: reload page → `loadFixture` → patch → the render loop picks up state on next frame.
- `loadFixture` freezes the game (verifyPaused=true, fixtureFrozen=true). Call `__HT_VERIFY__.tick(n)` to advance.
- `__HT_VERIFY__.getState()` returns a formatted snapshot without advancing ticks.
- The fixture is at tick 920 (countdown 01:28). Player has 71 tiles, enemy 69, 4 water.
- `take_screenshot` captures the current canvas state — ensure overlays are dismissed first.

## Output Format

Post a single GitHub comment to the issue using `gh issue comment <N> --body "..."`. The comment MUST follow this structure:

```markdown
## AC 逐条判定

1. ✅ <AC text>
   <one-sentence player-narrative observation>

2. ❌ <AC text>
   <what went wrong, in player voice>

3. ⚠️ <AC text>
   <couldn't test / not applicable — reason>

## QA 自主观察

<2-5 paragraphs of free-form observations in player voice: things noticed
beyond AC, design friction, highlights,手感 issues.>

## 总结

<one paragraph: overall verdict — ACCEPTED / REJECTED / BLOCKED.
If REJECTED: which AC failed and why.
If BLOCKED: what prevented testing.>

<details>
<summary>测试过程</summary>

- Loaded fixture: mid-battle-v1
- Patched: [list of state modifications]
- Screenshots taken: [list of paths]
- Ticks fast-forwarded: [count]
- Issues encountered: [tool failures, workarounds]

</details>
```

## Error Handling

| Situation | Action |
|-----------|--------|
| No `## Acceptance criteria` section in issue | FAIL immediately. Comment: "No acceptance criteria found. Add a `## Acceptance criteria` section with checkbox items." |
| Dev server not reachable (5173 and 3000 both dead) | Report BLOCKED. Don't start the server. |
| `__HT_VERIFY__` not available after 10s polling | Report BLOCKED: "Game not in DEV mode or VerifyAPI not mounted." |
| MCP tool failure (disconnect, screenshot fails) | Stop. Report what was tested so far. Don't retry blindly. |
| AC item not testable via browser (e.g., "unit tests pass") | Mark ⚠️ with reason. Don't run Vitest through DevTools. |
| Scenario can't be constructed from fixture | Mark that AC ⚠️ with explanation. Continue testing other items. |

## What This Skill Is NOT

- Not a bug fixer — it reports, doesn't repair
- Not a test runner — it doesn't replace Vitest or Playwright
- Not a pipeline — the LLM decides the testing strategy
- Not a checklist robot — the persona makes it read like a real player
