# BUGS

## Bug 1 — First level permanently locked (critical)

**File:** `src/main.ts`, `Ui.isUnlocked()` (line 2762)

**Reproduce:** Fresh save, click PLAY on title screen → nothing happens. Open LEVELS screen → level 1-1 (PLATFORM) is disabled and shows "LOCKED".

**Root cause:**
```js
// Before fix
isUnlocked(index) {
  if (index < 0)   // ← should be <= 0
    return true;
  const prev = this.levels[index - 1];
  return !!prev && !!this.save.levels[prev.id]?.cleared;
}
```
`this.levels[-1]` evaluates to `undefined` in JavaScript (arrays have no negative indices). So `!!undefined` is `false`, and level 0 is locked. `startLevel` calls `isUnlocked` and returns early, so the PLAY button on the title screen does nothing on a fresh save, and level 1-1 is shown as "LOCKED" in the levels menu.

**Fix:**
```js
// After fix
isUnlocked(index) {
  if (index <= 0)   // ← changed < to <=
    return true;
  const prev = this.levels[index - 1];
  return !!prev && !!this.save.levels[prev.id]?.cleared;
}
```
The first level (index 0) is always accessible. All subsequent levels unlock after their predecessor is cleared — that logic was already correct.

**Verification:** With the fix applied, clicking PLAY launches level 1-1 (PLATFORM) immediately, and the LEVELS screen shows 1-1 as an enabled button.
