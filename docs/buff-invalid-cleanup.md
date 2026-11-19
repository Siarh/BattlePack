# Buff widget cleanup for invalid buff records

## Root cause

The stripped LuaJIT 2.0 prototype `OnBuffsChanged` iterates the aggregated `EVENT_OBJECT_BUFFS_ELEMENT_CHANGED` payload and calls `object.GetBuffInfo(buffId)`. It immediately reads `ownerId` from the result without checking whether the buff still exists.

The Allods API documentation notes that an aggregated buff-change event can contain IDs whose objects or buffs are no longer valid by the time the handler runs. A missing result can therefore abort the handler before BuffsView state is reconciled.

## Fix

The handler now checks the result of `GetBuffInfo` before the first `ownerId` access:
- a valid buff record continues through the original stack/visibility update code unchanged;
- a nil/false result branches to the existing `OnBuffRemoved({ buffId = buffId })` routine, then resumes the inner event loop.

The existing removal routine destroys any tracked widget for that buff ID and clears its map entry. No additional event registration is introduced.

## Static validation

- Bytecode format and opcode table match LuaJIT 2.0 (header `1b 4c 4a 01`).
- All 33 prototypes remain present.
- Only `OnBuffsChanged` changed: 226 → 234 instructions; KGC count 19 → 21.
- Injected code was decoded from the built file; logical constants 19 and 20 resolve to `buffId` and `OnBuffRemoved`.
- All relative branch targets in the changed prototype were recalculated and checked to remain in range.

## Runtime status

The Allods client is unavailable here. This binary is statically validated but not yet confirmed in-game. Keep this change in draft until the addon loads and the post-combat UI is tested.
