# Life Quest — Inventory System Requirements

## Overview

This document describes the requirements for adding a real-world item inventory system to Life Quest. The core concept mirrors RPG consumable mechanics: completing tasks consumes associated items, inventory depletes over time, and low stock automatically generates a restocking quest.

---

## Goals

- Track real-world consumable items (e.g. toothpaste, toothbrush) by remaining uses
- Deduct inventory automatically when a task is completed
- Alert the user when stock falls below a threshold by generating a purchase side quest
- Allow the user to replenish stock by completing the purchase quest and entering a quantity

---

## Data Model

### New Collection: `inventory`

| Field | Type | Description |
|---|---|---|
| `itemId` | string | Unique identifier |
| `name` | string | Display name (e.g. "Toothpaste") |
| `totalUses` | number | Maximum uses when full (used for % calculation) |
| `remainingUses` | number | Current remaining uses |
| `threshold` | number | Alert threshold as a decimal (default: `0.2` = 20%) |

### Updated Collection: `tasks`

Add the following field to existing task documents:

| Field | Type | Description |
|---|---|---|
| `consumeItems` | array | List of items consumed on task completion |

Each entry in `consumeItems`:

```json
{
  "itemId": "toothpaste_01",
  "amount": 1
}
```

A single task may consume multiple items. The deduction amount is defined on the task side, not the inventory item, since the same item may be consumed differently across tasks.

---

## Core Logic

### On Task Completion

1. Read `consumeItems` from the completed task
2. For each entry, subtract `amount` from `remainingUses` in the corresponding inventory document
3. Check if `remainingUses / totalUses <= threshold`
4. If yes, and no active purchase quest exists for that item, generate a new side quest: **"Buy [item name]"**

### Duplicate Purchase Quest Prevention

Before generating a purchase quest, query active tasks for an existing **"Buy [item name]"** quest. If one already exists, skip creation.

### On Completing a Purchase Quest

1. Prompt the user with an input dialog: **"How many uses did you buy?"**
2. Add the entered value to the item's current `remainingUses` (additive, not a reset — supports partial stock top-ups)
3. Mark the purchase quest as complete

---

## Edge Cases

| Scenario | Handling |
|---|---|
| `remainingUses` reaches 0 | Allow it; no hard block on task completion |
| User buys before stock runs out | Additive replenishment handles this correctly |
| Multiple tasks consume the same item in quick succession | Each completion triggers an independent deduction and threshold check |
| Purchase quest already exists | Do not create a duplicate |

---

## Inventory Management Page

A dedicated page for managing inventory items. Tech stack: React + Firebase, consistent with existing Life Quest UI.

### Features

| Feature | Description |
|---|---|
| List all items | Display name, remaining uses, total uses, and remaining % for each item |
| Low stock indicator | Highlight items at or below threshold (≤ 20%) |
| Add item | Form to create a new item: name, totalUses |
| Edit item | Update name, totalUses, remainingUses, threshold |
| Delete item | Remove item and unlink from any tasks referencing it |

### Notes

- `threshold` defaults to `0.2` on creation; editable per item if needed in future
- Deleting an item should not break existing tasks — `consumeItems` entries referencing the deleted item are silently ignored at task completion time
- No bulk import required for v1

---

## Out of Scope (v1)

- Push notifications (reminders surface only within the app as side quests)
- Per-item custom thresholds (unified 20% threshold for all items)
- Inventory history / usage log
- Item categories or tagging

---

## Summary Flow

```
Complete Task
    ↓
Deduct consumeItems from inventory
    ↓
remainingUses / totalUses ≤ 0.2?
    ├── No  → Done
    └── Yes → Purchase quest already exists?
                  ├── Yes → Done
                  └── No  → Create "Buy [item]" side quest

Complete "Buy [item]" side quest
    ↓
Prompt: "How many uses did you buy?"
    ↓
remainingUses += input
```
