# Gallery Unlock Patch

## What this branch contains

A modified `Assembly-CSharp.dll` that bypasses the gallery unlock-code dialog in
*NTS Honey & Money*.

## What was changed

Exactly **2 bytes** in `Assembly-CSharp.dll`.

| | |
|-|-|
| File offset | `0x4B4C` |
| Original bytes | `03 02` |
| Patched bytes | `17 2A` |
| File size | unchanged (189440 bytes) |

Inside the assembly this is the body of MethodDef rid=320 (the SHA-256
verifier called from `GalleryUnlockDialog`). The original IL was:

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

After the patch the same method body starts with:

```
ldc.i4.1               // push true
ret                    // return
```

The remaining 25 bytes of the original body are now unreachable dead code, so
the fat method header (`code_size = 27`) does not need to change and no other
metadata is affected.

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
4. All gallery items that already exist in your `UnlockedMedia` list will
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
