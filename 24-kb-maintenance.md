# 24 — Maintaining the knowledge base

This base only makes sense if it grows along with real modding experience.
This file describes **when and how to update it**.

## The rule

> **Every new finding from working on a mod goes into the knowledge base — immediately.**

It applies to the AI agent and to the user alike. Knowledge that stays only in the chat window
is lost at the next session — and that is exactly the problem this base was
created for.

## Instructions for the AI agent

While working on a Civ VII mod in this project:

1. **At the start of a session** read [00-README.md](00-README.md) and the files relevant to
   the current task. Do not re-derive things that are already here.
2. **While working** note every finding that does not follow directly from the base.
3. **Before the end of the session** (or right after an important discovery) write them into the appropriate file.
4. **Do not ask for permission** to update the knowledge base — it is a standing task.
   Ask only when the structure would have to be rebuilt or a large section removed.
5. **Tell the user what you updated** — briefly, at the end of your reply.

## What qualifies for recording

✅ **Record:**
- something behaved **differently than described** in the base → fix the entry (do not add alongside it)
- a ⚠️ / ❓ **confirmed or refuted** in practice → change the marker and note how it was checked
- a bug whose solution took **more than a few minutes** → [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md)
- a new table, column, effect, requirement, API → the appropriate topic file + possibly
  [15-schema-reference.md](15-schema-reference.md) / [18-reference-enumerations.md](18-reference-enumerations.md)
- a working code fragment solving a non-obvious problem → the relevant cookbook
- a new external source → [13-references-and-community.md](13-references-and-community.md)
  (after assessing its reliability per [22-source-evaluation.md](22-source-evaluation.md))

❌ **Do not record:**
- things already described (check first, then write)
- one-off details specific to a single mod and not transferable
- speculation without a ⚠️/❓ marker
- content from external sources **without verification** in the game's files

## Where things go

| Kind of finding | File |
|---|---|
| Mod structure, actions, criteria, scope, LoadOrder | [01-architecture.md](01-architecture.md) |
| Tables, XML/SQL, database operations | [02-database.md](02-database.md) |
| Modifiers, effects, requirements | [03-modifiers-effects.md](03-modifiers-effects.md) |
| Ages, civilizations, traditions, trees | [04-ages-and-civilizations.md](04-ages-and-civilizations.md) |
| UI, JS, components, events | [05-ui-javascript.md](05-ui-javascript.md) |
| A specific step-by-step recipe | cookbooks [06](06-cookbook-new-civilization.md)–[09](09-cookbook-ui-mod.md) |
| Icons, graphics, assets | [12-assets-icons-localization.md](12-assets-icons-localization.md) |
| Texts, translations, inflection | [23-localization-i18n.md](23-localization-i18n.md) |
| **A trap / a surprise** | [14-quirks-and-gotchas.md](14-quirks-and-gotchas.md) |
| Debugging, logs, workflow | [19-workflow-and-debugging.md](19-workflow-and-debugging.md) |
| Something undocumented / an experiment | [17-advanced-and-undocumented.md](17-advanced-and-undocumented.md) |

If a topic fits nowhere — **create a new numbered file** and add it
to the table of contents in [00-README.md](00-README.md).

## How to update correctly

**A change of verification status** — note where the knowledge comes from:
```markdown
- ⚠️ ~~It is not known whether returning to the menu reloads mods.~~
+ ✅ Returning to the main menu reloads mods without restarting the game
+   (checked 2026-08-10: a change in data/units.xml visible after returning to the menu).
```

**Correcting an error** — do not delete it quietly, mark the correction:
```markdown
> ⚠️ **CORRECTION.** This used to say X. That was wrong — in reality it is Y.
```
That way it is visible that the topic was investigated, and you do not return to the same
wrong conclusion. The model: the "CORRECTION" section in [10-tools-frameworks.md](10-tools-frameworks.md).

**A new trap** — add it at the end of the list in `14`, with the next number, always with a marker
and something concrete (symptom → cause → fix).

## Regenerating generated files

Two files are produced by a script from the game's files — after a game update it is worth refreshing them:

- [15-schema-reference.md](15-schema-reference.md) — table columns
- [18-reference-enumerations.md](18-reference-enumerations.md) — effects, collections, requirements

The regeneration scripts are **inside those files**, at the end.

## Periodic review

Every now and then (e.g. after finishing a mod or after a big game patch):

- [ ] can any ⚠️/❓ be settled by now?
- [ ] is the list of open questions in [17](17-advanced-and-undocumented.md) up to date?
- [ ] did the schemas change after the game patch? (regenerate `15` and `18`)
- [ ] does the mod list in [11](11-distribution-and-managers.md) match the subscriptions?
- [ ] has any file grown enough to be worth splitting? (the model: i18n split out
      of `12` into `23`)
