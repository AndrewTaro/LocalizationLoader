# LocalizationLoader

A translation loader for World of Warships. It lets **any number of mods add or override UI
text at once**, with per-key fallback to the game's own translations.

## Please read before installing

- **This mod is approved by Wargaming and is safe to use as distributed here.**
- **Modifying it further is strictly prohibited.** If you alter the loader, or install a build
  that did not come from this release page, I will not be responsible for any loss or damage to
  your game or your account.

## Why

The usual approach, overwriting `global.mo` in `res_mods/texts/<locale>`, has had several issues even if it's just replacing one string:
- **Mods conflict**: Two translation mods cannot coexist.
- **Monolithic**: The whole .mo file must be shipped.
- **Version Dependency**: .mo file must be updated every patch.

This project replaces `bin64/gettext_x64r.dll` with a proxy that checks mod catalogs first and passes
everything else to the game's original library. Lookups become a **priority-ordered registry**:
every mod drops a `.mo` file into one folder, all of them load, and any key you don't define
falls through to the vanilla text. You never have to ship a full catalog to change three
strings.

## Install

1. Go to `<wows>\bin\<wows_version>\bin64`
1. **If `gettext_x64r_orig.dll` already exists, skip this step.** Otherwise rename
   `gettext_x64r.dll` to `gettext_x64r_orig.dll`.
2. Copy `gettext_x64r.dll` from this package into that `bin64\`.
3. Done!

> **Step 1's condition matters.** Renaming unconditionally works the first time and breaks the
> second: it renames *the loader* to `gettext_x64r_orig.dll`, which then tries to load itself.
> The loader detects that and refuses — untranslated text and a log line rather than a dead
> client — but your original is gone by then, and only Check and Repair brings it back.

**A game update replaces all of `bin\<build>\`, so install again after every patch.** Until
you do, the game just runs as normal without the loader; nothing breaks.

## Uninstall

In the same `bin64\` folder:

1. Delete `gettext_x64r.dll`.
2. Rename `gettext_x64r_orig.dll` back to `gettext_x64r.dll`.

That's all of it — `_orig` *is* the untouched original, so there is no backup to keep track of.
`gettext_proxy.log` and the `res_mods\texts` folder can go too. If you delete the wrong file
and the game stops starting, Game Center → **Check and Repair** puts everything back.

## How to Use

1. Create .mo file with the standard gettext tools, such as POEdit.
  - **It must only consist of the translations you modify**.
  - The conventional `.mo` mods that ship the unrelated entries are now actively harmful for other mods, and contradicsts the whole point of this project.
2. Drop the result in `bin\<wows_version>\res_mods\texts\<locale>\*`:
  - **Per-language.** A file under `texts\de\` applies only while the player is in German.
  - **Any depth.** Everything below `<locale>` is searched, so `LC_MESSAGES\` works and so does
  not bothering.
3. Done!

## Resolve Mod Conflicts
When two catalogs define the same key, a header field decides:

```
X-LocalizationLoader-Priority: 90
```

**Higher wins.** The rest of the rules:

- **Absent means 50**, so other mods can deliberately place themselves above or below you.
- **`global.mo` is ignored** in that folder and you must not use that name. The game's own overlay already applies a file by
  that name, and it replaces the whole vanilla catalog rather than merging. Name yours
  something else, e.g. `mymod.mo`.
- **`IDS_PLURAL_FORMS` is refused.** The client reads that answer back as a plural *form
  index*, not as text, so claiming it would break plural selection across the whole UI.
- **The log says so** if you hit either refusal.

## Performance

**The loader makes translation faster, not slower** — by roughly an order of magnitude. The
game's own gettext library has no translation cache: every lookup re-walks the locale list and
re-searches the full catalog, costing ~600 ns even for the same key in a tight loop. The client
translates well over 100,000 times per session, overwhelmingly repeats.

| | ns per call |
|---|---|
| the game's library, directly | ~640 |
| through the loader | **~60** |

An overridden key never reaches the library at all (~18 ns), and lookups scale across threads,
which matters because the client translates from several threads at once.

## Notes

- The library actually doing the translating is still the signed one your client shipped with —
  verify it with right-click → Properties → Digital Signatures on `gettext_x64r_orig.dll`. The
  only new binary is the loader, which is unsigned; its SHA-256 is on the release page — check
  your copy with `Get-FileHash .\gettext_x64r.dll -Algorithm SHA256`.
- Game Center's Check and Repair reverts it, like any other mod file.
- Troubleshooting: the loader writes `bin64\gettext_proxy.log` — which original it bound, which
  catalogs it loaded, how many keys each contributed, and the first several overrides applied.
  Delete it freely, it is recreated.
