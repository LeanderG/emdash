---
title: Stable keys for repeater items
type: feature-plan
status: proposed
authors:
  - Leander Gilles (@LeanderG)
discussion: null
created: 2026-10-19
---

# Stable keys for repeater items

## Problem and user case

Repeater items have no stored identity. The admin editor gives each item a temporary `_key` while it's open, then removes the key before saving. Stored values, API responses, and MCP results contain only the sub-field values, as the [field type reference](../docs/src/content/docs/reference/field-types.mdx) shows.

Without a stored key, nothing can tell which item is which across saves:

- Site code that renders repeater items has to use the array index as the component key. After an editor reorders or deletes items, frameworks reuse component state for the wrong item.
- The admin editor matches incoming items to its temporary keys by position. When the value changes from outside the editor, collapsed state follows the position instead of the item.
- Revision history has to guess which items correspond between versions. [Changes inside blocks and repeater fields in revision history](https://github.com/LeanderG/emdash/blob/design/revision-history-blocks-and-repeaters/proposals/revision-history-blocks-and-repeaters.md) pairs repeater items by identical content and then by position. Removing one item and editing its neighbor in the same save can read as an edit to the wrong item.

Blocks fields don't have this problem. Every block carries a `_key` that the server assigns when the client doesn't send one, and the server uses it to match blocks to their stored versions.

## Proposed behavior

Every repeater item that EmDash writes carries a `_key` string, whichever client sent it:

- The admin editor keeps item keys when it saves, and gives new items a new key.
- When content is created or updated through the REST API, MCP, or a seed, the server keeps each item's `_key` and fills in any that are missing.
- A `_key` stays with its item when the item is edited or moved, as long as the client sends it back.

The rules apply to top-level repeater fields and to repeater fields inside blocks.

For example, an MCP client creates an entry with a keyless repeater:

```json
"specifications": [
	{ "label": "Weight", "value": "1.2 kg" },
	{ "label": "Color", "value": "Black" }
]
```

EmDash stores and returns it with keys:

```json
"specifications": [
	{ "_key": "01JB7X6Q4N2K8M5R3T9V0W1Y2Z", "label": "Weight", "value": "1.2 kg" },
	{ "_key": "01JB7X6Q4P3L9N6S4V0W1X2Y3A", "label": "Color", "value": "Black" }
]
```

A later update that sends both items back in the opposite order, keys included, stores the same two keys in the new order.

## Goals

- Give every newly written repeater item a stable identifier that site code, the admin, and revision history can use.
- Assign keys on every write path, so `_key` doesn't depend on which client wrote the content.
- Keep every request that is valid today valid.

## Non-goals

- Requiring `_key` in requests. Clients that never send keys keep working.
- Backfilling keys into stored content with a migration. Existing items receive keys the next time their entry is saved.
- The stricter block rules, such as rejecting keyless updates to a field that already has items. Repeater items have no type or version for the server to protect.
- Using keys for single-item operations, such as restoring or commenting on one repeater item.
- Changing how revision history displays repeaters. The revision history plan can pair items by `_key` once keys exist.

## Interfaces and compatibility

- Stored repeater values and API and MCP responses gain a `_key` string on each item written after the change. Existing fields don't change. Code that reads sub-fields by name is unaffected. Code that iterates over every property of an item, or validates items with a strict schema, sees one more property.
- The schema API requires sub-field slugs to start with a lowercase letter, so `_key` can't collide with a sub-field defined through it.
- Request validation accepts `_key` on repeater items as an optional non-empty string. The repeater item schema already allows unknown properties, so requests that send `_key` today keep working.
- Generated TypeScript types declare `_key?: string` on repeater items. It stays optional because content saved before the change has no keys.
- The server fills in keys during the existing create and update work, using the entry data those handlers already load. No database migration and no extra queries.
- The admin change is in `@emdash-cms/admin`, and the server and type changes are in `emdash`. Both need changesets. The field type reference shows `_key` in the stored repeater shape and tells site authors to fall back to the index for items saved before the change.

## Errors and edge cases

- A `_key` that isn't a non-empty string fails validation with a `VALIDATION_ERROR` naming the item, as it does for blocks.
- A duplicate `_key` within one field keeps its first occurrence. Later items with that key get new keys. This differs from blocks, which reject duplicates, because clients that copy an item today would otherwise start failing.
- A client that sends a keyless array on every update gets new keys on every save. Clients that want stable keys send back the keys they read.
- Items saved before the change keep having no `_key` until their entry is saved again. Site code needs a fallback for them.
- When a translation copies a repeater from its source entry, the copied items keep the source's keys. Keys only need to be unique within one field of one entry.
- Applying a seed whose repeater items have no keys assigns keys like any other create. Keys in the seed are kept.

## Verification

- Admin component tests: stored keys survive edits, keyless items receive unique keys that stay stable across edits, and duplicate keys are replaced.
- Server tests for every supported dialect:
  - creating with keyless items stores a unique `_key` on each item;
  - updating with keys keeps them, including after a reorder;
  - a duplicate key is replaced on the second item;
  - an empty or non-string `_key` fails validation;
  - a repeater inside a block receives keys;
  - an entry saved before the change can still be read and updated.
- An MCP test that `content_create` with a keyless repeater returns keyed items.
- A type generation test for the optional `_key` on repeater items.

## Open questions

- Should the server reuse stored keys by position when an update sends a keyless array? That keeps keys stable for clients that never send them back, but a reorder would move each key to whichever item takes its position.
- Should duplicate keys be rejected, matching blocks, instead of replaced?
