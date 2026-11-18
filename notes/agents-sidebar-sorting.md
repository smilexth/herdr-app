# Agents sidebar: what orders the list, and what can change it

**Question:** What controls the order/sorting of agents in the agents sidebar, and what tools exist to change or customize it?

All paths below are relative to the repo root. Line numbers are from this checkout (branch `fix/spaces-stale-reconnect`, 2026-11-18).

## How the shipped agents sidebar orders agents today

The shipped mod is `defaults/mods/agents/main.js` (469 lines; header comment, line 1:
"Agents in grouped or priority views; parents take their most urgent sub-agent's place").
Every render rebuilds the whole order in `render()` (`defaults/mods/agents/main.js:219-244`).
The rule chain, in the order the code applies it:

1. **Base list: model order.** `agents()` returns live status-reporting terminals
   "in space, tab, pane order" (`defaults/herdr.d.ts:195`). Space order itself is
   "local machine first, then machines by name" (`defaults/AGENTS.md:101-102`), so the
   raw list is implicitly machine-grouped. `main.js:224`.
2. **Sub-agents nest under their parent.** Live sub-agents (children by `terminals()`
   `parent`, `state === "live"`) are collected depth-first into trees
   (`main.js:171-180`) and removed from the top-level list — *unless pinned*, or the
   sidebar is collapsed to the rail (no trees there) (`main.js:233-234`). A kept
   sub-agent (one with a pane) is in `agents()` too; where a tree shows it, it gets no
   row of its own (`main.js:231-234`).
3. **Sort mode: `shared("agents.sort")`**, either `"priority"` (default) or
   `"grouped"` — the `SORTS` table and `sortOf()` (`main.js:59-63`), switched from the
   header's sort button → overlay menu that calls `setShared("agents.sort", s.id)`
   (`main.js:140-164`, button at `main.js:437-443`). The key is the mod's own shared
   state (persisted in `shared.json`); it is not a settings.json key and is not
   documented outside the mod's code.
   - **Priority** (`main.js:240`): sort by urgency rank **descending**, then by
     recency of last status change **descending**, then by `seq` **descending**:
     `list.sort((x, y) => rankOf(y) - rankOf(x) || this.since(y) - this.since(x) || y.seq - x.seq)`.
     - Ranks: `RANK = { blocked: 4, done: 3, working: 2, idle: 1 }` (`main.js:52`) —
       the same urgency as the model's status rollup (`defaults/AGENTS.md:154-156`).
     - **A parent's urgency is the max of its own status and its live sub-agents'**
       (`main.js:238`, header comment line 1). Sub-agent status words map
       `error → blocked`, `done → idle` ("it reports to its parent, and the parent's
       done is the news") (`main.js:55-58`).
     - **Recency**: `track()` stamps each agent on every status *transition*
       (done and idle count as one state) into `localStorage["changed"]`, kept across
       restarts and pruned after 30 days (`main.js:97-110`, `since()` at `132-134`).
       `seq` is the model's change counter ("Higher = changed more recently",
       `defaults/herdr.d.ts:188-189`).
   - **Grouped**: no comparator at all — the model's space/tab/pane order stands
     (`main.js:237,240`; the `if (byPriority)` guard).
4. **Pinned first, always.** After sorting, pinned agents are lifted to the very
   front in both modes (`main.js:241-243`): `list = [...list.filter(a => a.pinned), ...rest]`.
   `pinned` comes from `agents()` (`defaults/herdr.d.ts:190-191`), set by
   `terminal.pin` and kept as the core label `herdr.pinned`, so every Herdr client
   agrees and the pin dies with the terminal (`defaults/AGENTS.md:661-663`,
   `docs/commands.md:161-163`). The mod's hover pin icon toggles it
   (`main.js:322-338`); legacy localStorage pins migrate to the label
   (`main.js:114-130`).
5. **Group headers** (display grouping over the sorted list, `main.js:376-384`):
   a "Pinned" section first if any (`main.js:378`), then — priority mode —
   consecutive same-rank runs become sections *Needs you / Done / Working / Idle*
   (`SECTION`, `main.js:64-65,380-383`); grouped mode — consecutive same-space runs
   become sections labeled with the space name (`sp-${a.space}`, `main.js:381-383`;
   "Agents of a space are next to each other in agents(), so neighbours make a
   group", `main.js:376-377`). Unknown rank falls into Idle (`rankOf(a) || RANK.idle`,
   `main.js:380`).
6. **No machine axis of its own**: a remote agent just shows a machine-name chip from
   its address prefix (`vps/t7` → "vps") (`main.js:302,369`). Machine grouping exists
   only implicitly, via space order, in "grouped" mode.
7. **The rail** (sidebar narrower than 160 px, `main.js:17`): groups aren't drawn;
   the same sorted, pinned-first list is a flat stack of numbered rows
   (`list.flatMap(withKids)`, `main.js:413-415`; the number is the row's 1-based
   position, `main.js:316`).
8. **On status change**: live data re-renders the mod immediately; `track()` stamps
   the transition so the agent jumps to its new rank section and wins the recency
   tiebreak; a 30 s timer refreshes the "how long ago" durations
   (`main.js:92`, shown at `main.js:319,372`). Fold state (groups and parent trees)
   is per-address display state only (`this.folded`, `main.js:84,249,272,388,403`) —
   it never affects the order.
9. **Finally, the mod publishes the order**: `setAgentOrder(list.map(a => a.address))`
   with the final top-level list (pinned first; nested sub-agents *excluded*)
   (`main.js:244`).

## The sorting-related knobs that already exist

### `setAgentOrder` — the real contract

`defaults/herdr.d.ts:240-243`:

```ts
/** Sets agent.next / agent.prev order to match the list a mod displays.
 *  Call when rendering the list. In memory only; missing agents are ignored, new agents
 *  are appended in model order. Without a supplied order, navigation uses model order. */
export function setAgentOrder(addresses: string[]): null;
```

The docs' phrasing is accurate. Rust semantics:

- The binding stores the array wholesale into a single app-global thread-local:
  `AGENT_ORDER: RefCell<Vec<String>>` (`src/mods.rs:599`), replaced on every call —
  `AGENT_ORDER.with(|s| *s.borrow_mut() = strings(a.get(0)))` (`src/mods.rs:1163-1166`).
  **There is one order for the whole app: the last mod to render wins** (same
  last-call-wins shape as `setWindowTitle`/`setBadge`). Two mods that both list agents
  would keep overwriting each other on every re-render.
- Navigation merges at read time (`src/mods.rs:471-480`, `pub fn agent_order`):
  stored order first, filtered to the currently available agents, then any available
  agents not mentioned are **appended in model order**. So nested sub-agents the
  sidebar never lists are still reachable by `agent.next` — after every listed agent.
- "Available" means every live address with a status (`src/window.rs:1431-1435`).
- In memory only: empty on restart; no persistence.
- Consumers:
  - `agent.next` / `agent.prev` (`src/window.rs:3541-3556`): walk the merged order
    with wraparound; the "current" agent is `shown_agent()` mapped through
    `root_agent()` (the top of the sub-agent's parent chain, `src/window.rs:1474-1485`),
    so navigating away from a sub-agent resumes at its parent's slot. Unknown current:
    `next` goes to the first, `prev` to the last.
  - `shown_agent()` (`src/window.rs:1437-1447`): the chosen agent while live, else
    the focused pane's agent, **else the first of the agent order** — so the sidebar's
    first row is also the default agent a `{ "agent": "focused" }` layout block shows.
- Pinned down by the unit test `navigation_follows_mod_order` (`src/mods.rs:1706-1717`)
  and by the bun tests below.

### `terminal.pin {terminal, pinned}`

`docs/commands.md:161-163`: "Pin an agent to the top of the agent lists (pinned true)
or unpin it (false). Pins are kept with the terminal, so every Herdr client shows
them, and end with it." Stored as the core label `herdr.pinned` (`defaults/AGENTS.md:661-663`).
This is the only manual positioning tool: binary, and always to the very top
(above both sort modes' sections).

### Status semantics that drive grouping

OSC 7501 statuses `working | blocked | done | idle`; "done" = finished unseen,
becomes `idle` once looked at; a space/tab's status is the most urgent of its agents,
blocked > done > working > idle (`defaults/AGENTS.md:154-156`, `defaults/herdr.d.ts:94,170-193`).
These statuses are the entire sort key of priority mode; an agent can't be given a
custom rank — the words are fixed vocabulary.

### `agents.sort` (shared key)

The mod's own sort setting, `"priority" | "grouped"`, persisted through
`setShared` into `shared.json` and switchable from the sort menu overlay
(`defaults/mods/agents/main.js:59-63,157-159`). Any mod can read or write it
(`shared("agents.sort")`) — it's shared state, not owned — but nothing outside the
shipped mod defines further sort ids.

## How a person (or their agent) customizes the order today

The sanctioned path is cloning the mod — shipped mods are read-only and replaced on
start ("Shipped mods are Herdr's; duplicate with `mod.clone` before editing",
`defaults/AGENTS.md:934`). The clone-command pattern, exactly as documented
(`defaults/AGENTS.md:196-197`):

```sh
herdr-app do layout.clone '{"name":"main","as":"my-layout"}'
herdr-app do mod.clone '{"name":"files","as":"my-files"}'      # pattern; for agents:
herdr-app do mod.clone '{"name":"agents","as":"my-agents"}'
```

Then:

1. Edit the sort in `~/.config/herdr-app/mods/my-agents/main.js` — the comparator at
   line 240 (priority mode), the `SORTS` table at lines 59-62 (add a mode; the sort
   menu lists whatever is there), or the pinned lift at lines 241-243.
2. Because the agents mod's `mod.json` says `"place": "none"`
   (`defaults/mods/agents/mod.json` — it lives only where a layout puts it), the clone
   must be referenced by a layout: clone the layout too and point the sidebar block's
   `content.mod` at `"my-agents"` ("A cloned layout keeps its mod references: clone a
   mod too and point your layout's `content.mod` to it to change code",
   `defaults/AGENTS.md:201-202`; the shipped sidebar block is
   `{ "id": "agents", "type": "block", "content": { "mod": "agents" } }`,
   `defaults/AGENTS.md` default-window JSON). Clones rewrite their own asset paths and
   are never updated by Herdr (`defaults/AGENTS.md:196-205`).
3. `herdr-app check ~/.config/herdr-app/mods/my-agents` after every change
   (`defaults/AGENTS.md`, "Check before you finish").
4. A clone that keeps calling `setAgentOrder` keeps `agent.next`/`agent.prev` and any
   rail numbering following *its* order (`defaults/AGENTS.md:166-168`):
   "**Agent navigation order** belongs to the mods. A mod displaying agents calls
   `setAgentOrder(list.map(a => a.address))` on render so `agent.next` / `agent.prev`
   follow its list."

Alternatively, write a fresh sidebar mod from `agents()` / `terminals()` /
`setAgentOrder` — nothing in the default interface is special (`defaults/AGENTS.md:12-15`).
A custom mod can persist its own sort choice via `shared()` (like `agents.sort`) or
its `localStorage`.

Ordering behaviors pinned down by tests (a behavior change to the shipped mod needs
these updated; the tests read `defaults/mods/agents/main.js` directly, so clones are
not covered) — `scripts/agents.test.ts`:

- Legacy pin migration and offline-pin retention (`scripts/agents.test.ts:20-50`).
- Both sorts call `setAgentOrder` with the full top-level list, and the order is
  stable across renders when nothing changed (`:150-176`, esp. `:169,175`).
- Per-row label rules per sort (`:178-210`).
- A kept sub-agent shows once, nested, and **the published order excludes it**
  (`expect(order).toEqual(["t1"])`, `:239-251`); it keeps its own row in the rail,
  when pinned, or when its parent ended (`:260-267`).
- The focused agent's row is marked selected in both widths (`:269-290`).

## What cannot be changed from a mod today (gaps), and upstream asks

Gaps in the current API:

- **`setAgentOrder` is write-only.** There is no read-back of the effective (merged)
  order, so a second mod can't align with the order the person sees
  (`defaults/herdr.d.ts:240-243` offers only the setter). Asked for in #167 and #891.
- **One global order, last render wins** (`src/mods.rs:599,1163-1166`): no per-mod
  orders; two visible agent lists would fight over `agent.next`/`agent.prev`.
- **No index-based focus command**: only `agent.next`, `agent.prev`, and
  `agent.focus {terminal}` (by address) (`docs/commands.md:377-393`). The rail
  *displays* row numbers (`main.js:316`) but nothing can target them.
- **No manual ordering**: pin is the only positioning tool and it is binary
  (top-of-list or sorted away); no drag-and-drop, no arbitrary rank, no persisted
  personal order for agents. (Spaces have `space.move`; agents have no equivalent.)
- **The sort menu has exactly two modes**; grouping by machine or by task-name prefix,
  or per-group priority, doesn't exist. `agents()`'s model order is fixed at
  space/tab/pane — every other ordering is mod-side code.

Upstream issues (repo `herdrdev/herdr-app`, all OPEN as of 2026-11-18):

- **#964** — Agents sidebar: add grouping by machine or by name prefix, with priority
  order inside each group (two more views or one grouping setting).
- **#715** — Sidebar: allow manual reordering (drag-and-drop or move up/down) of
  machines and of agent sessions in the Agents panel, persisted across restarts.
- **#891** — `agent.select {index}`: focus the Nth agent in the Agents list's order;
  notes `setAgentOrder` is write-only and proposes cloning `agents` as today's workaround.
- **#167** — No command to go to an agent by number (⌘⌥1–9); asks for
  `agent.select {index}` or a read for the current agent order.
- **#617** — Priority-mode rows lead with the space name; asks for a tab-name-only
  display option (labeling, not order, but same surface: `main.js:341-342`).
- Adjacent: **#573** — spaces sidebar's machine groups can't be reordered (spaces, not
  agents).

## Sources

- `defaults/mods/agents/main.js` — the shipped agents sidebar mod (read in full)
- `defaults/herdr.d.ts` — generated mod API: `agents()`/`AgentInfo` (170-196),
  `setAgentOrder` (240-243)
- `defaults/AGENTS.md` — model and rules: space order (101-102), Agents/status (154-156),
  Sub-agents (158-165), agent navigation order (166-168), clone commands (196-205),
  command list incl. `terminal.pin` (661-663, 678), shipped mods read-only (934)
- `docs/commands.md` — `terminal.pin` (161-163), `agent.focus`/`next`/`prev` (377-393)
- `scripts/agents.test.ts` — bun tests for the shipped mod (read in full)
- `src/mods.rs` — `agent_order()` merge (471-480), `AGENT_ORDER` thread-local (599),
  `setAgentOrder` binding (1163-1166), test `navigation_follows_mod_order` (1706-1717)
- `src/window.rs` — `agent_order()` / `shown_agent()` (1431-1447), `root_agent()`
  (1474-1485), `agent.next`/`prev` handling (3541-3556)
- `gh issue list/view -R herdrdev/herdr-app` — issues #964, #715, #891, #167, #617, #573
