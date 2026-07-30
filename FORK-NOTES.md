# Fork Notes

Personal notes for this fork (`NicoCastillo/DockDoor`). Not part of upstream.

## Setting up on a new machine

### 1. Build

There is no signing certificate configured, so build ad-hoc signed:

```bash
xcodebuild -project DockDoor.xcodeproj -scheme DockDoor -configuration Release \
  -destination 'platform=macOS' \
  CODE_SIGN_IDENTITY="-" CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO DEVELOPMENT_TEAM="" \
  build
```

The app lands in:

```
~/Library/Developer/Xcode/DerivedData/DockDoor-*/Build/Products/Release/DockDoor.app
```

Install it by quitting DockDoor, then `ditto`ing that bundle over `/Applications/DockDoor.app`.

### 2. Turn on the folder widget — easy to forget

**Upstream ships `enableFolderWidget` defaulting to `false`.** Hovering a folder in the
dock (Downloads, etc.) will show *nothing at all* until it is switched on, which looks
exactly like the feature being broken.

Quit DockDoor first, or the running app overwrites the value on exit:

```bash
osascript -e 'tell application "DockDoor" to quit'
defaults write com.ethanbills.DockDoor enableFolderWidget -bool true
open -a /Applications/DockDoor.app
```

Or in the GUI: **Settings → Dock Previews → enable the folder widget.** (It used to live
under Widgets; upstream moved it in `c60ec54`.)

Setting it explicitly also pins it, so a future upstream default change can't flip it back.

### 3. Repeat per macOS user account

Preferences are per-user. Each account that uses DockDoor needs its own
`enableFolderWidget` flip, even though `/Applications/DockDoor.app` is shared.

For another account, easiest is to toggle it in the GUI while logged in as them. From an
admin session, run it *as* that user so the plist stays owned by them:

```bash
sudo -u <username> defaults write com.ethanbills.DockDoor enableFolderWidget -bool true
```

### 4. Re-grant Accessibility after every install

Accessibility is a per-user TCC grant keyed to the binary's code hash. Replacing an
ad-hoc signed build invalidates it, so after installing, previews may not appear at all —
apps included, not just folders. Fix in **System Settings → Privacy & Security →
Accessibility**: toggle DockDoor off, then on.

If *only* folder previews are missing, that's step 2, not this.

## What this fork changes

- `5b5b9a0` — folder widget file rows are draggable to other apps / Finder
  (`FolderWidgetView.swift`)
- `56b2074` — folder widget preview stays anchored when the cursor moves onto it, instead
  of shifting toward the dock (`SharedPreviewWindowCoordinator.swift`)

The anchor fix is in otherwise-unmodified upstream code, so it is a candidate to submit
upstream. Doing so would remove the need to carry it across syncs.

## TODO — pending upstream PR for the anchor fix (not started)

Nothing has been pushed or opened. Decide later whether to bother; the fix works locally
either way.

**Do not include the screen recording of the bug.** Text repro below is enough, and is
more useful to a maintainer since they can run it.

### How to build the branch

Cherry-pick the fix alone onto upstream, so the branch contains neither the upstream merge
commit nor this file:

```bash
git fetch upstream
git checkout -b fix/folder-widget-preview-anchor upstream/main
git cherry-pick 56b2074
git push origin fix/folder-widget-preview-anchor
```

Then open the PR against `ejbills/DockDoor` (`gh pr create --repo ejbills/DockDoor`).

### Draft PR

Title: `fix: keep folder widget preview anchored when the cursor enters it`

Body:

> **What** — Hovering a folder in the dock shows the folder widget. As soon as the cursor
> moves onto the panel, it shifts toward the dock (~20pt in my setup, collapsing the gap)
> and its file list briefly reloads.
>
> **Repro** — 1. Enable the folder widget (`enableFolderWidget`). 2. Dock magnification on.
> 3. Hover a folder in the dock; the panel appears with a gap. 4. Move the cursor onto the
> panel; it jumps closer to the dock and the list flashes.
>
> Magnification appears to be the precondition for the visible jump; only verified on my
> own setup. The content flash looks independent of it.
>
> **Cause** — `showFolderWidget` rebuilds and re-anchors on every
> `AXSelectedChildrenChanged`, so the dock icon rect is read twice: once magnified, once
> not. The second, smaller rect moves the computed position. `performDisplay` already
> guards against this for app previews ("Skip re-display to prevent position bouncing from
> duplicate AX notifications"), but the folder path unconditionally overwrites
> `anchoredDockItem`, defeating the `anchorDockPreviewPosition` setting.
>
> **Fix** — Skip re-display when the same folder is already displayed and anchored,
> mirroring the app path. Also removes the reload flash. `FolderWidgetView` drives its own
> loading via `.task(id:)`, so contents do not go stale.
>
> **Note** — The guard is unconditional, matching `performDisplay`. Happy to gate it on
> `anchorDockPreviewPosition` if you would prefer.

### Known review risks

- The early return is **not** gated on `anchorDockPreviewPosition`. A user who turns that
  setting off is opting into the preview following the icon, and this guard blocks that for
  folders. Done unconditionally to match the app path; may be asked to gate it.
- Fixes the symptom, not the trigger. Why a duplicate `AXSelectedChildrenChanged` fires was
  never confirmed — it was inferred from the icon rect changing size. Do not overclaim it.
- Media and calendar widgets go through `performDisplay`, so they are already covered if
  asked about scope.

### Evidence it is a defect rather than intended behavior

- `anchorDockPreviewPosition` exists to prevent exactly this, and defaults to on.
- `performDisplay` already carries an anti-bouncing guard for the app path.
- The anchor mechanism landed `8653b8b` (2026-02-22); folder previews landed `d3c4a59`
  (2026-05-26), three months later, and never wired into it.
- `bufferFromDock` is already magnification-aware, so a stable gap was the intent.

## Syncing upstream

```bash
git fetch upstream
git merge-tree --write-tree HEAD upstream/main >/dev/null && echo "clean merge"
git merge upstream/main
```

Merge rather than rebase, to avoid rewriting the commits above. After merging, rebuild and
reinstall (steps 1 and 4), and re-check that upstream hasn't changed any defaults you rely
on:

```bash
git diff <old> <new> -- DockDoor/consts.swift
```

Default changes are silent and only affect keys you never set explicitly — that is exactly
how the folder widget disappeared during the 1.39.4 sync.
