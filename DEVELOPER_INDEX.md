# Life Quest — Developer Index
> Single-file app (`index.html`). All logic is inline JS. State is a global `S` object persisted to `localStorage` + Firebase RTDB.

---

## Module Map

### 1. Firebase / Cloud Sync  `L9–104`
Loaded asynchronously 500ms after `window.load` to avoid blocking render.

| Symbol | Line | Role |
|--------|------|------|
| `window.fbSave` stub | 11 | No-op until Firebase loads |
| `showSyncStatus(msg)` | 14 | Transient status overlay (auto-fades 3s) |
| `deserializeForImport(data)` | 19 | Used for pre-Firebase import path |
| Firebase module (dynamic `<script type="module">`) | 28 | Full init, real `fbSave`, cloud load |
| `serialize()` (inner) | 48 | Converts Sets → arrays for JSON storage |
| `deserialize(data)` (inner) | 51 | Restores Sets, fills missing defaults |
| `window.fbSave` (real) | 65 | Debounced 2s write to `lifequest/save` |
| Cloud ↔ local conflict resolution | 76 | Compares `lastSaved` timestamps; newer wins |

---

### 2. Data Constants / Schemas  `L484–608`

| Symbol | Line | Role |
|--------|------|------|
| `TYPES` | 485 | Task type configs: `daily` / `main` / `side` (colors, labels, emoji) |
| `TASK_ICONS` | 503 | 28-icon picker array |
| `TASK_LOCS` | 505 | 12 canonical location IDs used in task `loc` field |
| `DEFAULT_STATE` | 514 | Initial `S` shape — only used on first run or reset |
| `DEFAULT_USER_TASKS` | 534 | 10 seed tasks (IDs 1001–1010, sub-IDs 2001–2011) |
| `BASE_LOCS` | 554 | 3 built-in locations with room hierarchies (home/work/mall) |
| `STATS` | 577 | 6 stat definitions: `str exe foc hlt moo int` (daily vs long-term) |
| `ACHS` | 586 | 9 achievement definitions with `prog()` functions |
| `MILESTONES` | 598 | 6 streak milestone tiers (3/7/14/30/60/100 days) |
| `CUSTOM_ICONS` | 607 | 16-icon array for custom location picker |
| `LV_NAMES` | 608 | 7 level name strings (Lv1–7) |

---

### 3. Time System  `L611–722`

| Symbol | Line | Role |
|--------|------|------|
| `todayStr()` | 612 | Returns `"YYYY-M-D"` — used as day identity key |
| `timeUntilMidnight()` | 616 | Returns `"HH:MM:SS"` countdown string |
| `formatDate()` | 625 | Chinese date string for display badge |
| `checkNewDay()` | 630 | **Core day-rollover logic**: streak calc, selective task reset by repeat cycle, daily stat reset + long-stat decay (-1), sets `S._isNewDay` |
| `showNewDayPopup()` | 697 | Shows greeting popup if `S._isNewDay` flag is set |
| `updateDateBadge()` | 708 | Updates `#date-badge` element |
| `startClock()` | 712 | 1s interval: refreshes clock, triggers `checkNewDay` if date rolled mid-session |

**Repeat cycle reset logic** (inside `checkNewDay`, L666–681):
- `daily` → always reset
- `weekly` → reset if `dow === repeatDay`
- `biweekly` → weekly check + even week-number parity
- `monthly` → reset if `dom === repeatDate`
- `none` → never auto-reset

---

### 4. State / Persistence  `L724–759`

| Symbol | Line | Role |
|--------|------|------|
| `let S` | 725 | Global mutable state object |
| `loadState()` | 726 | Reads `lq_state_v2` from localStorage; deserializes Sets; falls back to `DEFAULT_STATE` + seed tasks |
| `saveState()` | 751 | Stamps `S.lastSaved`, serializes Sets → arrays, writes localStorage, calls `fbSave()` |

**Key `S` fields:**
```
xp, lv, streak, bestStreak
weekDone[7]         — weekly check-in booleans
taskDone: Set<id>   — completed task IDs
subDone: Set<id>    — completed subtask IDs (stored as strings)
unlocked: Set<id>   — unlocked achievement IDs
sv: {str,exe,foc,hlt,moo,int}  — current stat values
svBase: {str,exe,foc,moo}      — daily stat baselines
selectedLoc, selectedRoom
userTasks[]         — all tasks (replaces old room-keyed tasks)
customLocs[]        — user-added locations
customRooms{}       — legacy room-keyed task overrides (key: "locId_roomId")
taskNextId          — auto-increment for new task IDs
taskCurType         — active tab in Tasks view
lastActiveDate      — YYYY-M-D, drives checkNewDay
```

---

### 5. Helper Utilities  `L761–836`

| Symbol | Line | Role |
|--------|------|------|
| `LV_CAP()` | 762 | XP needed for next level = `S.lv * 100` |
| `allLocs()` | 763 | `BASE_LOCS + S.customLocs` |
| `getLoc(id)` | 764 | Find location by ID |
| `getRoom(lid, rid)` | 765 | Find room within a location |
| `getTasks(lid, rid)` | 766 | Legacy room-keyed task lookup via `S.customRooms` |
| `allTasks()` | 772 | Returns `S.userTasks` (primary task store) |
| `shouldShowToday(t)` | 779 | Evaluates repeat cycle vs today's date — filters task visibility |
| `repeatLabel(t)` | 799 | Human-readable repeat string for UI badge |
| `allTasksOld()` | 808 | **Dead code** — old location-scoped task aggregator |
| `sd(k)` | 817 | Lookup stat definition by key |
| `streakBonus()` | 818 | Returns XP multiplier: 1.0 / 1.2 / 1.5 / 2.0 / 3.0 based on streak |
| `lvName()` | 822 | Current level name from `LV_NAMES` |
| `updateXPBar()` | 823 | Refreshes XP strip + navbar badges |
| `showToast(msg)` | 831 | 2s bottom toast notification |

---

### 6. Achievement System  `L839–877`

| Symbol | Line | Role |
|--------|------|------|
| `_popQ` | 840 | Queue of pending achievement popups |
| `checkAchs()` | 841 | Checks all 9 achievements against state; pushes newly unlocked to `_popQ` |
| `showNextPop()` | 864 | Dequeues and displays popup overlay |
| `closePop()` | 873 | Closes popup; chains next from queue after 300ms |
| `moodDown()` | 874 | Decrements `S.sv.moo` by 20, saves, re-renders |

**Achievement IDs:** `first` `combo3` `combo10` `alldone` `streak3` `streak7` `lv5` `streak14` `streak30`

---

### 7. Map Rendering  `L879–950`

| Symbol | Line | Role |
|--------|------|------|
| `drawMap(cv)` | 880 | Canvas pixel-art world: sky, ground, road, clouds, trees, flowers, lake, buildings for home/work/mall + custom locations. Stick figure drawn at selected location. |

**Building hit-test positions** (reused in `_doRender` L1866):
```js
BPOS = [
  {id:'home', x:W*0.04, y:58, w:W/3-12, h:42},
  {id:'work', x:W*0.36, y:53, w:W/3-16, h:58},
  {id:'mall', x:W*0.67, y:63, w:W/3-12, h:42},
  // + custom locs at (24+i*44, 124)
]
```

---

### 8. Render Functions  `L952–1457`

All renderers return HTML strings injected into `#panel`.

| Symbol | Line | Role |
|--------|------|------|
| `taskItemWorld(t, bonus)` | 953 | Task card for World tab (no edit/delete buttons) |
| `taskItem(t, bonus)` | 981 | Task card for Tasks tab (with ✏️ 🗑 buttons) |
| `getWorldTasks()` | 1014 | Filters `userTasks` to current loc/room, daily only, sorted undone-first by time |
| `taskItemExpandable(t, bonus)` | 1042 | **Primary card component**: expandable with subtask progress bar and checklist; collapsed done-cards are lightweight |
| `renderWorld()` | 1110 | World tab: canvas + loc pills + room buttons + streak banner + task list + goals card |
| `renderGoalsCard(bonus)` | 1137 | Collapsible main/side goals card shown at bottom of World tab |
| `renderTasks()` | 1161 | Tasks tab: type tabs (daily/main/side) + banner + sorted task list + add button + form |
| `_renderTasksOld_unused()` | 1220 | **Dead code** — old location-grouped tasks view |
| `renderStats()` | 1245 | Stats tab: daily stats grid + mood button + long-term stats + attribute sources + XP total |
| `renderStreak()` | 1302 | Streak tab: hero card + countdown + weekly check-in + milestones grid + next milestone |
| `renderAch()` | 1360 | Achievements tab: 2-column grid with progress bars |
| `renderQuestBoard()` | 1378 | Main/side task board, embedded in Map tab |
| `renderMap()` | 1405 | Map tab: canvas + loc pills + quest board + location manager + save management |

---

### 9. Task Management  `L1459–1660`

**UI state variables:**
| Variable | Line | Role |
|----------|------|------|
| `_taskAddOpen` | 1460 | Whether add/edit form is visible |
| `_goalsOpen` | 1461 | Whether goals card in World tab is expanded |
| `_taskEditId` | 1462 | ID of task being edited (null = new task) |
| `_taskAF` | 1463 | Add-form working object (draft task) |
| `_taskExpandId` | 1464 | Currently expanded task card ID |
| `_pendingDeleteId` | 1465 | Task ID awaiting delete confirmation (stored as string) |

**Functions:**
| Symbol | Line | Role |
|--------|------|------|
| `toggleGoals()` | 1467 | Toggle `_goalsOpen` |
| `setTaskType(t)` | 1468 | Switch task type tab, close form, auto-expand |
| `openTaskAdd(type)` | 1469 | Reset `_taskAF`, open form |
| `closeTaskAdd()` | 1470 | Close form, clear edit ID |
| `setTaskAFType(t)` | 1471 | Set draft task type |
| `setTaskAFIco(ic)` | 1472 | Set draft task icon |
| `toggleTaskAFAttr(k)` | 1473 | Toggle stat attr on draft task |
| `addTaskSub()` | 1474 | Add subtask to draft (reads `#tsub-inp`) |
| `delTaskSub(i)` | 1478 | Remove subtask from draft by index |
| `saveNewTask()` | 1479 | **Create or update**: reads form DOM, merges into `_taskAF`, pushes/updates `S.userTasks` |
| `editTask(id)` | 1498 | Populate `_taskAF` from existing task, open form |
| `deleteUserTask(id)` | 1503 | Store `_pendingDeleteId` (as string), show confirm bar |
| `confirmDelete()` | 1514 | Remove task from `S.userTasks`, clean up `taskDone` |
| `cancelDelete()` | 1524 | Clear pending ID, hide confirm bar |
| `toggleTaskExpand(id)` | 1528 | Toggle expanded card |
| `toggleSubTask(sid, parentId, parentXp)` | 1529 | Subtask check: auto-complete parent when all subs done; auto-undo parent when unchecking a sub |
| `renderTaskAddForm()` | 1574 | Returns add/edit form HTML (type picker, icon grid, XP slider, repeat config, attr chips, sub-task builder) |

---

### 10. Location & Quick Actions  `L1662–1738`

| Symbol | Line | Role |
|--------|------|------|
| `selLoc(id)` | 1663 | Select location, default to first room, save, auto-expand, re-render |
| `selLocAndNav(id)` | 1669 | `selLoc` then navigate to World tab |
| `selRoom(id)` | 1670 | Select room within current location |
| `toggleDay(i)` | 1671 | Toggle weekly check-in day; recalculates streak from trailing run |
| `toggleTask(id, xp)` | 1677 | **Core XP engine**: toggle done/undone, apply/reverse XP with streak bonus, apply stat gains, level-up check, popup/toast, run achievement check |
| `qAdd()` | 1714 | Quick-add daily task from World view (reads `#wi`) |
| `setAddIcon(ic)` | 1723 | Select icon in map location creator |
| `addLoc()` | 1724 | Create custom location from map form |
| `delLoc(id)` | 1734 | Remove custom location; reset selection if active |

---

### 11. Save Management  `L1740–1799`

| Symbol | Line | Role |
|--------|------|------|
| `exportSave()` | 1741 | Clipboard copy of JSON; fallback to file download |
| `exportFallback(data)` | 1748 | Creates `<a>` download link for `life-quest-save.json` |
| `importSave()` | 1755 | Opens `#import-overlay` textarea |
| `doImport()` | 1759 | Parses pasted JSON, deserializes Sets, overwrites `S`, saves + Firebase push |
| `cancelImport()` | 1782 | Hides import overlay |
| `resetConfirm()` | 1786 | Shows reset confirmation bar |
| `doReset()` | 1790 | Clears `lq_state_v2`, reloads defaults via `loadState()` |
| `cancelReset()` | 1797 | Hides reset confirmation bar |

---

### 12. Navigation & Render Loop  `L1802–1887`

| Symbol | Line | Role |
|--------|------|------|
| `nav(t)` | 1803 | Switch tab: update `_curTab`, sync nav button `.on` classes, scroll to top, auto-expand, render |
| `autoExpandFirst()` | 1812 | Finds first undone task with subtasks in current view, sets `_taskExpandId` |
| `_renderPending` | 1829 | RAF debounce flag — prevents duplicate renders within same frame |
| `render()` | 1830 | Schedules `_doRender` via `requestAnimationFrame` |
| `_doRender()` | 1838 | Dispatches to tab renderer, injects HTML into `#panel`, calls `updateXPBar`, re-initializes canvas + click handler |

**Init sequence** (L1881–1887):
```js
loadState()       // restore S from localStorage
checkNewDay()     // handle day rollover
autoExpandFirst() // pre-set expanded task
render()          // initial DOM paint
updateXPBar()     // sync XP strip
startClock()      // 1s interval
showNewDayPopup() // greet if new day
```

---

## Key Data-Flow Patterns

### Task Completion
```
toggleTask(id, xp)
  → streakBonus() → XP calc → S.taskDone.add/delete
  → STATS gains/losses on S.sv
  → level-up check (S.xp >= LV_CAP)
  → popup (main task) or toast
  → checkAchs()
  → saveState() → updateXPBar() → autoExpandFirst() → render()
```

### Subtask Auto-Complete
```
toggleSubTask(sid, parentId, parentXp)
  → S.subDone.add/delete
  → if all parent.subs done → auto toggleTask logic inline
  → saveState() → render()
```

### New Day Rollover
```
startClock [every 1s] → checkNewDay()
  → streak ± → weekDone reset
  → per-task repeat logic → S.taskDone.delete
  → daily stats reset to svBase, long stats decay -1
  → S._isNewDay = true → saveState()
  → render() → showNewDayPopup()
```

### Firebase Sync
```
saveState() → fbSave() [debounced 2s]
  → Firebase set(ref, serialize(S))
  → showSyncStatus('✅ 已同步')
```

---

## IDs / Keys Reference

| Range | Type |
|-------|------|
| 1001–1010 | Seed `userTasks` IDs |
| 2001–2011 | Seed subtask IDs |
| 500+ | `S.nextTId` — legacy room-task IDs |
| 2100+ | `S.taskNextId` — new `userTasks` IDs |
| `c10+` | `S.nextCId` — custom location IDs |

**localStorage key:** `lq_state_v2`  
**Firebase path:** `lifequest/save`

---

## Dead Code

| Symbol | Line | Note |
|--------|------|------|
| `allTasksOld()` | 808 | Old location-scoped task aggregator — superseded by `allTasks()` |
| `_renderTasksOld_unused()` | 1220 | Old task view — replaced by `renderTasks()` |
| `DEFAULT_TASKS` | referenced 769 | Referenced in `getTasks()` but never defined — `customRooms` fallback always returns `[]` for built-in rooms not pre-seeded |
| `taskItemWorld(t, bonus)` | 953 | Defined but `renderWorld` now uses `taskItemExpandable` |
| `window.fbLoad` stub | 12 | Assigned but never implemented or called |
