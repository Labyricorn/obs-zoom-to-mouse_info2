# obs-zoom-to-mouse — Fixes applied

Date: 2025-11-20

This file explains the compatibility fix(es) applied to `obs-zoom-to-mouse.lua` to stop the script crashing after updating OBS Studio and to make it work across multiple OBS versions.

## Problem

After updating OBS Studio, the script produced this runtime error while loading (or when reloading settings):

```
[string "D:/.obs/obs-zoom-to-mouse.lua"]:435: attempt to call field 'obs_sceneitem_get_info' (a nil value)
```

Root cause: OBS's sceneitem transform API was changed/renamed in newer builds. Newer builds expose `obs_sceneitem_get_info2` / `obs_sceneitem_set_info2` while older builds use `obs_sceneitem_get_info` / `obs_sceneitem_set_info`. The script had unconditional calls to the older symbol(s), which were nil in the updated OBS build, causing the crash.

## Fix implemented

1. Runtime API detection and selection

   - The script now detects and prefers the newer functions if present, otherwise falls back to the older names. This is done once at script load and used everywhere the transform API is required.

   Key lines (conceptual):

   ```lua
   -- prefer the newer _info2 api, otherwise use the older api
   local sceneitem_get_info_fn = obs.obs_sceneitem_get_info2 or obs.obs_sceneitem_get_info
   local sceneitem_set_info_fn = obs.obs_sceneitem_set_info2 or obs.obs_sceneitem_set_info
   local has_sceneitem_getinfo = sceneitem_get_info_fn ~= nil
   local has_sceneitem_setinfo = sceneitem_set_info_fn ~= nil
   ```

   - All previous direct calls like `obs.obs_sceneitem_get_info(...)` and `obs.obs_sceneitem_set_info(...)` were replaced with `sceneitem_get_info_fn(...)` and `sceneitem_set_info_fn(...)` respectively. This ensures the script calls whichever symbol is present in the running OBS Lua binding.

2. Fallback behavior & logging

   - If neither API function exists on the user's OBS build (very old or unusual builds), the script no longer calls a nil value. Instead it logs a warning and uses conservative fallback transform values when reading transforms, and logs a warning when it cannot restore transforms on exit.

   Example warnings you may see in the script log:

   - `WARNING: obs_sceneitem_get_info not available — using fallback transform info.`
   - `WARNING: obs_sceneitem_set_info not available — cannot restore transform info on this OBS version.`

   These warnings mean the script couldn't use the full API in that OBS version; it will continue to run, but transform restoration may be degraded.

3. Restore bug fix

   - While editing, the restore path was fixed so that the script calls the correct "set" function when restoring the original transform (previous code mistakenly attempted to call the getter). Now the script restores transform using `sceneitem_set_info_fn(...)` when available.

## Files changed

- `d:\.obs\obs-zoom-to-mouse.lua`
  - Added runtime API detection (prefer `_info2` functions).
  - Replaced direct calls to sceneitem transform functions by calling the selected function variables.
  - Added guarded fallbacks and helpful warning log messages.
  - Fixed the transform restore call path to use the setter function.

No new scripts or binaries were added.

## How to test

1. Backup the existing script (optional):

```powershell
copy 'd:\.obs\obs-zoom-to-mouse.lua' 'd:\.obs\obs-zoom-to-mouse.lua.bak'
```

2. Reload the script in OBS:
   - Tools → Scripts → select `obs-zoom-to-mouse.lua` → Remove then Add (or restart OBS).

3. Enable debug logging in the script properties (check "Enable debug logging") to see more diagnostics.

4. Try the script hotkeys:
   - Zoom toggle hotkey (to zoom in/out).
   - Follow toggle hotkey (start/stop following while zoomed).

5. Open the script log (View → Logs / or the OBS script console) and confirm you see either normal operation messages (Zoomed in/out, Tracking mouse) or, if an API was not present, the warning messages noted above.

6. If you see the original crash again, collect these items and send them back for troubleshooting:
   - OBS version string from the script log (the script logs the version when debug logging is enabled), or run `obs.obs_get_version_string()` from the OBS Lua console.
   - Exact script log lines including any warning/errors.

## What to expect

- If your OBS build exposes the newer `obs_sceneitem_get_info2` / `obs_sceneitem_set_info2`, the script will automatically use them and behave normally.
- If your OBS build exposes the older `obs_sceneitem_get_info` / `obs_sceneitem_set_info`, the script will use those and behave normally.
- If your OBS build exposes neither symbol, the script will continue to run, but will use fallback transform values and log a warning indicating limited restore capabilities. In that case, you may see transform/crop restore not be exact.

## Suggested follow-ups (optional)

- If you want perfect restoration on OBS builds that don't expose the sceneitem info API, the script can attempt a best-effort reconstruction by calling individual getters (`obs_sceneitem_get_pos`, `obs_sceneitem_get_scale`, `obs_sceneitem_get_rot`, `obs_sceneitem_get_bounds`, etc.) and re-assembling the transform on restore. This is more work and could be added if you need it.

- If many users run a very recent OBS version and the old symbols are gone everywhere, we could simplify the script to use `_info2` only — but the current approach keeps compatibility across mixed environments.

## Contact / reporting

If anything still fails after applying this fix and reloading the script, please include:
- The OBS version string (from `obs.obs_get_version_string()` logged by the script or from About → Help in OBS).
- The script log lines showing warnings or errors.

With that I can either add a more advanced fallback or recommend the minimum OBS version for full fidelity.

---

Generated by automated edit on the script on 2025-11-20; file: `d:\.obs\obs-zoom-to-mouse.lua` was updated to prefer `obs_sceneitem_get_info2`/`obs_sceneitem_set_info2` when available.
