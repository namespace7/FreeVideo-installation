# FreeVideo on macOS: Installation Troubleshooting and Uninstall Notes

**Date:** 3 November 2026  
**Machine:** Apple-silicon Mac M4, 16 GB unified memory  
**FreeVideo UI version observed:** v0.4.28  
**Engine:** `~/FreeVideo/FreeVideo-engine`

> **Outcome:** The FreeVideo interface launched and a generation workflow started, but no video was confirmed successfully completed. Generation showed a memory warning and an estimate of about 197 minutes for a 10-second clip.

## 1. Summary

- The automatic Python download failed through the configured routes.
- We manually downloaded Python 3.12.3 for Apple silicon and placed it in the bootstrap layout FreeVideo checks.
- The interpreter and bundled `uv` were verified successfully.
- The FreeVideo UI later launched.
- Generation was blocked once by a memory preflight error and later showed low memory with an impractically long ETA.
- Uninstalling is reasonable if the goal is useful local video generation without hours of waiting.

## 2. Installation facts

| Item | Observed value |
|---|---|
| Setup-plan root | `/Users/yashwant/FreeVideo/FreeVideo-engine` |
| Model directory in plan | `/Users/yashwant/FreeVideo/FreeVideo-engine/models/vdn` |
| Disk mode | `normal` |
| Engine folder | 52 GB |
| `/Applications/FreeVideo.app` | 215 MB |
| `~/FreeVideo/manual-downloads` | 194 MB |
| `~/Library/Application Support/FreeVideo` | 51 MB |
| `~/FreeVideo/ComfyUI` | Found; size/ownership not yet checked |

## 3. Issue: Python download failed

The installer reported `Python download failed on every route`.

Earlier setup logs showed:
- Official/inherited GitHub route: timeout.
- `ghfast.top` route: HTTP 403 Forbidden.

Relevant logs:
- `setup-runs/20261102T132752Z-96079/00-python-official-inherited.log`
- `setup-runs/20261102T132752Z-96079/01-python-ghfast-inherited.log`
- Launcher log under `launcher/source/2026.11.1.133217-09631d1b1f3e1124/.freevideo/comfy-setup/20261102T132736Z-9e422682/launcher.log`

### Workaround

We manually downloaded the Python 3.12.3 ARM64 standalone archive and extracted it under:

`~/FreeVideo/manual-downloads/python-3.12.3-test/python/`

We copied it into FreeVideo’s bootstrap layout:

`~/FreeVideo/FreeVideo-engine/.freevideo/bootstrap/python/cpython-3.12.3-macos-aarch64-none/`

Verified:
- Python version `3.12.3`
- Architecture `arm64`
- `ssl` and `sqlite3` imports worked
- Bundled `uv 0.9.0` found the interpreter with Python downloads disabled

The installer source contained a reuse branch that checks the bootstrap directory, asks `uv` to find the requested Python, and verifies its version. Our directory layout matched that logic.

**Evidence limit:** We confirmed the interpreter and `uv` discovery, but did not capture a definitive installer log line saying `Reusing verified bootstrap Python`. The UI later launching is consistent with the workaround helping, but does not prove every setup step succeeded without further downloads.

## 4. Issue: generation memory and speed

An early generation attempt failed at preflight with:

`MemoryError: Native generation needs at least [value redacted] GiB working unified memory after the system reserve`

The exact required-memory value was removed from the copied report, so the exact shortfall is unknown.

Later UI screenshots showed:
- FreeVideo v0.4.28
- 10-second duration
- Light quality
- Two-pass enabled
- A low-memory warning
- Estimated remaining time around 192–197 minutes, at step 1 of 8 in the first pass

One screenshot showed landscape 1344 × 768; another showed square 1024 × 1024. Do not assume the resolution was identical in both.

### Memory diagnostics recorded

- Physical memory: `17179869184` bytes (16 GiB)
- System-wide free memory percentage at the time: 30%
- Swap total: 2048 MB
- Swap used: approximately 1106.31 MB
- Swap free: approximately 941.69 MB

These values are a snapshot; system-wide free memory does not mean all of it is available to the model. macOS, other apps, and AI generation share unified memory.

### Interpretation

The UI launching is not the same as successful video generation. FreeVideo’s preflight check and low-memory warning indicate that this workload may exceed the available working memory. The very long ETA made this configuration impractical. We did not bypass the preflight check.

## 5. Lessons learned

1. Installation and inference are separate tests.
2. The Python workaround addressed the Python bootstrap problem only; it did not solve memory requirements.
3. Retain logs until troubleshooting is complete.
4. Do not assume a folder named `ComfyUI` is disposable; it may contain workflows, custom nodes, models, or outputs.
5. A long ETA can reflect hardware and workflow constraints, not necessarily a broken installation.
6. Inspect generated outputs before deleting the engine directory.

## 6. Paths found in the uninstall audit

| Path | Size | Action |
|---|---:|---|
| `/Applications/FreeVideo.app` | 215 MB | Remove if no longer wanted |
| `~/FreeVideo/FreeVideo-engine` | 52 GB | Main installation; inspect outputs, then remove if nothing needs preserving |
| `~/FreeVideo/manual-downloads` | 194 MB | Contains our manual archive/test copies; inspect, then remove if no longer needed |
| `~/Library/Application Support/FreeVideo` | 51 MB | Inspect before removing |
| `~/FreeVideo/ComfyUI` | Not measured | **Do not delete until ownership and contents are confirmed** |

## 7. Safe uninstall procedure

### Step A — Stop generation and quit

1. Click **Cancel current** in FreeVideo.
2. Wait for generation to stop.
3. Quit FreeVideo and its associated windows.
4. Do not force-quit while files are actively being written unless it is unresponsive.

### Step B — Inspect first (read-only)

Run:

```bash
echo "===== ComfyUI folder size ====="
du -sh "$HOME/FreeVideo/ComfyUI" 2>/dev/null
echo ""
echo "===== ComfyUI top-level contents ====="
ls -lah "$HOME/FreeVideo/ComfyUI"
echo ""
echo "===== FreeVideo Application Support contents ====="
find "$HOME/Library/Application Support/FreeVideo" -maxdepth 3 -print
echo ""
echo "===== Engine top-level contents ====="
ls -lah "$HOME/FreeVideo/FreeVideo-engine"
echo ""
echo "===== Likely generated videos inside engine ====="
find "$HOME/FreeVideo/FreeVideo-engine" -type f \( -iname '*.mp4' -o -iname '*.mov' -o -iname '*.webm' \) -print 2>/dev/null
```

Copy any videos, workflows, or personal files you want to keep somewhere safe before deleting the engine directory.

### Step C — Remove confirmed FreeVideo-owned paths

**These commands are destructive. Do not run them until you have reviewed Step B and preserved anything you want.** They remove only the exact paths shown.

Main engine (52 GB, including models, caches, logs and anything stored inside it):

```bash
rm -rf "$HOME/FreeVideo/FreeVideo-engine"
```

Application bundle:

```bash
rm -rf "/Applications/FreeVideo.app"
```

Manual Python downloads and test extractions we created:

```bash
rm -rf "$HOME/FreeVideo/manual-downloads"
```

Application Support data — only after confirming it contains FreeVideo-specific state you no longer need:

```bash
rm -rf "$HOME/Library/Application Support/FreeVideo"
```

Do **not** remove `~/FreeVideo/ComfyUI` until you confirm it belongs exclusively to this FreeVideo installation and has no workflows, custom nodes, models, or outputs to preserve. Avoid `sudo` unless necessary and you understand the exact target.

### Step D — Verify what remains

```bash
echo "===== Remaining FreeVideo paths ====="
for P in   "/Applications/FreeVideo.app"   "$HOME/FreeVideo/FreeVideo-engine"   "$HOME/FreeVideo/manual-downloads"   "$HOME/Library/Application Support/FreeVideo"   "$HOME/.local/share/freevideo"
do
  if [ -e "$P" ]; then
    echo "STILL EXISTS: $P"
    du -sh "$P" 2>/dev/null
  else
    echo "Not present: $P"
  fi
done

echo ""
echo "===== Remaining ~/FreeVideo contents ====="
ls -lah "$HOME/FreeVideo" 2>/dev/null
```

If `~/FreeVideo/ComfyUI` remains, that is intentional until it has been inspected.

## 8. Keep this record outside the installation

Recommended destination:

`~/Documents/projects/FreeVideo-installation-troubleshooting.md`

Keeping this file outside `~/FreeVideo/FreeVideo-engine` means uninstalling the engine will not remove the notes.

## 9. Final status

- [x] Python 3.12.3 downloaded and verified as ARM64.
- [x] Bundled `uv` could find the managed Python without downloading.
- [x] FreeVideo interface launched.
- [x] Memory preflight failure and long ETA documented.
- [ ] Desired video successfully generated.
- [ ] Engine outputs and separate ComfyUI directory inspected.
- [ ] Uninstall completed and remaining paths verified.

**Conclusion:** The Python workaround made the required interpreter available, and the UI became usable. The goal of generating a video at a practical speed was not achieved. Uninstalling is reasonable, but complete cleanup should wait until the separate ComfyUI folder, support data, and any outputs have been inspected.
