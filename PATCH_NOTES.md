# Gallery Unlock Patch

## What this branch contains

A modified `Assembly-CSharp.dll` that bypasses the gallery unlock-code dialog
*and* the secondary length check that gates `GalleryCanvas`'s auto-unlock
routine in *NTS Honey & Money*.

## What was changed

Six bytes total in `Assembly-CSharp.dll` — three independent two-byte patches.

| Method (rid) | Purpose | File offset | Original | Patched |
|-|-|-|-|-|
| 320 | SHA-256 verifier called from `GalleryUnlockDialog` | `0x4B4C` | `03 02` | `17 2A` |
| 322 | Length check called from `GalleryCanvas` (`input.Length == _reference`) | `0x4BF9` | `7E 13` | `17 2A` |
| 324 | Hash-length sanity check (always true anyway, patched for safety) | `0x4C29` | `7E 13` | `17 2A` |

`17 2A` is the IL pair `ldc.i4.1; ret` — each patched method now unconditionally
returns `true`. File size is unchanged (189440 bytes) and no metadata or
other methods are affected.

The original method 320 IL was:

```
ldarg.1
ldarg.0
ldfld    _saltCache
call     ComputeSha256Base64
stloc.0
ldarg.0
ldfld    _codeHashCache
ldloc.0
ldc.i4.4               // StringComparison.Ordinal
call     String.Equals
ret
```

After the patch all three method bodies begin with:

```
ldc.i4.1               // push true
ret                    // return
```

The remaining bytes of each original body are now unreachable dead code, so
the method headers (`code_size`) do not need to change.

### Why three patches?

The verification of the entered code is split across two paths:

1. `GalleryUnlockDialog.Submit` calls method 320 (SHA-256 hash compare). On
   success it sets a flag on the `GalleryUnlockData` ScriptableObject and shows
   the success dialog.
2. The next time `GalleryCanvas` is shown, it sees the flag and *separately*
   checks that the entered code is the right length (method 322) and that the
   produced hash has the same length as the stored hash (method 324) before it
   actually walks `UnlockedMedia` and flips every entry's `LockedState` to
   "unlocked".

Patching only method 320 makes the dialog say "success" but leaves the
gallery still locked, because method 322 fails for any input whose length
does not match `_reference`. Patching all three makes both paths accept any
input.

## How to install

1. Locate your game install folder. Typically:
   `<Steam>\steamapps\common\NTS Honey and Money\NTS_HM_Data\Managed\`
   (path may differ for non-Steam versions; look for the folder that contains
   `Assembly-CSharp.dll`).
2. **Make a backup of the original `Assembly-CSharp.dll`** (rename it to
   `Assembly-CSharp.dll.bak`, for example). You will want this if anything
   goes wrong or after a game update.
3. Copy the patched `Assembly-CSharp.dll` from this branch over the original.
4. Launch the game.

## How to use it in-game

1. Open the gallery and trigger the "enter unlock code" dialog as usual.
2. Type **anything** (even a single letter) and submit.
3. The game will accept the input, set the internal "gallery unlocked" flag,
   and the `GalleryCanvas` will run the unlock routine on its next refresh.
4. Close and re-open the gallery (or switch tabs) to trigger the refresh.
5. All gallery items that already exist in your `UnlockedMedia` list will
   become viewable. Items belonging to chapters you have never played will
   still not appear, because they are added to `UnlockedMedia` only when the
   chapter runs. To populate them, also set
   `"ForceUnlockAllChapters": true` in
   `%LOCALAPPDATA%Low\Tasty_soap_bubbles\NTS_HM\slot{N}_SaveData.json`
   and replay the chapters (rollback works).

## Caveats

* The DLL is processed by **GUPS Obfuscator**. This particular build does not
  appear to embed an integrity check (no HMAC over the assembly, no Mono.Cecil
  self-verifier was found in the IL), but if a future game update adds one,
  the patched DLL will fail to load and you will need to revert to the
  backup.
* Game updates ship a new `Assembly-CSharp.dll`. The patch will need to be
  re-applied (offset may shift) after every update.
* This patch only disables the verification step. It does not modify save
  files, does not phone home, and does not change any other game logic.
